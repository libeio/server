
## 文件说明
+ include/
	- 基本库，第三方，测试，自定义
+ dataparse/
	- 数据解析，数据测试
+ manager/
	- 服务器管理，数据库管理
+ message/
	- 消息定义，消息解析
+ support/
	- 函数支援，类支援 （该文件夹下的所有文件最好应相互独立）
+ wrapper/
	- 第三方库的封装类
+ serverbusiness/
	- 业务服务器
+ serverdatabase/
	- 数据库
+ serverforward/
	- 核心转发器
+ servergateway/
	- 网关服务器


## 整体说明
### 整体拓扑
+ 涉及到的实体对象主要有 客户端，网关服务器，转发服务器，数据库服务器，日志服务器，业务服务器 等。 
+ 转发器监听(listen)来自其他服务器（如业务服务器、网关服务器、数据库服务器等）的连接(connect)，也就是
  说其他服务器是转发器的客户端。
+ 网关监听(listen)来自客户端的连接(connect)。
+ 其拓扑如下(A ---> B 表示 A 连接到 B，B 监听 A)：

		+----------+                                      +----------+
		| 客户端1  +----------+              +------------+  业务1   |
		+----------+          ↓              |            +----------+
		+----------+    +-----+---+          |            +----------+
		| 客户端2  +--->+  网关1  +-------+  |  +---------+  业务2   |
		+----------+    +---------+       |  |  |         +----------+
		+----------+                      |  |  |         +----------+
		| 客户端3  +----------+           |  |  |  +------+  业务3   |
		+----------+          ↓           ↓  ↓  ↓  ↓      +----------+
		+----------+    +-----+----+    +-+--+--+--+-+    +----------+
		| 客户端4  +--->+  网关2   +--->+   转发器   +<---+  业务... |
		+----------+    +-----+----+    +-+--+--+--+-+    +----------+
		+----------+          ↑           ↑  ↑  ↑  ↑      +----------+
		| 客户端5  +----------+           |  |  |  +------+  数据库  |
		+----------+                      |  |  |         +----------+
		+----------+    +----------+      |  |  |         +----------+
		| 客户端...+--->+  网关... +------+  |  +---------+   日志   |
		+----------+    +-----+----+         |            +----------+
		+----------+          ↑              |            +----------+
		| 客户端...+----------+              +------------+    ...   |
		+----------+                                      +----------+

### 服务器架构

		+----------------------------------------------------------------------------------------------------------------------------------+
		|                                                         ServerManager                                                            |  管理层
		+----------------------------------------------------------------------------------------------------------------------------------+
		+----------------------------------------------------------------------------------------------------------------------------------+
		|                                                         AbstractServer                                                           |  抽象层
		+----------------------------------------------------------------------------------------------------------------------------------+
		+----------+   +----------+   +----------+   +----------+   +----------+   +----------+   +----------+   +----------+   +----------+
		|   网关   |   |  转发器  |   |  业务1   |   |  业务2   |   |  业务3   |   |  业务... |   |  数据库  |   |   日志   |   |   ...    |  实体层
		+----------+   +----------+   +----------+   +----------+   +----------+   +----------+   +----------+   +----------+   +----------+

+ 通过抽象层派生出实体层，在实体层中对抽象层中定义的方法进行实现，管理层通过调用抽象层方法实现对实体服务器的管理。
+ 服务器之间，服务器与客户端之间通过基于套接字的 bufferevent 实现消息流传递。
	- 服务器外发数据流程：管理层将 Message 转化为字符串风格的 message，实体层将 message 序列化为流，写入对应的对端套接字(Message -> message -> stream)；
	- 服务器接收数据流程：实体层从对应的对端套接字读取流，将流转化为字符串风格的 message，管理层将 message 结构化为 Message(stream -> message -> Message)。
+ 管理层、实体层、套接字底层三者之间的消息转化示意

		+---------------------------------------------+
		|   管理层(结构化消息 Message)                |
		|       +-----------------------------------+ |
		|       |  实体层(字符串风格消息 message)   | |
		|       |     +---------------------------+ | |
		|       |     |   套接字底层(stream)      | | |
		|       |     +---------------------------+ | |
		|       +-----------------------------------+ |
		+---------------------------------------------+ 

### 心跳机制
+ 相对于转发器而言，其他服务器是其客户端。为保持长连接，转发器需要定期向其他服务器发送心跑包，以检测对端服务器是否与转发器保持连接状态。
+ 网关服务器会定期向客户端发送心跳包，以检测客户端是否与网关保持连接状态。
+ 心跳机制可以进行升级，目前所知有两种方式：一种方式是使用客户端发送心跳，一种是使用 time-wheeling
 
## 细节说明
### 监听服务器和连接服务器
+ 只有两个监听服务器，分别是网关和转发器，而网关请求连接转发器，也作为转发器的客户端存在。
+ 所以对于转发器而言，一般只需要实现处理客户端请求的方法即可
	- net_client_connected
	- net_client_disconnected
	- net_client_message_handle
+ 对于数据库服务器、日志服务器、业务服务器等连接服务器而言，因为其是转发器的客户端，所以一般只需要实现处理服务器消息的方法即可
	- net_server_connected
	- net_server_disconnected
	- net_server_message_handle
+ 对于网关，因为其既是监听服务器（监听来自真正客户端的请求），又作为转发器的客户端，所以需要实现以上两类方法。
### 对于 ClientInfo 的理解
+ 网关中的 ClientInfo 保存的是真实客户端的信息，转发器中的 ClientInfo 保存的是 网关、数据库、日志、业务等服务器的信息。	
