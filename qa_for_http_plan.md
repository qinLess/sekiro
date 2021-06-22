
其实一直不想回答这个问题的，因为Sekiro和http转发调用可以说就完全没有比较的必要。
但是依然有太多人再没有了解sekiro和sekiro类似方案的原理的情况问我Sekiro相比NPS+frida/NPS + NanoHTTP的优势，
说实话每次回答这样的问题我感觉是受到了极大的侮辱。

为了避免再有人侮辱我让我生气，我这里统一的回答一下这个问题。


首先Http+NanoHttp方案是我在2018年在国内某大厂内玩剩下的方案，Sekiro前生方案是hermes [https://gitee.com/virjar/hermesagent](https://gitee.com/virjar/hermesagent),
hermes就是标准的Http+NanoHttp模式，并且Hermes相比NaoHttp做了太多的优化，而且做了大量工程化的封装工作。当时效果是50台手机将当时我在的厂的一层楼的办公网络打瘫痪，业务RPC大约250qps、签名QPS2000，占用20台2核4G服务器作为转发层，服务器使用python（Django）开发。
即使这样，线上Hermes系统负载和报警都非常高，系统经常雪崩，只要有较大业务调用的时候我就不能好好休息。20台服务器经常CPU占用70%以上。

那么很多人说，我用nps或者frp做内网穿透，挂载NanoHttp黑盒调用，压测的时候支持的qps可以到上千，性能看起来是非常高，为啥和我说的hermes的表现不一致呢。
所以我来给你解释下，为啥你压测可以qps上千。

### 要搞清楚通道流量压测和业务调用的区别，搞清楚签名计算和业务黑盒调用的区别
如果你的压测是空调用，比如在NanoHttp中只是简单返回一个字符串或者简单实现一个算法函数调用。

1. 你的整个通道的上面的单报文其实是非常小的，请求只是两三个参数，返回就一个签名md5。所以不会有吞吐堵塞问题
2. 调用计算是不耗时的，手机在同步计算场景下都可以满足几千qps。此时你完全不需要考虑多线程以及多线程下任务堵塞问题

你可以简单改一下你的压测代码，每次调用的内容你返回一个500k的大json文件，每次调用你让线程休眠20秒。你再看看你的压测qps能不能还有这么高
(大json和休眠是线上真的会出现的情况，也是最普遍的需求)

### 要搞清楚短时稳定和长期稳定的区别
demo短时跑和作为一个稳定长久运行，对于一个方案来说挑战是非常大的。手机不像服务器，没有专门的运维，没有7*24小时运行保证机制，没有稳定的物理网卡环境。你会遇到多种真实挑战

1. OOM（内存溢出）：手机你要是用来跑业务，两三天不重启就容易各种鬼畜了，对于app差不多一个小时就应该重启一次。要不然跑着跑着就进程僵死了。此时NPS基本感知不到app的ANR事件，你的调用就是无限timeout
2. 物理网卡和Wi-Fi环境的区别，如果你用的是开发版子，连接的网线。那可能网络还可以一些。但是如果是真实手机，Wi-Fi没多久就会抖动。甚至当同一个Wi-Fi下面手机挂的多了点儿，Wi-Fi网络很容易直接瘫痪。如果Wi-Fi信号发生了一分钟的瘫痪，那么手机Wi-Fi重连大概需要在90秒之后，这期间这个Wi-Fi下面的所有手机会全部掉线。也是调用雪崩了


## Hermes在http上遇到的真实挑战
作为一个曾经经历过大流量长期压测的线上系统，我这里大概讲一下当时面临的真实问题（所有NPS/frp + NanoHttp在长期高压状态都会遇到）。

### 1. 单机房Wi-Fi信号稳定性
作为一个牛逼大厂，每个办公区都有待空调的的小机房。有专门运维配置内网穿透，业务服务器到小机房的网络走专线所以是非常稳定的。但是架不住我们的调用量太大，
无线信道很容易被占满，单一AP也非常容易打满。

所以经常遇到路由器直接被打死的情况发生，一旦路由器被打死，整个RPC调用会直接瘫痪。

### 2. 心跳不及时
在多台手机上面维护一个统一的接口服务，并且实现多台服务器的自动加入和离开集群。是一个服务治理和负载均衡处理的问题。这个在行业内都是需要有专门框架管理和维护才能做的比较完美的。
常见的框架：zookeeper、doubbo等。

由于是http转发，手机在线与离线，需要通过一种机制来进行管理。常见思路有两种

1. 通过手机上报心跳，报告自己的ip和状态。超时未上报则认为机器下线。这是hermes当时的方式。这样有一个非常大的问题，就是手机app在上报间隙重启，此时中心服务认为手机在线，然后就会发送调用。但是手机其实并没有启动服务。中心服务会socket connect timeout，如果单纯是timeout也还可以接受，主要是服务器 转发是blocking的，timeout会导致服务器卡线程池资源，一旦有大量线程被占用无响应。服务器就无法接受新的tcp请求，在之后就是502 bad gateway
2. 通过调用探测感知。这是网关LB一般的做法，比如ngnix。具体思路就是探测下游手机的调用情况，如果连续多次失败，那么对特定手机进行调度频率降权。然后这样还是有挑战，首先手机本身业务就有可能是耗时调用，不能依赖手机业务调用结果作为服务器熔断判断基准，一般大公司会是定制ng，单独开发healthcheckAPI用于链路探测，healthcheckAPI不做业务，每次都是简单返回特定静态文件。这样healthcheckAPI就不是业务API，不会被业务调用失败导致手机下线。解决了healthcheckAPI之后，还有另一个复杂问题，就是一般的服务器熔断场景下，节点下线是非常不频繁的，节点下线只有在系统发布和故障期间。但是我们的场景，节点下线只是app重启，切换设备id，app被Android系统回收导致的重启等。下线频率太大了，所以探测机制如何才能做得灵敏，这个问题基本无法解决的。在框架层面我们基本要允许10%的流量就是因为app重启导致下线的。

### 3. BIO转发和Django单进程模型
说实话，python在大多数情况真的不考虑性能问题。目前sekrio只使用一台2核4G就可以完成支撑当时hermes 20台2核4G服务器集群的业务。并且不会存在当时那么多问题。
这里主要的就是BIO转发和DJango框架uwsgi单进程问题。由于是BIO转发的，每个http调用转发都会有线程被占用，直到手机方面返回数据该线程才会被释放。当手机下线服务器无感知和手机接受业务返回前重启app，以及手机接受业务但是返回数据耗时很长的情况下。服务器都会占用线程资源。
我记得当时公司的uwsgi配置是一台服务器40个线程（我后来调整到了80个），这就说 20 * 80 = 1600，整个集群最大的并发数量只有一千六百个，这里正常都有30%由于手机掉线等原因导致调用连续不通达形成的线程占用。

所以毫无疑问，在黑盒调用框架中心调度端来说，只要你是做黑盒调用框架，必然需要NIO来解决。否则吹再牛逼都是不牛逼的

单进程模型也是导致hermes性能瓶颈的一个大问题，当时如果使用的java tomcat实现，肯定也会比Django好很多。每个链接单个进程导致服务器的内存占用非常高

### 4. 雪崩
先解释下，雪崩就是指系统长期处于亚健康状态，看起来是正常接受和处理请求，但是其实负荷非常高，当遇到一些不稳定因素（请求突然变多，下游某个系统短期突然不稳定），系统就直接崩溃。并且由于系统被打垮之后，上游调用没有减少（甚至上游由于失败增加重试导致调用量翻倍），系统无法正常恢复，因为每一个节点在上线一瞬间都会接受本来应该分布在多个节点的全部请求，一瞬间被打死然后下线。
这样除非上游系统停止调用，否则系统永远无法恢复正常服务。

hermes系统在当时是经常雪崩的，当然后来我做了很多保护和优化机制。但是我并没有从根本解决导致雪崩的本因。

导致雪崩的原因：

1. Wi-Fi网络抖动，中心服务无法监控到机房Wi-Fi抖动，接受业务请求转发到Wi-Fi网络但是无响应，socket timeout之后才会放行。之后所有服务器线程被占用，LB层转发502 bad gateway
2. 单台手机无保护无法上线，雪崩状态手机一台一台的上线。每台手机上线之后马上接受到大量处理能力之外的请求，然后手机被打死下线
3. redis并发限制，调度中心使用redis调度多台手机，相当于多台手机使用了同一个redis锁。当qps达到2000之后其实redis锁对并发的影响已经非常大了（每秒对同一个redis操作2000次，同时需要考虑多台服务器网络延时问题）。
4. 心跳不及时导致上线状态判断不及时。有些业务会封杀设备id，所以在抓取一段时间之后，会重启app切换设备ID。这个时候http服务下线，但是服务器不知道，服务器还是会转发请求，当然这个转发不会成功，但是会大量占用线程资源。导致服务器的线程资源被占用完了，正常的调用转发无线程可用。

对于hermes来说，天然太多问题需要处理，甚至有部分问题无法处理。最开始sekrio只是hermes内部的一个子模块，我希望通过长链接的方式解决 timeout频繁问题，再后来sekiro独立从hermes中抽取，仅仅保留RPC功能，变成了大家看到的开源版Sekiro。
所以Sekiro是踩在NanoHttpD方案的尸体上演变出来的一套框架，两种方案已经没有比较的意义了。

## NanoHttp方案的弱点

### NanoHttp是BIO独立线程工作的
说NanoHttpD高性能的，我觉得你可能真的对网络了解太少。NanoHTTPD本来设计就一个非常精简的demo，整个NanoHTTPD可就是一个不到2500行的java文件，
而作为常用的java http服务器tomcat代码量不知道多少万行，要说作为一个不考虑性能，或者demo场景下讨论NanoHTTPD还行，但是如果要说NanoHTTPD可以高性能那就是扯淡。

NanoHTTPD会为每个请求创建一个独立线程，而不是使用线程池。我们都知道创建线程和创建对象成本不是同一个数量级的，线程创建会有对应的运行时、堆栈空间、参与CPU资源调度。NanoHTTPD没有做线程资源复用，大并发一定存在瓶颈

### NanoHTTPD没有心跳
NanoHTTPD本身支持keepAlive，但是每一条TCP通道不会有心跳，所以tcp无法长久复用，NanoHTTPD调用需要建立多条tcp通道。
如果你开发了多台手机的NanoHTTPD网关转发，那么网关层需要做好tcp复用（Django这种一个请求一个进程的就基本不会有keep alive了，除非你手动做隧道关联）

### NanoHTTPD需要内网穿透配合
也不能说有内网穿透不好，但是多一个新的软件，你做keepalive就存在一定不稳定了。大多数情况内网穿透需要专门运维配合。如果手机换地方了可能就不行了。当然nps可以把内网穿透搞到手机里面，但是也得绑定到特定服务器的特定端口

### NanoHTTPD无法做到群控调度
每个NanoHTTPD手机都是暴露成一个http接口，拥有不同的http地址。如果你想要线上使用那么一定的有中间层，实现手机的http地址自动注册、管理、下线
当然你说就一台手机写死到路由器里面这就当我没说，如果你只有一台手机，怎么玩儿都无所谓。

如果是http接口暴露，一旦有多台手机，那么他们的状态管理和轮训策略还有元数据注册都是相当复杂的。还是那个问题，如果短时间启动一台手机跑算法会很快，但是多台服务器搞集群，长久运行就有相当多的的工作了

### NanoHTTPD需要再次封装API框架
他只是一个http框架，作为黑盒调用还需要二次封装。但是毫无疑问你们没有太多工程经验，做不了优雅的API包装

# 结语
如果你们团队并不是长久运行黑盒调用服务，只是临时启动一个手机跑个签名算法，抓完了就收起来歇菜。直接NanoHTTPD随便玩儿。如果你们公司稍微大一点儿，至少愿意花钱买代理，那么基本会要求你做一个长久的服务，
那么建议你使用Sekiro。如果你们公司拥有长期大量的爬虫业务，且qps足够大、手机足够多、办公点足够多那么Sekiro商业版将会是你最好的选择