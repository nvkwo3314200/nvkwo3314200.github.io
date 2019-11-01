---
title: Spring Resource设计与实现
layout: post
categories: Spring源码分析
tags: Spring源码分析 Resource设计与实现
excerpt:  最近开始写关于Spring源码分析的相关内容， Spring是一个很优秀的开源框架，希望在分析的过程序，让自己能借鉴其中的编程思想。
文章有些非原创，会加以整理并写明原文链接。 此文介绍Spring Resource 相关的设计 
---

### 写在前面
几乎所有的项目都会，用到关于配置文件的解读，Spring 框架也是。现在就一起在解读Spring中是怎么设计并实现，多样化的资源文件的操作的。


原文出处：https://blog.csdn.net/fyzlucky2015/article/details/77943994

### 分析Spring的资源抽象设计和实现

Spring把其资源做了一个抽象，底层使用统一的资源访问接口来访问Spring的所有资源。也就是说，不管什么格式的文件，也不管文件在哪里，到Spring 底层，都只有一个访问接口，Resource。整个Spring的资源定义在spring-core 的包中。我们先来看Resource整个的类图：
![Srping Resource 类图](/assets/imgs/20191111SpringResourceClassPic.png)

Resource的抽象比较简单，由几个重要的接口和相关抽象类及其实现的类组成。

### 一.资源接口类 Resource、InputStreamSource、WritableResource、ContextResource

#### 1、重要接口 Resource和InputStreamSource
Resource接口，是整个Spring框架对资源的抽象访问接口。它继承于InputStreamSource接口。Spring把资源抽象了，那么到底抽象成什么了呢？看代码一目了然，所有资源高度抽象为二进制流，也就是不管你资源文件是什么格式，也不管你资源在哪里，Spring底层访问的都是文件的二进制流，这样就可以统一访问了。因此，Resource并不是资源的根接口，根接口是 InputStreamSource
该接口非常简单，只有一个方法  ：获取资源的二进制流对象，所有的不同的资源类型都要去实现该接口
```java_holder_method_tree
    InputStream getInputStream() throws IOException;
```
Resource接口，继承了InputStreamSource，拥有的方法如下,这些方法就构成了Spring对资源的访问能力
```java_holder_method_tree
    public interface Resource extends InputStreamSource
    {
       /**
        * 判断资源实际上是否物理存在
        */
       boolean exists();
       /**
        * 判断资源是否可读
        */
       default boolean isReadable()
       {
          return true;
       }
       /**
        * 判断资源是否打开
        */
       default boolean isOpen()
       {
          return false;
       }
       /**
        * 判断资源是否是文件
        */
       default boolean isFile()
       {
          return false;
       }
       /**
        * 获取资源的URL的句柄
        */
       URL getURL() throws IOException;
       /**
        * 获取资源的URI的句柄
        */
       URI getURI() throws IOException;
       /**
        * 获取资源的文件句柄File
        */
       File getFile() throws IOException;
       /**
        *获取资源的可读字节管道
        */
       default ReadableByteChannel readableChannel() throws IOException
       {
          return Channels.newChannel(getInputStream());
       }
       /**
        * 获取资源的长度
        */
       long contentLength() throws IOException;
       /**
        * 获取资源的最后一次修改时间
        */
       long lastModified() throws IOException;
       /**
        *用于创建相对于当前Resource代表的底层资源的资源，
        * 比如当前Resource代表文件资源“d:/test/”则createRelative（“test.txt”）
        * 将返回表文件资源“d:/test/test.txt”Resource资源。
        */
       Resource createRelative(String relativePath) throws IOException;
       /**
        * 返回当前Resource代表的底层文件资源的文件路径，
        * 比如File资源“file://d:/test.txt”将返回“d:/test.txt”，
        * 而URL资源http://www.javass.cn将返回“”，因为只返回文件路径。
        */
       @Nullable
       String getFilename();
       /**
        * 返回当前Resource代表的底层资源的描述符，
        * 通常就是资源的全路径（实际文件名或实际URL地址）。
        * @see Object#toString()
        */
       String getDescription();
    }
```
#### 2、WritableResource和ContextResource

Resource接口定义了资源的可访问等一系列的操作，但有些资源需要可写入的，如文件对象，因此还定义了WritableResource接口，继承了Resource接口，表示资源具有可写入的能力。WritableResource接口拥有三个方法，如下：
```java_holder_method_tree
/**
 * Spring资源的扩展接口，该接口支持Spring资源的可写入
 */
public interface WritableResource extends Resource
{
   /**
    * 判断资源是否可写入
    */
   default boolean isWritable()
   {
      return true;
   }
   /**
    * 获取资源写入的二进制流
    */
   OutputStream getOutputStream() throws IOException;
   /**
    * 获取资源可写入的字节管道对象
    */
   default WritableByteChannel writableChannel() throws IOException
   {
      return Channels.newChannel(getOutputStream());
   }
}
```
ContextResource 接口也继承了Resource接口，表示可以从关闭的上下文Context中获取资源的路径，这样应用程序上下文也就有了返回上下文路径的能力。
```java_holder_method_tree

/**
 * Spring的资源扩展接口，支持从关闭的上下文中获取资源的路径
 */
public interface ContextResource extends Resource
{
   /**
    * 从关闭的上下文Context中获取资源的路径
    */
   String getPathWithinContext();
}
```

Spring资源的访问接口，介绍完了，下面看看 Resource的抽象实现



### 二、Resource接口的抽象实现类 AbstractResource

AbstractResource 是个抽象类，Resource接口的大部分方法的默认实现。具体分析看代码洛：

```java_holder_method_tree
/**
 * Spring资源的抽象实现类。
 * 该抽象类实现了大部分的资源操作
 *   该抽象类未能实现 资源的根接口，即InputStreamSource ,该接口由具体资源子类去实现
 */
public abstract class AbstractResource implements Resource
{
   /**
    * 判断资源是否存在
    */
   @Override
   public boolean exists()
   {
      //先尝试判断文件是否存在，如果资源是文件形式，判断文件是否存在
      try
      {
         return getFile().exists();
      } catch (IOException ex)
      {
         //如果资源不是文件，回溯到二进制流，看二进制流是否能打开，如果可以，则就存在了
         try
         {
            InputStream is = getInputStream();
            is.close();
            return true;
         } catch (Throwable isEx)
         {
            return false;
         }
      }
   }
   /**
    * 判断资源是否可读，抽象类中总是认为资源是可读的
    */
   @Override
   public boolean isReadable()
   {
      return true;
   }
   /**
    * 判断资源是否打开，抽象类中总是认为资源是关闭的
    * This implementation always returns {@code false}.
    */
   @Override
   public boolean isOpen()
   {
      return false;
   }
   /**
    * 判断资源是否是文件
    */
   @Override
   public boolean isFile()
   {
      return false;
   }
   /**
    * 该资源解析为URL，需要具体的子类去实现，这里抽象类假设都不能解析URL
    */
   @Override
   public URL getURL() throws IOException
   {
      throw new FileNotFoundException(getDescription() + " cannot be resolved to URL");
   }
   /**
    * 该资源解析为URI
    */
   @Override
   public URI getURI() throws IOException
   {
      //获取资源的URL，判断是否可以转换为 URI
      URL url = getURL();
      try
      {
         return ResourceUtils.toURI(url);
      } catch (URISyntaxException ex)
      {
         throw new NestedIOException("Invalid URI [" + url + "]", ex);
      }
   }
   /**
    * 该资源解析为Fiel 抽象类假设都不嫩解析为File
    */
   @Override
   public File getFile() throws IOException
   {
      throw new FileNotFoundException(getDescription() + " cannot be resolved to absolute file path");
   }
   /**
    * 获取资源的字节管道
    */
   @Override
   public ReadableByteChannel readableChannel() throws IOException
   {
      return Channels.newChannel(getInputStream());
   }

   /**
    * 获取资源的长度，通过获取资源二进制流计算资源的长度  字节为单位
    */
   @Override
   public long contentLength() throws IOException
   {
      //先获取资源的输入流对象
      InputStream is = getInputStream();
      try
      {
         //将资源对象全部遍历一次，获取资源的字节数，来求得资源的大小
         long size = 0;
         byte[] buf = new byte[255];
         int read;
         while ((read = is.read(buf)) != -1)
         {
            size += read;
         }
         return size;
      } finally
      {
         try
         {
            is.close();
         } catch (IOException ex)
         {
         }
      }
   }
   /**
    * 获取资源最后的修改时间
    */
   @Override
   public long lastModified() throws IOException
   {
      long lastModified = getFileForLastModifiedCheck().lastModified();
      if (lastModified == 0L)
      {
         throw new FileNotFoundException(getDescription() +
               " cannot be resolved in the file system for resolving its last-modified timestamp");
      }
      return lastModified;
   }
   /**
    *获取文件用于获取资源最后的修改时间
    */
   protected File getFileForLastModifiedCheck() throws IOException
   {
      return getFile();
   }
   /**
    *用于创建相对于当前Resource代表的底层资源的资源， 子类重写
    */
   @Override
   public Resource createRelative(String relativePath) throws IOException
   {
      throw new FileNotFoundException("Cannot create a relative resource for " + getDescription());
   }
   /**
    * 获取资源的路径  子类重写
    */
   @Override
   @Nullable
   public String getFilename()
   {
      return null;
   }
   /**
    * 重写 toString（）
    */
   @Override
   public String toString()
   {
      return getDescription();
   }
   /**
    * 重写equals
    */
   @Override
   public boolean equals(Object obj)
   {
      return (obj == this ||
            (obj instanceof Resource && ((Resource) obj).getDescription().equals(getDescription())));
   }
   /**
    * 重写HashCode
    */
   @Override
   public int hashCode()
   {
      return getDescription().hashCode();
   }
}
```
##### 总结：

①、没有实现资源的根接口 InputStreamSource ，方法getInputStream() 留给具体的子类去实现

②、没有实现Resource接口的 getDescription() 方法，留给子类去实现，资源文件默认的equals()、hashCode() 都通过这个来判断



### 三、资源的具体实现类分析

在定义资源访问接口和默认的抽象实现之后，需要针对具体的资源设计其实现方式。Spring将常见的资源按照资源类型和路径分为了7大组，分别如下：

FileSystemResource         代表文件系统资源，以操作系统文件路径的方式访问

PathResource       代表文件系统资源，以Path对象访问

AbstractFileResolvingResource   代表需要解析的路径资源，如类资源Class  、URL资源等等  是个抽象类，有三个具体的实现

ByteArrayResource   代表字节数组资源

VfsResource  代表JBoss的虚拟文件系统VFS

InputStreamResource  代表输入二进制流的资源

DescriptiveResource 代表资源描述的资源，可以理解为资源的元数据(元资源) 不指向任何的实际资源对象

以上7类具体的资源或抽象的资源全部继承了 资源抽象类 AbstractResource。

#### 1、FileSystemResource         

代表文件系统资源，以操作系统文件路径的方式访问，继承了 AbstractResource抽象类，并实现了 资源可写入接口 WritableResource 

内部实现中，以 文件对象File和文件路径字符串 path  组成。也就是通过文件路径，转换为文件对象File，是java.io.File 的一个封装。实现了 资源的根接口，获取了文件的 输入流：
```java_holder_method_tree
/**
 * 实现 InputStreamSource 接口
 * @see java.io.FileInputStream
 */
@Override
public InputStream getInputStream() throws IOException
{
   return Files.newInputStream(this.file.toPath());
}
/**
 *实现资源的可写入，资源输出流
 * @see java.io.FileOutputStream
 */
@Override
public OutputStream getOutputStream() throws IOException
{
   return Files.newOutputStream(this.file.toPath());
}
```
#### 2、PathResource

代表文件系统资源，以Path对象访问，和FileSystemResource类似，也是表示文件资源，只不过该类是通过Path对象访问，可以理解为对java.nio.file.Path 对象的封装，本质上还是File对象处理的。实现类比较简单，这里不在做说明了

#### 3、ByteArrayResource 

该类是字节数组资源的实现，也就是说，该类资源是以字节数组表示的。内部实现也比较简单，使用不可变的字节数组存储 和一个不可变的描述对象，代码也不贴了。

#### 4、VfsResource

资源以虚拟文件VFS的形式存在，可以使用该实现类。VFS是一个虚拟文件系统，Linux的系统中所有文件的顶层都设计为虚拟的VFS，它能一致的访问物理文件系统、jar资源、zip资源、war资源等，VFS能把这些资源一致的映射到一个目录上，访问它们就像访问物理文件资源一样，而其实这些资源不存在于物理文件系统。

该实现类，封装了一个Object对象，所有的操作都是通过这个包装的对象的反射来实现的。当然，内部具体实现细节，可以通过工具类 VfsUtils 调用。

#### 5、InputStreamResource

资源以输入的二进制流的形式存在，内部实现是以不可变的InputStream加上不可变的描述符组成。比较简单，类似与 ByteArrayResource

#### 6、DescriptiveResource

资源的描述的资源，也就是说如果一个资源仅仅是以文本描述的方式存在，可以使用该类，可以理解为资源的元数据吧？

它不指向任何具体的实际物理资源。觉得没什么用

#### 7、AbstractFileResolvingResource

代表需要解析的路径资源，如类资源Class  、URL资源等等，是个抽象类，有3个常用的子类。带有路径解析的资源类似这样：http://....; ftp://.......; file://......; classpath://....; jar://........;war://.......  等等，因此需要一个抽象类，把这些需要解析的在形式化上统一。该类，使用Java的 统一资源定位符，URL对象，来表示类似这些需要解析的对象。

该类有常见的3个子类的实现：

ClassPathResource、UrlResource、ServletContextResource

##### ①、ClassPathResource

ClassPathResource这个资源类表示的是类路径下的资源，资源以相对于类路径的方式表示。这个资源类有3个成员变量，分别是一个不可变的相对路径、一个类加载器、一个类对象。这个资源类可以相对于应用程序下的某个类或者相对于整个应用程序，但只能是其中之一，取决于构造方法有没有传入Class参数。是一个常用的资源对象。他有4个构造函数：
```java_holder_method_tree
/**
 * 初始化，以类的路径作为参数
 */
public ClassPathResource(String path)
{
   this(path, (ClassLoader) null);
}
/**
 * 初始化，以类的路径和类加载器作为参数
 */
public ClassPathResource(String path, @Nullable ClassLoader classLoader)
{
   Assert.notNull(path, "Path must not be null");
   String pathToUse = StringUtils.cleanPath(path);
   if (pathToUse.startsWith("/"))
   {
      pathToUse = pathToUse.substring(1);
   }
   this.path = pathToUse;
   this.classLoader = (classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader());
}
/**
 * 初始化，以类的路径和类的字节码作为参数
 */
public ClassPathResource(String path, @Nullable Class<?> clazz)
{
   Assert.notNull(path, "Path must not be null");
   this.path = StringUtils.cleanPath(path);
   this.clazz = clazz;
}
/**
 * 初始化，以类的路径、类加载器和类的字节码作为参数
 */
protected ClassPathResource(String path, @Nullable ClassLoader classLoader, @Nullable Class<?> clazz)
{
   this.path = StringUtils.cleanPath(path);
   this.classLoader = classLoader;
   this.clazz = clazz;
}
```
重要方法：
```java_holder_method_tree
/**
 * 获取类路径下的资源的二进制流
 */
@Override
public InputStream getInputStream() throws IOException
{
   InputStream is;
   if (this.clazz != null)
   {
      is = this.clazz.getResourceAsStream(this.path);
   } else if (this.classLoader != null)
   {
      is = this.classLoader.getResourceAsStream(this.path);
   } else
   {
      is = ClassLoader.getSystemResourceAsStream(this.path);
   }
   if (is == null)
   {
      throw new FileNotFoundException(getDescription() + " cannot be opened because it does not exist");
   }
   return is;
}
/**
 * 获取类路径下的资源描述符
 */
@Override
public String getDescription()
{
   StringBuilder builder = new StringBuilder("class path resource [");
   String pathToUse = path;
   if (this.clazz != null && !pathToUse.startsWith("/"))
   {
      builder.append(ClassUtils.classPackageAsResourcePath(this.clazz));
      builder.append('/');
   }
   if (pathToUse.startsWith("/"))
   {
      pathToUse = pathToUse.substring(1);
   }
   builder.append(pathToUse);
   builder.append(']');
   return builder.toString();
}
```

##### ②、UrlResource

UrlResource这个资源类封装了可以以URL表示的各种资源。它有3个属性，URI、URL，以及规范化后的URL，用于资源间的比较以及计算HashCode。他是对java.net.URL的封装。

对比一下其构造函数：
```java_holder_method_tree
/**
 * 原始的URI，如果可用，使用URI和文件访问
 */
@Nullable
private final URI uri;
/**
 * 原始的URL，用于实际访问
 */
private final URL url;
/**
 *标准化的URL，用于比较
 */
private final URL cleanedUrl;
/**
 * 初始化，给定URI路径
 */
public UrlResource(URI uri) throws MalformedURLException
{
   Assert.notNull(uri, "URI must not be null");
   this.uri = uri;
   this.url = uri.toURL();
   this.cleanedUrl = getCleanedUrl(this.url, uri.toString());
}

/**
 * 初始化，给定URL路径
 */
public UrlResource(URL url)
{
   Assert.notNull(url, "URL must not be null");
   this.url = url;
   this.cleanedUrl = getCleanedUrl(this.url, url.toString());
   this.uri = null;
}

/**
 * 初始化，给定物理路径，然后转换到URL路径，当给定的物理路径需要可用
 */
public UrlResource(String path) throws MalformedURLException
{
   Assert.notNull(path, "Path must not be null");
   this.uri = null;
   this.url = new URL(path);
   this.cleanedUrl = getCleanedUrl(this.url, path);
}

/**
 * 初始化，给定路径样式描述符和路径字符串
 */
public UrlResource(String protocol, String location) throws MalformedURLException
{
   this(protocol, location, null);
}

/**
 * 初始化，给定路径样式描述符、路径和格式化符号
 */
public UrlResource(String protocol, String location, @Nullable String fragment) throws MalformedURLException
{
   try
   {
      this.uri = new URI(protocol, location, fragment);
      this.url = this.uri.toURL();
      this.cleanedUrl = getCleanedUrl(this.url, this.uri.toString());
   } catch (URISyntaxException ex)
   {
      MalformedURLException exToThrow = new MalformedURLException(ex.getMessage());
      exToThrow.initCause(ex);
      throw exToThrow;
   }
}
```
再看看，获取输入流的方法：
```java_holder_method_tree
/**
 * 获取URL的二进制输入流
 */
@Override
public InputStream getInputStream() throws IOException
{
   //使用URL打开URL连接
   URLConnection con = this.url.openConnection();
   //设置是否需要使用缓存
   ResourceUtils.useCachesIfNecessary(con);
   try
   {
      //获取二进制流
      return con.getInputStream();
   } catch (IOException ex)
   {
      // 如果打开了资源，需要关闭Http连接
      if (con instanceof HttpURLConnection)
      {
         ((HttpURLConnection) con).disconnect();
      }
      throw ex;
   }
}
```

##### ③、ServletContextResource

Web容器上下文的资源，相对于Web应用程序根目录的路径加载资源。该类的实现在Spring-web 的包中。继承抽象类AbstractFileResolvingResource和实现了接口 ContextResource。

该类在实现方式上，内部使用不可变的ServetContext 对象和一个String类型的路径对象。类中所有的操作都是基于该对象。

核心方法：
```java_holder_method_tree
/**
 * 获取web应用程序上下文的资源二进制流
 */
@Override
public InputStream getInputStream() throws IOException {
   InputStream is = this.servletContext.getResourceAsStream(this.path);
   if (is == null) {
      throw new FileNotFoundException("Could not open " + getDescription());
   }
   return is;
}
```


### 四、资源中最后一个类EncodedResource

EncodedResource类是辅助类，从名字上可以看出，它是一个编码类。资源加载的时候，是采用操作系统默认的编码方式，为解决编码不统一的问题，Spring的IOC获取资源后，需要把资源重新编码一下。例如，在Spring应用程序上下文的 XmlBeanDefinitionReader 类中，获取了 资源后，需要对资源进一步解析，在解析之前，调用 new EncodedResource();解析资源重新编码：
```java_holder_method_tree
/**
 * 读取Resource解析
 */
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException
{
   //先将资源Resource进行编码
   return loadBeanDefinitions(new EncodedResource(resource));
}
EncodedResource 类，内部采用一个Resource属性、一个Charset属性和一个表示编码类型的String属性。看下几个核心的方法：

/**
 * 使用 编码规则 获取 {@code java.io.Reader}读取对象 
 */
public Reader getReader() throws IOException
{
   if (this.charset != null)
   {
      return new InputStreamReader(this.resource.getInputStream(), this.charset);
   } else if (this.encoding != null)
   {
      return new InputStreamReader(this.resource.getInputStream(), this.encoding);
   } else
   {
      return new InputStreamReader(this.resource.getInputStream());
   }
}

/**
 * 获取编码后的二进制流对象
 */
@Override
public InputStream getInputStream() throws IOException
{
   return this.resource.getInputStream();
}
```

五、总结

Spring框架中对资源的设计和实现，大致就是这样了，从设计到实现上并不复杂。关键的是要善于使用抽象。不得不说，Spring框架的Resource工具是十分强大的，整个实现都在spring-code 包中，以后如果要读取文件，就可以直接使用该包了。当然，资源访问只是Spring的一小部分，资源访问的应用是在应用程序上下文ApplicationContext中调用ResourceLoad进行资源的加载。这里等后面分析ApplicationContext的初始化过程的时候再来具体分析。



