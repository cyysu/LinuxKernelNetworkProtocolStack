## 1. 套接字基础
![](./pic/kingkang1.jpg)

### 1.1 系统调用入口

进程与内核交互是通过一组定义好的函数函数来进行的，这些函数成为系统调用。

### 1.2 系统调用机制（linux在i386上的实现）

1.  每一个系统调用均被编号， 成为系统调用号
2.  当进程进行一个系统调用时， 要通过终端指令 **INT 80H**, 从用户空间进入内核空间，并将系统调用号作为参数传递给内核函数
3.  在 linux 系统中所有的系统调用都会进入系统的同一个地址， 这个地址称为 **system_call**
4.  最终根据系统调用号， 调用系统调用表 **sys_call_table** 中的某一个函数
5. socket系统调用在套接字层、传输层、网络层之间的调用关系（以 `sendmsg()` 为例）

![](./pic/7.1_1.jpg)

### 1.3 套接字的系统调用

- **建立**
	- `socket`:	 	在指明的通信域内产生一个 未命名 的套接字
	- `bind`:		 	分配一个本地地址给套接字
- **服务器** 	
	- `listen`:	 	套接字准备接收连接请求
	- `accept`:	 	等待接受连接
- **客户** 	
	- `connect`: 	 	同外部套接字建立连接
- **输入** 	
	- `read`:		 	接收数据到一个缓冲区
	- `readv`: 	 	接收数据到多个缓冲区
	- `recv`: 	 	指明选项接收数据
	- `recvfrom`:  	接收数据和发送者的地址
	- `redvmsg`:	 	接收数据到多个缓存中, 接收控制信息和发送者地址;指明接收选项
- **输出** 	
	- `write`:	 	发送一个缓冲区的数据
	- `writev`:	 	发送多个缓冲区的数据
	- `send`:		 	指明选项发送数据
	- `secdto`:	 	发送数据到指明的地址
	- `sendmsg`:	 	从多个缓存发送数据和控制信息到指明的地址; 指明发送选项
- **I/O**
	- `select`: 	 	等待 I/O 事件
- **终止** 	
	- `shutdown`:  	终止一个或者连个方向上的连接
	- `close`: 	 	终止连接并释放套接字
- **管理** 	
	- `fcntl`: 	 	修改 I/O 语义
	- `ioctl`:	 	各类套接字的操作
	- `setsockopt`:	设置套接字或者协议选项
	- `getsockopt`:	获得套接字或者协议选项
	- `getsockname`:	得到分配给套接字的本地地址
	- `getpeername`:	得到分配给套接字的远端地址


### 1.4 socket系统调用号
![](./pic/7.1_2.png)
	
	
### 1.5 `sys_socketcall()`
		系统中所有的 socket 系统调用总入口为 sys_socketcall() 有两个参数 call , args
		
- **参数说明**

	- **call**，  **操作码**，函数中通过操作码跳转到真正的系统调用函数
	- **args**， **指向一个数组的指针 **，指向用户空间，表示系统调用的参数

- **工作流程**

0. [compat.c/compat_sys_socketcall()](./compat.c)中注释了源代码
1. 判断参数 call 是否在操作码所表示的范围内
2. 将参数 args 所表示的用户空间的数据（ 总共`nas(call)`,nas定义在[compat.c](./compat.c)文件中 ）个拷贝到内核空间的变量中
3. 通过一个 `switch(call)`，如果不同的操作码来调用相应的套接字的系统调用（如：`sys_socket()`, `sys_bind()`,etc）
4. 该函数的返回结果就是以上系统调用的返回结果
5. call 参数所代表的不同的系统调用的**通常**调用流程图

![7.1_3](./pic/7.1_3.png)


-----------------------------------------------------------------------


## 2. `socket`系统调用
![](./pic/kingkang2.jpg)
----
### 2.1 `sys_socket()`
	sys_socket() 把套接字的创建 和 与此套接字关联的文件描述符的分配做了简单的封装，完成创建套接字的功能

 - **参数说明**

	- **family**,  待创建套接字的协议族，如 *PF_INET, `PF_UNIX*
	- **type**, 待创建套接字的类型，如*SOCK_STREAM, SOCK_DGRAM, SOCK_RAW* 等
	- **protocol**，传输层协议，如*IPPROTO_TCP, IPPROTO_UDP*等


- **工作流程**

0. [sys_socket.c/SYSCALL_DEFINE3()](./sys_socket.c)中注释了源码
1. 创建一个 `struct socket` 类型的指针 `sock`
2. 将 `sock` 的地址传入 `sock_create()`
	- `sock_create()`函数内部调用 `__sock_create()`
		- 首先调用`sock_alloc()`
		- 调用`pf->create()`
			- `sk_alloc()`
			- `sk->sk_prot->hash()`, TCP: `tcp_v4_hash()`, RAW: `raw_v4_hash()`
			- `sk->sk_prot->init()`, TCP: `tcp_v4_init_sock()`, RAW: `raw_init()`
	- `sock_map_fd()` 为创建好的套接字分配一个文件描述符，并绑定
3. 返回错误值
	
##  2.2 `__sock_create()`
	sock_create() 内部对__sock_create() 做了调用，

- **参数说明**

	- **family**,  待创建套接字的协议族，如 *PF_INET, `PF_UNIX*
	- **type**, 待创建套接字的类型，如*SOCK_STREAM, SOCK_DGRAM, SOCK_RAW* 等
	- **protocol**，传输层协议，如*IPPROTO_TCP, IPPROTO_UDP*等
	- **res**,  表示的就是上述的 *struct socket*结构体指针的地址, 用来输出套接字
	- **kern** ， 用来表示创建套接字的是内核还是普通进程


- **工作流程**

0. [sys_socket.c/__sock_create() ](./sys_socket.c),中注释了源码
1. 检查`family`， `type` 是否合法范围内
2. 由于`SOCK_PACKET`类型的套接字已经废除， 而在系统外增加一个`PF_PACKET`类型的协议族，将前者强转成后者
3. 安全模块对台阶自的创建做检查 , `security_socket_create()`
4. `sock_alloc()` 在 `sock_inode_cache` 缓存中分配 **i节点** 和 **套接字**，同时初始化 i节点 和 套接字（ **i节点用来标识此文件并与套接字关联，让套接字可以向一般的文件对他进行读写）**，如果分配失败则会给出警告：`socket: no more sockets`，并根*据套接字的类型参数设置创建的套接字的类型*
5. 根据参数`family` 获取已经注册在`net_families`中的对应的`net_proto_family`指针( **pf** ),*需要读写锁的保护*
6. `try_module_get(net_families[family])`,`family` 标识的类型的协议族`net_proto_family`是以内核模块加载并**动态**注册到net_families中，则需要对内核模块引用计数加一，防止创建过程中此内核模块被动态卸载， 并对读写锁解锁
7. `pf->create(sock, protocol)`, 继续对套接字初始化（调用IPv4协议族中的**inet_create()**），同时创建传输控制块
8. `try_module_get(sock->ops->owner)`, 如果`sock->ops`是以内核模块的方式动态加载，并且注册到内核中的，则需要对内核模块引用计数加一（ ），防止创建过程中此内核模块被动态卸载
9. `module_put(pf->owner)`, 完成对IPv4协议族中的`inet_create()`调用完后，对模块的引用计数减一, 进行一系列错误检查创建完成
	
### 2.3 `inet_create()`
		 与协议族有关的套接字以及传输控制块创建过程在这个函数之后全部结束，返回到创建套接字的统一接口中

- **参数说明**

	- **sock**， 已经在__sock_create()中创建的套接字
	- **protocol**， 表示创建套接字的协议号


- **工作流程**

0. [sys_socket.c/inet_create() ](./sys_socket.c),中注释了源码
1. `sock->state = SS_UNCONNECTED`, 将套接字的状态注册成**SS_UNCONNECTED**
2. `list_for_each_rcu(),`**将sock->type作为关键字遍历inetsw散列表**
3. `list_entry()`，通过计算偏移的方法获取指向 inet_protosw 的结果的指针
4. 根据参数类型获取匹配的 inet_protosw 结构体的实例
5. 如果不能再 inetsw 中获得匹配的 inet_protosw 结构的实例，则需加载相应的内核模块，再返回第五步，（最多尝试两次，失败则会退出）
6. 判断当前进程是否有*answer->capability*（保存在进程的描述符中国）的能力，如果没有则不能创建套接字
7. `sock->ops = answer->ops`, 用来设置套接字层 和 传输层之间的接口ops
8. `sk_alloc()`, 用来**分配一个传输控制块**，返回值放在**sk**中
9. 设置传输模块是否需要校验(**sk->sk_no_check**) 和 是否可以重用地址和端口标志（**sk->sk_reuse**）
10. 设置 **inet_sock** 块中的 **is_icsk** ,用来标识是否为面向连接的传输控制块
11. 如果套接字为原始类型，则设置本地端口为协议号 并且 **inet->hdrincl** 表示需要自己构建 IP 首部
12. 设置传输模块是否支持 PMTU(动态发现因特网上任意一条路径的最大传输单元(MTU)的技术)
13. `sock_init_data(sock, sk)`, 对传输控制块进行了初始化。
14. 初始化**sk->destruct**, 在套接字释放时回调，用来清理和回收资源，设置传输控制字协议族(**sk->sk_family**)和协议号标识(**sk->sk_protocol**)
15. 设置传输控制块 单播的TTL, 是否法相回路标志，组播TTL, 组播使用的本地接口索引，传输控制块组播列表
16. 如果传输控制块中的num设置了本地端口号，则设置传输控制块中的sport网络字节序格式的本地端口号； **调用传输层接口上的hash(),把传输控制块加入到管理的散列表中**；（TCP: `tcp_v4_hash()`, UDP:`udp_lib_hash()`）
17. 如果sk->sk_prot->init指针已经被设置，则会调用sk->sk_prot->init(sk)来进行具体传输控制块的初始化（TCP: `tcp_v4_init_sock()`,无UDP）


-----------------				 
## 3. `bind` 系统调用
![](./pic/kingkang3.jpg)

### 3.1 `sys_bind()`
		bind 系统调用将一个本地的地址 及 传输层的端口 和 套接字 关联起来
		一般作为客户端进程并不关心它的本地地址和端口是什么，所以没有必要调用 bind()，内核将会自动为其选择一个本地地址和端口
		
- **参数说明**

	- **fd**, 进行绑定的套接字的文件描述符
	- **umayaddr**, 进行绑定的地址
	- **addrlen**，进行绑定地址的长度.(不同协议族的地址描述结构不一样， 所以需要长度)

- **工作流程**

0. [sys_socket.c/SYSCALL_DEFINE3()](./sys_socket.c)中注释了源码
1. 首先创建 **struct socket** 类型的指针 **sock**
2. `sockfd_lookup_light()`,  根据文件描述符 **fd** 获取套接字的指针， 并且返回是否需要对文件引用计数的标志
3. `move_addr_to_kernel(umyaddr, addrlen, address)`,  **address**字符型数组用来保存地址从用户空间传进来的绑定地址
4. `security_socket_bind()`，安全模块对套接字bind做检查
5. `sock->ops->bind()`, 在 **inet_create()** 第 8 步中设置了套接字层与传输层之间的接口 **ops** ,所有类型的套接字的 bind 接口是统一的即 `inet_bind()` ，他将实现传输层接口 bind 的有关调用
	- **RAW:** `sk->sk_prot->bind()` ---> `raw_bind()`
	- **TCP/UDP:** `sk->sk_prot->get_port()`
		- **TCP:** `tcp_v4_get_port()`
		- **UCP:** `rdp_v4_get_port()`
6. `fput_light()`, 根据第二步中获得标志， 对文件的引用计数进行操作


### 3.2 `inet_bind()`
	所有类型的套接字的 bind 接口是统一的即 `inet_bind()` ，他将实现传输层接口 bind 的有关调用， 是 bind 系统调用的套接字层的实现

- **参数说明**

	- **sock**, 套接字指针类型，`sys_bind()` 中根据 fd 获得的套接字指针
	- **uaddr**, 绑定地址，`struct sockaddr` 结构体类型指针，即	`（struct sockaddr *）address` , address 是一个字符数组从 用户空间  拷贝到 内核空间
	- **addr_len**, 绑定地址的长度

- **工作流程**

0. [sys_socket.c/inet_bind()](./sys_socket.c)中注释了源码
1. **addr**、**sk** 、**inet**指针， 分别是`（struct socketaddr_in）uaddr`、`sock->sk` 、`inet_sk(sk)`
2. `sk->sk_prot`, 在`inet_create()` 中 `sk_alloc()` 中被初始化，如果是TCP套接字该值就是**tcp_prot**，只有RAW 类型（**SOCK_RAW**）的套接字才可以直接调用传输层接口上的 bind()， 即当前套接字在传输层接口上有 bind 的实现( `raw_bind()` )，`sk->sk_prot->bind` 为真，完成后可以直接返回
3. 若没有 bind 的实现，则需要对绑定地址的长度进行合法性检查
4. `inet_addr_type(addr->sin_addr.sin_addr)`， 根据**绑定地址中的地址参数**得到地址的类型(组播，广播，单播...)
5. 对上一步得到的地址类型进行检查，判断是否可以进行地址和端口的绑定
6. 将绑定地址中的网络字节序的端口号转换成本机的字节序，并对端口进行合法性校验，并且还要判断是否允许绑定小于1024的特权端口，保存在 **snum** 中
7. `sk->sk_state != TCP_CLOSE || inet->inet_num`, 如果套接字的状态不是TCP_CLOSE或者已经是绑定过的套接字则会返回错误
8. `inet->inet_rcv_saddr = inet->inet_saddr = addr->sin_addr.s_addr`,   将传入的绑定地址设置到传输控制块中
9. `sk->sk_prot->get_port(sk, snum)`,  调用传输层 `get_port`进行具体的传输层的地址绑定，该 **get_port()** 对应的**TCP:** `tcp_v4_get_port()`, **UCP:** `rdp_v4_get_port()`
10. `sk->sk_userlocks |= SOCK_BINDADDR_LOCK;sk->sk_userlocks |= SOCK_BINDPORT_LOCK;`   标识了传输控制块已经绑定了 **本地地址** 和 **本地端口**
11. `inet->sport = htons(inet->num)` 设置本地端口.   再初始化目的地址和目的端口为0