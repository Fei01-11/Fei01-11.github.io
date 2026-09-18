## 概述
**该项目采用的模型为主从reactor多线程模型** 
`主从reactor多线程模型:`
[五分钟快速理解 Reactor 模型-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1811347)
![image.png](https://xxxcac-pic.oss-cn-hangzhou.aliyuncs.com/%E4%B8%BB%E4%BB%8Ereactor.png)

本项目的大致结构图：
主线程维护一个反应堆，反应堆里面有任务队列。任务队列的个数=反应堆实例个数=线程个数（主线程+子线程个数）
![image.png](https://xxxcac-pic.oss-cn-hangzhou.aliyuncs.com/webserver%E6%A1%86%E6%9E%B6%E5%9B%BE.png)

（1）主从reactor:主线程负责监听和连接，把通信的文件描述符交给线程池。在线程池里存在多个子线程，每个子线程也有反应堆模型，反应堆模型也是poll/epoll/select三选一。主线程将通信描述符封装成channel后，随机放入一个子线程的任务队列中，子线程会遍历任务队列，将任务注册到反应堆模型，反应堆的select/poll/epoll检测到读写事件时返回fd，用fd找到channel后根据channel调用对应的读写回调。
`程序还考虑了一个极端情况：如果线程池中没有子线程的话，主线程会自己监听和处理，结构退化成单reactor模型`

（3）执行读任务函数主要是完成一个非阻塞读取调用直到读完，将数据缓存在用户缓冲区中，接着执行一个消息解析的操作，根据HTTP解析是否成功的判断来决定重新注册写事件还是读事件。如果解析失败那么重新注册读事件等待下次读取更多数据直到一个完整的HTTP请求。如果是解析成功的话就制作响应报文并且注册写事件，等待内核缓冲区可写触发事件时，将其写入内核缓冲区。

（4）主线程持续监听，当有新连接来时会重复上面的步骤。

**注意主线程和子线程相关定义和交互**
主线程就是负责监听，子线程就是工作线程。所谓反应堆，就是select/poll/epoll三者组成的dispatcher。主线程和子线程都有各自的反应堆。每个子线程都有一个任务队列，由主线程和当前子线程共同操作，主线程主要是添加，子线程主要是删改。主线程将封装好的channel随机添加入任何一个子线程的任务队列中，子线程将任务队列中的任务注册入自己的反应堆中，由反应堆来检测读写事件并进行相关的读写操作。

## 主要的类
   Channel:将文件描述符、对应的事件、回调函数进行封装
   Dispatcher：基类，声明了一些虚函数
   EventLoop：
	   1. 通过枚举(ElemType)的方式定义处理channel的方式：ADD/DELETE/MODIFY
	   2. 定义任务队列(ChannelElement)的节点：处理channel的方式type、channel*  

## 反应堆实例
==channel、Dispatcher(Poll,Epoll,Select)、EventLoop==
### Channel
**Channel类** :将文件描述符、对应的事件、回调函数进行封装
    1、通过枚举(FDEvent)定义文件描述符的读写事件 (ReadEvent、WriteEvent)
    2、channel类：
        1）通过可调用对象包装器打包：读回调、写回调、销毁回调
        2）对于写事件->判断是否需要检测写事件、是否要修改写事件

### Dispatcher
**Dispatcher类**：基类，声明了一些虚函数
    1、定义了一些列的虚函数：事件添加add、删除remove、修改modify、检测dispatch
    2、定义了设置channel的函数
```c++
add()
remove()
modify()
dispatcher(channel*)
```

### EventLoop
**三大组件：任务队列、channelMap、dispatcher**
**EventLoop：**
    ==1、通过枚举(ElemType)的方式定义处理channel的方式：ADD/DELETE/MODIFY==
    ==2、定义任务队列(ChannelElement)的节点：处理channel的方式type、channel==
    ==3、EventLoop类中的操作:==
```cpp
//启动反应堆
run()
//处理激活的文件描述符
eventActive(int fd, int event)
//添加任务到任务队列
addTask(channel*, ElemType)
//处理dispatcher中的节点
add/remeove/modify(Channel*)
//释放channel
freechannel(channel*)
//读取数据
readMessage()
```

#### EventLoop中操作的相关细节
**任务队列**：
queue<ChannelElement*>，在执行过程会对文件描述符的事件进行，增加、删除、修改，将这些需求记录到任务队列中。遍历任务队列，进行事件的处理

**channelMap**：
map<int,channel*>，基于文件描述符找到channel

**互斥锁**：只允许一个线程对任务队列进行操作

**socketPair**:
```
基本用法：
1. 这对套接字可以用于全双工通信，每一个套接字既可以读也可以写。例如，可以往sv[0]中写，从sv[1]中读；或者从sv[1]中写，从sv[0]中读；
2. 如果往一个套接字(如sv[0])中写入后，再从该套接字读时会阻塞，只能在另一个套接字中(sv[1])上读成功。
3. 项目中用于激活epoll/select/poll

拓展：
读、写操作可以位于同一个进程，也可以分别位于不同的进程，如父子进程。如果是父子进程时，一般会功能分离，一个进程用来读，一个用来写。因为文件描述符sv[0]和sv[1]是进程共享的，所以读的进程要关闭写描述符, 反之，写的进程关闭读描述符。
```

*上述细节的应用：*
1. 在EventLoop初始化时，通过指向子类实例的指针(m_dispatcher)，选择合适的IO复用方式。
2. 构造函数中：对socketpair进行初始化，指定0：发送数据，1：接收数据。创建channel类的channel实例，传进文件描述符1，读事件，读回调函数(readMessage)。在这一步通过bind绑定readMessage。之后通过addTask()将channel实例加入任务队列。
3. 在添加任务节点到任务队列的时候需要加锁，保护共享资源，多个线程都可以访问这个任务队列，之后就是处理该节点。
```
处理细节：
1、对于节点的添加：可能是当前线程也可能是其他线程
   1)修改fd 的事件，当前子线程发起，当前子线程处理
   2)添加新的fd，添加任务节点的操作是由主线程发起。
2、不能让主线程处理任务队列，需要由当前的子线程去处理（实现的方法是通过比较线程ID）。
```

`子线程处理(processTask):`
	从任务队列中取出节点(ChannelElement)，每个节点中都存储着channel,以及其对应的处理方式，包括添加，删除，修改    ```
```
添加、删除、修改操作，涉及到对channelMap的操作
这三个函数传入的是一个channel实例，通过该实例得到文件描述符fd，通过该文件描述符在map中find,查看该channel是否在map中。
1）添加：若没有，添加到channleMap中。然后调用dispatcher实例的setChannel方法，设置channel,然后调用add方法,将其添加到所选择的IO复用方式中
2）删除：若有则删除，想调用dispatcher实例的set方法设置channel，在调用remove方法
3）修改：若有则修改，想调用dispatcher实例的set方法设置channel，在调用modify方法
```

**channelMap**:
使用map进行存储：fd, channel*。在dispatcher中存在一个设置channel的方法setChannel。该方法可以被epoll类继承，通过channel取出fd。channelMap的作用是为了将channel和fd做映射，channel里面含有对fd操作的函数。

`具体处理:`
假如接收到了一个fd，需要基于这个文件描述符进行事件的处理，就要找到fd对应的channel,channel内部有文件描述符和事件处理的回调函数。
```c
struct epoll_event ev;
    ev.data.fd = m_channel->getSocket();
    int events = 0;
    if (m_channel->getEvent() & (int)FDEvent::ReadEvent)
    {
        events |= EPOLLIN;
    }
    if (m_channel->getEvent() & (int)FDEvent::WriteEvent)
    {
        events |= EPOLLOUT;
    }
    ev.events = events;
    int ret = epoll_ctl(m_epfd, op, m_channel->getSocket(), &ev);
```
`这些方法都是虚函数，通过比如epoll类继承dispatcher类，在epoll类中重写了这些方法。`

**主线程(taskWakeup)**：
告诉子线程处理任务队列中的任务，这时候子线程在干啥：(1)在工作，(2)子线程阻塞
主要是通过 socketpair()函数创建一对无名的、相互连接的套接字。该套接字在反应堆初始化时已经加入到了任务队列中，由子线程将其添加到了epoll检测的事件中。主线程通过往套接字中写入一些数据，触发读事件，解除子线程的阻塞。
1. **run()** 循环进行事件的处理，由子线程处理。
	通过调用dispatcher实例的dispatcher方法,  检测事件，由读写事件发生时，调用eventActive()激活反应堆。     
2. **eventActive()**
	通过传入的第一个参数fd，到channelMap中寻找channel实例。传入的第二个参数是event。这个函数是在epoll中dispatcher方法中调用，当事件发生时，epoll_wait解除阻塞，通过与EPOLLIN,EPOLLOUT进行比较，调用不同的eventActive(),传进来的参数是fd,event。eventActive根据传进来的event调用不同的回调函数，该回调函数在channel中注册。

==4、EpollDispatcher==
该类继承自dispatcher类，重写了add/remove/modify/dispatcher虚函数
```c
m_epfd = epoll_create(10);
// 参数已经弃用，但是必须大于零。返回引用新epoll实例的文件描述符。该文件描述符用于随后的所有对epoll的调用接口。每创建一个epoll句柄，会占用一个fd，因此当不再需要时，应使用close关闭epoll_create（）返回的文件描述符，否则可能导致fd被耗尽。当所有文件描述符引用已关闭的epoll实例，内核将销毁该实例并释放关联的资源以供重用。
m_events = new struct epoll_event[m_maxNode];
```
前三个方法，调用epoll_ctl()，采用按位与的方式。

**dispatcher方法**：
  1)调用epoll_wait:检测事件的发生，阻塞2秒
```c
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

/*作用： 等待监听的所有fd相应事件的产生。
1.2、参数详解：
1) int epfd： epoll_create()函数返回的epoll实例的句柄。
2) struct epoll_event * events： 接口的返回参数，epoll把发生的事件的集合从内核复制到 events数组中。events数组是一个用户分配好大小的数组，数组长度大于等于maxevents。（events不可以是空指针，内核只负责把数据复制到这个 events数组中，不会去帮助我们在用户态中分配内存）
3) int maxevents： 表示本次可以返回的最大事件数目，通常maxevents参数与预分配的events数组的大小是相等的。
4) int timeout： 表示在没有检测到事件发生时最多等待的时间，超时时间(>=0)，单位是毫秒ms，-1表示阻塞，0表示不阻塞。*/
```

2)细节：注意EPOLLERR和EPOLLHUP
`EPOLLERR：发生错误
`EPOLLHUP：连接关闭
   对方已经关闭了连接，需要将fd 删除。
3)对于EPOLLIN EPOLLOUT事件
        调用eventActive(int fd, int event）:event:readEvent/writeEvent激活反应堆。
    对于反应堆模型，每个线程各有一个，子线程要执行的任务都在其反应堆模型的任务队列中
    
==5、工作线程==
    线程结构体中：线程实例，互斥锁、条件变量、反应堆模型
    作用：主线程，创建子线程，并为其指定回调函数，在回调函数中创建反应堆模型（加锁，条件变量），创建完成后用条件变量唤醒阻塞的主线程，再通过eventLoop的run方法，启动反应堆。在创建子线程时需要阻塞主线程。让当前的函数不会直接结束：方法是通过条件变量。  目的：保证反应堆模型能够创建完成，主线程执行创建子线程的函数，而回调函数(反应堆创建）由子线程去完成，在这里evLoop就是一个共享资源，需要加互斥锁。

==6、线程池==
1. 构造函数中设置线程的数量，以及主线程。
	 **目的**：如果线程池中线程数量为零，线程池就不能工作，这时候可以将所有的任务放到主线程的反应堆模型中，这时候就是单反应堆模型；如果线程池中线程的数量是大于0的，就是多反应堆模型 。
2. 由主线程创建子线程，创建的子线程调用run方法创建属于子线程的反应堆模型，然后将子线程存入vector中。
3. 主线程调用用于从线程池中取出一个子线程的方法。
	通过线程的数量进行判断，如果线程的数量为0，取出的就是主线程，否则就是子线程。然后，从子线程中取出反应堆模型。

==6、Buffer==
1. 存在两个计数：读数据位置、写数据位置
	 扩容：存在几种情况：
                case1:写数据内存充足(capacity - writepos) 不需要扩容
                case2:写数据内存不充足(capacity - writepos)但是readable(已读内存+可写内存)充足，需要合并内存memcpy(data,data+readpos, readable)
                case3:扩容
2. struct iovec vec\[2]
	struct iovec是一个用于存放指向数据缓冲区的指针和长度信息的结构体.在Linux系统中，readv() 和 writev() 系统调用就使用了struct iovec结构体，它可以使得在一次系统调用中读取或写入多个非连续的缓冲区，从而减少了系统调用的次数，提高了性能。
	 struct iovec定义了一个向量元素。通常，这个结构用作一个多元素的数组。对于每一个传输的元素，指针成员iov_base指向一个缓冲区，这个缓冲区是存放的是readv所接收的数据或是writev将要发送的数据。成员iov_len在各种情况下分别确定了接收的最大长度以及实际写入的长度。
3. 处理读入的数据
	Get / HTTP/1.0:
          请求行以\\r\\n结尾：使用memmem,类似于strstr
4. 发送数据
	 linux下当连接断开，还发数据的时候，不仅send()的返回值会有反映，而且还会向系统发送一个异常消息，如果不作处理，系统会出BrokePipe，程序会退出，这对于服务器提供稳定的服务将造成巨大的灾难。为此，send()函数的最后一个参数可以设MSG_NOSIGNAL，禁止send()函数向系统发送异常消息。为了防止粘包进行了延迟处理；
```cpp
int Buffer::sendData(int socket)
{
    // 判断有无数据
    int readable = readableSize();
    if (readable > 0)
    {
        int count = send(socket, m_data + m_readPos, readable, MSG_NOSIGNAL);
        if (count > 0)
        {
            m_readPos += count;
            usleep(1);
        }
        return count;
    }
    return 0;
}
```
##  疑问
### 1. 套接字和谁一起封装的？封装的意义是什么？
套接字主要是操作函数、事件类型（读/写）和数据一起进行封装。channel是为了实现任务队列。任务队列中的任务就是指channel。

### 2.任务队列中的事件如何算已就绪？事件循环的意义？
所谓的就绪就是指任务是否能接收处理（事件是否已经在任务队列中），判断的依据是事件循环是否属于当前线程。主线程并不负责事件的处理，事件的处理是由子线程来解决的（主从reactor模型的特点），如果当前pid还是主线程，说明还没有准备就绪。事件循环是为了及时更新已就绪事件的需求（读/写）。


### 3.http报文处理部分是怎么和事件的读写联系上的？
http报文的处理主要是依赖于channel事件中的handleFunc。
```cpp
//channel的构造函数，注意http报文处理部分之所以能被调用是因为handleFunc readFunc
//和handleFunc writeFunc
Channel(int fd, FDEvent events, handleFunc readFunc, handleFunc writeFunc, handleFunc destroyFunc, void* arg)
```

### 4.主从reactor的好处
项目高并发的优化可以从线程池和网络设计模式这两部分入手。
首先是单Reactor模型，可以采用Proactor模型或者多Reactor模型进行改进。Proactor模型对异步IO有要求，而在Linux上没有相关的异步IO系统调用，一般都是采用同步模拟（由主线程完成IO操作）去实现，相比之下强行模拟的效果不如采用单Reactor（这个时候主线程承担了太多工作）。
主从多Reactor模型的主线程只负责新连接到达的监听以及新连接的建立，对于新到达的连接通过生产者消费者模型分发给子Reactor（另起线程），由子Reactor完成已建立连接的读写事件监听任务。这样当有瞬间的高并发连接时，也不会出现新连接丢失的情况。

### 5.线程开多少比较合适？为什么？
本项目是开了四个线程，应该是考虑到了2核的原因。
线程池的线程数量在构造的时候就已经确定下来，当机器核心数量改变时需要通过修改才能改变线程数量去匹配，不符合开闭原则，线程数量这一块可以通过C++17新特性动态获得机器的核心数量`std::thread::hardware_concurrency()`，来使程序更好的匹配机器实现更好的性能【避免核心数量过少进行频繁的上下文切换以及核心数量过多被闲置】。或者通过普通线程（临时创建用来响应多余的任务）和核心线程（一直存在）来优化【实现较为复杂】

### 6.http报文的处理
http请求报文是由三部分组成: `请求行, 请求头和请求体
**请求行**：    
     **方法**：如 GET、POST、PUT、DELETE等，指定要执行的操作。
     **请求 URl**（统一资源标识符）：请求的资源路径，通常包括主机名、端口号（如果非默认）、路径和查询字符串。
    **HTTP 版本**：如 HTTP/1.1 或 HTTP/2。    
    请求行的格式示例：==GET /index.html HTTP/1.1==
    
**请求头**：    
    包含了客户端环境信息、请求体的大小（如果有）、客户端支持的压缩类型等。
    常见的请求头包括`Host`、`User-Agent`、`Accept`、`Accept-Encoding`、`Content-Length`等。
**空行**：    
    请求头和请求体之间的分隔符，表示请求头的结束。
**请求体**（可选）：    
    在某些类型的HTTP请求（如 POST 和 PUT）中，请求体包含要发送给服务器的数据。
`请求实例`
```java
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Encoding: gzip, deflate
Connection: keep-alive
```


**状态行**（Status Line）：
    **HTTP 版本**：与请求消息中的版本相匹配。
    **状态码**：三位数，表示请求的处理结果，如 200 表示成功，404 表示未找到资源。
    **状态信息**：状态码的简短描述。    
    状态行的格式示例：==HTTP/1.1 200 OK==
    
**响应头**（Response Headers）：    
    包含了服务器环境信息、响应体的大小、服务器支持的压缩类型等。
    常见的响应头包括`Content-Type`、`Content-Length`、`Server`、`Set-Cookie`等。
**空行**：    
    响应头和响应体之间的分隔符，表示响应头的结束。
**响应体**（可选）：
    包含服务器返回的数据，如请求的网页内容、图片、JSON数据等。
`响应实例`
```java
HTTP/1.1 200 OK
Date: Wed, 18 Apr 2024 12:00:00 GMT
Server: Apache/2.4.1 (Unix)
Last-Modified: Wed, 18 Apr 2024 11:00:00 GMT
Content-Length: 12345
Content-Type: text/html; charset=UTF-8

<!DOCTYPE html>
<html>
<head>
    <title>Example Page</title>
</head>
<body>
    <h1>Hello, World!</h1>
    <!-- The rest of the HTML content -->
</body>
</html>
```
`状态码说明`
```
1xx
(Informational) 信息性状态码，表示正在处理。

2xx
(Success) 成功状态码，表示请求正常。200：请求被成功处理。
204：该状态码表示服务器接收到的请求已经处理完毕，但是服务器不需要返回响应体。
206： 该状态码表示客户端进行了范围请求，而服务器成功执行了这部分的GET请求。

3xx
(Redirection) 重定向状态码，表示客户端需要进行附加操作.
301：永久性重定向。
302 ：临时性重定向。  

4xx
(Client Error) 客户端错误状态码，表示服务器无法处理请求。
400：指出客户端请求中的语法错误。
401：该状态码表示发送的请求需要有认证。
403：该状态码表明对请求资源的访问被服务器拒绝了。
404 ：该状态码表明服务器上无法找到指定的资源。
 
5xx
(Server Error) 服务器错误状态码，表示服务器处理请求出错。
500：该状态码表明服务器端在执行请求时发生了错误。
502 ：该状态码表明服务器网关错误。
503 ： 该状态码表明服务器暂时处于超负载或正在进行停机维护，现在无法处理请求。
```

每个部分之间都通过特殊界限符划分。在我们获得一个数据包（以\r\n结尾的数据包）的时候可以根据 **状态机** 的状态变量判断如何处理当前的数据包，并且在执行完相应操作后设置状态变量进行状态转移完成整个报文的解析工作。
状态机的三个状态：解析请求头->解析请求行->解析请求体

### 7.buffer动态扩容（集中读，分散写）
此处涉及到 **消费者（读数据）和生产者（写数据）问题**
buffer是指读取数据的缓冲区。
buffer包括m_data(指向缓冲区的指针)、readPos、writePos、capacity（缓冲区容量大小）四个参数。
![image.png](https://xxxcac-pic.oss-cn-hangzhou.aliyuncs.com/buffer.png)
主要存在下面四种状态：
1. 未写入任何数据（内存都可用）。此时 **readPos = writePos = 0** 。
![image.png](https://xxxcac-pic.oss-cn-hangzhou.aliyuncs.com/buffer_nodata.png)

2. 写入部分数据但写入数据未读。此时 **readPos=0,0<writePos<capacity**
![image.png](https://xxxcac-pic.oss-cn-hangzhou.aliyuncs.com/buffer_dataRead.png)

3. 写入部分数据但写入数据只读部分。此时 **0<readPos<writePos<capacity**
![image.png](https://xxxcac-pic.oss-cn-hangzhou.aliyuncs.com/buffer_data_noread.png)
此状态需要调整,将写入未读的数据覆盖在内存数据已读的位置上，然后调整readPos和writePos的位置。
![image.png](https://xxxcac-pic.oss-cn-hangzhou.aliyuncs.com/20240720204709.png)

4. 未写入任何数据但内存都是未读数据。**readPos=0,writePos=capacity**
![image.png](https://xxxcac-pic.oss-cn-hangzhou.aliyuncs.com/20240720204852.png)

### 8.LFU实现定时器
改进，用于关闭不活跃连接

### 9.主线程是怎么分配任务给子线程的？
通过下面程序随机抽取一个子线程。
```cpp
EventLoop* ThreadPool::takeWorkerEventLoop()
{
    assert(m_isStart);
    if (m_mainLoop->getThreadID() != this_thread::get_id())
    {
        exit(0);
    }
    // 从线程池中随机找一个子线程, 然后取出里边的反应堆实例
    EventLoop* evLoop = m_mainLoop;
    if (m_threadNum > 0)
    {
        evLoop = m_workerThreads[m_index]->getEventLoop();
        m_index = ++m_index % m_threadNum;
    }
    return evLoop;
}
```

将新连接的fd放入取出的子线程的eventloop中
```cpp
int TcpServer::acceptConnection(void* arg)
{
    TcpServer* server = static_cast<TcpServer*>(arg);
    // 和客户端建立连接
    int cfd = accept(server->m_lfd, NULL, NULL);
    // 从线程池中取出一个子线程的反应堆实例, 去处理这个cfd
    EventLoop* evLoop = server->m_threadPool->takeWorkerEventLoop();
    // 将cfd放到 TcpConnection中处理
    new TcpConnection(cfd, evLoop);
    return 0;
}

TcpConnection::TcpConnection(int fd, EventLoop* evloop)
{
    m_evLoop = evloop;
    m_readBuf = new Buffer(10240);
    m_writeBuf = new Buffer(10240);
    // http
    m_request = new HttpRequest;
    m_response = new HttpResponse;
    m_name = "Connection-" + to_string(fd);
    m_channel = new Channel(fd, FDEvent::ReadEvent, processRead, processWrite, destroy, this);
    evloop->addTask(m_channel, ElemType::ADD);
}
```

## 难点及如何解决
### 难点1与解决
难点：整个项目的规划以及框架的设计，以及围绕高并发进行的一系列优化。
通过采用从局部到整体的设计思想。先使用单一线程完成串行的HTTP连接建立、HTTP消息处理和HTTP应答发送，然后围绕高并发这个核心扩展多个模块。首先就是缓冲区模块的一个设计，这里优先实现是为了下面各个模块的调试方便，记录各个模块运行的状况和打印输出模块运作情况来排除明显的BUG。然后是引入IO多路复用实现单线程下也能在一次系统调用中同时监听多个文件描述符，再进一步搭配线程池实现多客户多任务并行处理，这是高并发的核心部分。

### 难点2与解决
难点：该项目最先是使用单reactor多线程模型操作的，主线程里使用一个复用IO承担了所有事件的监听，发现访问量上来后有些新连接会丢失。
解决：对设计模式进行优化，采用主从reactor模型，mainReactor只负责新连接到达的监听以及新连接的建立，subReactor模型处理新到达的连接，由主Reactor完成已建立连接的读写事件监听任务，子reactor处理读写事件。这样当有瞬间的高并发连接时，也不会出现新连接丢失的情况。

### 难点3与解决
难点：存在部分连接不活跃的情况，导致资源浪费。
解决：在应用层实现了心跳机制，通过定时器（LFU）实现非活跃连接的一个检测和中断，减少系统资源（内存）不必要的浪费。
该机制应用在子线程中。子线程对遍任务队列时会对定时器链表进行管理。遍历完任务队列后对定时链表进行遍历，如果存在超时节点则将该fd打上“delete"，删除超时节点，将其加入任务队列中。

参考：[C++高并发异步定时器的实现 - 个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000044377770)
[C++手动实现定时器_c++ 定时器-CSDN博客](https://blog.csdn.net/haokan123456789/article/details/136986153?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-136986153-blog-134049574.235^v43^pc_blog_bottom_relevance_base2&spm=1001.2101.3001.4242.1&utm_relevant_index=3)
1. 定时器类：
```c++
class util_timer {
public:
    util_timer() : prev(NULL), next(NULL){}
public:
   time_t expire;   // 任务超时时间，这里使用绝对时间
   void (*cb_func)( client_data* ); // 任务回调函数，回调函数处理的客户数据，由定时器的执行者传递给回调函数
   client_data* user_data;
   util_timer* prev;    // 指向前一个定时器
   util_timer* next;    // 指向后一个定时器
};
```

2. 定时器链表
    定时器链表是一个升序、双向链表，带有头结点，尾节点指针
    1）将目标定时器添加到链表add_timer：如果目标定时器的超时时间小于当前链表中所有定时器的超时时间，则把该定时器插入链表头部,作为链表新的头节点，否则就需要调用重载函数 add_timer(),把它插入链表中合适的位置，以保证链表的升序特性。
    2)调整函数adjust_timer：当某个定时任务发生变化时，调整对应的定时器在链表中的位置。这个函数只考虑被调整的定时器的超时时间延长的情况，即该定时器需要往链表的尾部移动。 如果被调整的目标定时器处在链表的尾部，或者该定时器新的超时时间值仍然小于其下一个定时器的超时时间则不用调整。如果目标定时器不是链表的头节点，则将该定时器从链表中取出，然后插入其原来所在位置后的部分链表中
     3）删除del_timer：只有一个定时器，直接删除。如果有至少两个，且头结点为目标删除节点，将下一个节点重置为头结点。若尾部为目标节点，则将前一个节点重置为尾节点。若在中间，删除后调整
    4）tick()：SIGALARM信号每次被触发就在其信号处理函数中执行一次tick()函数，以处理链表上的到期任务。首先获得当前系统时间，从链表头结点开始依次处理每个定时器，直到一个尚未处理的定时器。每个定时器都是用绝对时间作为超时值，所以可以把定时器的超时值和系统当前时间比较以判断定时器是否到期。没到期，调用定时器的处理函数，处理完将该节点删除。
3. 用户数据结构类：记录fd, 定时器类，读缓存，客户端地址
4. 过程：
	任务超时时间，这里使用绝对时间：当前系统时间 + 3 * 5。
	当有数据到来时，通过epoll_wait检测，若是监听描述符，则accept创建通信描述符，并创建定时器，设置其回调函数，绑定定时器和用户数据（涉及用户数据类、定时器类），最后将定时器添加到链表（定时器链表）。
	创建一个sockpair(fd\[0]接收，fd\[1]发送) 属性都设置为非阻塞， 将 fd\[0] 添加到epoll检测列表，事件为EPOLLIN || EPOLLET。
	使用的信号为SIGALRM和SIGTERM，SIGTERM是kill或killall命令发送到进程的默认信号。它会导致进程终止，但与SIGKILL信号不同，进程可以捕获并解释（或忽略）它。因此，SIGTERM类似于要求进程很好地终止，允许清理和关闭文件。这两个信号都为其指定了回调函数，信号设置了SA_RESTART标记，那么当执行某个阻塞系统调用时，收到该信号时，进程不会返回，而是重新执行该系统调用。
```c++
void addsig( int sig )
{
    struct sigaction sa;
    memset( &sa, '\0', sizeof( sa ) );
    sa.sa_handler = sig_handler;
    sa.sa_flags |= SA_RESTART;
    sigfillset( &sa.sa_mask );
    assert( sigaction( sig, &sa, NULL ) != -1 );
}
void sig_handler( int sig )
{
    int save_errno = errno;
    int msg = sig;
    send( pipefd[1], ( char* )&msg, 1, 0 );
    errno = save_errno;
}
```

`过程细节`
首先：定时，5秒产生一个SIGALRM信号--》alarm(5)，调用回调函数 sig_handler，向fd\[1]发送数据(将信号作为数据传输），epoll_wait检测到数据到来，通过判断fd是否为fd\[0]；若是则接收信息，接收到的信息。对接受信息进行判断。
如果是SIGALRM,则将标志timeout设为true,在最后处理定时事件，因为I/O事件有更高的优先级。当然，这样做将导致定时任务不能精准的按照预定的时间执行。定时处理任务，实际上就是调用tick()函数,因为一次 alarm 调用只会引起一次SIGALARM 信号，所以我们要重新定时，以不断触发SIGALARM信号。
```c++
for( int i = 0; i < ret; ++i ) {
   switch( signals[i] )  {
       case SIGALRM:
	   {
     // 用timeout变量标记有定时任务需要处理，但不立即处理定时任务
     // 这是因为定时任务的优先级不是很高，我们优先处理其他更重要的任务。
	       timeout = true;
	       break;
	    }
        case SIGTERM:
	    {
	        stop_server = true;
	    }
	}
}
void timer_handler()
{
    // 定时处理任务，实际上就是调用tick()函数
    timer_lst.tick();//会调用定时器的回调函数cb_func
    // 因为一次 alarm 调用只会引起一次SIGALARM 信号，所以我们要重新定时，以不断触发  SIGALARM信号。
    alarm(TIMESLOT);
}
// 定时器回调函数，它删除非活动连接socket上的注册事件，并关闭之。
void cb_func( client_data* user_data )
{
    epoll_ctl( epollfd, EPOLL_CTL_DEL, user_data->sockfd, 0 );
    assert( user_data );
    close( user_data->sockfd );
    printf( "close fd %d\n", user_data->sockfd );
}
```
 如果是SIGTERM信号则结束时间循环，退出。
其次，如果是通信描述符数据到来：如果接收到的数据个数小于零，发生了错误，则关闭连接并移除其对应的定时器。如果为0，是对方关闭了连接，我们也要关闭连接，并移除对应的定时器。如果客户端上有数据可读，则需要调整对应的定时器，以延迟该连接被关闭的时间，就是从新设置任务超时时间，然后调整定时器。

### 难点4及解决
关于任务队列，最先开始是每个子线程和主线程一起维护一个任务队列，构成一个典型的单生产者多消费者的场景，但是会造成惊群：主线程添加任务后，锁释放后多个阻塞线程会进行竞争，谁先拿到锁谁访问资源。
解决：设计了每个子线程都会有一个任务队列，这样最多会有两个线程（主线程和一个子线程）对任务队列操作，只会阻塞一个线程，不太会有死锁，也避免惊群发生。唤醒1个使用notify.one()，唤醒多个使用notify.all()。
`此处需要阻塞主线程的还有一个理由：有一种极端情况是主线程添加任务时任务队列还未创建，此时连接会挂掉，所以需要阻塞主线程。`
该方案同时带来了一个阻塞的新场景：线程从任务队列拿到任务事件并注册到检测集合，但是select/poll/epoll的检测集合是没有这个任务的fd，尽管设置了timeout去重新发起检测，仍然存在任务的fd没有加入检测集合的情况，select/poll/epoll依旧阻塞，连带着子线程一起阻塞，此时需要使用socketpair构建套接字对，把这个套接字的一端fd放进检测集合，阻塞时往写端写数据来激活select/poll/epoll，使select/poll/epoll检测到读写事件的发生。
`补个知识：` [阻塞I/O、非阻塞I/O和I/O多路复用、怎样理解阻塞非阻塞与同步异步的区别？ - myseries - 博客园 (cnblogs.com)](https://www.cnblogs.com/myseries/p/11756335.html#:~:text=%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E6%98%AF%E6%8C%87%E4%BD%BF%E7%94%A8,%E5%88%99%E9%98%BB%E5%A1%9E%E7%9B%B4%E5%88%B0%E8%B6%85%E6%97%B6%E3%80%82)

### 缺点和改进
关于线程池线程数量设定的优化。线程池的线程数量在构造的时候就已经确定下来，当机器核心数量改变时需要通过修改才能改变线程数量去匹配，不符合开闭原则，线程数量这一块可以通过C++17新特性动态获得机器的核心数量`std::thread::hardware_concurrency()`，来使程序更好的匹配机器实现更好的性能【避免核心数量过少进行频繁的上下文切换以及核心数量过多被闲置】。或者通过普通线程（临时创建用来响应多余的任务）和核心线程（一直存在）来优化。
	
2. 项目中怎么使用多线程？
	多线程主要是主线程和子线程对任务队列的访问。主线程向子线程的任务队列中添加任务，子线程会在任务队列中修改、删除、添加，两个线程操作时需要同步，只需要用一把锁完成主线程和子线程的同步即可。

3. 难点4那一块
