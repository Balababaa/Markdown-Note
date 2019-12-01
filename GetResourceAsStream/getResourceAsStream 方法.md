# getResourceAsStream 方法

​        在使用 Hikari 连接池的时候遇到了一个小问题，如下 **HikariConfig** 对象有个构造函数需要接受一个字符串 propertyFileName ，这个字符串是配置文件的地址。

```Java
public HikariConfig(String propertyFileName)
{
   this();

   loadProperties(propertyFileName);
}
```

​        一开始在我的代码中我是这样写的，但是这样写一直是报错，提示找不到文件。后来进入到 HikariConfig 的源码才发现它是这样加载配置文件的。

```java
HikariConfig hikariConfig = new HikariConfig("hikari.properties");

java.lang.IllegalArgumentException: Cannot find property file: hikari.properties
```

​        项目的目录结构如下所示。

```
├─src
│  └─main
│      ├─java
│      │  └─com
│      │      └─xiaobai
│      │              Application.java
│      │
│      └─resources
│              hikari.properties
│                      
└─target
    └─classes //classpath
        │  hikari.properties
        │  
        └─com
            └─xiaobai
                    Application.class
```

​        首先，HikariConfig 的构造函数拿到配置文件的地址后，就去调用 loadProperties(propertyFileName) 方法。这个方法首先会用拿到的目录创建一个 File 文件对象，但是项目当前的根目录下并没有 hikari.properties 这个文件，因此就会调用 this.getClass().getResourceAsStream(propertyFileName) ，但是这个方法也无法加载 hikari.properties 这个文件，因此就会抛出 IllegalArgumentException。 之后我就很难受，咋整都加载不出这个文件，后来了解到 getResourceAsStream() 这个方法加载文件的一些规则。

```java
private void loadProperties(String propertyFileName)
{
   final File propFile = new File(propertyFileName);
   try (final InputStream is = propFile.isFile() ? new FileInputStream(propFile) : this.getClass().getResourceAsStream(propertyFileName)) {
      if (is != null) {
         Properties props = new Properties();
         props.load(is);
         PropertyElf.setTargetFromProperties(this, props);
      }
      else {
         throw new IllegalArgumentException("Cannot find property file: " + propertyFileName);
      }
   }
   catch (IOException io) {
      throw new RuntimeException("Failed to read property file", io);
   }
}
```

### Class 对象和 ClassLoader 对象的 getResourceAsStream 方法

​        class 对象的这个方法在参数为绝对路径时，会以 classpath 为起点查找文件，当参数为相对路径时，会在 class 文件所在的目录为起点查找文件。

​        ClassLoader  对象的这个方法默认从 classpath 下查找对象，只接收相对路径参数，不接收绝对路径参数。

​        hikari 使用的是前者。当我传入参数为 hikari.properties 时，首先在根目录下查找，发现没有找到；之后又去 HikariConfig.class 所在的位置查找 hikari.properties ，当然会找不到这个文件。因此需要传入一个绝对路径，让这个方法在 classpath 下查找这个文件。

```java
HikariConfig hikariConfig = new HikariConfig("/hikari.properties");
```

### 类加载器

​    但是当我执行如下代码的时候，取到的 InputStream 却为空，这是什么鬼。

```java
Object.class.getClassLoader().getResourceAsStream("hikari.properties");
```

​    Java 的类在加载的时候需要类加载器来参与，而 Java 类加载器分为 4 种。

![](E:\DesktopFile\Java 设计模式\GetResourceAsStream\ClassLoader.png)

​    Bootstrap Class Loader  会加载这些 jar 包下的类，Object就位于 rt.jar 种。但是为什么还是取不到资源呢？因为 Bootstrap Class Loader 是使用 C++ 语言实现的，在执行 Object.class.getClassLoader() 时，取到的ClassLoader 其实是空的（但是为什么没有抛出 NullPointerException 异常），这就是为什么无法用上面的代码来获取 hikari.properties 文件。

```
jre/lib/resources.jar
jre/lib/rt.jar
jre/lib/sunrsasign.jar
jre/lib/jsse.jar
jre/lib/jce.jar
jre/lib/charsets.jar
jre/lib/jfr.jar
jre/classes
```

