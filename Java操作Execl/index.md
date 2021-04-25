---
title: Java操作Execl
description: apache poi，easypoi，easyExecl三者的对比及基本应用。
date: 2021-04-09
tags: 
    - Execl
    - JAVA
categories: 
    - JAVA
---

### 一、介绍

在平时的业务系统开发中，少不了需要用到导出、导入excel功能，今天来总结一下，如果你正为此需求感到困惑，那么阅读完本文，你一定会有所收获！

本文对比`apache poi`，`easypoi`，`easyExecl`，给出它们的基本应用方式。



### 二、apache poi

[Apache POI](https://poi.apache.org/)

`Apache POI`是一种流行的API，它允许程序员使用Java程序创建，修改和显示MS Office文件。这由Apache软件基金会开发使用Java分布式设计或修改Microsoft Office文件的开源库。它包含类和方法对用户输入数据或文件到MS Office文档进行解码。

大概在很久很久以前，微软的电子表格软件 Excel 以操作简单、存储数据直观方便，还支持打印报表，在诞生之初，可谓深得办公室里的白领青睐，极大的提升了工作的效率，不久之后，便成了办公室里的必备工具。

随着更多的新语言的崛起，例如我们所熟悉的 java，后来便有一些团队开始开发一套能与 Excel 软件无缝切换的操作工具！

这其中就有我们所熟悉的 apache 的 poi，其前身是 Jakarta 的 POI Project项目，之后将其开源给 apache 基金会！

当然，在java生态体系里面，能与Excel无缝衔接的第三方工具还有很多，因为 ```apache poi``` 在业界使用的最广泛，因此其他的工具不做过多介绍！



#### 2.1 、首先引入 apache poi 的Maven依赖

```xml
<dependencies>
    <!--xls(03)-->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi</artifactId>
        <version>4.1.2</version>
    </dependency>
    <!--xlsx(07)-->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml</artifactId>
        <version>4.1.2</version>
    </dependency>
    <!--时间格式化工具-->
    <dependency>
        <groupId>joda-time</groupId>
        <artifactId>joda-time</artifactId>
        <version>2.10.6</version>
    </dependency>
</dependencies>
```



#### 2.2、导出Execl

导出操作，即使用 Java 写出数据到 Excel 中，常见场景是将页面上的数据导出，这些数据可能是财务数据，也可能是商品数据，生成 Excel 后返回给用户下载文件。

在 poi 工具库中，导出 api 可以分三种方式

- **HSSF方式**：这种方式导出的文件格式为office 2003专用格式，即`.xls`，优点是导出数据速度快，但是最多65536行数据
- **XSSF方式**：这种方式导出的文件格式为office 2007专用格式，即`.xlsx`，优点是导出的数据不受行数限制，缺点导出速度慢
- **SXSSF方式**：SXSSF 是 XSSF API的兼容流式扩展，主要解决当使用 XSSF 方式导出大数据量时，内存溢出的问题，支持导出大批量的excel数据



##### 2.2.1、HSSF方式导出

HSSF方式最多支持导出65536条数据！


```java
public class ExcelWrite2003Test {
    public static String PATH = "D:/Desktop/";

    public static void main(String[] args) throws Exception {
        //时间
        long begin = System.currentTimeMillis();

        //创建一个工作簿
        Workbook workbook = new HSSFWorkbook();
        //创建表
        Sheet sheet = workbook.createSheet();
        //写入数据
        for (int rowNumber = 0; rowNumber < 65536; rowNumber++) {
            //创建行
            Row row = sheet.createRow(rowNumber);
            for (int cellNumber = 0; cellNumber < 10; cellNumber++) {
                //创建列
                Cell cell = row.createCell(cellNumber);
                cell.setCellValue(cellNumber);
            }
        }
        System.out.println("ExcelWrite2003 导出完成......");

        FileOutputStream fileOutputStream = new FileOutputStream(PATH + "用户信息表03.xls");
        workbook.write(fileOutputStream);
        fileOutputStream.close();
        long end = System.currentTimeMillis();
        // 输出用时(s)
        System.out.println("用时 " + (double) (end - begin) / 1000 + " 秒");
    }
}

// ExcelWrite2003 导出完成......
// 用时 1.387 秒
```



##### 2.2.2、XSSF方式导出

XSSF方式支持大批量数据导出，所有的数据先写入内存再导出，容易出现内存溢出！

```java
public class ExcelWrite2007Test {
    public static String PATH = "D:/Desktop/";

    public static void main(String[] args) throws Exception {
        //时间
        long begin = System.currentTimeMillis();

        //创建一个工作簿
        Workbook workbook = new XSSFWorkbook();
        //创建表
        Sheet sheet = workbook.createSheet();
        //写入数据
        for (int rowNumber = 0; rowNumber < 65537; rowNumber++) {
            Row row = sheet.createRow(rowNumber);
            for (int cellNumber = 0; cellNumber < 10; cellNumber++) {
                Cell cell = row.createCell(cellNumber);
                cell.setCellValue(cellNumber);
            }
        }
        System.out.println("ExcelWrite2007 导出完成......");

        FileOutputStream fileOutputStream = new FileOutputStream(PATH + "用户信息表07.xlsx");
        workbook.write(fileOutputStream);
        fileOutputStream.close();
        long end = System.currentTimeMillis();
        System.out.println("用时 " + (double) (end - begin) / 1000 + " 秒");
    }
}

// ExcelWrite2007 导出完成......
// 用时 5.382 秒
```



##### 2.2.3、SXSSF方式导出

SXSSF方式是XSSF方式的一种延伸，主要特性是低内存，导出的时候，先将数据写入磁盘再导出，避免报内存不足，导致程序运行异常，缺点是运行很慢！

```java
public class ExcelWriteSXSSFTest {
    public static String PATH = "D:/Desktop/";

    public static void main(String[] args) throws Exception {
        //时间
        long begin = System.currentTimeMillis();

        //创建一个工作簿
        Workbook workbook = new SXSSFWorkbook();

        //创建表
        Sheet sheet = workbook.createSheet();

        //写入数据
        for (int rowNumber = 0; rowNumber < 100000; rowNumber++) {
            Row row = sheet.createRow(rowNumber);
            for (int cellNumber = 0; cellNumber < 10; cellNumber++) {
                Cell cell = row.createCell(cellNumber);
                cell.setCellValue(cellNumber);
            }
        }
        System.out.println("ExcelWriteSXSSFTest 导出完成......");

        FileOutputStream fileOutputStream = new FileOutputStream(PATH + "用户信息表2007.xlsx");
        workbook.write(fileOutputStream);
        fileOutputStream.close();

        long end = System.currentTimeMillis();
        System.out.println("用时 " + (double) (end - begin) / 1000 + " 秒");
    }
}

// ExcelWriteSXSSFTest 导出完成......
// 用时 2.621 秒
```



#### 2.3、导入Execl

导入操作，即将 excel 中的数据采用java工具库将其解析出来，进而将 excel 数据写入数据库！

同样，在 poi 工具库中，导入 api 也分三种方式，与上面的导出一一对应！

##### 2.3.1、HSSF方式导入

```java
public class ExcelRead2003Test {
    public static String PATH = "D:/Desktop/";

    public static void main(String[] args) throws Exception {
        //获取文件流
        FileInputStream inputStream = new FileInputStream(PATH + "用户信息表.xls");

        //1.创建工作簿,使用excel能操作的这边都看看操作
        Workbook workbook = new HSSFWorkbook(inputStream);
        //2.得到表
        Sheet sheet = workbook.getSheetAt(0);
        //3.得到行
        Row row = sheet.getRow(0);
        //4.得到列
        Cell cell = row.getCell(0);
        getValue(cell);
        inputStream.close();
    }

    public static void getValue(Cell cell){
        //匹配类型数据
        if (cell != null) {
            CellType cellType = cell.getCellType();
            String cellValue = "";
            switch (cellType) {
                case STRING: //字符串
                    System.out.print("[String类型]");
                    cellValue = cell.getStringCellValue();
                    break;
                case BOOLEAN: //布尔类型
                    System.out.print("[boolean类型]");
                    cellValue = String.valueOf(cell.getBooleanCellValue());
                    break;
                case BLANK: //空
                    System.out.print("[BLANK类型]");
                    break;
                case NUMERIC: //数字（日期、普通数字）
                    System.out.print("[NUMERIC类型]");
                    if (HSSFDateUtil.isCellDateFormatted(cell)) { //日期
                        System.out.print("[日期]");
                        Date date = cell.getDateCellValue();
                        cellValue = new DateTime(date).toString("yyyy-MM-dd");
                    } else {
                        //不是日期格式，防止数字过长
                        System.out.print("[转换为字符串输出]");
                        cell.setCellType(CellType.STRING);
                        cellValue = cell.toString();
                    }
                    break;
                case ERROR:
                    System.out.print("[数据类型错误]");
                    break;
            }
            System.out.println(cellValue);
        }
    }
}
```

##### 2.3.2、XSSF方式导入

```java
public class ExcelRead2007Test {
    public static String PATH = "D:/Desktop/";

    public static void main(String[] args) throws Exception {
        //获取文件流
        FileInputStream inputStream = new FileInputStream(PATH + "用户信息表.xlsx");

        //1.创建工作簿,使用excel能操作的这边都看看操作
        Workbook workbook = new XSSFWorkbook(inputStream);
        //2.得到表
        Sheet sheet = workbook.getSheetAt(0);
        //3.得到行
        Row row = sheet.getRow(0);
        //4.得到列
        Cell cell = row.getCell(0);
        getValue(cell);
        inputStream.close();
    }

    public static void getValue(Cell cell){
        //匹配类型数据
        if (cell != null) {
            CellType cellType = cell.getCellType();
            String cellValue = "";
            switch (cellType) {
                case STRING: //字符串
                    System.out.print("[String类型]");
                    cellValue = cell.getStringCellValue();
                    break;
                case BOOLEAN: //布尔类型
                    System.out.print("[boolean类型]");
                    cellValue = String.valueOf(cell.getBooleanCellValue());
                    break;
                case BLANK: //空
                    System.out.print("[BLANK类型]");
                    break;
                case NUMERIC: //数字（日期、普通数字）
                    System.out.print("[NUMERIC类型]");
                    if (HSSFDateUtil.isCellDateFormatted(cell)) { //日期
                        System.out.print("[日期]");
                        Date date = cell.getDateCellValue();
                        cellValue = new DateTime(date).toString("yyyy-MM-dd");
                    } else {
                        //不是日期格式，防止数字过长
                        System.out.print("[转换为字符串输出]");
                        cell.setCellType(CellType.STRING);
                        cellValue = cell.toString();
                    }
                    break;
                case ERROR:
                    System.out.print("[数据类型错误]");
                    break;
            }
            System.out.println(cellValue);
        }
    }
}
```

##### 2.3.3、SXSSF方式导入

```java
public class ExcelReadSXSSFTest {
    public static String PATH = "D:/Desktop/";

    public static void main(String[] args) throws Exception {
        //获取文件流

        //1.创建工作簿,使用excel能操作的这边都看看操作
        OPCPackage opcPackage = OPCPackage.open(PATH + "用户信息表2007.xlsx");
        XSSFReader xssfReader = new XSSFReader(opcPackage);
        StylesTable stylesTable = xssfReader.getStylesTable();
        ReadOnlySharedStringsTable sharedStringsTable = new ReadOnlySharedStringsTable(opcPackage);
        // 创建XMLReader，设置ContentHandler
        XMLReader xmlReader = SAXHelper.newXMLReader();
        xmlReader.setContentHandler(new XSSFSheetXMLHandler(stylesTable, sharedStringsTable, new SimpleSheetContentsHandler(), false));
        // 解析每个Sheet数据
        Iterator<InputStream> sheetsData = xssfReader.getSheetsData();
        while (sheetsData.hasNext()) {
            try (InputStream inputStream = sheetsData.next();) {
                xmlReader.parse(new InputSource(inputStream));
            }
        }
    }

    /**
     * 内容处理器
     */
    public static class SimpleSheetContentsHandler implements XSSFSheetXMLHandler.SheetContentsHandler {
        protected List<String> row;

        /**
         * A row with the (zero based) row number has started
         * @param rowNum
         */
        @Override
        public void startRow(int rowNum) {
            row = new ArrayList<>();
        }

        /**
         * A row with the (zero based) row number has ended
         * @param rowNum
         */
        @Override
        public void endRow(int rowNum) {
            if (row.isEmpty()) {
                return;
            }
            // 处理数据
            System.out.println(row.stream().collect(Collectors.joining("   ")));
        }

        /**
         * A cell, with the given formatted value (may be null),
         * and possibly a comment (may be null), was encountered
         * @param cellReference
         * @param formattedValue
         * @param comment
         */
        @Override
        public void cell(String cellReference, String formattedValue, XSSFComment comment) {
            row.add(formattedValue);
        }

        /**
         * A header or footer has been encountered
         * @param text
         * @param isHeader
         * @param tagName
         */
        @Override
        public void headerFooter(String text, boolean isHeader, String tagName) {
        }
    }
}
```



### 三、easypoi

[easypoi](http://doc.wupaas.com/docs/easypoi/easypoi-1c0u6ksp2r091)

以前的以前，有个大佬程序员，跳到一家公司之后就和业务人员聊上了，这些业务员对excel报表有着许许多多的要求，比如想要一个报表，他的表头是一个多行表头，过几天之后，他想要给这些表头添加样式，比如关键的数据标红，再过几天，他想要再末尾添加一条合计的数据，等等！

起初还好，都是copy、copy，之后发现系统中出现大量的重复代码，于是有一天真的忍受不了了，采用注解搞定来搞定这些定制化成程度高的逻辑，将公共化抽离出来，于是诞生了 easypoi！

easypoi 的底层也是基于 apache poi 进行深度开发的，它主要的特点就是将更多重复的工作，全部简单化，避免编写重复的代码！

下面，我们就一起来了解一下这款高大上的开源工具：easypoi

#### 3.1、首先添加依赖包

```xml
<dependencies>
    <dependency>
        <groupId>cn.afterturn</groupId>
        <artifactId>easypoi-base</artifactId>
        <version>4.1.0</version>
    </dependency>
    <dependency>
        <groupId>cn.afterturn</groupId>
        <artifactId>easypoi-web</artifactId>
        <version>4.1.0</version>
    </dependency>
    <dependency>
        <groupId>cn.afterturn</groupId>
        <artifactId>easypoi-annotation</artifactId>
        <version>4.1.0</version>
    </dependency>
</dependencies>
```



#### 3.2、采用注解导出导入

easypoi 最大的亮点就是基于注解实体类来导出、导入excel，使用起来非常简单！

首先，我们创建一个实体类`UserEntity`，其中`@Excel`注解表示导出文件的头部信息。

```java
public class UserEntity {
    @Excel(name = "姓名")
    private String name;

    @Excel(name = "年龄")
    private int age;

    @Excel(name = "操作时间", format = "yyyy-MM-dd HH:mm:ss", width = 20.0)
    private Date time;

	// 省略 get/set方法
}
```

接着，我们来编写导出服务！

```java
public class EasypoiExcelExport {
    public static void main(String[] args) throws IOException {
        List<UserEntity> dataList = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            UserEntity userEntity = new UserEntity();
            userEntity.setName("张三" + i);
            userEntity.setAge(20 + i);
            userEntity.setTime(new Date(System.currentTimeMillis() + i));
            dataList.add(userEntity);
        }
        //生成excel文档
        Workbook workbook = ExcelExportUtil.exportExcel(new ExportParams("用户", "用户信息"),
                UserEntity.class, dataList);
        FileOutputStream fos = new FileOutputStream("D:/Desktop/easypoi-user1.xls");
        workbook.write(fos);
        fos.close();
    }
}
```

结果预览：

![easypoi1](https://gitee.com/ycfxhsw/picture/raw/master/easypoi1.png)

对应的导入操作，也很简单，源码如下：

```java
public class importEasypoi {
    public static void main(String[] args) {
        ImportParams params = new ImportParams();
        params.setTitleRows(1);
        params.setHeadRows(1);
        long start = new Date().getTime();
        List<UserEntity> list = ExcelImportUtil.importExcel(new File("D:/Desktop/easypoi-user1.xls"),UserEntity.class, params);
        System.out.println(new Date().getTime() - start);
    }
}
```

#### 3.3、自定义数据结构导出导入

easypoi 同样也支持自定义数据结构导出导入excel。

- 自定义数据导出 excel

```java
public class CustomExport {
    public static void main(String[] args) throws Exception {
        //封装表头
        List<ExcelExportEntity> entityList = new ArrayList<ExcelExportEntity>();
        entityList.add(new ExcelExportEntity("姓名", "name"));
        entityList.add(new ExcelExportEntity("年龄", "age"));
        ExcelExportEntity entityTime = new ExcelExportEntity("操作时间", "time");
        entityTime.setFormat("yyyy-MM-dd HH:mm:ss");
        entityTime.setWidth(20.0);
        entityList.add(entityTime);
        //封装数据体
        List<Map<String, Object>> dataList = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            Map<String, Object> userEntityMap = new HashMap<>();
            userEntityMap.put("name", "张三" + i);
            userEntityMap.put("age", 20 + i);
            userEntityMap.put("time", new Date(System.currentTimeMillis() + i));
            dataList.add(userEntityMap);
        }
        //生成excel文档
        Workbook workbook = ExcelExportUtil.exportExcel(new ExportParams("学生", "用户信息"), entityList, dataList);
        FileOutputStream fos = new FileOutputStream("D:/Desktop/user2.xls");
        workbook.write(fos);
        fos.close();
    }

```

- 导入 excel

```java
public class CustomImport {
    public static void main(String[] args) {
        ImportParams params = new ImportParams();
        params.setTitleRows(1);
        params.setHeadRows(1);
        long start = new Date().getTime();
        List<Map<String, Object>> list = ExcelImportUtil.importExcel(new File("D:/Desktop/user2.xls"),Map.class, params);
        System.out.println(new Date().getTime() - start);
    }
}
```



### 四、easyexcel

[EasyExecl](https://alibaba-easyexcel.github.io/index.html)

easyexcel 是阿里巴巴开源的一款 excel 解析工具，底层逻辑也是基于 apache poi 进行二次开发的。不同的是，再读写数据的时候，采用 sax 模式一行一行解析，在并发量很大的情况下，依然能稳定运行！

下面，我们就一起来了解一下这款新起之秀！

#### 4.1、首先添加依赖包

```xml
<dependencies>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>easyexcel</artifactId>
        <version>2.2.6</version>
    </dependency>
 <!--常用工具库-->
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>29.0-jre</version>
    </dependency>
</dependencies>
```

#### 4.2、采用注解导出导入

easyexcel 同样也支持采用注解方式进行导出、导入！

首先，我们创建一个实体类`UserEntity`，其中`@ExcelProperty`注解表示导出文件的头部信息。

```java
public class UserEntity {

    @ExcelProperty(value = "姓名")
    private String name;

    @ExcelProperty(value = "年龄")
    private int age;

    @DateTimeFormat("yyyy-MM-dd HH:mm:ss")
    @ExcelProperty(value = "操作时间")
    private Date time;

	// 省略 get/set方法
}
```

接着，我们来编写导出服务！

```java
public class EasyEmport {
    public static void main(String[] args) {
        List<UserEntity> dataList = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            UserEntity userEntity = new UserEntity();
            userEntity.setName("张三" + i);
            userEntity.setAge(20 + i);
            userEntity.setTime(new Date(System.currentTimeMillis() + i));
            dataList.add(userEntity);
        }
        EasyExcel.write("D:/Desktop/easyexcel-user1.xls", UserEntity.class).sheet("用户信息").doWrite(dataList);
    }
}
```

对应的导入操作，也很简单，源码如下：

```java
public class EasyImport {
    public static void main(String[] args) {
        String filePath = "D:/Desktop/easyexcel-user1.xls";
        List<UserEntity> list = EasyExcel.read(filePath).head(UserEntity.class).sheet().doReadSync();
        System.out.println(JSONArray.toJSONString(list));
    }
}
```



#### 4.3、自定义数据结构导出导入

easyexcel 同样也支持自定义数据结构导出导入excel。

- 自定义数据导出 excel

```java
public static void main(String[] args) {
    //表头
    List<List<String>> headList = new ArrayList<>();
    headList.add(Lists.newArrayList("姓名"));
    headList.add(Lists.newArrayList("年龄"));
    headList.add(Lists.newArrayList("操作时间"));

    //数据体
    List<List<Object>> dataList = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        List<Object> data = new ArrayList<>();
        data.add("张三" + i);
        data.add(20 + i);
        data.add(new Date(System.currentTimeMillis() + i));
        dataList.add(data);
    }
    EasyExcel.write("D:/Desktop/easyexcel-user2.xls").head(headList).sheet("用户信息").doWrite(dataList);
}
```

- 导入 excel

```java
public static void main(String[] args) {
    String filePath = "D:/Desktop/easyexcel-user2.xls";
    UserDataListener userDataListener = new UserDataListener();
    EasyExcel.read(filePath, userDataListener).sheet().doRead();
    System.out.println("表头：" + JSONArray.toJSONString(userDataListener.getHeadList()));
    System.out.println("数据体：" + JSONArray.toJSONString(userDataListener.getDataList()));
}
```

### 五、小结

总体来说，easypoi和easyexcel都是基于apache poi进行二次开发的。

不同点在于：

1、`easypoi` 在读写数据的时候，优先是先将数据写入内存，优点是读写性能非常高，但是当数据量很大的时候，会出现OOM，当然它也提供了 sax 模式的读写方式，需要调用特定的方法实现。

2、`easyexcel` 基于sax模式进行读写数据，不会出现OOM情况，程序有过高并发场景的验证，因此程序运行比较稳定，相对于 easypoi 来说，读写性能稍慢！

easypoi 与 easyexcel 还有一点区别在于，easypoi 对定制化的导出支持非常的丰富，如果当前的项目需求，并发量不大、数据量也不大，但是需要导出 excel 的文件样式千差万别，那么我推荐你用 easypoi；反之，使用 easyexcel ！



### 六、参考

1、[apache poi](https://poi.apache.org/) - 接口文档

2、[easypoi](http://doc.wupaas.com/docs/easypoi/easypoi-1c0u6ksp2r091) - 接口文档

3、[easyexcel](https://alibaba-easyexcel.github.io/index.html) - 接口文档