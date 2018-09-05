## 1. 触发方式
> 1. 水平触发(Level-Trigger, LT)：**只要**socket所在缓冲区writeable或者readable，就触发事件。
> 2. 边缘触发(Edge-Trigger, ET)：**只有**socket所在缓冲区从unwritable/unreadable切换为writable/readable时，才触发事件。

## 2. recv调用
**recv一般用于面向连接的socket通信**

在tcp通信中，recv收到的只是有序的字节流。应用层网络收发模块需要自行完成协议的组包，才能交付其他模块使用。组包时首先需要判断当前收取的字节流是否能构成一个完整的应用层协议，否则需要继续收取，直至满足。
应用层协议通常也是以**head+body**的方式组合，组包过程大致如下：
>1. 判断当前应用层缓冲区的大小是否满足 sizeof(head)，如果不满足，则等待下次收包再判断；否则继续步骤2。
>2. 反序列化出head，取出head.body_len。
>3. 判断当前应用层缓冲区的大小是否满足 sizeof(head) + head.body_len，如果不满足，则返回；否则继续步骤4。
>4. 此时收到了完整的应用层协议包，可将head起始指针及协议包大小，或者body指针及body大小，交付至其他模块。
>5. 根据body指针及大小，反序列化出应用层结构，继续业务逻辑的处理。

UDP通信时不需要组包的流程。因为UDP没有网络控制，每次sendmsg发送的数据会全部发送到对端去，但是如果发送数据大小超过链路MTU，会导致IP层的拆包和合包。所以在游戏业务中，为了降低网络延迟，发送数据的两端会保证单个协议包的长度小于MTU-UDP.head_len。
```c
#include <sys/types.h>
#include <sys/socket.h>
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

### 2.1. 阻塞I/O
>1. 如果socket接收缓冲区空。recv调用将阻塞，直到有数据到达。
>2. 如果socket接收缓冲区非空。recv将数据从socket缓冲区拷贝至应用层缓冲区。**如果应用层缓冲区大小小于socket缓冲区的数据大小，应用层可多次读取，直到再次阻塞。**

### 2.2. 非阻塞I/O
>1. 如果socket缓冲区空。recv调用直接返回-1，并设置errno为**EAGAIN**或者**EWOULDBLOCK**。
>2. 如果socket缓冲区非空。正常读取数据，直到socket缓冲区空，或者应用层缓冲区满。

### 2.3. 多路I/O复用
为了更好的检查socket是否有可读事件发生，可以采用多路I/O复用的技术，常使用的方式有select，poll，或者epoll（**linux特有**）。

#### LT方式
当以LT方式监听事件时，因为只要可读就通知，所以只要触发可读事件，即可调用recv读取数据。

首先，此事件触发表示缓冲区一定有数据可读。如果应用层缓存区大小大于socket缓冲区中数据的大小，则会读取socket缓冲区中所有的数据，并返回读取的字节数；如果应用层缓冲区小于socket缓冲区中数据的大小，则会读取应用层缓冲区最多允许的字节数，并在下次轮询时再次触发可读事件，或者以其他方式让出应用层缓冲区以继续读取数据。

#### ET方式
因为ET方式只有在缓冲区从unreadable转换为readable时才触发事件，所以在此模式下需要一直读，直到返回-1，errno为EAGAIN或者EWOULDBLOCK。否则，后续即使有新的数据到达，也不再会触发事件。

考虑到上述情况，在ET触发方式下，如果应用层缓冲区不足以存储socket缓冲区的数据，需要动态扩充应用层缓冲区大小，以允许接收所有socket缓冲区数据，或者以其他方式让出应用层缓冲区以继续读取数据。否则，只能close掉当前socket连接，因为该socket已经不会在触发事件了。

## 3. send调用
在tcp中，send调用正常返回只表示数据由应用层缓存区拷贝到了socket发送缓冲区，但是还未发送到网络中。具体的发送流程依赖于tcp的拥塞控制算法。**因此发送端调用一次send，接收端可能需要多次recv才能收到完整的数据。**
```c
#include <sys/socket.h>
ssize_t send(int socket, const void *buffer, size_t length, int flags);
```

### 3.1. 阻塞I/O
>1. 如果socket发送缓冲区大小小于length。send调用将阻塞，直至socket缓冲区大小满足发送数据的长度。
>2. 如果socket发送缓冲区大小大于等于length。send调用将应用层缓冲区的数据拷贝至socket缓冲区，返回拷贝的字节数。

### 3.2. 非阻塞I/O
>1. 如果socket发送缓冲区大小小于length。send调用直接返回-1，并设置errno为**EAGAIN**或者**EWOULDBLOCK**。
>2. 如果socket发送缓冲区大小大于等于length。send调用将应用层缓冲区的数据拷贝至socket缓冲区，返回拷贝的字节数。

### 3.3. 多路I/O复用
为了更好的检查socket是否有可写等事件发生，同样可以采用多路I/O复用的技术。

#### LT方式
当以LT方式监听事件时，因为只要可写就通知，所以只要触发可写事件，即可调用send发送数据。
但是，在此模式下，如果socket可写，就会一直触发事件，不可接受，所以一般采用如下方式。
>1. 不监听socket，直接send。
>2. send返回失败，errno为EAGAIN或者EWOULDBLOCK，表示发送缓冲区满，此时在应用层将socket设置为不可写，并使用I/O复用技术监听套接字的可写事件。
>3. 套接字返回可写事件。将套接字移除监听列表。继续步骤1。

#### ET方式
因为ET方式只有在缓冲区从unwriteable转换为writeable时才触发事件，所以在此模式下需要一直写，直到返回-1，errno为EAGAIN或者EWOULDBLOCK。否则，后续socket可写，也不再会触发事件。
所以在ET触发方式下，只要应用层缓冲区有数据就直接调用send，如果socket缓冲区满导致send返回失败，则在下次触发事件时再次send即可。但是，如果socket一直不可写，则会导致多次无效的send调用。

#### 注意：
>因为单次send可能无法发送所有应用层缓冲区的数据，所以为了保证网络模块能够正确的发送其它模块的协议数据，需要应用层网络模块自行维护一个应用层的发送队列，用来缓冲其它模块的数据，并负责将发送队列的数据写入socket缓冲区。此场景也适用于接收数据的情况。
