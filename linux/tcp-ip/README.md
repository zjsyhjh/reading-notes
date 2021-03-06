### 理解网络编程和套接字

网络编程中接受连接请求的套接字创建过程如下

- 第一步：调用`socket`函数创建套接字
- 第二步：调用`bind`函数分配`IP`地址和端口号
- 第三步：调用`listen`函数转为可接受请求状态
- 第四步：调用`accept`函数受理连接请求

```c
#include <sys/socket.h>
/*
 * 套接字生成, 成功时返回文件描述符，失败时返回-1
 * domain - 套接字中使用的协议族（Protocol Family）信息，常用的如下
 	- PF_INET：IPv4互联网协议族
 	- PF_INET6：IPv6互联网协议族
 * type - 套接字数据传输类型，2种具有代表性的数据传输方式
 	- SOCK_STREAM：面向连接的套接字（特点：可靠、按序传输、传输的数据不存在数据边界）
 	- SOCK_DGRAM：面向消息的套接字（特点：传输速度快、传输的数据有边界）
 * protocol - 通信使用的协议信息
 	- 满足IPv4协议族、面向连接这两个条件的协议只有IPPROTO_TCP
 	- 满足Ipv4协议族、面向消息这两个条件的协议只有IPPROTO_UDP
 */
int socket(int domain, int type, int protocol);

/*
 * 给创建好的套接字分配地址信息（ip地址和端口）
 * 成功时返回0，失败时返回-1
 */
int bind(int sockfd, struct sockaddr *myaddr, socklen_t addrlen);\
/*
 * 表示IPv4地址的结构体
 * 一般sin_port以及sin_addr需要使用字节序转换函数（如htons)将主机字节序转换为网络字节序
 * 可利用常数INADDR_ANY自动分配服务器端的IP地址
 */
struct sockaddr_in {
  sa_family_t	sin_family; //地址族
  uint16_t		sin_port; //16位TCP/UDP端口号
  struct in_addr sin_addr; //32位IP地址
  char			sin_zero[8]; //不使用
};
/*
 * 存放32位IP地址
 */
struct in_addr {
  In_addr_t s_addr; //32位IPv4地址
};

/*
 * 注意到bind的第二个参数是sockaddr结构体变量
 * 此结构体成员sa_data保存的地址信息中包含IP地址和端口号，剩余部分应填充0
 * 直接填充信息比较麻烦，通过先填写sockaddr_in结构体变量，然后转化为sockaddr型的结构体变量
 */
struct sockaddr {
  sa_family_t	sin_family; //地址族
  char 			sa_data[14]; //地址信息
};

/*
 * 将套接字转化为可接受连接的状态
 * 成功时返回0， 失败时返回-1
 * 只有调用了listen函数，客户端才能进入可发起连接请求的状态
 * sockfd - 希望进入等待连接请求状态的套接字文件描述符
 * backlog - 连接请求等待队列（Queue）的长度，若为5，则队列长度为5，表示最多使5个连接请求进入队列
 */
int listen(int sockfd, int backlog);

/*
 * 受理连接请求
 * 成功时返回文件描述符，失败时返回-1
 * sockfd - 服务器套接字的文件描述符
 * addr - 保存发起连接请求的客户端地址信息的变量地址值，调用函数后向传递来的地址变量参数填充客户端地址信息
 * addrlen - 第二个参数addr结构体的长度，但是存有长度的变量地址。函数调用完成后，该变量即被填入客户端地址长度
 */
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

网络编程中发起连接请求的套接字创建过程如下

- 第一步：调用`socket`函数创建套接字
- 第二步：调用`connect`函数向服务器端发送连接请求

```c
#include <sys/socket.h>

/*
 * 发起连接请求
 * 成功时返回0，失败时返回-1
 * 在服务器端调用listen函数创建连接请求等待队列之后，客户端即可通过connect函数发起连接请求
 * sockfd - 客户端套接字文件描述符
 * serv_addr - 保存目标服务器端地址信息的变量地址值
 * addrlen - 以字节为单位传递已传递给第二个结构体参数serv_addr的地址变量长度
 */
int connect(int sockfd, struct sockaddr *serv_addr, socklen_t addrlen)；
```

客户端调用`connect`函数后，发生以下情况之一才会返回

- 服务器端接受连接请求（所谓接受连接请求并不意味着服务器端调用`accept`函数，其实是服务器端把连接请求信息记录到等待队列）
- 发生断网等异常情况而中断连接请求

实现服务器端必经过程之一就是给套接字分配IP和端口号，但客户端实现过程中并未出现套接字地址分配，而是创建套接字后立即调用`connect`函数。那么客户端套接字何时、如何分配地址？

- 何时？调用`connect`函数
- 如何？IP用主机的IP地址，端口随机

非阻塞`connect`：在一个TCP套接字被设置为非阻塞之后，调用`connect`函数后会立即返回，如果连接不能马上建立（返回-1），则`errno`设置为`EINPROGRESS`，表示连接正在进行，但仍未完成，于此同时TCP三次握手操作仍在继续。此时可以调用`select`检测非阻塞`connect`是否完成

非阻塞`connect`有三种用途

- 我们可以在TCP三次握手的同时做一些其它的处理。`connect`操作需要一个往返时间才能完成，从几个毫秒（局域网）到几百毫秒或几秒（广域网），这段时间可以同时处理其他的事
- 可以用这种技术同时建立多个连接（在Web浏览器中很普遍）
- 由于我们使用`select`来等待连接的完成，因此可以给`select`设置一个时间限制，从而缩短`connect`的超时时间，在大多数实现中，`connect`的超时时间在75秒到几分钟之间。有时候应用程序想要一个更短的超时时间，使用非阻塞`connect`就是一种方法
  -  `select`与非阻塞`I/O`相关的规则
    - 当连接建立成功时，套接字描述符变为**可写**
    - 当连接建立出错时，套接字描述符变为**既可读又可写**

处理非阻塞`connect`的步骤

- 创建`socket`，返回套接字描述符
- 调用`fcntl`或`ioctlsocket`把套接字描述符设置为非阻塞
- 调用`connect`开始建立连接
- 判断连接是否成功建立
  - 如果`connect`返回0，说明连接成功
  - 否则，调用`select`来判断连接建立是否成功
    - 如果`select`返回0，则表示在`select`的超时时间内未能成功建立连接，需要返回超时错误给用户，同时关闭连接，以防TCP三次握手继续进行下去
    - 如果`select`返回大于0的值，则说明检测到可读或可写或异常的套接字描述符存在，此时应该通过调用`getsockopt(Unix平台下)`来检测集合中的套接字接口上是否存在待处理的错误。如果连接建立是成功的，则`errno`的值为0，否则，就是连接建立错误

