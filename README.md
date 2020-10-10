# 缓存

## 缓存的作用
重用已获取的资源，减少延迟与网络阻塞，进而减少显示某个资源所用的时间，借助 HTTP 缓存，Web 站点变得更具有响应性。

其实我们对于页面静态资源的要求就两点
1. 静态资源加载速度
2. 页面渲染速度

页面渲染速度建立在资源加载速度之上，但不同资源类型的加载顺序和时机也会对其产生影响，所以缓存的可操作空间非常大

## 缓存的分类

![image](https://github.com/HankBass/front-end-UI-comparison/blob/master/images/cache-type.png?raw=true)

## 浏览器缓存
### HTTP 缓存
常见的 HTTP 缓存只能存储 GET 响应，对于其他类型的响应则无能为力。缓存的关键主要包括request method和目标URI（一般只有GET请求才会被缓存）。
浏览器在每次GET URL时都会先检查URL对应的缓存，除非你指定不使用缓存（强制刷新或者Disable Cache等）

通常浏览器缓存策略分为两种：`强缓存`和`协商缓存`

* 基本原理

  * 浏览器在加载资源时，根据请求头的expires和cache-control判断是否命中强缓存，是则直接从缓存读取资源，不会发请求到服务器。
  * 如果没有命中强缓存，浏览器一定会发送一个请求到服务器，通过last-modified和etag验证资源是否命中协商缓存，如果命中，服务器会将这个请求返回，但是不会返回这个资源的数据，依然是从缓存中读取资源
  * 如果前面两者都没有命中，直接从服务器加载资源

* 相同点

如果命中，都是从客户端缓存中加载资源，而不是从服务器加载资源数据；

* 不同点
强缓存不发请求到服务器，协商缓存会发请求到服务器

#### 缓存的优点
1. 缓存可以减少冗余的数据传输。
2. 缓存可以缓解网络瓶颈的问题。
3. 缓存可以降低对原始服务器的要求。
4. 缓存可以降低请求的距离时延。


#### 强缓存
强缓存通过`Expires`和`Cache-Control`两种响应头实现

##### Expires

`Expires`是`HTTP 1.0`提出的一个表示资源过期时间的`header`，它描述的是一个绝对时间，由服务器返回。
`Expires` 服务器的时间和浏览器的时间可能并不一致，所以HTTP1.1提出新的字段代替它
![image](https://github.com/HankBass/Cache-Guide/blob/master/images/Expires.png?raw=true)

##### Cache-Control

`Cache-Control` 出现于 `HTTP 1.1`，优先级高于 Expires ,表示的是相对时间
Cache-Control包括：
* max-age

max-age（单位为s）指定设置缓存最大的有效时间，定义的是时间长短。当浏览器向服务器发送请求后，在max-age这段时间里浏览器就不会再向服务器发送请求了。
我们来找个资源看下。比如shang.qq.com上的css资源，max-age=2592000，也就是说缓存有效期为2592000秒（也就是30天）。于是在30天内都会使用这个版本的资源，即使服务器上的资源发生了变化，浏览器也不会得到通知。max-age会覆盖掉Expires。

![image](https://github.com/HankBass/Cache-Guide/blob/master/images/max-age.png?raw=true)

* s-maxage

s-maxage（单位为s）同max-age，只用于共享缓存（比如CDN缓存）。
比如，当s-maxage=60时，在这60秒中，即使更新了CDN的内容，浏览器也不会进行请求。也就是说max-age用于普通缓存，而s-maxage用于代理缓存。如果存在s-maxage，则会覆盖掉max-age和Expires header。

* public

public 指定响应会被缓存，并且在多用户间共享。也就是下图的意思。如果没有指定public还是private，则默认为public。

![image](https://github.com/HankBass/Cache-Guide/blob/master/images/public.png?raw=true)

* private

private 响应只作为私有的缓存（见下图），不能在用户间共享。如果要求HTTP认证，响应会自动设置为private。

![image](https://github.com/HankBass/Cache-Guide/blob/master/images/private.png?raw=true)

* no-cache

no-cache 指定不缓存响应，表明资源不进行缓存。
但是设置了no-cache之后并不代表浏览器不缓存，而是在缓存前要向服务器确认资源是否被更改。因此有的时候只设置no-cache防止缓存还是不够保险，还可以加上private指令，将过期时间设为过去的时间。

![image](https://github.com/HankBass/Cache-Guide/blob/master/images/no-cache.png?raw=true)

![image](https://github.com/HankBass/Cache-Guide/blob/master/images/no-cache-2.png?raw=true)


* no-store

no-store 绝对禁止缓存。一看就知道如果用了这个命令当然就是不会进行缓存啦～每次请求资源都要从服务器重新获取。
* must-revalidate

must-revalidate指定如果页面是过期的，则去服务器进行获取。

* ...


#### 协商缓存

当浏览器对某个资源的请求没有命中强缓存，就会发一个请求到服务器，验证协商缓存是否命中，如果协商缓存命中，请求响应返回的http状态为`304`并且会显示一个`Not Modified`的字符串

协商缓存是利用的是【`Last-Modified`，`If-Modified-Since`】和【`ETag`、`If-None-Match`】这两对Header来管理的

Last-modified
服务器端文件的最后修改时间，需要和`cache-control`共同使用，是检查服务器端资源是否更新的一种方式。当浏览器再次进行请求时，会向服务器传送`If-Modified-Since`报头，询问`Last-Modified`时间点之后资源是否被修改过。如果没有修改，则返回码为`304`，使用缓存；如果修改过，则再次去服务器请求资源，返回码和首次请求相同为`200`，资源为服务器最新资源。

如下图，最后修改时间为2014年12月19日星期五2点50分47秒

![image](https://github.com/HankBass/Cache-Guide/blob/master/images/Last-modified.png?raw=true)

ETag

`ETag`就像一个指纹，资源变化都会导致`ETag`变化，跟最后修改时间没有关系，`ETag`可以保证每一个资源是唯一的

`If-None-Match`的`header`会将上次返回的`ETag`发送给服务器，询问该资源的`ETag`是否有更新，有变动就会发送新的资源回来

![image](https://github.com/HankBass/Cache-Guide/blob/master/images/ETag.png?raw=true)

> ETag的优先级比Last-Modified更高

使用`ETag`可以解决`Last-modified`存在的一些问题：
```
a、某些服务器不能精确得到资源的最后修改时间，这样就无法通过最后修改时间判断资源是否更新
b、如果资源修改非常频繁，在秒以下的时间内进行修改，而Last-modified只能精确到秒
c、一些资源的最后修改时间改变了，但是内容没改变，使用ETag就认为资源还是没有修改的。
```
整体流程图

![image](https://github.com/HankBass/Cache-Guide/blob/master/images/liucheng.png?raw=true)

状态码
* 200：强缓Expires/Cache-Control存失效时，返回新的资源文件
* 200(from cache): 强缓Expires/Cache-Control两者都存在，未过期，Cache-Control优先Expires时，浏览器从本地获取资源成功
* 304(Not Modified )：协商缓存Last-modified/Etag没有过期时，服务端返回状态码304

#### 缓存实现方式

方式一：客户端指定

1. request header 携带cache-control
2. 服务端接收到 header cache-control字段，并且使用该字段设置response 的 cache-control
3. 浏览器接收到response header,根据cache-control 设置缓存

方法二：服务端指定

1. 服务端接收到请求，设置response 的 cache-control并返回response
2. 浏览器接收到response header,根据cache-control 设置缓存

其他情况
1. 浏览器可以根据用户操作直接跳过缓存，比如勾选了控制台的Disable cache

#### 工程中遇到的问题

缓存可以说是性能优化中简单高效的一种优化方式了。缓存策略可以缩短网页请求资源的距离，减少延迟，并且由于缓存文件可以重复利用，还可以减少带宽，降低网络负荷。
同时，也会给升级迭代中的项目带来不便，也就是用户(包括测试人员)在没有清除缓存时，无法获取到我们更新过的静态资源，测试方面，无法及时测试线上已修复BUG及增加了与开发人员之间沟通成本。用户方面，由于缓存而无法及时得到项目升级的功能乃至于出现页面报错、样式错乱等问题，降低了用户体验。

> <font size=4 color="SteelBlue">以下内容摘自嘉德永丰前端团队--王挺</font>

##### 解决方案

###### 前后端未分离

方案一：手动配置字符后缀

通过在静态资源引用链接后面添加字符后缀参数，更改资源url，让浏览器无法用过url匹配本地对应的缓存资源而达到获取最新文件，阻止缓存的效果。

```
HTML

<script type="text/javascript" src="./js/jquery.js?rd=20190724" ></script>
<link type="text/javascript" src="./js/jquery.js?rd=20190724" />
```
> <font  color="green">优点</font> 能够灵活针对单个更新的文件进行缓存控制，优化迭代项目更新初次访问速度。

> <font  color="red">缺点</font> ： 需要每次去寻找对应引用更新文件的页面进行手动更改，对于比较大的项目、页面模块相对较多的页面需要浪费较大的工作量以及维护成本。

方案二：后端配置资源请求拦截器

后端添加一个拦截器，拦截js，css文件请求，加后缀进行重定向。

```
JAVA

String requestURL = req.getRequestURL().toString();
String queryStr = req.getQueryString();
if(requestURL != null && (requestURL.endsWith(".js") || requestURL.endsWith(".css"))){
  String newURL = null;
  if(StringUtils.isBlank(queryStr)){
    newURL = requestURL + "?rd=" + Constant.TIME;
    response.sendRedirect(newURL);
  }
}
```

> <font  color="green">优点</font> ：只需要写一个拦截的代码，无需手动配置文件引用后缀，每次更新修改配置变量值(Constant.TIME)即可,或者直接赋予版本号。

> <font  color="red">缺点</font> ： 每个静态文件请求会浪费200ms的重定向时间，影响页面加载速度。

方案三：使用后端变量添加后缀

后端配置变量，使用框架模板语法在文件引用处加上后缀。

```
HTML

<link href="index02/css/index.css?rd=<%=Constant.TIME%>" rel="stylesheet" type="text/css"/>
```
```
JAVA

public class Constant {
  public static final String TIME = "201907221537";
}
```

> <font  color="green">优点</font> ：配置完之后，每次更新修改配置变量值(Constant.TIME)即可,或者直接赋予版本号。

> <font  color="red">缺点</font> ： 第一次配置需要找到所有js/css文件的引用地方，比较麻烦。


###### 前后端分离

方案一：配置服务器禁止缓存html根文件
前后端分离项目一般不会存在缓存问题，因为项目打包的时候，webpack为自动为文件加上hash值，主要是浏览器缓存了html根文件，导致项目整体都不会获取最新文件。解决方案是配置服务器，禁止缓存根文件。

```
Nginx

location = /index.html {
  add_header Cache-Control "no-cache, no-store";
}
```

##### 服务器配置


方案一：配置`ETag`

`ETag`是`HTTP`协议提供的若干机制中的一种Web缓存验证机制，并且允许客户端进行缓存协商。这就使得缓存变得更加高效，而且节省带宽。如果资源的内容没有发生改变，Web服务器就不需要发送一个完整的响应。`ETag`也可用于乐观并发控制，作为一种防止资源同步更新而相互覆盖的方法。
由Web服务器根据URL上的资源的特定版本而指定。如果那个URL上的资源内容改变，一个新的不一样的`ETag`就会被分配。用这种方法使用`ETag`即类似于指纹，并且他们能够被快速地被比较，以确定两个版本的资源是否相同。


`ETag`典型用法：

当一个URL被请求，Web服务器会返回资源和其相应的`ETag`值，它会被放置在`HTTP`的“`ETag`”字段中： `ETag`: "686897696a7c876b7e", 然后，客户端可以决定是否缓存这个资源和它的`ETag`。以后，如果客户端想再次请求相同的URL，将会发送一个包含已保存的`ETag`和“`If-None-Match`”字段的请求。 `If-None-Match`: "686897696a7c876b7e" 客户端请求之后，服务器可能会比较客户端的`ETag`和当前版本资源的`ETag`。如果`ETag`值匹配，这就意味着资源没有改变，服务器便会发送回一个极短的响应，包含`HTTP` “`304` 未修改”的状态。`304`状态告诉客户端，它的缓存版本是最新的，并应该使用它。

##### 总结

> 目前来看ETag是最优解决缓存方案，能够根据文件修改时间自动判断是否获取最新资源文件，无需人工手动干预，但是，对于ETag服务器配置这一块经验尚缺，在项目的应用上还不能保证稳定性（经试验，在tomact上默认使用Etag无需配置，但是可能由于项目内代码的影响导致浏览器缓存机制无效，目前不能很好的重现，本地测试时有时无）。
对于jsp这类框架而言，实际项目目前采用方案三解决缓存问题，目前也建议采用该方案，即在每次打包测试，发布版本时，修改配置变量值即可。
