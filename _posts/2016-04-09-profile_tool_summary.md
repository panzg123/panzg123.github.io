---
layout: post
title: 分析工具总结
categories: 网络编程
---
# 一、前端性能分析工具

## 1.1概述

常见的Web前端性能分析工具主要有以下几种：

**1\. 在线网站类：**

*   WebPageTest,站点性能评测网站，地址[http://www.webpagetest.org/](http://www.webpagetest.org/).
*   ShowSlow, yslow的数据收集与展示平台, 它是一个开源的php项目可以用来与firefox的yslow插件、page speed插件或者dynatrace通信，收集插件或程序所发送过来的信息并集中展示。只需要在dynatrace安装目录下进行一些设置，即可自动实现上传结果到showslow平台作为存档、分析及监控。

**2\. 浏览器插件类：**

*   FireBug,它可以对页面进行调试，可以记录所有网页的访问耗时，下载的资源等。
*   Page Speed,基于firebug的web页面优化的评测工具，同时还有支持chrome的插件，因为是google产的。直接打开FF的firebug或chrome的开发人员工具，切换到page speed标签，浏览一个网页然后点击分析即可，分析完成后会针对规则打出一个成绩，并告诉你哪些规则你没有符合。
*   Speed Tracer,基于chrome的插件，同样是有google产的，这个是web前端页的性能记录和分析工具，同时还提供一个规则和建议的评测。使用:[https://developers.google.com/web-toolkit/speedtracer/get-started](https://developers.google.com/web-toolkit/speedtracer/get-started)。 这个工具收集的东西主要是资源或事件消耗的时间，它会实时的记录某个页面的加载过程，并且一直跟踪所有的事件；在易用性方面数据可以到出来，还有可以根据时间轴来分析具体某端的性能规则和建议。
*   Yslow：基于firebug的评测分析工具，yahoo产；和page speed类似工具，会给出页面的评分和优化说规则，同时会提供页面下载资源的统计分析功能，还提供了一些小工具，如js运行检测，图片的优化工具，未符合规则的资源有哪些等等。总的来说是page speed的增强版。

**3\. 独立程序类：**

*   DynaTrace Ajax Edition：基于IE，firefox的插件，对于FF需要版本支持，需要独立安装文件（50多M）。其可支持到函数级的度量分析，此外其它工具能支持的功能这个工具都支持的。
*   Fiddler，Microsoft的一款web调试工具，它会记录所有本地的http通信。同时支持ie插件版
*   HttpAnalyzer，和fiddler原理一样的工具，不过功能比fiddler更加易用。同时支持ie，ff插件版，此外独立版程序提供http调试功能，写基于http通信的程序使用这个调试比较不错，之前写接口测试工具时就用的这个调试的。
*   HttpWatch：这个工具更适合用于站点的页面性能分析。

在本次报告中，主要演示YSlow和dynaTraceAjaxEdition两个功能强大的工具。

## 1.2 YSlow

YSlow 根据一组高性能 web 页面准则，通过检查页面上所有组件，包括由 JavaScript创建的，来分析 web 页面性能。YSlow 是一个集成了 Firebug web 开发工具的Firefox 插件；它可提供提升页面性能的建议、总结组件性能、显示页面统计数据并提供性能分析的工具。

**1) 启动界面如下：**

[![image](http://images0.cnblogs.com/blog/442949/201505/300949167825242.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949157198284.png)

选择合适的性能分析规则集，对页面进行评分，可以根据自己的需求来选择：

l YSlow(V2) 包含了所有 23 条规则，针对大型站点，进行全面的性能分

l Classic(V1) 包含起初经典的 13 条规则（参考：Steve Souders的《High Performance Web Sites 14 Rules for Faster Page》），针对大型网站的，只是当时 Steve Souders 的研究还没有现在V2这么完善；

l Small site or Blog 这个规则集包含 15 个规则，适用于小型网站或博客；

YSlow 的测试规则是很灵活的，除了默认的3条规则集选项外，你还可以根据自己的需要点击“Edit”按钮，编辑自己的自定义规则集，如下图：

**2) YSlow的Grade等级视图:**

进入 Grade 分析结果页面，你会看到如下的界面（本站首页 V2 规则集的分析结果）：

[![image](http://images0.cnblogs.com/blog/442949/201505/300949187514045.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949179239158.png)

雅虎的前端工程师们曾经针对网站速度提出了非常著名34条准则：《Best Practices for Speeding Up Your Web Site》。而现在将34条精简为更加直观的23条，并针对每一条给出从F~A的评分以及最终的总分。完整的23条规则在附录。

**3) 网页评分结果：**

从上图中可以看出，该博客得分是82分，其中关于后端打分比较低。同时，YSlow也给出了得分低的标签和相应的建议，比如：

1\. Use a Content Delivery Network (CDN) – 没有CND支持的。

2\. Add Expires headers – F：服务器的缓存

3\. Compress components with gzip – D：服务器端gzip压缩文件，

4\. Configure entity tags (ETags) – F：服务器端清除ETags，也是无能为力；

**4) Yslow 的 Components 组件视图**

[![image](http://images0.cnblogs.com/blog/442949/201505/300949201416802.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949194854201.png)

通过 Components 视图会以表格的显示显示网页总的 HTTP 请求数量，页面体积以及各个元素占用的流量的大小。图表的数据是死的，关键需要我们自己分析图标数据反映出来的问题，然后针对 YSlow 的规则建议去优化相关的性能，解决存在的问题。

例如上图，该博客中JavaScript 文件和 CSS 的背景图片占的整个网站的流量比较大，我就要想办法用 YUI 的 Compressor 去压缩JS文件体积，使用 CSS Sprites 技巧合并图片还有利用 Smush.it 等工具压缩背景的体积。稍后介绍 Tools 界面的时候，我会继续介绍一些前端代码的压缩工具和图片压缩工具的。

**5) YSlow 的 Statistics 饼状统计信息视图**

[![image](http://images0.cnblogs.com/blog/442949/201505/300949215637803.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949208762661.png)

YSlow 除了提供 Components 的表格分析统计图外，另外还提供了 Statistics 饼状视图供我们分析性能数据。这种视图更加直观，反映出各种元素在页面中所占的流量比例。并且提供了页面在无缓存和有缓存情况下的流量请求对比。比如上图，我的首页在有缓存的时候用户只需要请求 73.2K 的html页面就可以了。这也是下载的规则中为什要求：Add Expires headers 添加缓存的原因。

另外 Statistics 视图也是我们做前端性能分析报表的一个好工具，可以直接用这个里饼图做我的性能分析报表的饼图的。

**6) YSlow 的 Tools 辅助工具**

[![image](http://images0.cnblogs.com/blog/442949/201505/300949228764005.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949221574917.png)

YSlow 不光只为我们分析前端性能，而且还提供了几款使用的辅助工具：

*   JSLint： Douglas Crockford 用 JavaScript 开发的 JavaScript 代码质量分析工具。很好，很强大，也很打击人！
*   All JS：显示页面中所有的 JavaScript 代码的路径和具体的代码。
*   All JS Beautified：格式化所有的 JavaScript 代码，让你分析阅读其他网站的 JavaScript 代码更加方便。
*   All JS Minified：用 Minify 算法压缩所有的JavaScript 代码。
*   All CSS：显示页面中所有的 CSS 文件的路径和具体的代码。
*   YUI CSS Compressor： YAHOO 代码的 CSS 压缩工具，压缩的比例还是跟高的。如果再 gzip 一下，那就十分的到位了。
*   All Smush.it™： 可以通过这个在线工具无损坏的压缩网站页面上的所有有图片，并且提供打包下载。只是原来的非 JPG 格式的图片都会压缩成 PNG 格式的图片。
*   Printable View：可以通过它打印出Grade、Components 和 Statistics 的分析报表。

## 1.3 [dynaTrace Ajax Edition](http://www.cnblogs.com/justinw/archive/2009/11/26/1611397.html)

dynaTrace Ajax Edition 是一个强大的底层追踪、前端性能分析工具，该工具不仅能够记录浏览器的请求在网络中的传输时间、前端页面的渲染时间、DOM 方法执行时间以及 JavaScript 代码的解析和执行时间，还可以跟踪 JavaScript 从执行开始，经过本地的 XMLHttpRequest、发送网络请求、再到请求返回的全过程。

dynaTrace Ajax 目前有两个版本，免费版和商业版，它们之间的区别可查看 版本比较，本文主要是针对免费版本的介绍。在 3.0 之前的版本只支持运行在 IE 浏览器下，包括 IE6、IE7、IE8， 在 3.0 Beta 版之后可同时支持在 IE 和 Firefox 浏览器上的性能跟踪。

下面就一个例子来演示dynaTraceAjax的功能：

a) **配置好dynaTraceAjax**后，点击Start Tracing，在google搜索栏目中输入“dynaTrace”，选择其中的google maps，让地址显示这个地址。然后我们可以看到从浏览器上搜集到的信息。

b) **打开summary视图**，可以看到如下信息。

[![image](http://images0.cnblogs.com/blog/442949/201505/300949239854976.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949233917861.png)

Summary视图显示了session记录中所有访问过的URL链接的信息。点击顶部表格中某个具体的url下方就会更新图表和时间线来显示所选链接的数据。在这个视图中我们可以得到以下信息：

*   l 载入页面所耗时间：Page Load Time[ms]（页面载入时间[毫秒]）栏显示从页面开始载入到浏览器派发onload事件所经历的时间；
*   l 网络请求花了多长时间：下方NetWork饼图从DNS解析、网络连接、服务器响应以及网络传输方面详细分解网络请求过程，由于网络内容在这段时间里是并行下载的，所以NetWork[ms]栏则显示的是所有网络请求时间的总和；
*   l 下载了多少以及什么类型的资源文件，对比有多少资源是从浏览器缓存读取的（Resource条形图）；
*   l 通过JavaScript触发器（脚本载入、载入完毕、鼠标、键盘等事件）和JavaScript API或库执行的所有JavaScript函数一共耗时多长时间；
*   l 渲染页面所占时间。浏览器必须计算布局并渲染页面显示。浏览器重新计算布局和重绘的时间取决于你的HTML、样式表以及动态DOM操作。Rendering[ms]栏显示了页面在渲染工作上实际消耗的时间。
*   l 屏幕下方的时间轴图显示精确的页面生命周期：该图反映了页面进程中网络资源下载、JavaScript执行、页面发生渲染的时间，CPU占用情况，以及发生了哪些事件。例如：onLoad事件、鼠标或键盘交互、XmlHttpRequests等。

在我们的例子中，以下内容引起了我的注意：

*   l maps.google.com页面的**页面载入时间为6.5秒**：这是页面在派发onload事件之前浏览器初始化html和所有引用的对象所消耗的时间；
*   l 这页面的**网络时间耗时12秒**。当我观察该页面的Network饼图时，我发现50%多的时间消耗在传输内容（这也可能意味着我的网速慢）上，42%的时间花在服务器响应上（指过了多长时间服务器开始响应），以及8%的时间消耗在与服务器建立物理连接上。
*   l 总耗时3.6秒的JavaScript也是重要角色。JavaScript trigger饼图显示时间的具体消耗情况：载入script耗时2.1秒，onload事件派发耗时1.3秒，剩下的由鼠标点击事件处理占用。
*   l 时间轴还显示页面发出了2个XmlHttpRequest请求。它由一个小图标标注在event行中请求发生的时间点上。下一节将进行更详细的讨论。

**c) Timeline视图（时间轴视图）——近距离观察页面生命周期事件**

时间轴视图可以通过双击Cockpit面板中的Timeline节点打开，或者在Summary视图中通过在某个URL上点击右键，选择“Drill Down-TimeLine”打开。

[![image](http://images0.cnblogs.com/blog/442949/201505/300949252981178.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949246735820.png)

我们可以在此视图下做如下观测：

1.网络请求并行下载来自6个不同域的内容；

2.到浏览器派发onload事件大约需要6.5秒（图中由IE图标标识）；

3.从maps.gstatic.com下载main.js耗时2.41秒（鼠标悬停在这段上可以看到详细信息）；

4.main.js下载完成后，可以看到脚本实际执行耗时1.1秒，并触发两个JavaScript文件的下载（1秒）和另外2个JavaScript的执行（2秒）；

5.CPU占用率显示JavaScript执行占用的浏览器CPU时间；

6.Event轴显示了鼠标点击事件，XmlHttpRequest事件和onUnload事件。

**d) PurePath视图（路径视图）——JavaScript、DOM和Ajax问题的详细说明**

在PurePath列表中选择一个活动，PurePath或JavaScript追踪树将更新显示当前所选活动的信息。PurePath树显示了JavaScript代码执行过程，包括每个方法执行的时间和调用的参数以及返回值（我们在第二步中开启了参数捕获选项）。代码跟踪也追踪计时器调用，并把这些调用当做树的一部分。我们不仅能看到JavaScript方法，也能看到DOM访问和AJAX请求。

**e) Network视图（网络视图）——分析“对话”**

[![image](http://images0.cnblogs.com/blog/442949/201505/300949299238224.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949259856320.png)

Network视图显示了发生在浏览器或各自页面中的所有网络请求。从每个请求上我们可以到的资源是否来自浏览器缓存（Cached栏），请求类型（Network或Ajax），HTTP状态，Mime类型，大小，在DNS、网络连接、服务器响应、网络传输和等待上消耗的时间。界面底部显示了HTTP请求和响应头以及返回的实际内容。这个页面中有趣的地方是从mt0.google.com和mt1.google.com取回数据时的等待时间。每个浏览器针对每个域名都有一个连接数上限。

**f) Hotspot视图（热点视图）——哪些地方降低了性能**

打开HotSpot视图来分析我访问过的页面中所有的JavaScript、DOM和渲染动作。

[![image](http://images0.cnblogs.com/blog/442949/201505/300949313767468.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949306104367.png)

上方的表格以聚合的方式显示所有JavaScript、DOM和渲染动作。我们可以看到130次的Drawing动作，946次Reflow动作以及在一个div上调用了一个匿名函数1293次。这个列表是按总的执行时间排序的，性能越高排序越靠上。双击这些中的一个，将显示它执行前后的追踪结果。Back Traces表显示了谁来调用这些动作，Forward Traces表显示的是这个动作又触发了哪些动作。界面底部显示了在Back Traces树或Forward Traces树中选中的JavaScript的源码。

g) 自动化数据集

除了用dynaTrace手动收集数据，也可以用脚本工具代替人工方式驱动浏览器自动收集数据。当你用像Selenium、Watir、WebAii这样的工具运行测试脚本是，dynaTrace可以自动从每个浏览器session中收集性能信息。这里有篇博文《5步实现性能自动分析》，教你如何用Watir配合dynaTrace自动分析。

h) 与你的同事分享数据

收集信息并制成离线版分析数据是dynaTrace的功能之一，上面你已经领略到了。如果你在别人的代码中发现了问题，或者你想跟你的同事分享你的发现，你需要一种简单的方式共享你收集到的数据。这可以通过把你的session导出为session文件实现。在Cockpit面板中的右键菜单或者工具条里的导出按钮能完成这一工作。导入文件的操作与此类似。

# 二、云端性能分析工具

## 2.1 概述

Linux 平台上的性能工具有很多，系统性能专家 Brendan D. Gregg用下面三张图片分别总结了 Linux 各个子系统以及监控、测试、优化这些子系统所用到的工具。

#### **监控**

[![image](http://images0.cnblogs.com/blog/442949/201505/300949326732198.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949320486840.png)

#### **测试**

[![image](http://images0.cnblogs.com/blog/442949/201505/300949337519627.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949332668311.png)

#### **优化**

[![image](http://images0.cnblogs.com/blog/442949/201505/300949348444827.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949342821255.png)

我们只需要简单的工具就可以对 Linux 的性能进行监测，他们各有所长，可以灵活应用在不同的地方，详情如下：

![](/images/server_profile_tool.png)

下面我用工具tcpdump和vmstat为例来介绍系统性能监测方法。

## 2.2 tcpdump

Tcpdump是使用最广泛的命令行网络数据包分析器或数据包嗅探程序之一，用于捕捉或过滤在网络上通过某个接口接收或传输的TCP/IP数据包。

tcpdump采用命令行方式，它的命令格式为：

```
tcpdump [ -AdDeflLnNOpqRStuUvxX ] [ -c count ]
[ -C file_size ] [ -F file ]
[ -i interface ] [ -m module ] [ -M secret ]
[ -r file ] [ -s snaplen ] [ -T type ] [ -w file ]
[ -W filecount ]
[ -E spi@ipaddr algo:secret,... ]
[ -y datalinktype ] [ -Z user ]
[ expression ]
```

下面介绍tcpdump的常见用法：

**1\. 默认启动**

*   l tcpdump

普通情况下，直接启动tcpdump将监视第一个网络接口上所有流过的数据包。

**2\. 监视指定网络接口的数据包**

*   l tcpdump -i eth1

如果不指定网卡，默认tcpdump只会监视第一个网络接口，一般是eth0，下面的例子都没有指定网络接口。　

**3\. 监视指定主机的数据包**

*   l 打印所有进入或离开sundown的数据包

tcpdump host sundown

*   l 也可以指定ip,例如截获所有210.27.48.1 的主机收到的和发出的所有的数据包

tcpdump host 210.27.48.1

*   l 截获主机hostname发送的所有数据

tcpdump -i eth0 src host hostname

*   l 监视所有送到主机hostname的数据包

tcpdump -i eth0 dst host hostname

**4. 监视指定主机和端口的数据包**

*   l 如果想要获取主机210.27.48.1接收或发出的telnet包，使用如下命令

tcpdump tcp port 23 host 210.27.48.1

*   l 对本机的udp 123 端口进行监视 123 为ntp的服务端口

tcpdump udp port 123

**5. 监视指定网络的数据包**

*   l 打印本地主机与Berkeley网络上的主机之间的所有通信数据包(nt: ucb-ether, 此处可理解为'Berkeley网络'的网络地址,此表达式最原始的含义可表达为: 打印网络地址为ucb-ether的所有数据包)

tcpdump net ucb-ether

*   l 打印所有通过网关snup的ftp数据包(注意, 表达式被单引号括起来了, 这可以防止shell对其中的括号进行错误解析)

tcpdump 'gateway snup and (port ftp or ftp-data)'

*   l 打印所有源地址或目标地址是本地主机的IP数据包

tcpdump ip and not net localnet

**6. 监视指定协议的数据包**

*   l 打印TCP会话中的的开始和结束数据包, 并且数据包的源或目的不是本地网络上的主机.(nt: localnet, 实际使用时要真正替换成本地网络的名字))

tcpdump 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0 and not src and dst net localnet'

## 2.3 vmstat

vmstat是一个提供报告虚拟内存统计的工具。它包括了系统内存、交换和实时处理器利用率。为了运行vmstat，只需在控制台输入vmstat。

### 1\. 不带参数运行vmstat会显示vmstat的默认结果。

[![image](http://images0.cnblogs.com/blog/442949/201505/300949356577241.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949352352413.png)

其中各指标的信息如下：

*   l Procs：procs有 r列和b列。r列代表等待访问CPU进程的数量。而b列意味着睡眠进程的数量。在这些列的下面，是它们的值。从上面的截图中，我门有2个进程正在等待访问CPU，0个睡眠进程。
*   l Memory：memory有swpd、 free、 buff 和 cache 这些列。这些信息和命令free -m相同。swpd列显示了有多少内存已经被交换到了交换文件或者磁盘。free列显示了未分配的可用内存。buff列显示了使用中的内存。cache列显示了有多少内存可以被交换到交换文件或者磁盘上如果一些应用需要他们。
*   l Swap：swap显示了从交换系统上发送或取回了多少内存。si列告诉我们每秒有多少内存被从swap移到真实内存中（In）。so列告诉我们每秒有多少内存被从真实内存移到swap中（Out）。
*   l I/O：io依据块的读写显示了每秒输入输出的活动。bi列告诉我们收到的块数量，bo列告诉我们发送的块数量。
*   l System：system显示了每秒的系统操作数量。in列显示了系统每秒被中断的数量。cs列显示了系统为了处理所以任务而上下文切换的数量。
*   l CPU：CPU告诉了我们CPU资源的使用情况。us列显示了处理器在非内核程序消耗的时间。sy列显示了处理器在内核相关任务上消耗的时间。id列显示了处理器的空闲时间。wa列显示了处理器在等待IO操作完成以继续处理任务上的时间。

### **2. 按间隔时间运行vmstat：**

作为一个统计工具，使用vmstat最好的方法是使用间隔时间。你可以间断地捕捉系统状态。让我假设以5秒的间隔运行vmstat。只需要在你的控制台中输入vmstat 5就行。

[![image](http://images0.cnblogs.com/blog/442949/201505/300949408605631.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949398135445.png)

命令将会每5秒运行一次，直到你按下Ctrl-C来终止它。你也可以使用第二个参数来控制vmstat运行的次数。下面的命令会以5秒的间隔运行7次vmstat。

[![image](http://images0.cnblogs.com/blog/442949/201505/300949419384061.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949414077731.png)

### **3. 显示活跃和非活跃内存**

在vmstat后加入-a选项，这是个示例：

[![image](http://images0.cnblogs.com/blog/442949/201505/300949429541706.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949424543620.png)

### **4. 显示磁盘统计数据总结**

vmstat也可以打印系统磁盘活动统计，使用-D选项就行。

[![image](http://images0.cnblogs.com/blog/442949/201505/300949441268163.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949434079075.png)

### **5. 显示单位**

可以选择你想打印的显示单位字符。在-S后跟上k (小写，1000)、 K (大写，1024)、 m (小写，1000000)、 M (大写，1048576) 字节. 如果你不想选择单位，默认使用的是K (1024)。

### **6. 显示某个磁盘分区的详细统计数据**

使用-p选项跟上设备名。这里有个例子。

[![image](http://images0.cnblogs.com/blog/442949/201505/300949449855591.png "image")](http://images0.cnblogs.com/blog/442949/201505/300949445948006.png)

### **7. 文件**

vmstat实际上是使用这些文件获取的数据。

/proc/meminfo

/proc/stat

/proc/*/stat

# 三、附录：

## 3.1《Best Practices for Speeding Up Your Web Site》23条规则

1\. Minimize HTTP Requests

2\. Use a Content Delivery Network

3\. Avoid empty src or href

4\. Add an Expires or a Cache-Control Header

5\. Gzip Components

6\. Put StyleSheets at the Top

7\. Put Scripts at the Bottom

8\. Avoid CSS Expressions

9\. Make JavaScript and CSS External

10\. Reduce DNS Lookups

11\. Minify JavaScript and CSS

12\. Avoid Redirects

13\. Remove Duplicate Scripts

14\. Configure ETags

15\. Make AJAX Cacheable

16\. Use GET for AJAX Requests

17\. Reduce the Number of DOM Elements

18\. No 404s

19\. Reduce Cookie Size

20\. Use Cookie-Free Domains for Components

21\. Avoid Filters

22\. Do Not Scale Images in HTML

23\. Make favicon.ico Small and Cacheable

## 3.2 影响页面Loading时间的因素

在 Web2.0 应用程序中，JavaScript 的执行常常会阻碍浏览器端资源的下载和增加页面的 Loading 的时间，导致这个问题的因素主要有：

1) 浏览器本身的因素，例如在 IE 浏览器下 ,CSS Selectors 的查找速度相比其他浏览器如 Firefox 相对会慢很多

2) CSS 对相同对象的查询次数太多

3) 存在太多 Ajax 的 XMLHttpRequest 请求

4) JS、CSS、图片数量过多，增加了网络传输开销

5) DOM 的尺寸太大，一方面会增加内存的占用，另一方也会影响页面的性能，例如 CSS 的查询操作

6) 丰富的 DOM 操作，例如创建新的 DOM 元素或是作为 HTML 形式添加新的元素等

7) 过多的事件处理绑定（Event Handler Bindings）等

# 四、参考

csdn,[WEB前端性能分析--工具篇](http://blog.csdn.net/five3/article/details/7686376), [http://blog.csdn.net/five3/article/details/7686376](http://blog.csdn.net/five3/article/details/7686376 "http://blog.csdn.net/five3/article/details/7686376")

开源中文社区，Linux性能监测专题，[http://linux.cn/topic-linux-system-performance-monitoring.html](http://linux.cn/topic-linux-system-performance-monitoring.html "http://linux.cn/topic-linux-system-performance-monitoring.html")

**作者**：[panzg123](https://github.com/panzg123)

**出处**：同步自博客[西芒xiaoP](http://www.cnblogs.com/panweishadow/)

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。