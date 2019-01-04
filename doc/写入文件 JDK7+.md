# 写入文件（JDK 7+）

1.追加文件（文件不存在报错）

```java
Path path = Paths.get("1.txt");
String content = "老王";
// 追加文件
Files.write(path, content.getBytes(StandardCharsets.UTF_8), StandardOpenOption.APPEND);
```

判断文件，如果不在创建

```java
Path path = Paths.get("1.txt");
if (!Files.exists(path)) {
    Files.createFile(path); // 创建文件
}
```

2.新写入文件

```java
Path path = Paths.get("1.txt");
String content = "老王";
Files.write(path, content.getBytes(StandardCharsets.UTF_8), StandardOpenOption.CREATE);
```

每次都会覆盖重新文件，文件不存在会自动创建，文件存在则会复制重写。