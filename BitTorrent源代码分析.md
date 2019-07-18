# [guoqp](https://www.cnblogs.com/guoqp/)



随笔 - 12, 文章 - 2, 评论 - 0, 引用 - 0

## [BitTorrent源代码分析](https://www.cnblogs.com/guoqp/p/6381838.html)



       tracker服务器是BT下载中必须的角色。一个BT client 在下载开始以及下载进行的过程中，要不停的与 tracker 服务器进行通信，以报告自己的信息，并获取其它下载client的信息。这种通信是通过 HTTP 协议进行的，又被称为 tracker  HTTP 协议，它的过程是这样的：

​       client 向 tracker 发一个HTTP 的GET请求，并把它自己的信息放在GET的参数中；这个请求的大致意思是：我是xxx（一个唯一的id），我想下载yyy文件，我的ip是aaa，我用的端口是bbb。。。

​       tracker 对所有下载者的信息进行维护，当它收到一个请求后，首先把对方的信息记录下来（如果已经记录在案，那么就检查是否需要更新），然后将一部分（并非全部，根据设置的参数已经下载者的请求）参与下载同一个文件（一个tracker服务器可能同时维护多个文件的下载）的下载者的信息返回给对方。

​       Client在收到tracker的响应后，就能获取其它下载者的信息，那么它就可以根据这些信息，与其它下载者建立连接，从它们那里下载文件片断。



关于client和tracker之间通信协议的细节，在“BT协议规范”中已经给出，这里不再重复。下面我们具体分析 tracker服务器的实现细节。



从哪里开始？

​       要建立一个 tracker服务器，只要运行 bttrack.py 程序就行了，它最少需要一个参数，就是 –dfile，这个参数指定了保存下载信息的文件。Bttrack.py 调用 track.py 中的 track()函数。因此，我们跟踪到 track.py 中去看track() 函数。



Track.py：track()

​       这个函数首先对命令行的参数进行检查；然后将这些参数保存到 config 字典中。在BT中所有的工具程序，都有类似的处理方式。



接下来的代码：

1. 
2. 
3. ​       r = RawServer(Event(), config['timeout_check_interval'], config['socket_timeout'])
4. 
5. ​    t = Tracker(config, r)
6. 
7. ​    r.bind(config['port'], config['bind'], True)
8. 
9. ​    r.listen_forever(HTTPHandler(t.get, config['min_time_between_log_flushes']))
10. 
11. ​    t.save_dfile()
12. 
13. 
14. 

*复制代码*


首先是创建一个 RawServer 对象，这是一个服务器对象，它将实现一个网络服务器的一些细节封装起来。不仅tracker服务器用到了 RawServer，我们以后还可以看到，由于每个 client端也需要给其它 client 提供下载服务，因此也同时是一个服务器，client的实现中，也用到了RawServer，这样，RawServer的代码得到了重用。关于 RawServer的详细实现，在后面的小节中进行分析。

接着是创建一个 Tracker对象。

然后让RawServer绑定在指定的端口上（通过命令行传递进来）。

最后，调用 RawServer::listen_forever() 函数，使得服务器投入运行。

最后，在服务器因某些原因结束运行以后，调用 Tracker::save_dfile() 保存下载信息。这样，一旦服务器再次投入运行，可以恢复当前的状态。





其它信息：

1、  BT源码的分布：

把BT的源码展开之后，可以看到有一些python程序，还有一些说明文件等等，此外还有一个BitTorrent目录。这些 python程序，实际是一些小工具，比如制作 metafile的btmakemetafile.py、运行tracker服务器的bttrack.py、运行BT client端的 btdownloadheadless.py 等等。而这些程序中，用到的一些 python 类的实现，都放在子目录 BitTorrent 下面。我们的分析工作，通常是从工具程序入手，比如 bttrack.py，而随着分析的展开，则重点是看 BitTorrenet子目录下的代码。

BT作者 Bram Cohen 在谈到如何开发可维护的代码的一篇文章中（http://www.advogato.org/article/258.html），其中提到的一条就是开发一些小工具以简化工作，我想BT的这种源码结构，也正是作者思想的一种体现吧。



2、  我们看到，python和我们以前接触的 c/c++ 不一样的第一个地方就是它的函数在定义的时候，不用指定参数类型。既然这样，那么，在调用函数的时候，你可以传递任意类型的参数进来。例如这样的函数：

1. 
2. def foo(arg):
3. 
4. ​        print type(arg)
5. 
6. ​      
7. 
8. ​       你可以这样来调用：
9. 
10. ​       a = 100
11. 
12. ​       b = “hello world”
13. 
14. ​       foo(a)
15. 
16. ​       foo(b)
17. 
18. 
19. 
20. ​       输出结果是：
21. 
22. ​       <type ‘int’>;
23. 
24. ​       <type ‘str’>;
25. 
26. 
27. 

*复制代码*


       这是因为，第一次调用 foo()的时候，传递的是一个整数类型，而第二次调用的时候，传递的是一个字符串类型。



这种参数具有动态类型的特性，是 c/c++等传统的语言是所不具备的。这也是 python 被称为动态语言的一个原因吧。C++的高级特性模板，虽然也使得参数类型可以动态化，但使用起来，远没有python这么简单方便。

Tracker 服务器源码分析之三：HTTPHandler 类

本篇文章分析 HTTPHandler类，它在 HTTPHandler.py 文件中。

上一篇我们讲到， RawServer 只负责网络 I/O，也就是从网络上读取和发送数据，至于读到的数据如何分析，以及应该发送什么样的数据，则交给 Handler 类来处理。如果是用 c++ 来实现的话，那么 Handler 应该是一个接口类（提供几个虚函数作为接口），但是 python 动态语言的特性，并不需要专门定义这么一个接口类，所以实际上并没有 Handler 这么一个类。任何一个提供了以下成员函数的类，都可以作为一个 Handler 类来与 RawServer 配合，它们是：

external_connection_made()：在建立新的连接的时候被调用

data_came_in()：连接上有数据可读的时候被调用

connection_flushed()：当在某个连接上发送完数据之后被调用

​       HTTPHandler 就是这样一个 Handler 类，它具备以上接口。

​       HTTPHandler 代码很少，因为它把主要工作又交给 HTTPConnection 了。

​       我们看 HTTPHandler 类的这几个函数：

l         external_connection_made()：

每当新来一个连接的时候，就创建一个 HTTPConnection 类。

l         data_came_in()：

当连接上有数据可读的时候，调用 HTTPConnection::data_came_in()。我们接下去看HTTPConnection::data_came_in()。

我们知道，BT client端与 tracker服务器之间是通过tracke HTTP 协议来进行通信的。HTTP协议分为请求（request）和响应（response），具体的协议请看相关的 RFC 文档。我这里简单讲一下。

对 tracke 服务器来说，它读到的数据是 client 端的HTTP 请求。

HTTP请求以行为单位，行的结束符是“回车换行”，也就是 ascii 字符 “\r”和“\n”。
第一行是请求的 URL，例如：

GET              /announce?ip=aaaaa;port=bbbbbbb       HTTP/1.0
这行数据被空格分为三部分，

第一部分GET表示命令，其它命令还有POST、HEAD等等，常用的就是GET了。

第二部分是请求的URL，这里是 /announce?ip=aaaaa;port=bbbbbbb。如果是普通的上网浏览网页，那么URL 就是我们要看的网页在该web服务器上的相对路径。但是，这里的URL仅仅是交互信息的一种方式，client 端把要报告给 tracker 的信息，放在URL中，例子里面是 ip 和 port，更详细的信息请看“BT协议规范”中 tracker 协议部分。

第三部分是HTTP协议的版本号，在程序中忽略。

接下来的每一行，都是HTTP协议的消息头部分，例如：

Host:www.sina.com.cn

Accept-encoding:gzip

通过消息头，tracker服务器可以知道 client端的一些信息，这其中比较重要的就是 Accept-encoding，如果是 gzip ，那么说明 client 可以对 gzip 格式的数据进行解压，那么tracker服务器就可以考虑用 gzip 把响应数据压缩之后再传回去，以减少网络流量。我们可以在代码中看到相应的处理。

在消息头的最后，是一个空行，表示消息头结束了。对GET和HEAD命令来说，消息头的结束，也就意味着整个client端的请求结束了。而对 POST 命令来说，可能后面还跟着其它数据。由于我们的 tracker服务器只接受 GET 和 HEAD 命令，所以在协议处理过程中，如果遇到空行，那么就表示处理结束。

HTTPConnection::data_came_in() 用一个循环来进行协议分析：

首先是寻找行结束符号：
i = self.buf.index('\n')
（我认为仅仅找 “\n”并不严谨，应该找 “\r\n”这个序列）。
如果没有找到，那么 index() 函数会抛出一个异常，而异常的处理是返回 True，表示数据不够，需要继续读数据。

如果找到了，那么 i  之前的字符串就是完整的一行。于是调用协议处理函数，代码是：

self.next_func = self.next_func(val)
在 HTTPConnection 的初始化的时候，有这么一行代码：
self.next_func = self.read_type
next_func 是用来保存协议处理函数的，所以，第一个被调用的协议处理函数就是 read_type()。它用来分析client端请求的第一行。在 read_type() 的最后，我们看到：
return self.read_header
这样，在下一次调用 next_func 的时候，就是调用 read_header()了，也就是对 HTTP 协议的消息头进行分析。
下面先看 read_type()，
它首先把 GET 命令中的 URL 部分保存到 self.path中，因为这是 client端最关键的信息，后面要用到。
然后检查一下是否是GET或者HEAD命令，如果不是，那么说明数据有错误。返回None，否则return self.read_header
接下来我们看read_header()，
这其中，最重要的就是对空行的处理，因为前面说了，空行表示协议分析结束。
在检查完 client 端是否支持 gzip 编码之后，调用：
r = self.handler.getfunc(self, self.path, self.headers)
通过一层层往后追查，发现 getfunc() 实际是 Tracker::get()，也就是说，真正对 client 端发来的请求进行分析，以及决定如何响应，是由 Tracker 来决定的。是的，这个 Tracker 在我们tracker 服务器源码分析系列的第一篇文章中就已经看到了。在创建 RawServer 之后，马上就创建了一个 Tracker 对象。所以，要了解 tracker 服务器到底是如何工作的，需要我们深入进去分析 Tracker 类，那就是我们下一篇文章的工作了。

在调用完 Tracker::get() 之后，返回的是决定响应给 client 端的数据，
if r is not None:
self.answer(r)
最后，调用 answer() 来把这些数据发送给 client 端。
对 answer() 的分析，我们在下一篇分析 Tracker类的文章中一并讲解。
l         connection_flushed()：
tracker服务器用的是非阻塞的网络 I/O ，所以不能保证在一次发送数据的操作中，把要发送的数据全部发送出去。

这个函数，检查在某个连接上需要发送的数据，是否已经全部被发送出去了，如果是的话，那么关闭这个连接的发送端。（为什么仅仅关闭发送端，而不是完全关闭这个连接了？疑惑）。

本篇文章分析 Tracker 类，它在 track.py 文件中。
在分析之前，我们把前几篇文章的内容再回顾一下，以理清思路。
BT的源码，主要可以分为两个部分，一部分用来实现 tracker 服务器，另一部分用来实现BT的客户端。我们这个系列的文章围绕 tracker 服务器的实现来展开。
BT 客户端与 tracker 服务器之间，通过 track  HTTP协议进行通信，而BT客户端之间以BT对等协议进行通信。
Tracker 服务器的职责是搜集客户端的信息，并帮助客户端相互发现对方，从而使得客户端之间能够相互建立连接，进而互相能下载所需的文件片断。
在实现 tracker 服务器的时候，首先是通过 RawServer 类来实现网络服务器的功能，然后由 HTTPHandler 类来完成对协议数据的第一层分析。因为 track  HTTP 协议是以HTTP协议的形式交互的，所以 HTTPHandler 按照HTTP的协议对客户端的请求进行第一层处理（也就是取得URL和HTTP 消息头），然后把URL和 HTTP消息头进一步交给 Tracker类来进行第二层分析，并把分析的结果按照 HTTP协议的格式封装以后，发给客户端。

Tracker 类对 track HTTP协议做第二层分析，它根据第一层分析后的URL以及HTTP消息头，进一步得到客户端的信息（包括客户端的ip地址、端口、已下载完的数据以及剩余数据等等），然后综合当前所有下载者的情况，生成一个列表，这个列表记录了下载同一个文件的其它下载者的信息（但不是所有的下载者，只是选择一部分），并把这个列表交给 HTTPHandler，由它进一步返回给客户端。

如此，整个 tracker 服务器的实现，在层次上就比较清晰了。
       为了分析 Tracker类，首先要理解“状态文件”。
l         状态文件：
在第一篇文章中，我们说到，要启动一个 tracker 服务器，至少要指定一个参数，就是状态文件。在 Tracker 的初始化函数中，主要就是读取指定的状态文件，并根据该文件做一些初始化的工作。所以必须弄清楚状态文件的作用：
\1.         状态文件的作用：
tracker 服务器如果因为某些意外而停止，那么所有的下载者不仅不能继续下载，而且先前所做的努力都前功尽弃。这种情况是不能容忍的，因此，必须保证在 tracker 重新启动之后，所有的下载者还能继续工作。Tracker 服务器周期性的将当前系统中必要的下载状态信息保存到状态文件中，在它因故停止，而后又重新启动的时候，可以根据这些信息重新恢复“现场”，从而使得下载者可以继续下载。
\2.         状态文件的格式：

状态文件的信息对应着一个比较复杂的4级嵌套的字典。

要详细分析这个字典类型，必须理解一点：一个 tracker 服务器，可以同时为下载不同文件的几批下载者提供服务。
我们知道，一批下载同一个文件的下载者，它们必然拥有同样的 torrent 文件，它们能根据 torrent 文件找到同一个 tracker 服务器。而下载另一个文件的一批下载者，必然拥有另外一个 torrent 文件，但是这两个不同的 torrent 文件，可能指向的是同一个 tracker 服务器。所以说“一个 tracker 服务器，可以同时为下载不同文件的几批下载者提供服务。”
实际上，那些专门提供 bt 下载的网站，都是架设了一些专门的 tracker 服务器，每个服务器可以同时为多个文件提供下载跟踪服务。
理解了这一点，我们继续分析状态文件的格式。
第一级字典：

在 Tracker 的初始化函数中，有这样的代码，
if exists(self.dfile):

h = open(self.dfile, 'rb')

ds = h.read()

h.close()

tempstate = bdecode(ds)

else:

tempstate = {}



​       这段代码是从从状态文件中读取信息，由于读到的是经过 Bencoding 编码后的数据，所以还需要经过解码，解码后就得到一个字典类型的数据，保存到 template 中，这就是第一级字典。它有两个关键字，peers 和 completed，分别用来记录参与下载的peer的信息和已经完成了下载的peer的信息（凡是出现在 completed的peer，也必然出现在 peers中）。这两个关键字对应的数据类型都是字典，我们重点分析 peers 关键字所对应的第二级字典。

第二级字典：

关键字：torrent文件中 info 部分的 SHA hash

数据：第三级字典
一个被下载的文件，唯一的被一个 torrent 文件标识，tracker通过计算torrent文件中 info 部分的 SHA hash，这是一个20字节的字符串，它可以唯一标识被下载文件的信息。第二级字典以此字符串作为关键字，保存下载此文件的下载者们的信息。

第三级字典：

关键字：下载者的 peer id

数据：第四级字典

解释：每个下载者，都创建一个唯一标识自己的20字节的字符串，称为 peer id。第三级字典以次为关键字，保存每个下载者的信息。

第四级字典：

关键字： ip、port、left等

数据：分别保存下载者的 ip地址、端口号和未下载完成的字节数

另外还有两个可选的关键字given ip 和nat，它们是用于 NAT 的，关于NAT的情况，后面会再提到。

理解了这个4级嵌套的字典，对 Tracker 的分析才好继续进行下去。

下面我们挨个看 Tracker 类的成员函数。
l         初始化函数 __init__()：
开始是一些参数的初始化，其中比较难理解的有：
self.response_size = config['response_size']
self.max_give = config['max_give']
要理解这两个参数，必须看那份更详细的BT协议规范中对“numwant”关键字的解释：
· numwant: Optional. Number of peers that the client would like to receive from the tracker. This value is permitted to be zero. If omitted, typically defaults to 50 peers.

If a client wants a large peer list in the response, then it should specify the numwanted parameter.
意思就是说，默认情况下，tracker 服务器给下载者响应的 peers 个数是 response_size 个，但有时候，下载者可能希望获得更多的 peers 信息，那么它必须在请求中包含 numwant 关键字，并指定希望获得 peers 的个数。例如是 300，tracker 取 300和 max_give中较小的一个，作为返回给下载者的 peers 的个数。
self.natcheck = config['nat_check']
self.only_local_override_ip = config['only_local_override_ip']
这两个参数是和 NAT 相关的，我们终于必须要说到 NAT 了。
我们知道，如果一个 BT 客户端处在局域网中，通过 NAT 之后连到 tracker 服务器的话，那么 tracker 服务器从连接中获得的该客户端的 IP 地址是一个公网IP，如果其它客户端通过这个 IP 试图连接该客户端的话，肯定会被 NAT 拒绝的。

通过一些 NAT 穿越的技术，在某些情况下，可以让一些客户端穿过 NAT，与处在局域网中的客户端建立连接，具体的技术资料我已经贴在论坛上了，大家有兴趣可以去看一看。原来我以为 BT 也用到了一些 NAT 穿越技术，但现在发现并没有，可能是技术实现上比较复杂，而且不能保证在任何情况下都有效的原因吧。

我们来看那份比较详细的协议规范中，对“ip”关键字的解释：
· ip: Optional. The true IP address of the client machine, in dotted quad format. Notes: In general this parameter is not necessary as the address of the client can be determined from the IP address from which the HTTP request came. The parameter is only needed in the case where the IP address that the request came in on is not the IP address of the client. This happens if the client is communicating to the tracker through a proxy (or a transparent web proxy/cache.) It also is necessary when both the client and the tracker are on the same local side of a NAT gateway. The reason for this is that otherwise the tracker would give out the internal (RFC191![img](http://bbs.chinaunix.net/static/image/smiley/default/icon_cool.gif) address of the client, which is not routeable. Therefore the client must explicitly state its (external, routeable) IP address to be given out to external peers. Various trackers treat this parameter differently. Some only honor it only if the IP address that the request came in on is in RFC1918 space. Others honor it unconditionally, while others ignore it completely.

在客户端发给 tracker 服务器的请求中，可能包含“ip”，也就是指定自己的 IP 地址。你可能有疑问了，客户端为什么要通知 tracker服务器自己的 ip 地址了？tracker 服务器完全可以从连接中获得这个 ip 啊。嗯，实际的网络情况是非常复杂的，如果客户端是在局域网内通过 NAT 后上网，或者客户端是通过某个代理服务器之后，再与 tracker 服务器建立连接，那么 tracker 从连接中获得的 ip 地址并不是客户端真实的 ip 地址，为了获得真实的ip，必须让客户端主动在协议中通知tracker。因此，就出现了两个 ip 地址，一个是从连接中获得的 ip 地址，我把它叫做“连接ip”，另一个是客户端通过请求传递过来的 ip，我叫它“真实ip”。显然，tracker 应该把客户端的“真实ip”记录下来，并把这个“真实ip”通知给其它下载者。

这个“ip”参数又是可选的，也就是说，如果客户端拥有一个公网的ip，而且并没有通过NAT或者代理，那么，它并不需要传递这个参数，“连接ip”就是“真实ip”。
按协议规发的说法，“ip”这个参数在以下两种情况下有用：

1、客户端可能拥有一个公网IP，但它又是通过一个代理服务器与tracker服务器建立连接的，它需要传递“ip”。

2、客户端在某个局域网中，恰好tracker也在同一个局域网中，。。。（这种情况又会怎么样了？我还没有弄明白 ：）

回过头来看 natcheck 和 only_local_override_ip，

natcheck ：how many times to check if a downloader is behind a NAT (0 = don't check)

only_local_override_ip：如果从 GET 参数中传递过来的 ip，是一个公网 ip，是否忽略它？它的默认值是 1。

现在还不好理解它的意思，我们看后面代码的时候，再来理解它。

self.becache1 = {}

self.becache2 = {}

self.cache1 = {}

self.cache2 = {}

self.times = {}


这里出现5个字典，其中times 用来，而其它4个字典的作用是什么？

嗯，还是让我们先来看看在“BT移植邮件列表”中，Bram Cohen 发的一个帖子，
There are two new GET parameters for the tracker in the latest release. They are –
key=xxxx - this is like peer id, but it's only known to the client and the tracker. It allows clients to be behind dynamic IP. If a peer announced a key previously, then it's accepted if and only if it gives the same key again. If no key was given, then the fallback is checking that the IP hasn't changed. If the IP has changed, mainline currently will give a peer list but not change any data related to that peer, so that peers behind dynamic IP using old clients will continue to work okay. Currently mainline generates the value associated with key as eight random hex values, and the tracker accepts any string from clients.
compact=1 - when a client sends this, the 'peers' return value is a single string whose length is a multiple of 6 rather than a dict. To extract peer information from the string, chop it into substrings of length 6. For each substring, the first four bytes are the IP and the last two are the port, encoded big-endian. This results in huge bandwidth savings.
Everybody developing ports should implement these keys, they're very useful.
-Bram

BT 在不停的向前发展，所以协议规范也在发展之中，新引入了两个关键字，其中一个是 compact，如果客户端请求中 compact=1，表示紧凑模式，也就是tracker给客户端响应的数据，采用一种比原来更紧凑的形式，这样可以有效的节约带宽。

Becache1 和 cache1用于普通模式，而 becache2和 cache2用于紧凑模式。我们马上能看到它们的初始化操作。
if exists(self.dfile):

h = open(self.dfile, 'rb')

ds = h.read()

h.close()

tempstate = bdecode(ds)

else:

tempstate = {}
if tempstate.has_key('peers'):

self.state = tempstate

else:

self.state = {}

self.state['peers'] = tempstate

self.downloads = self.state.setdefault('peers', {})

self.completed = self.state.setdefault('completed', {})

statefiletemplate(self.state)

这部分代码是读取状态文件，初始化 downloads和completed这两个字典，并检查读取的数据是否有效。

现在，downloads里面是保存了所有下载者的信息，而 completed保存了所有完成下载的下载者的信息。
for x, dl in self.downloads.items():

self.times[x] = {}

​        for y, dat in dl.items():

​               self.times[x][y] = 0

​            if not dat.get('nat',1):

​                   ip = dat['ip']

​                gip = dat.get('given ip')

​                if gip and is_valid_ipv4(gip) and (not self.only_local_override_ip or is_local_ip(ip)):

​                       ip = gip

self.becache1.setdefault(x,{})[y] = Bencached(bencode({'ip': ip, 'port': dat['port'], 'peer id': y}))

​                self.becache2.setdefault(x,{})[y] = compact_peer_info(ip, dat['port'])

这里，对 times、becache1、becache2初始化。它们都是2级嵌套的字典，第一级的关键字是 torrent 文件中的 info 部分的 hash，第二级关键字是下载者的 peer id，becache1保存的是一个 Bencached 对象，而 becache2 保存的是一个字符串，它是把 ip和port 组合成的一个字符串。

参数设置完之后，有：

rawserver.add_task(self.save_dfile, self.save_dfile_interval)

add_task() 我们已经见到过好多次了，这表示每隔一段时间，需要调用 save_dfile() 来保存状态文件。

再后面的代码，我没有仔细看了，象 allow_get 和 allowed_dir 等的意义，还需要看相关的代码才能明白，如果你仔细看了这些部分，希望能补充一下。

初始化以后，就是 Tracker 的最重要，也是代码最长的函数： get() 。

l         get()：

在第三篇文章中，我们已经看到，在由 HTTPHandler 对 track HTTP协议进行第一层分析之后，就是调用 Tracker::get() 来进行第二层分析的。它的参数是 URL 和 HTTP 消息头。
在这个函数中，首先调用 urlparse() 对 URL 进行解析，例如这样的 URL ：
/announce?ip=192.168.112.1&port=9999&left=2000
解析之后，就获得了 path，是announce，还有参数，包括：
ip：192.168.112.1
port：9999

left：2000



然后，根据 path 的不同，分别处理。
一般来说，客户端发给 tracker 的请求中，path 都是 announce，但有时候，第三方可能也想查询一下 tracker 服务器的状态，那么它可以通过其它的 path 来向 tracker 服务器请求，例如 scrape。在一些专门提供 bt 下载的网站上，我们可以看到不停更新的下载者、种子个数等信息，就是用这种方式从 tracker 服务器处获得的。

我们只看 path 是 announce 的情况。

首先是对客户端传递来的参数的有效性进行检查，包括是不是有 info_hash 关键字？ip地址是否合法等等。

然后，
ip = connection.get_ip()
这样得到的 ip ，是根据客户端与 tracker 服务器建立的连接中获取的 ip，就是“连接ip”了。

接下来，

ip_override = 0

if params.has_key('ip') and is_valid_ipv4(params['ip']) and (not self.only_local_override_ip or is_local_ip(ip)):

ip_override = 1
这段代码的意图，是为了判断在随后保存客户端的 ip 地址的时候，是否要用“真实ip”来取代“连接ip”。如果 ip_override 为1，那么就保存“真实ip”，也就是“连接ip”被“真实ip”覆盖（override）了。

分析源码的过程其实就是揣测作者意图的过程，我的揣测是这样的：

如果客户端从请求中传递了“真实ip”，那么对 tracker来说，，既然客户端都已经报告了“真实ip”了，那么当然就保存“真实ip”就好了。可如果“真实 ip ”是个公网 ip，而且only_local_override_ip=1，也就是说，忽略“真实ip”为公网ip的情况，那么，保存的是“连接”ip。

说句实话，为什么要设置 only_local_override_ip 这么一个参数，我还是没有弄明白。
if peers.has_key(myid):

myinfo = peers[myid]

if myinfo.has_key('key'):

if params.get('key') != myinfo['key']:

​                  return (200, 'OK', {'Content-Type': 'text/plain', 'Pragma': 'no-cache'},

​                        bencode({'failure reason': 'key did not match key supplied earlier'}))

​        confirm = 1

elif myinfo['ip'] == ip:

​           confirm = 1

else:

​       confirm = 1



这段代码涉及到身份验证吧，我没有仔细看了，关于 “key”的解释，请看上面Bram Cohen的帖子。
接下来，如果验证通过，而且事件不是“stopped”，那么就把客户端的信息保存下来。如果已经存在该客户端的信息，那么就更新一下。注意这里 ip_override 派上了用场，也就是如果覆盖，那么保存的是“真实ip”，否则保存的是“连接ip”。

if port == 0:

peers[myid]['nat'] = 2**30

elif self.natcheck and not ip_override:

to_nat = peers[myid].get('nat', -1)

if to_nat and to_nat < self.natcheck:

NatCheck(self.connectback_result, infohash, myid, ip, port, self.rawserver)

​       else:

peers[myid]['nat'] = 0



​       第一个 port == 0 的情况，不知道是什么意思？

第二个表示要检查 NAT的情况。大概意思就是 tracker服务器主动用 BT对等协议与该客户端进行握手，如果握手成功，那么说明该客户端是可以被直接连接的。这一点很重要，如果 tracker 服务器无法和客户端直接建立连接的话，那么其它下载者也无法和该客户端建立连接。

这里用到的 NatChecker 类，也是一个 Handler 类，具体细节，大家自己分析吧。

data = {'interval': self.reannounce_interval} 
从这到最后，就是根据紧凑模式和普通模式两种不同情况，分别从 becache1或者 becache2中，返回随机的 peers 的信息。
在这里，我们来总结一下 cache1、becache1、cache2、becache2的用处。我感觉 cache1和 cache2 好像没什么作用，因为从代码中没有看到它们两的意义。Becache1和 becache2则分别用于普通模式和紧凑模式情况下，对 peers 的信息进行缓存。它们从状态文件中初始化自己；如果有新的 peer 出现，被添加到这两个缓存中；如果是“stopped”事件，那么从缓存中删除对应的 peer。最后，tracker 根据情况，从其中一个缓存取得随机的 peers 的信息，返回给客户端。
l         connectback_result()
这个函数，用于 NatCheck 类作为回调函数。它根据 tracker 服务器主动与客户端建立连接的结果做一些处理。其中的参数 result，是表示tracker 与客户端建立连接是否成功。如果建立成功，显然对方不在 NAT 后面，否则就是在 NAT 后面了。record['nat'] += 1 这没看懂，为什么不是直接 record['nat'] = 1 ？最后，如果建立连接成功，那么更新一下 becache1 和 becache2。

摘自：http://bbs.chinaunix.net/thread-556756-1-1.html

