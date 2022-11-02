面试官：为什么 SpringBoot 的 jar 可以直接运行？



## 前言

`SpringBoot`提供了一个插件`spring-boot-maven-plugin`用于把程序打包成一个可执行的`jar`包。在`pom`文件里加入这个插件即可：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

打包完生成的`executable-jar-1.0-SNAPSHOT.jar`内部的结构如下：

```makefile
├── META-INF
│   ├── MANIFEST.MF
│   └── maven
│       └── spring.study
│           └── executable-jar
│               ├── pom.properties
│               └── pom.xml
├── lib
│   ├── aopalliance-1.0.jar
│   ├── classmate-1.1.0.jar
│   ├── spring-boot-1.3.5.RELEASE.jar
│   ├── spring-boot-autoconfigure-1.3.5.RELEASE.jar
│   ├── ...
├── org
│   └── springframework
│       └── boot
│           └── loader
│               ├── ExecutableArchiveLauncher$1.class
│               ├── ...
└── spring
    └── study
        └── executablejar
            └── ExecutableJarApplication.class
```

然后可以直接执行jar包就能启动程序了：

`java -jar executable-jar-1.0-SNAPSHOT.jar`

打包出来fat jar内部有4种文件类型：

- META-INF文件夹：程序入口，其中`MANIFEST.MF`用于描述jar包的信息
- lib目录：放置第三方依赖的jar包，比如`Spring Boot`的一些jar包
- `spring boot loader`相关的代码
- 模块自身的代码

MANIFEST.MF文件的内容：

```makefile
Manifest-Version: 1.0
Implementation-Title: executable-jar
Implementation-Version: 1.0-SNAPSHOT
Archiver-Version: Plexus Archiver
Built-By: Format
Start-Class: spring.study.executablejar.ExecutableJarApplication
Implementation-Vendor-Id: spring.study
Spring-Boot-Version: 1.3.5.RELEASE
Created-By: Apache Maven 3.2.3
Build-Jdk: 1.8.0_20
Implementation-Vendor: Pivotal Software, Inc.
Main-Class: org.springframework.boot.loader.JarLauncher
```

我们看到，它的`Main-Class`是`org.springframework.boot.loader.JarLauncher`，当我们使用java -jar执行jar包的时候会调用`JarLauncher`的main方法，而不是我们编写的`SpringApplication`。

那么JarLauncher这个类是的作用是什么的？

它是SpringBoot内部提供的工具Spring Boot Loader提供的一个用于执行Application类的工具类(fat jar内部有spring loader相关的代码就是因为这里用到了)。相当于Spring Boot Loader提供了一套标准用于执行SpringBoot打包出来的jar

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/WRLYshvT39z52fcLiaX1SuruexDXnZDkP2YngEktCZJtibicDaWoelCe3O8coSXQyyYiaIMQyCbaROSuicibb1Jwrggw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### Spring Boot Loader抽象的一些类

抽象类`Launcher`：各种`Launcher`的基础抽象类，用于启动应用程序；跟Archive配合使用；目前有3种实现，分别是`JarLauncher`、`WarLauncher`以及`PropertiesLauncher`。·

Archive：归档文件的基础抽象类。`JarFileArchive`就是jar包文件的抽象。它提供了一些方法比如getUrl会返回这个`Archive`对应的URL；`getManifest`方法会获得Manifest数据等。`ExplodedArchive`是文件目录的抽象

JarFile：对jar包的封装，每个`JarFileArchive`都会对应一个`JarFile`。`JarFile`被构造的时候会解析内部结构，去获取jar包里的各个文件或文件夹，这些文件或文件夹会被封装到Entry中，也存储在JarFileArchive中。如果Entry是个jar，会解析成`JarFileArchive`。

比如一个JarFileArchive对应的URL为：

```
jar:file:/Users/format/Develop/gitrepository/springboot-analysis/springboot-executable-jar/target/executable-jar-1.0-SNAPSHOT.jar!/
```

它对应的JarFile为：

```
/Users/format/Develop/gitrepository/springboot-analysis/springboot-executable-jar/target/executable-jar-1.0-SNAPSHOT.jar
```

这个JarFile有很多Entry，比如：

```
META-INF/
META-INF/MANIFEST.MF
spring/
spring/study/
....
spring/study/executablejar/ExecutableJarApplication.class
lib/spring-boot-starter-1.3.5.RELEASE.jar
lib/spring-boot-1.3.5.RELEASE.jar
...
```

JarFileArchive内部的一些依赖jar对应的URL(SpringBoot使用org.springframework.boot.loader.jar.Handler处理器来处理这些URL)：

```
jar:file:/Users/Format/Develop/gitrepository/springboot-analysis/springboot-executable-jar/target/executable-jar-1.0-SNAPSHOT.jar!/lib/spring-boot-starter-web-1.3.5.RELEASE.jar!/

jar:file:/Users/Format/Develop/gitrepository/springboot-analysis/springboot-executable-jar/target/executable-jar-1.0-SNAPSHOT.jar!/lib/spring-boot-loader-1.3.5.RELEASE.jar!/org/springframework/boot/loader/JarLauncher.class
```

我们看到如果有jar包中包含jar，或者jar包中包含jar包里面的class文件，那么会使 用 !/ 分隔开，这种方式只有org.springframework.boot.loader.jar.Handler能处 理，它是SpringBoot内部扩展出来的一种URL协议。

### JarLauncher的执行过程

JarLauncher的main方法：

```java
public static void main(String[] args) {
    // 构造JarLauncher，然后调用它的launch方法。参数是控制台传递的
    new JarLauncher().launch(args);
}  
```

`JarLauncher`被构造的时候会调用父类`ExecutableArchiveLauncher`的构造方法。

`ExecutableArchiveLauncher`的构造方法内部会去构造`Archive`，这里构造了`JarFileArchive`。构造`JarFileArchive`的过程中还会构造很多东西，比如`JarFile，Entry …`

```java
//JarLauncher的launch方法：
protected void launch(String[] args) {
  try {
    // 在系统属性中设置注册了自定义的URL处理器：org.springframework.boot.loader.jar.Handler。如果URL中没有指定处理器，会去系统属性中查询
    JarFile.registerUrlProtocolHandler();
    // getClassPathArchives方法在会去找lib目录下对应的第三方依赖JarFileArchive，同时也会项目自身的JarFileArchive
    // 根据getClassPathArchives得到的JarFileArchive集合去创建类加载器ClassLoader。这里会构造一个LaunchedURLClassLoader类加载器，这个类加载器继承URLClassLoader，并使用这些JarFileArchive集合的URL构造成URLClassPath
    // LaunchedURLClassLoader类加载器的父类加载器是当前执行类JarLauncher的类加载器
    ClassLoader classLoader = createClassLoader(getClassPathArchives());
    // getMainClass方法会去项目自身的Archive中的Manifest中找出key为Start-Class的类
    // 调用重载方法launch
    launch(args, getMainClass(), classLoader);
  }
  catch (Exception ex) {
    ex.printStackTrace();
    System.exit(1);
  }
}

// Archive的getMainClass方法
// 这里会找出spring.study.executablejar.ExecutableJarApplication这个类
public String getMainClass() throws Exception {
  Manifest manifest = getManifest();
  String mainClass = null;
  if (manifest != null) {
    mainClass = manifest.getMainAttributes().getValue("Start-Class");
  }
  if (mainClass == null) {
    throw new IllegalStateException(
        "No 'Start-Class' manifest entry specified in " + this);
  }
  return mainClass;
}

// launch重载方法
protected void launch(String[] args, String mainClass, ClassLoader classLoader)
    throws Exception {
      // 创建一个MainMethodRunner，并把args和Start-Class传递给它
  Runnable runner = createMainMethodRunner(mainClass, args, classLoader);
      // 构造新线程
  Thread runnerThread = new Thread(runner);
      // 线程设置类加载器以及名字，然后启动
  runnerThread.setContextClassLoader(classLoader);
  runnerThread.setName(Thread.currentThread().getName());
  runnerThread.start();
}
```

MainMethodRunner的run方法：

```java
@Override
public void run() {
  try {
    // 根据Start-Class进行实例化
    Class<?> mainClass = Thread.currentThread().getContextClassLoader()
        .loadClass(this.mainClassName);
    // 找出main方法
    Method mainMethod = mainClass.getDeclaredMethod("main", String[].class);
    // 如果main方法不存在，抛出异常
    if (mainMethod == null) {
      throw new IllegalStateException(
          this.mainClassName + " does not have a main method");
    }
    // 调用
    mainMethod.invoke(null, new Object[] { this.args });
  }
  catch (Exception ex) {
    UncaughtExceptionHandler handler = Thread.currentThread()
        .getUncaughtExceptionHandler();
    if (handler != null) {
      handler.uncaughtException(Thread.currentThread(), ex);
    }
    throw new RuntimeException(ex);
  }
}
```

Start-Class的main方法调用之后，内部会构造Spring容器，启动内置Servlet容器等过程。这些过程我们都已经分析过了。

### 关于自定义的类加载器LaunchedURLClassLoader

LaunchedURLClassLoader重写了loadClass方法，也就是说它修改了默认的类加载方式(先看该类是否已加载这部分不变，后面真正去加载类的规则改变了，不再是直接从父类加载器中去加载)。LaunchedURLClassLoader定义了自己的类加载规则：

```java
private Class<?> doLoadClass(String name) throws ClassNotFoundException {

  // 1) Try the root class loader
  try {
    if (this.rootClassLoader != null) {
      return this.rootClassLoader.loadClass(name);
    }
  }
  catch (Exception ex) {
    // Ignore and continue
  }

  // 2) Try to find locally
  try {
    findPackage(name);
    Class<?> cls = findClass(name);
    return cls;
  }
  catch (Exception ex) {
    // Ignore and continue
  }

  // 3) Use standard loading
  return super.loadClass(name, false);
}
```

加载规则：

- 如果根类加载器存在，调用它的加载方法。这里是根类加载是`ExtClassLoader`
- 调用`LaunchedURLClassLoader`自身的`findClass`方法，也就是`URLClassLoader`的`findClass`方法
- 调用父类的`loadClass`方法，也就是执行默认的类加载顺序(从`BootstrapClassLoader`开始从下往下寻找)

`LaunchedURLClassLoader`自身的`findClass`方法：

```java
protected Class<?> findClass(final String name)
     throws ClassNotFoundException
{
    try {
        return AccessController.doPrivileged(
            new PrivilegedExceptionAction<Class<?>>() {
                public Class<?> run() throws ClassNotFoundException {
                    // 把类名解析成路径并加上.class后缀
                    String path = name.replace('.', '/').concat(".class");
                    // 基于之前得到的第三方jar包依赖以及自己的jar包得到URL数组，进行遍历找出对应类名的资源
                    // 比如path是org/springframework/boot/loader/JarLauncher.class，它在jar:file:/Users/Format/Develop/gitrepository/springboot-analysis/springboot-executable-jar/target/executable-jar-1.0-SNAPSHOT.jar!/lib/spring-boot-loader-1.3.5.RELEASE.jar!/中被找出
                    // 那么找出的资源对应的URL为jar:file:/Users/Format/Develop/gitrepository/springboot-analysis/springboot-executable-jar/target/executable-jar-1.0-SNAPSHOT.jar!/lib/spring-boot-loader-1.3.5.RELEASE.jar!/org/springframework/boot/loader/JarLauncher.class
                    Resource res = ucp.getResource(path, false);
                    if (res != null) { // 找到了资源
                        try {
                            return defineClass(name, res);
                        } catch (IOException e) {
                            throw new ClassNotFoundException(name, e);
                        }
                    } else { // 找不到资源的话直接抛出ClassNotFoundException异常
                        throw new ClassNotFoundException(name);
                    }
                }
            }, acc);
    } catch (java.security.PrivilegedActionException pae) {
        throw (ClassNotFoundException) pae.getException();
    }
}
```

下面是LaunchedURLClassLoader的一个测试：

```java
// 注册org.springframework.boot.loader.jar.Handler URL协议处理器
JarFile.registerUrlProtocolHandler();
// 构造LaunchedURLClassLoader类加载器，这里使用了2个URL，分别对应jar包中依赖包spring-boot-loader和spring-boot，使用 "!/" 分开，需要org.springframework.boot.loader.jar.Handler处理器处理
LaunchedURLClassLoader classLoader = new LaunchedURLClassLoader(
        new URL[] {
                new URL("jar:file:/Users/Format/Develop/gitrepository/springboot-analysis/springboot-executable-jar/target/executable-jar-1.0-SNAPSHOT.jar!/lib/spring-boot-loader-1.3.5.RELEASE.jar!/")
                , new URL("jar:file:/Users/Format/Develop/gitrepository/springboot-analysis/springboot-executable-jar/target/executable-jar-1.0-SNAPSHOT.jar!/lib/spring-boot-1.3.5.RELEASE.jar!/")
        },
        LaunchedURLClassLoaderTest.class.getClassLoader());

// 加载类
// 这2个类都会在第二步本地查找中被找出(URLClassLoader的findClass方法)
classLoader.loadClass("org.springframework.boot.loader.JarLauncher");
classLoader.loadClass("org.springframework.boot.SpringApplication");
// 在第三步使用默认的加载顺序在ApplicationClassLoader中被找出
classLoader.loadClass("org.springframework.boot.autoconfigure.web.DispatcherServletAutoConfiguration");
```

### Spring Boot Loader的作用

`Spring Boot`在可执行jar包中定义了自己的一套规则，比如第三方依赖jar包在/lib目录下，jar包的URL路径使用自定义的规则并且这个规则需要使用`org.springframework.boot.loader.jar.Handler`处理器处理。它的Main-Class使用`JarLauncher`，如果是`war`包，使用`WarLauncher`执行。这些`Launcher`内部都会另起一个线程启动自定义的`SpringApplication`类，这些特性通过`spring-boot-maven-plugin`插件打包完成。

