---
title: FIle类常用工具方法整理（持续更新）
tags: [Java,File]
categories:
  - Java
date: 2018-08-18 13:14:19
---

整理记录工作中经常用到的文件相关的工具方法。

- 递归遍历一个目录，获取所有文件名（也可以取到绝对路径）
```java
public static void traverse(String filePath, List<String> files) {
        if (StringUtils.isBlank(filePath)){
            return ;
        }
        try{
            File superFile = new File(filePath);
            if (superFile.exists()) {
                File[] fileList = superFile.listFiles();
                if (null != files && fileList.length > 0) {
                    for (File file : fileList) {
                        // 还是文件夹
                        if (file.isDirectory()) {
                            traverse(file.getAbsolutePath(),files);
                        } else {
                            files.add(file.getName());    //文件名
                            //files.add(file.getAbsolutePath());    //文件绝对路径
                        }
                    }
                }
            }
        }catch (Exception e){
            //log
        }
        return ;
    }
```

- 获取文件大小，自动用K、M、G表示<!-- more -->
```java
public static String parseSize(long length){
        StringBuilder size = new StringBuilder();
        DecimalFormat format = new DecimalFormat("###.0");
        if (length < 1024) {
            size.append((int) length).append(" B");
        }else if (length >= 1024 && length < 1024 * 1024) {
            double i = (length / (1024.0));
            size.append(format.format(i)).append(" K");
        }else if (length >= 1024 * 1024 && length < 1024 * 1024 * 1024) {
            double i = (length / (1024.0 * 1024.0));
            size.append(format.format(i)).append(" M");
        }else if (length >= 1024 * 1024 * 1024) {
            double i = (length / (1024.0 * 1024.0 * 1024.0));
            size.append(format.format(i)).append(" G");
        }
        return size.toString();
    }
```

- Multipart文件转存为本地的File
```java
public static void multipartToFile(MultipartFile file, String fileFolder){
        FileOutputStream outputStream = null;
        try {
            File newFileFolder = new File(fileFolder);
            if (!newFileFolder.exists()) {
                newFileFolder.mkdirs();
            }

            fileFolder = newFileFolder.getAbsolutePath() + File.separator + file.getOriginalFilename();

            outputStream = new FileOutputStream(new File(fileFolder));
            IOUtils.copy(file.getInputStream(), outputStream);
        } catch (Exception e) {
            // log
        } finally {
            IOUtils.closeQuietly(outputStream);
        }
    }
```

- 清理指定目录下一天前（时间可以指定）的文件
```java
public static void cleanDirectory(String dir, long ttl) {
        File file = new File(dir);
        String[] subDirNames = file.list(new FilenameFilter() {
            @Override
            public boolean accept(File current, String name) {
                return new File(current, name).isDirectory();
            }
        });
        if (subDirNames != null) {
            for (String name : subDirNames) {
                File subDir = new File(dir + File.separator + name);
                if (System.currentTimeMillis() - subDir.lastModified() > ttl) {
                    try {
                        FileUtils.deleteDirectory(subDir);  //import org.apache.commons.io.FileUtils;
                    } catch (Exception e) {
                        // log
                    }
                }
            }
        }
    }
```

- 把字符串存入指定文件
```java
public static void strToFile(String content, File outFile) {
        OutputStream os = null;
        try {
            File parent = outFile.getParentFile();
            if (!parent.exists()) {
                parent.mkdirs();
            }
            if (!outFile.exists()) {
                outFile.createNewFile();
            }
            os = new FileOutputStream(outFile);
            IOUtils.write(content, os);
        } catch (Exception e) {
            // log
        } finally {
            IOUtils.closeQuietly(os);
        }
    }
```

<hr />