---
title: Jar包内容读取和修改的技巧
tags: [ java, 运维能力 ]
keywords: [ java, 运维能力 ]
date: 2019-12-29 22:39:48
categories:
- 运维能力
index_img: /img/blog/jar-file-inspect-and-modify-skill/index.jpeg
banner_img: /img/blog/jar-file-inspect-and-modify-skill/banner.jpg
---
一些Linux上关于Jar包的实用小技巧
<!-- more -->

有过Linux服务器上Java问题定位经验的同学恐怕都碰到过一些疑难杂症问题，有些时候我们免不了会有把jar包打开看看的想法，以便检查一下里面的内容是不是真的如自己在本地环境里看到的那样。本文介绍了一些比较实用的方法，可以在Linux服务器环境达到这种效果 : )

## 准备工作

为了演示下文的内容，先使用maven创建工程，把可执行jar包打出来。

先找个空目录，执行maven创建空工程的命令：
```shell
mvn archetype:generate -DgroupId=test -DartifactId=demo -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```
在当前目录下就会出现一个名为demo的目录，里面就是创建出来的新工程了，进入demo目录内，编辑pom文件，在里面加入打可执行jar包所需的插件配置，如下：
```xml
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.1</version>
        <configuration>
          <compilerArgument>-parameters</compilerArgument>
          <encoding>UTF-8</encoding>
          <source>1.8</source>
          <target>1.8</target>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-dependency-plugin</artifactId>
        <version>2.10</version>
        <executions>
          <execution>
            <id>copy-dependencies</id>
            <phase>package</phase>
            <goals>
              <goal>copy-dependencies</goal>
            </goals>
            <configuration>
              <outputDirectory>target/lib</outputDirectory>
            </configuration>
          </execution>
        </executions>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>2.6</version>
        <configuration>
          <archive>
            <manifest>
              <addClasspath>true</addClasspath>
              <classpathPrefix>./lib/</classpathPrefix>
              <mainClass>test.App</mainClass>
              <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
              <addDefaultSpecificationEntries>true</addDefaultSpecificationEntries>
            </manifest>
            <manifestEntries>
              <Class-Path>.</Class-Path>
            </manifestEntries>
          </archive>
        </configuration>
      </plugin>
    </plugins>
  </build>
```
这里配置了三个插件，`maven-compiler-plugin`指定使用1.8的Java版本进行编译构建，使用`UTF-8`编码，并且编译后的方法参数名为源码中定义的参数名。`maven-dependency-plugin`用于将本工程所依赖的所有jar包复制到`target/lib`目录下。`maven-jar-plugin`插件用于打包，它指定了`test.App`作为jar包的main启动类，将`target/lib`目录下的第三方依赖jar包逐个加入到可执行jar包的classpath中，并且额外将可执行jar包所在目录（`.`目录）也加入到classpath中了。

maven自动生成的工程已经在`App`类里面打印了一句“Hello World!"了：

```java
package test;

/**
 * Hello world!
 *
 */
public class App
{
    public static void main( String[] args )
    {
        System.out.println( "Hello World!" );
    }
}
```

我们把`App`类复制为`App2`类：

```shell
/tmp/jarfile/demo/src/main/java/test# cp App.java App2.java
/tmp/jarfile/demo/src/main/java/test# ls -l
total 8
-rw-r--r-- 1 root root 180 Dec 29 16:30 App2.java
-rw-r--r-- 1 root root 167 Dec 29 15:37 App.java
```

把修改一下`App2`类的内容：

```java
package test;

/**
 * Hello world!
 *
 */
public class App2
{
    public static void main( String[] args )
    {
        System.out.println( "Hello World! - from App2" );
    }
}
```

然后在这个工程中执行`mvn clean package`把可执行jar包打出来。此时target目录里大概是这样的：

```shell
/tmp/jarfile/demo/target# ll
total 44
drwxr-xr-x 10 root root 4096 Dec 29 16:02 ./
drwxr-xr-x  4 root root 4096 Dec 29 16:02 ../
drwxr-xr-x  3 root root 4096 Dec 29 16:02 classes/
-rw-r--r--  1 root root 2475 Dec 29 16:02 demo-1.0-SNAPSHOT.jar
drwxr-xr-x  3 root root 4096 Dec 29 16:02 generated-sources/
drwxr-xr-x  3 root root 4096 Dec 29 16:02 generated-test-sources/
drwxr-xr-x  2 root root 4096 Dec 29 16:02 lib/
drwxr-xr-x  2 root root 4096 Dec 29 16:02 maven-archiver/
drwxr-xr-x  3 root root 4096 Dec 29 16:02 maven-status/
drwxr-xr-x  2 root root 4096 Dec 29 16:02 surefire-reports/
drwxr-xr-x  3 root root 4096 Dec 29 16:02 test-classes/
```

`demo-1.0-SNAPSHOT.jar`就是打出来的可执行jar包了。执行`java -jar demo-1.0-SNAPSHOT.jar`，由于我们在`maven-jar-plugin`插件中指定的main类是`test.App`，所以可以看到控制台输出的是"Hello World!"：

```shell
/tmp/jarfile/demo/target# java -jar demo-1.0-SNAPSHOT.jar
Hello World!
```

可执行jar包就打好了，准备工作完成~

> 这种打法把第三方依赖包放在单独的lib目录里了，也有其他的打包方式可以把依赖jar打到可执行jar包内（如Spring Boot自带的jar包打包插件）。

## 查看和修改Jar包内容的方法

**注意**: 下文的每一小节都是独立的内容。为防止修改jar包的操作影响后文的效果，本文假设每一小节都是基于上文“准备工作”打出来的jar包进行的，互相不影响文件内容。

### 使用vim

使用vim命令也可以查看jar包的文件内容，就像打开普通的文本文件一样，输入`vim <jar包名>`，vim会展示jar包内的文件列表：

```
" zip.vim version v28
" Browsing zipfile /tmp/jarfile/demo/target/demo-1.0-SNAPSHOT.jar
" Select a file with cursor and press ENTER

META-INF/
META-INF/MANIFEST.MF
test/
test/App.class
test/App2.class
META-INF/maven/
META-INF/maven/test/
META-INF/maven/test/demo/
META-INF/maven/test/demo/pom.xml
META-INF/maven/test/demo/pom.properties
```

把光标移动到对应的行上敲回车，就可以打开这个文件看到它的内容了。

vim也可以编辑jar包内的文本文件，例如打开`META-INF/MANIFEST.MF`文件，将里面的"Main-Class"一行修改一下，把启动jar包的main类改为`App2`，如下：

```
Manifest-Version: 1.0
Implementation-Title: demo
Implementation-Version: 1.0-SNAPSHOT
Archiver-Version: Plexus Archiver
Built-By: root
Specification-Title: demo
Implementation-Vendor-Id: test
Class-Path: .
Main-Class: test.App2
Created-By: Apache Maven 3.5.2
Build-Jdk: 1.8.0_191
Specification-Version: 1.0-SNAPSHOT
Implementation-URL: http://maven.apache.org
```

输入`:wq`命令保持并退出，跟我们编辑普通的文本文件一样操作。可能在将修改写入jar包并退出的时候会碰到报错，这种时候执行`:q!`命令强制退出就行了。再次执行`java -jar demo-1.0-SNAPSHOT.jar`可以看到此时打印的内容变成了"Hello World! - from App2"，说明对MANIFEST文件的修改生效了。

```shell
## 基于源码打出来的原始jar包
/tmp/jarfile/demo/target# java -jar demo-1.0-SNAPSHOT.jar
Hello World!
## 使用vim修改manifest文件
/tmp/jarfile/demo/target# vim demo-1.0-SNAPSHOT.jar
## 再次执行jar包，可以看到修改生效了
/tmp/jarfile/demo/target# java -jar demo-1.0-SNAPSHOT.jar
Hello World! - from App2
```

### 使用unzip和zip命令

jar包，其实是zip包（不是tar包呢……）。所以如果想要看zip中的内容，可以直接把jar包当zip包解压开来看看，像这样：

```shell
/tmp/jarfile/demo/target# unzip demo-1.0-SNAPSHOT.jar -d extracted/
Archive:  demo-1.0-SNAPSHOT.jar
   creating: extracted/META-INF/
  inflating: extracted/META-INF/MANIFEST.MF
   creating: extracted/test/
  inflating: extracted/test/App.class
  inflating: extracted/test/App2.class
   creating: extracted/META-INF/maven/
   creating: extracted/META-INF/maven/test/
   creating: extracted/META-INF/maven/test/demo/
  inflating: extracted/META-INF/maven/test/demo/pom.xml
  inflating: extracted/META-INF/maven/test/demo/pom.properties
```

所有的文件都被解压到`extract`目录中了。如果只是想看看jar包里面有什么文件，也可以加上`-l`参数，不解压只是把文件内容都列出来：

```shell
/tmp/jarfile/demo/target# unzip -l demo-1.0-SNAPSHOT.jar
Archive:  demo-1.0-SNAPSHOT.jar
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2019-12-29 16:32   META-INF/
      374  2019-12-29 16:32   META-INF/MANIFEST.MF
        0  2019-12-29 16:32   test/
      553  2019-12-29 16:32   test/App.class
      568  2019-12-29 16:32   test/App2.class
        0  2019-12-29 16:32   META-INF/maven/
        0  2019-12-29 16:32   META-INF/maven/test/
        0  2019-12-29 16:32   META-INF/maven/test/demo/
     2311  2019-12-29 16:02   META-INF/maven/test/demo/pom.xml
      107  2019-12-29 16:32   META-INF/maven/test/demo/pom.properties
---------                     -------
     3913                     10 files
```

然后只解压特定的文件内容：

```shell
/tmp/jarfile/demo/target# unzip -l demo-1.0-SNAPSHOT.jar | grep MANIF
      374  2019-12-29 17:08   META-INF/MANIFEST.MF
/tmp/jarfile/demo/target# unzip demo-1.0-SNAPSHOT.jar META-INF/MANIFEST.MF
Archive:  demo-1.0-SNAPSHOT.jar
  inflating: META-INF/MANIFEST.MF
/tmp/jarfile/demo/target# ll META-INF/MANIFEST.MF
-rw-r--r-- 1 root root 374 Dec 29 17:08 META-INF/MANIFEST.MF
```

我们可以把MANIFEST文件编辑一下，把启动类改为`App2`：`sed -i 's/App/App2/' META-INF/MANIFEST.MF`。

然后使用zip命令把修改后的文件替换回jar包中：`zip -u demo-1.0-SNAPSHOT.jar META-INF/MANIFEST.MF`。`-u`命令是把磁盘上的文件更新到jar包内，磁盘文件还是保留着的。也可以使用`-m`命令，把文件移动到jar包内，磁盘文件也就随之被删除了。

再次执行jar包可以看到文件修改已经生效了。

全流程如下：

```shell
## 打包，进入target目录执行jar包查看效果
/tmp/jarfile/demo# mvn clean package 2>&1 1>/dev/null
/tmp/jarfile/demo# cd target/
/tmp/jarfile/demo/target# java -jar demo-1.0-SNAPSHOT.jar
Hello World!
## 解压出manifest文件
/tmp/jarfile/demo/target# unzip demo-1.0-SNAPSHOT.jar META-INF/MANIFEST.MF
Archive:  demo-1.0-SNAPSHOT.jar
  inflating: META-INF/MANIFEST.MF
## 修改manifest文件的内容
/tmp/jarfile/demo/target# sed -i 's/App/App2/' META-INF/MANIFEST.MF
## 将修改后的manifest更新到jar包中
/tmp/jarfile/demo/target# zip -u demo-1.0-SNAPSHOT.jar META-INF/MANIFEST.MF
updating: META-INF/MANIFEST.MF
        zip warning: Local Entry CRC does not match CD: META-INF/MANIFEST.MF
 (deflated 42%)
## 再次执行jar包可以看到修改生效了
/tmp/jarfile/demo/target# java -jar demo-1.0-SNAPSHOT.jar
Hello World! - from App2
```

### 使用jar命令

既然是jar包，当然也可以使用`jar`命令来进行处理了。这个看上去最“正统”的命令之所以被排到最后，是因为很多时候我们所面对的环境里根本没有安装jdk，所以只能使用前面那些相对“偏门”的方法了。（看看孩子都被逼成啥样儿了！）

`jar`命令可以用来创建、解压、更新jar包，这里只讲一点简单的。

- `jar`命令有些用法和`tar`命令类似，使用`jar tf <jar包名>`命令可以查看jar包内的文件列表，也可以再加个`-v`参数获取更详细的信息。

- 使用`jar xf <jar包名> <jar包内文件路径>`命令可以将特定的文件解压出来。**注意**：解压和更新文件的时候文件路径都是相对的，相对于jar包内根目录和磁盘上的当前目录。

- 使用`jar uf <jar包名> <文件路径>`命令可以把特定文件更新到jar包内。

  **注意**：单纯的`-u`参数可以修改jar包内的普通文件（配置、`.class`文件等），但是修改不了MANIFEST文件。要修改MANIFEST还需要加上`-m`参数，像这样：`jar ufm demo-1.0-SNAPSHOT.jar MANIFEST.MF`（注意`-m`参数在`-f`参数的后面，否则会报错！），由于磁盘上的MANIFEST文件的部分配置项的key与jar包内MANIFEST有重复，你可能还会看到一大堆的告警信息。

例子：

```shell
## 重新打包
/tmp/jarfile/demo# mvn clean package 2>&1 1>/dev/null
/tmp/jarfile/demo# cd target/
/tmp/jarfile/demo/target# java -jar demo-1.0-SNAPSHOT.jar
Hello World!
## 查看jar包内容
/tmp/jarfile/demo/target# jar tf demo-1.0-SNAPSHOT.jar
META-INF/
META-INF/MANIFEST.MF
test/
test/App.class
test/App2.class
META-INF/maven/
META-INF/maven/test/
META-INF/maven/test/demo/
META-INF/maven/test/demo/pom.xml
META-INF/maven/test/demo/pom.properties
## 解压出manifest文件
/tmp/jarfile/demo/target# jar xf demo-1.0-SNAPSHOT.jar META-INF/MANIFEST.MF
## 把入口类改成test.App2
/tmp/jarfile/demo/target# sed -i 's/App/App2/' META-INF/MANIFEST.MF
## 把新的manifest文件更新到jar包内，忽略告警信息
/tmp/jarfile/demo/target# jar uvfm demo-1.0-SNAPSHOT.jar META-INF/MANIFEST.MF 2>/dev/null
updated manifest
## 再次执行jar包，可以看到结果已经变了
/tmp/jarfile/demo/target# java -jar demo-1.0-SNAPSHOT.jar
Hello World! - from App2
```

## 命令行编译class文件并更新到jar包内

也许有的时候我们还需要临时修改一下某些类，比如为了获取定位问题的辅助信息多打点日志什么的。这个时候我们还需要修改源代码重新打包。但如果开发环境和程序部署环境是相互隔离的，而且部署环境恰好有jdk，我们也可以命令行手动编译class文件更新到jar包内。

作为例子，我们先在target目录里面创建一份`App`类的源码（注意目录结构与package声明的一致）：

```shell
/tmp/jarfile/demo# mvn clean package 2>&1 1>/dev/null
/tmp/jarfile/demo# cd target/
root@desktop-0023:/tmp/jarfile/demo/target# mkdir test
root@desktop-0023:/tmp/jarfile/demo/target# vim test/App.java
root@desktop-0023:/tmp/jarfile/demo/target# cat test/App.java
package test;

public class App
{
    public static void main( String[] args )
    {
        System.out.println( "Hello World!" );
        App2.main( args );
    }
}
```

这里我们在`App`类里面调用了`App2#main`方法，所以`App`类是依赖`App2`类的。此时就不能直接使用`javac <java源文件>`的方式编译了，会报依赖找不到的错误：

```shell
root@desktop-0023:/tmp/jarfile/demo/target# javac test/App.java
test/App.java:8: error: cannot find symbol
        App2.main( args );
        ^
  symbol:   variable App2
  location: class App
1 error
```

还需要用`-cp`参数指定一下依赖：

```shell
## 更新jar包前
root@desktop-0023:/tmp/jarfile/demo/target# java -jar demo-1.0-SNAPSHOT.jar
Hello World!
## 指定 demo-1.0-SNAPSHOT.jar 为classpath，编译App.java所需的依赖在这个jar里面有
root@desktop-0023:/tmp/jarfile/demo/target# javac -cp demo-1.0-SNAPSHOT.jar test/App.java
## 更新jar包内的App.class文件
root@desktop-0023:/tmp/jarfile/demo/target# jar uvf demo-1.0-SNAPSHOT.jar test/App.class
adding: test/App.class(in = 450) (out= 309)(deflated 31%)
## 运行更新后的jar包
root@desktop-0023:/tmp/jarfile/demo/target# java -jar demo-1.0-SNAPSHOT.jar
Hello World!
Hello World! - from App2
```

此时jar包的执行逻辑已经被我们新编辑的`App.java`源码修改了。

-----------------
Banner图源：Photo on <a href="https://visualhunt.com/re6/5296d2aa">VisualHunt.com</a>
