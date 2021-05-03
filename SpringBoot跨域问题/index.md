---
title: SpringBoot跨域问题
description: SpringBoot中通过CORS解决跨域问题。
date: 2021-04-13
tags: 
    - Spring
    - SpringBoot
    - JAVA
categories: 
    - Spring
    - SpringBoot
    - JAVA

---



### 一、同源策略

同源策略是由Netscape提出的一个著名的安全策略，它是浏览器最核心也最基本的安全功能，现在所有支持JavaScript的浏览器都会使用这个策略。所谓同源是指`协议`、`域名`以及`端口`要相同。同源策略是基于安全方面的考虑提出来的，这个策略本身没问题，但是我们在实际开发中，由于各种原因又经常有跨域的需求，传统的跨域方案是JSONP，JSONP虽然能解决跨域但是有一个很大的局限性，那就是只支持GET请求，不支持其他类型的请求，而今天我们说的CORS（跨域源资源共享）（CORS，Cross-origin resource sharing）是一个W3C标准，它是一份浏览器技术的规范，提供了Web服务从不同网域传来沙盒脚本的方法，以避开浏览器的同源策略，这是JSONP模式的现代版。

在Spring框架中，对于CORS也提供了相应的解决方案。

#### 非同源限制

- 无法读取非同源网页的 Cookie、LocalStorage 和 IndexedDB
- 无法接触非同源网页的 DOM
- 无法向非同源地址发送 AJAX 请求



### 二、java后端实现CORS跨域请求的方式

1. 返回新的CorsFilter
2. 重写 WebMvcConfigurer
3. 使用注解 @CrossOrigin
4. 手动设置响应头 (HttpServletResponse)
5. 自定web filter 实现跨域

**注意：**

- `CorFilter / WebMvConfigurer / @CrossOrigin` 需要 SpringMVC 4.2以上版本才支持，对应springBoot 1.3版本以上
- 上面前两种方式属于全局 CORS 配置，后两种属于局部 CORS配置。如果使用了局部跨域是会覆盖全局跨域的规则，所以可以通过 @CrossOrigin 注解来进行细粒度更高的跨域资源控制。
- 其实无论哪种方案，最终目的都是修改响应头，向响应头中添加浏览器所要求的数据，进而实现跨域



#### 1.返回新的 CorsFilter(全局跨域)

在任意配置类，返回一个 新的 CorsFIlter Bean ，并添加映射路径和具体的CORS配置路径。

```java
@Configuration
public class GlobalCorsConfig {
    @Bean
    public CorsFilter corsFilter() {
        //1. 添加 CORS配置信息
        CorsConfiguration config = new CorsConfiguration();
        //放行哪些原始域
        config.addAllowedOrigin("*");
        //是否发送 Cookie
        config.setAllowCredentials(true);
        //放行哪些请求方式
        config.addAllowedMethod("*");
        //放行哪些原始请求头部信息
        config.addAllowedHeader("*");
        //暴露哪些头部信息
        config.addExposedHeader("*");
        //2. 添加映射路径
        UrlBasedCorsConfigurationSource corsConfigurationSource = new UrlBasedCorsConfigurationSource();
        corsConfigurationSource.registerCorsConfiguration("/**",config);
        //3. 返回新的CorsFilter
        return new CorsFilter(corsConfigurationSource);
    }
}
```



#### 2. 重写 WebMvcConfigurer(全局跨域)

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
              //是否发送Cookie
              .allowCredentials(true)
              //放行哪些原始域
              .allowedOrigins("*")
              .allowedMethods(new String[]{"GET", "POST", "PUT", "DELETE"})
              .allowedHeaders("*")
              .exposedHeaders("*");
    }
}
```



#### 3. 使用注解 (局部跨域)

在控制器(类上)上使用注解 `@CrossOrigin`:，表示该类的所有方法允许跨域。

```java
@RestController
@CrossOrigin(origins = "*")
public class HelloController {
    @RequestMapping("/hello")
    public String hello() {
        return "hello world";
    }
}
```

在方法上使用注解 @CrossOrigin:

```java
@RequestMapping("/hello")
@CrossOrigin(origins = "*")
//@CrossOrigin(value = "http://localhost:8081") //指定具体ip允许跨域
public String hello() {
    return "hello world";
}
```



#### 4. 手动设置响应头(局部跨域)

使用 HttpServletResponse 对象添加响应头(Access-Control-Allow-Origin)来授权原始域，这里 Origin的值也可以设置为 “*”,表示全部放行。

```java
@RequestMapping("/index")
public String index(HttpServletResponse response) {
    response.addHeader("Access-Allow-Control-Origin","*");
    return "index";
}
```



#### 5. 使用自定义filter实现跨域

首先编写一个过滤器，可以起名字为MyCorsFilter.java

```java
@Component
public class MyCorsFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        HttpServletResponse response = (HttpServletResponse) res;
        response.setHeader("Access-Control-Allow-Origin", "*");
        response.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE");
        response.setHeader("Access-Control-Max-Age", "3600");
        response.setHeader("Access-Control-Allow-Headers", "x-requested-with,content-type");
        chain.doFilter(req, res);
    }
    public void init(FilterConfig filterConfig) {}
    public void destroy() {}
}
```

在web.xml中配置这个过滤器，使其生效

```xml
<!-- 跨域访问 START-->
<filter>
  <filter-name>CorsFilter</filter-name>
  <filter-class>com.mesnac.aop.MyCorsFilter</filter-class>
</filter>
<filter-mapping>
  <filter-name>CorsFilter</filter-name>
  <url-pattern>/*</url-pattern>
</filter-mapping>
<!-- 跨域访问 END  -->
```



### 三、例子

首先创建两个普通的SpringBoot项目，第一个命名为provider提供服务，第二个命名为consumer消费服务，第一个配置端口为8080，第二个配置配置为8081，然后在provider上提供两个hello接口，一个get，一个post，如下：

```java
@RestController
public class Provider {
    @GetMapping("/hello")
    public String hello() {
        return "hello";
    }

    @PostMapping("/hello")
    public String hello2() {
        return "post hello";
    }
}
```

在consumer的resources/templates目录下创建一个html文件，发送一个简单的ajax请求，如下：

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <title>CORS</title>
  </head>
  <body>
    <div id="app"></div>
    <input type="button" onclick="btnClick()" value="get_button">
    <input type="button" onclick="btnClick2()" value="post_button">
    <script>
        function btnClick() {
            $.get('http://localhost:8080/hello', function (msg) {
                $("#app").html(msg);
            });
        }

        function btnClick2() {
            $.post('http://localhost:8080/hello', function (msg) {
                $("#app").html(msg);
            });
        }
    </script>
  </body>
</html>
```

然后分别启动两个项目，发送请求按钮，观察浏览器控制台如下：

> Access to XMLHttpRequest at 'http://localhost:8080/hello' from origin 'http://localhost:8081' has been    blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource

可以看到，由于同源策略的限制，请求无法发送成功。

使用CORS可以在前端代码不做任何修改的情况下，实现跨域，那么接下来看看在provider中如何配置。首先可以通过@CrossOrigin注解配置某一个方法接受某一个域的请求，如下：

```java
@RestController
public class Provider {
    @CrossOrigin(value = "http://localhost:8081")
    @GetMapping("/hello")
    public String hello() {
        return "hello";
    }

    @CrossOrigin(value = "http://localhost:8081")
    @PostMapping("/hello")
    public String hello2() {
        return "post hello";
    }
}
```

这个注解表示这两个接口接受来自http://localhost:8081地址的请求，配置完成后，重启provider，再次发送请求，浏览器控制台就不会报错了，consumer也能拿到数据了。

provider上，每一个方法上都去加注解未免太麻烦了，在Spring Boot中，还可以通过全局配置一次性解决这个问题，全局配置只需要在配置类中重写addCorsMappings方法即可，如下：

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("http://localhost:8081")
                .allowedMethods("*")
                .allowedHeaders("*");
    }
}
```

`/**`表示本应用的所有方法都会去处理跨域请求，allowedMethods表示允许通过的请求数，allowedHeaders则表示允许的请求头。经过这样的配置之后，就不必在每个方法上单独配置跨域了。



### 四、存在的问题

了解了整个CORS的工作过程之后，我们通过Ajax发送跨域请求，虽然用户体验提高了，但是也有潜在的威胁存在，常见的就是CSRF（Cross-site request forgery）跨站请求伪造。跨站请求伪造也被称为one-click attack 或者 session riding，通常缩写为CSRF或者XSRF，是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法，举个例子：

> 假如一家银行用以运行转账操作的URL地址如下：`http://icbc.com/aa?bb=cc`，那么，一个恶意攻击者可以在另一个网站上放置如下代码：`<img src="http://icbc.com/aa?bb=cc">`，如果用户访问了恶意站点，而她之前刚访问过银行不久，登录信息尚未过期，那么她就会遭受损失。

基于此，浏览器在实际操作中，会对请求进行分类，分为简单请求，预先请求，带凭证的请求等，预先请求会首先发送一个options探测请求，和浏览器进行协商是否接受请求。默认情况下跨域请求是不需要凭证的，但是服务端可以配置要求客户端提供凭证，这样就可以有效避免csrf攻击。