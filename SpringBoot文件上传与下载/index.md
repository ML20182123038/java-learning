---
title: SpringBoot文件上传与下载
description: SpringBoot单文件与多文件的上传，文件下载。
date: 2021-04-27
tags: 
    - Spring
    - SpringBoot
    - JAVA
categories: 
    - Spring
    - SpringBoot
    - JAVA
---

###  前端页面

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
  <head>
    <meta http-equiv="Content-type" content="text/html; charset=UTF-8">
    <title>文件上传</title>
  </head>
  <body>
    <form method="post" th:action="@{/monofileUpload}" enctype="multipart/form-data">
      单文件：<input type="file" name="monofile">
      <input type="submit" value="提交">
    </form>

    <form method="post" th:action="@{/multifileUpload}" enctype="multipart/form-data">
      <!-- 要选择多个文件需要写 multiple -->
      多文件：<input type="file" name="multifile" multiple>
      <input type="submit" value="提交">
    </form>
  </body>
</html>
```



### 文件上传、下载

```java
/**
 * 文件上传、下载
 */
@Controller
public class FileTest {
    @GetMapping("/upload")
    public String upload() {
        return "file";
    }

    /**
     * 单文件上传
     * MultipartFile 自动封装上传过来的文件
     * @param monofile 单个文件
     * @return f/t
     */
    @ResponseBody
    @PostMapping("/monofileUpload")
    public String upload(@RequestParam("monofile") MultipartFile monofile) throws IOException {
        // 判断上传文件是否为空，若为空则返回错误信息
        if (monofile.isEmpty()) {
            return "上传失败！";
        }

        // 获取文件原名
        String originalFilename = monofile.getOriginalFilename();
        // 创建一个新的File对象用于存放上传的文件
        File fileNew = new File("D:\\Desktop\\" + originalFilename);
        monofile.transferTo(fileNew);
        return "上传完成！";
    }

    /**
     * 多文件上传
     * @param multifile 多个文件
     * @return f/t
     */
    @ResponseBody
    @PostMapping("/multifileUpload")
    public String upload(@RequestParam("multifile") MultipartFile[] multifile) throws IOException {
        if (multifile.length > 0) {
            for (MultipartFile file : multifile) {
                // 获取文件原名
                String originalFilename = file.getOriginalFilename();
                // 创建一个新的File对象用于存放上传的文件
                File fileNew = new File("D:\\Desktop\\" + originalFilename);
                file.transferTo(fileNew);
            }
            return "上传完成";
        }
        return "上传失败！";
    }
    
    /**
     * 文件下载
     * @param response
     */
    @GetMapping("/download")
    public void download(HttpServletResponse response) {
        // 通过response输出流将文件传递到浏览器
        // 1、获取文件路径
        String fileName = "test.png";
        // 2.构建一个文件通过Paths工具类获取一个Path对象
        Path path = Paths.get("D:\\Desktop\\", fileName);
        //判断文件是否存在
        if (Files.exists(path)) {
            //存在则下载
            //通过response设定他的响应类型
            //4.获取文件的后缀名
            String fileSuffix = fileName.substring(fileName.lastIndexOf(".") + 1);
            // 5.设置contentType ,只有指定contentType才能下载
            response.setContentType("application/" + fileSuffix);
            // 6.添加http头信息
            // 因为fileName的编码格式是UTF-8 但是http头信息只识别 ISO8859-1 的编码格式
            // 因此要对fileName重新编码
            try {
                response.addHeader("Content-Disposition", "attachment;filename=" + new String(fileName.getBytes("UTF-8"), "ISO8859-1"));
            } catch (UnsupportedEncodingException e) {
                e.printStackTrace();
            }
            // 7.使用  Path 和response输出流将文件输出到浏览器
            try {
                Files.copy(path, response.getOutputStream());
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```



### 多文件上传中遇到的问题

```bash
org.apache.tomcat.util.http.fileupload.FileUploadBase$FileSizeLimitExceededException: The field fileName exceeds its maximum permitted size of 1048576 bytes.
```

Spring Boot默认文件上传大小为2M，多文档上传中总是出现文件大小超出限度

**解决方法：**

a、在application.properties文件中设置文件大小

```properties
#文件大小阈值，当大于这个阈值时将写入到磁盘，否则在内存中。默认值为0
spring.servlet.multipart.max-request-size=100MB   
#设置上传文件总大小为100MB
spring.servlet.multipart.max-file-size=100MB
```



b、在java文件中配置Bean来设置文件大小

```java
    /**  
     * 文件上传配置  
     * @return  
     */  
    @Bean  
    public MultipartConfigElement multipartConfigElement() {  
        MultipartConfigFactory factory = new MultipartConfigFactory();  
        //单个文件最大  
        factory.setMaxFileSize("10240KB"); //KB,MB  
        /// 设置总上传数据总大小  
        factory.setMaxRequestSize("102400KB");  
        return factory.createMultipartConfig();  
    }  
```