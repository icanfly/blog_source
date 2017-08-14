title: SpringBoot文件上传解析
date: 2017-05-03

tags:
 - SpringBoot
 - Spring
 - 问题解析
categories:
 - 原创文章
---

在以往的开发过程中，Spring体系的文件上传一直使用的是commons-fileupload，在我们的项目中也是一样的，这两天在做公司的大数据查询平台，其中在做大文件上传时遇到了一些问题，记录如下：

# 问题现象

在开发环境、测试环境中，我们的环境是直接部署的jetty，也就是所有的访问直接经由浏览器后到达jetty服务器，运行良好。

但是当程序上到预发布环境时，文件上传出现了问题，文件上传不成功。服务器返回了一个错误，大体意思就是文件上传的请求体大于服务器端的最大设置，从响应的header看是nginx响应回来的，初步定位到是nginx文件大小上传受限，运维帮助修改了nginx配置后，这个错误不再出现。然而又出现了新的问题，Java程序在上传文件的过程中内存溢出了！

{% asset_img jetty.jpeg [Jetty内存溢出] %}

都知道在springmvc中，上传文件的话如果太大会写入临时磁盘文件来避免内存溢出的问题。当然我们的程序中也设置了：

{% asset_img 应用配置1.png [应用配置] %}

以上的配置信息的解析是：

```
文件上传的最大大小为256M
HTTP请求的最大大小为257M，因为HTTP请求包含了文件上传，故要大于文件上传大小。
```

<!--more-->

# 问题分析

一直以为该配置是commons-fileupload的组件的配置，但是从报错的信息来看，似乎并没有写入到文件中，而是不断地在内存中写入堆积，导致内存不断被占用导致最终的内存溢出。

而且从报错的地方看是调用了jetty的Multipart的解析器，而并没有看到有使用到commons-fileupload的文件上传解析，出于解决问题的必要，我在项目中强行依赖了jetty的包，然后在IDE中打开了内存溢出部位的源码：

{% asset_img jetty-multipart-write.png [Jetty处理文件上传] %}

上图中write方法负责文件上传请求解析过程中数据的管道写入，这个方法的大体意思是：
```
如果设置了最大的文件上传大小，并且读取的实际大小大于了最大限制，则抛出异常
如果设置了文件上传大小因子fileSizeThreshold(大于0),并且读取的实际大小大于fileSizeThreshold，并且文件为空，那么就初始化一个文件。
```
比较吸引人的是有一个“createFile”函数的调用，我在整个类中找了一遍，发现有几个地方有调用：

在parse方法中有创建MultiPart类的调用：

{% asset_img multipart-create.png [MultipartInputStreamParser.parse方法] %}

以及MultiPart内部类的open方法：

{% asset_img multipart-open.png [MultiPart.open方法] %}

isWriteFilesWithFilenames() 是一个属性值判断，在MultiPartInputStreamParser类的初始化过程中并没有对此进行设值，故可以判断该处返回false，反推可以知道在这里并没有调用createFile()这个函数。
同时对_out对象进行了初始化，默认是初始化为一个ByteArrayOutputStream2的内存缓冲。

那么调用createFile()这个函数的地方就只剩下write方法了。

我们再来看看createFile这个方法的实现：

{% asset_img multipart-createfile.png [MultipartInputStreamParser.createFile方法] %}

在这里方法里面，对parser的_file文件对象进行了初始化，同时该该文件包装了带缓冲的输出流bos对象，如果之前有在_out中的缓冲数据，则将缓冲数据写入到bos对象中，最后，最关键的一步：将_out对象原有的引用替换为bos对象，即将输出流指向了文件输出。

# 柳暗花明

到这里，整个文件上传的线条都非常明显了：


<b>
1、Jetty检测到文件上传标识时(multipart)，实例化MultiPartInputStreamParser，并调用parse方法进行解析。
2、parse方法进行一系列的逻辑处理，并初始化MultiPart，调用MultiPart对象的open方法，open方法先将_out输出流先初始化为内存缓冲流。
3、parse方法完成一系列初始化后，进行上传数据读取，并调用MultiPart对象的写入方法write，并在write方法中进行了逻辑判断与处理：当内存缓冲达到最大值时，改为写入文件的方式（防止内存占用过大）
</b>

那么现在最大的问题是内存溢出了，那么最大的可能性就是write的时候并没有引导进入createFile这个调用，而是一直在向内存缓冲中写入数据，导致的内存溢出。

```
if (MultiPartInputStreamParser.this._config.getFileSizeThreshold() > 0 && _size + length > MultiPartInputStreamParser.this._config.getFileSizeThreshold() && _file==null)
```
这个条件中唯一的可能性就是MultipartConfig中并没有对fileSizeThreshold进行设值。

到此问题已经浮现，那么现实中就是要找到jetty的配置，并进行设值处理。这个值的配置我其实找了好久，包括翻看了jetty的etc目录下的所有文件。

最后发现，其实这是Servlet 3.0的文件上传规范：http://www.blogjava.net/yongboy/archive/2011/01/15/346202.html

应用进行设置后，所有符合Servlet 3.0规范的WEB服务器会自动加载这个配置。但是我想Jetty中应该也是可以配置的，具体的细节就没有去考究了。

# 锦上添花

而springboot是直接支持修改这个参数的：

{% asset_img 应用配置.png [应用配置] %}

# 峰回路转

我翻看了springboot的文档时，文档上的错误注释误导了我：

{% asset_img springboot文档.png [springboot文档] %}

上图红圈中的文字表述： file-size-threshold指明了上传内容会被写入到文件中的最大因子，默认为0，这表示上传内容会被立即写入到磁盘文件中。

我之前的配置中并没有配置file-size-threshold，然而默认值可以认为是都是写入文件。这并不符合我现在看到的现象，最后在我怀疑的精神下，我试着将该配置修改为file-size-threshold=1MB后，发到预发布环境验证，居然一切OK了。

也由此反证官方代码注释文档存在错误引导，并且实际上的Jetty代码中也体现了当设置为0时并不会创建文件，而一直写入内存缓冲导致内存溢出的问题。

为此我也给官方提交了一个[issue](https://github.com/spring-projects/spring-boot/issues/9073)，等待官方的答复。

<hr>20170504 更新</hr>
官方回复了，确实存在这样的问题：

> This is somewhat complicated as the behaviour with a value of zero varies by container:
> 
> Tomcat will write to disk immediately
> Jetty will never write to disk
> Undertow ignores the file size threshold entirely
> So the javadoc is right if you're using Tomcat, but wrong if you're using Jetty or Undertow.

所以也警示我们，虽然大厂的开源东西比较可靠，但是也要有怀疑精神！

还有值得一说的就是，我最初在使用springboot的时候，引入了commons-fileupload包，我一直以为springboot使用的是这个文件上传包，直到内存溢出错误发生时，我才发现这个包根本没有用到。而springboot使用到了的是web容器Servlet 3.0自带的文件上传解析。这个配置从springboot的配置项：spring.http.multipart.enabled（默认为true）可以设置开启或者关闭。在没有显式设置的情况下，这个选项是默认开启的，也就有了我上面的问题的出现。


# 延伸思考

那么问题又来了，如何让springboot使用commons-fileupload组件进行文件上传呢。这里我就不多讲了，通过网上我已经搜索到了比较多的资料：

1、https://github.com/bobbylight/file-upload-example
2、http://stackoverflow.com/questions/32782026/springboot-large-streaming-file-upload-using-apache-commons-fileupload

# 总结

产生问题的原因很多，自己也有原因，妄自认为和springmvc的文件上传没有区别。归根到底还是对springboot的文档和机制不熟悉。对springboot的使用还没有太深入，这个有待加强。

## 文件上传几大限制

- 负载均衡器（反向代理）限制： 一些负载均衡器或者反向代理服务器存在请求大小最大限制
- 应用服务器限制： 一些WEB应用服务器存在请求最大大小限制
- 应用程序限制： 应用程序中进行了上传大小限制

## SpringBoot文件上传

SpringBoot默认自己开启了Servlet3.0规范的文件上传，不再需要commons-fileupload等第三方文件上传包
同时配置项如下：
```
spring.http.enabled=true （默认为true）
spring.http.multipart.max-request-size=     #这里设置最大请求大小，该大小必须大于max-file-size
spring.http.multipart.max-file-size=        #这里设置最大的文件上传大小
spring.http.multipart.file-size-threshold=  #这里设置当文件大小超过多大后，由内存缓冲改由写入磁盘文件
```

## SpringBoot使用CommonsFileUpload

使用方式见文末所说。
```
spring.http.enabled=false
```
并引入Commons Fileupload包，并进行相关初始化。

## 关于怀疑精神

任何时候都要对开源精神充满敬畏，同时少不了怀疑探索精神！






