---
title: Java的IO系统初步总结
date: 2020-05-17 14:09:44
tags: 
    - Java
categories:
    - Java基础
---

### 1. 文章结构

![io](/public/images/JavaIO系统初步总结.png)


### 2. 核心类

#### File类

Java的老IO系统中的类，新开发的软件请使用Path类代替File类

文件和目录的path操作工具，相当于Node.js中的Path模块，但有不限制于路径操作，在指向文件路径时又可以指代文件操作

转换为Path类型有`toPath()`方法



#### Path类

与File类之间的转换，`toFile()`

用于表示操作系统文件系统的文件或文件的路径。之后如果要表示路径应该优先使用Path类，如果要和老IO系统的API交互时可以使用`toFile()`方法转换为File类型



####  FileInputStream类、FileOutputStream类

用于文件的流的读写操作，如果要从文件系统读写文件与Java的IO系统交互就应该使用这个类。

创建从文件系统的输入流

```java
new FileInputStream(Path.of("/Users/apple/Desktop/CharacterFileOut.txt").toFile())
```



#### BufferedInputStream类、BufferedOutputStream类

带缓冲带读写流，可以在缓冲区缓冲一部分数据后在进行系统的读写调用。因为实时的读写操作需要磁盘频繁的IO操作，浪费系统资源，比较低效。而使用带缓冲区的读写可以批量的读写至缓冲区，当缓冲区满时候，批量的读出或写入到文件系统

创建带缓冲写的输出流

```java
new BufferedOutputStream(
  new FileOutputStream(Path.of("/Users/apple/Desktop/CharacterFileOut.txt").toFile())
)
```



#### ZipIntputStream类、ZipOutputStream类

用于创建压缩文件或文件夹和解压缩文件或文件夹。Gzip算法类似有GZIPInputStream、GZIPOutputStream类

使用到的中间类

`ZipEntry`, `ZipFile`



#### Reader、Writer家族派生类

用于补充传统的`InputStream`, `OutputStream`派生子类，因为这些类只能读写人不可读的字节数据。而业务开发中经常有读写字符文件数据的需要需求，而Reader，和Writer的派生之类家族就是用来简化读写字符流的类。还有一个重要的原因是老的IO流在读写字节流时候仅支持8位的字节流，而现在常用的Unicode字符是16位的。因此开发流Reader和Writer的派生类来解决字符流读写的国际化问题。

所以在业务开发过程中，如果有读写字符文件的需求，应优先使用时Reader和Writer的派生子类。

常见的子类

- FileReader，FileWriter：用于字符文件的读写，相对应的老IO中的`FileinputStream`和`FileOutputStream`
- StringReader，StringWriter：用于读写字符串类型的数据流，对应的是`StringBufferInputStream`
- CharArrayReader，CharArrayWriter：用于字符数组类型的读写数据流，对应的是`ByteArrayInputStream`, `ByteArrayOutputStream`
- BufferedReader, BufferedWriter: 用于字符类型的缓冲读写，对应`BufferedInputStream`, `BufferedOutputStream`
- PrintWriter: 用于数据的可视化输出，支持Java的的数据类型。和使用`System.out`的使用接口相同

#### ProcessBuilder

用于执行操作系统的其它程序，比如调用Linux的`Shell`命令

```java
ProcessBuilder pb = new ProcessBuilder("ls -ahl".split(" ")).start();
```



### 3. 经典的使用方式

#### 3.1.字符类型的文件读写

```java
public class CharacterFileIO {
    public static void main(String[] args) throws FileNotFoundException, IOException {
        String filename = "/Users/apple/Desktop/CharacterFileOut.txt";
        writer(filename, "Harry Hacker: 72000\n");
        String readResult = read(filename);
        System.out.println("ReadResult: " + readResult);
        writerSimple(filename, "Harry Hacker: 72001\n");
        String readSimpleResult = readSimple(filename);
        System.out.println("readSimpleResult: " + readSimpleResult);
    }

    // 读文件
    public static String read(String filename) throws IOException{
        Path path = Paths.get(filename);
        try(BufferedReader in = new BufferedReader(new FileReader(path.toFile(), StandardCharsets.UTF_8))) {
            String s;
            StringBuilder sb = new StringBuilder();
            while ((s = in.readLine()) != null) {
                sb.append(s);
            }
            return sb.toString();
        }
    }

    // 使用Files类读文件
    public static String readSimple(String filename) throws IOException{
        Path path = Paths.get(filename);
        Stream<String> lines = Files.lines(path, StandardCharsets.UTF_8);
        StringBuilder sb = new StringBuilder();
        lines.forEach(i -> {
            sb.append(i).append("\n");
        });
        return sb.toString();
    }

    // 写文件
    public static void writer(String filename, CharSequence content) throws FileNotFoundException {
        Path path = Paths.get(filename);
        try (PrintWriter printWriter = new PrintWriter(new BufferedOutputStream(new FileOutputStream(path.toFile())))) {
            printWriter.println(content);
        }
    }

    // 使用Files类写文件
    public static void writerSimple(String filename, String content) throws IOException{
        Path path = Paths.get(filename);
        Files.write(path, content.getBytes(StandardCharsets.UTF_8));
    }
}
```



#### 3.2. 二进制类文件的读写

```java
/**
 * 二进制文件读读写
 */
public class ByteFileIO {
    public static void main(String[] args) throws IOException {
        String filepath = "/Users/apple/Desktop";
        String filename = "ByteFileIOOut.jpg";

        // 读
        byte[] readData = read(Path.of(filepath, filename).toString());

        // 写
        String filenameOut = "ByteFileIOOut2.jpg";
        writer(Path.of(filepath, filenameOut).toString(), readData);

        // Files读
        byte[] simpleReadData = readSimple(Path.of(filepath, filename).toString());

        // Files写
        String filenameSimpleOut = "ByteFileIOSimpleOut.jpg";
        writerSimple(Path.of(filepath, filenameSimpleOut).toString(), simpleReadData);
    }

    public static byte[] read(String filename) throws IOException {
        Path path = Paths.get(filename);
        try(BufferedInputStream fileIn = new BufferedInputStream(new FileInputStream(path.toFile()))) {
            int size = fileIn.available();
            if (size <= 0) {
                return new byte[0];
            }
            byte[] data = new byte[size];
            int totalSize = fileIn.read(data);
            System.out.println("read "+ path.getFileName() +" finished total size: "+ totalSize +" bytes");
            return data;
        }
    }

    public static byte[] readSimple(String filename) throws IOException {
        Path path = Paths.get(filename);
        final byte[] bytes = Files.readAllBytes(path);
        System.out.println("readSimple " + path.getFileName() + " finished total size: " + bytes.length + " bytes");
        return bytes;
    }

    public static void writer(String filename, byte[] data) throws IOException {
        Path path = Paths.get(filename);
        try(BufferedOutputStream out = new BufferedOutputStream(new FileOutputStream(path.toFile()))) {
            out.write(data);
            System.out.println("writer " + path.getFileName() + " finished total size: " + data.length + " bytes");
        }
    }

    public static void writerSimple(String filename, byte[] data) throws IOException {
        final Path path = Paths.get(filename);
        Files.write(path, data);
        System.out.println("writerSimple " + path.getFileName() + " finished total size: " + data.length + " bytes");
    }
}

/*
// 输出
read ByteFileIOOut.jpg finished total size: 200960 bytes
writer ByteFileIOOut2.jpg finished total size: 200960 bytes
readSimple ByteFileIOOut.jpg finished total size: 200960 bytes
writerSimple ByteFileIOSimpleOut.jpg finished total size: 200960 bytes
*/
```



#### 3.3. 文件或文件夹的压缩和解压缩

```java
public class ZipFileIO {
    public static void main(String[] args) throws IOException, IllegalAccessException {
        // 压缩测试
        System.out.println("=======运行压缩=======");
        zip(Path.of("/Users/apple/Desktop/python数据分析与机器学习实战"), Path.of("/Users/apple/Desktop/python数据分析与机器学习实战.bak.zip"));
        // 解压测试
        System.out.println("=======运行解压=======");
        unZip(Path.of("/Users/apple/Desktop/python数据分析与机器学习实战.bak.zip"), Path.of("/Users/apple/Desktop/1"));
    }

    public static void zip(Path sourcePath, Path targetPath) throws IOException {
        final File sourceFile = sourcePath.toFile();
        final File targetFile = targetPath.toFile();
        try (
                final ZipOutputStream zipOut = new ZipOutputStream(new BufferedOutputStream(new FileOutputStream(targetFile)));
        ) {
            compress(zipOut, sourceFile, sourceFile.getName());
        }
    }

    public static void compress(ZipOutputStream zipOut, File source, String sourceName) throws IOException {
        boolean isDir = source.isDirectory();
        if (!isDir) {
            System.out.println(sourceName);
            zipOut.putNextEntry(new ZipEntry(sourceName));
            try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream(source));) {
                int b;
                while ((b = bis.read()) != -1) {
                    zipOut.write(b);
                }
            }
        }else {
            // 目录
            File[] files = source.listFiles();
            if (files == null || files.length <= 0) {
                zipOut.putNextEntry(new ZipEntry(Path.of(sourceName, "").toString()));
            }else {
                for (File file : files) {
                    final String nextFilename = Path.of(sourceName, file.getName()).toString();
                    compress(zipOut, file, nextFilename);
                }
            }
        }
    }

    public static void unZip(Path sourcePath, Path targetPath) throws IllegalAccessException, IOException {
        final File sourceFile = sourcePath.toFile();
        if (!sourceFile.exists()) {
            throw new IllegalAccessException(sourceFile.toString() + " not exist");
        }
        ZipFile zipFile = new ZipFile(sourcePath.toFile());
        final Enumeration<? extends ZipEntry> entries = zipFile.entries();
        while (entries.hasMoreElements()) {
            final ZipEntry zipEntry = entries.nextElement();
            System.out.println(zipEntry.getName());
            if (zipEntry.isDirectory()) {
                Paths.get(targetPath.toString(), zipEntry.getName()).toFile().mkdirs();
            }else {
                final File zipEntryFile = Paths.get(targetPath.toString(), zipEntry.getName()).toFile();
                final File parentFile = zipEntryFile.getParentFile();
                if (!parentFile.exists()) {
                    parentFile.mkdirs();
                }
                zipEntryFile.createNewFile();
                try(
                        final BufferedInputStream bis = new BufferedInputStream(zipFile.getInputStream(zipEntry));
                        final BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream(zipEntryFile))
                ) {
                    int b;
                    while ((b = bis.read()) != -1) {
                        bos.write(b);
                    }
                }
            }
        }
    }
}
```



### 4. Java的IO类的使用原则

IO类库的设计使用的是**装饰器模式**

比如要读取一个二进制文件，那么可以确定需要使用Input类型的类，比如从文件系统读取，那需要`FileInputStream`类。而且希望能够高效的读取，减少系统开销。所以需要使用缓冲区功能，那需要`BufferedInputStream`类。

那应该如何将用到的两个类组合起来使用呢？ 通过嵌套流过滤器来实现，通过组合不同功能的流过滤器来实现不同功能的数据流

```java
new BufferedInputStream(new FileInputStream(file))
```

这种通过多个流过滤器的组合起来使用的方式，将带来极大的灵活性。但是灵活性、易用性这两个方面想都完美是不可能的。这就涉及到取舍了，二Java就选择了前者。但是随着JDK的新版本出来，只目前的新版本中也出现了高封装成都的IO类，比如从JDk1.7引入的`Files`类，就包含很多简单的读写封装

### 参考资料

[《Java核心技术：卷1》](https://book.douban.com/subject/26880667/)

[《Thinking in Java》](https://book.douban.com/subject/2130190/)