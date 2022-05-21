我的理解

网络模型属于应用的编码实现，一种范式，其根基一定是os内核针对tcp/ip协议栈的支持

上层使用的需求推进着底层的支持力度，底层支持方式作用着上层的使用形式

 

**同步**IO-应用程序自己去解决数据读取的过层，应用过程既关注过程，也关注结果

**异步**IO-应用程序向内核发送数据读取的需求，过程由os操作，应用程序只关注结果

因此IO是同步还是异步，一定是由内核开放的api决定的，最终应用程序通过系统调用决定使用哪种方式，如果内核不支持异步，应用代码写出花来也没办法异步

 

**阻塞**IO-应用程序操作socket的accept、read、write这些过程存在阻塞点

**非阻塞**IO-应用程序可以不需要阻塞在上面的步骤上

显而易见，是否阻塞也一定是内核支持的，比如`socket.accpet()`对应的某个内核实现就是阻塞的，不支持非阻塞，那么应用程序也只能阻塞

 

Java中

BIO就是同步阻塞IO

NIO就是同步非阻塞IO

 

假使现在内核仅仅支持阻塞方式的IO，那么在编写应用代码时能有什么选择，通过需求驱动，增加对网络模型的理解

 

## 一 BIO

### 1 一个线程版本

```java
package debug.io.model;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.ServerSocket;
import java.net.Socket;

/**
 * <p>功能实现就是获取请求的连接，获取该连接发送的消息，开展业务逻辑</p>
 * <p>设计到的阻塞操作是<tt>accept</tt>和<tt>read</tt></p>
 * <p>在整个轮询中，假使获取到了一个<tt>socket</tt>之后，后续的操作整个被阻塞住，并且与此同时有源源不断的连接进来，但是因为main线程一直阻塞，导致请求无法处理</p>
 * @since 2022/5/20
 * @author dingrui
 */
public class BIOModelByMainThread {

    private static final int PORT = 9991;

    public static void main(String[] args) throws IOException {
        // 在os底层做了bind和listen
        ServerSocket server = new ServerSocket(PORT);
        while (true) {
            /**
             * 第1个阻塞点 拿到连接的socket
             */
            Socket socket = server.accept();
            InputStream in = socket.getInputStream();
            BufferedReader reader = new BufferedReader(new InputStreamReader(in));
            while (true) {
                /**
                 * 第2个阻塞点
                 */
                String msg = reader.readLine();
                // TODO: 2022/5/21 业务逻辑
            }
        }
    }
}
```

这个版本的问题很明显，因为阻塞导致请求无法被处理，那么正常情况下可能就会朝着多线程的防线发展，为每个请求连接分配一个线程

 

### 2 多线程版本

```java
package debug.io.model;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.ServerSocket;
import java.net.Socket;

/**
 * <p>多线程</p>
 * <p>为每个连接都创建一个线程 每个线程中的<tt>read</tt>操作是阻塞点<ul>
 *     <li>假使读操作这样的一个阻塞近乎于不阻塞，也就是一个线程创建后，拿到cpu执行时间片后可以立马执行，执行完后进行线程销毁</li>
 *     <li>假使读操作近乎于无限阻塞，就是一个线程创建后，一直被阻塞</li>
 * </ul>
 * 上面是两个极限情况，实际情况即使没那么糟糕也明显存在的问题就是<ul>
 *     <li>线程创建和销毁都是比较重的os开销</li>
 *     <li>线程创建过多占用内存资源很大</li>
 *     <li>线程之间上下文切换占用os资源</li>
 * </ul>
 * </p>
 * @since 2022/5/21
 * @author dingrui
 */
public class BIOModelMultipleThread {

    private static final int PORT = 9991;

    public static void main(String[] args) throws IOException {
        ServerSocket server = new ServerSocket(PORT);
        while (true) {
            Socket socket = server.accept();
            new Thread(() -> {
                try {
                    InputStream in = socket.getInputStream();
                    BufferedReader reader = new BufferedReader(new InputStreamReader(in));
                    while (true) {
                        String msg = reader.readLine();
                        // TODO: 2022/5/21 业务逻辑
                    }
                } catch (Exception ignored) {
                }
            }).start();
        }
    }
}
```

有了多线程版本的问题后，可能会想着是不是可以用线程池实现，解决繁重的线程创建和销毁的问题，让线程得以服用

 

### 3 线程池版本

```java
package debug.io.model;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.*;

/**
 * <p>多线程</p>
 * <p>为每个连接都创建一个线程 每个线程中的<tt>read</tt>操作是阻塞点<ul>
 *     <li>假使读操作这样的一个阻塞近乎于不阻塞，也就是一个线程创建后，拿到cpu执行时间片后可以立马执行，执行完后进行线程销毁</li>
 *     <li>假使读操作近乎于无限阻塞，就是一个线程创建后，一直被阻塞</li>
 * </ul>
 * 上面是两个极限情况，实际情况即使没那么糟糕也明显存在的问题就是<ul>
 *     <li>线程创建和销毁都是比较重的os开销</li>
 *     <li>线程创建过多占用内存资源很大</li>
 *     <li>线程之间上下文切换占用os资源</li>
 * </ul>
 * </p>
 * @since 2022/5/21
 * @author dingrui
 */
public class BIOModelByThreadPool {

    private static final int PORT = 9991;

    private static final ExecutorService myTP = new ThreadPoolExecutor(2, 5, 60L, TimeUnit.SECONDS, new LinkedBlockingQueue<>(Integer.MAX_VALUE));

    /**
     * 任务对象
     */
    private static class MyTask implements Runnable {

        private Socket socket;

        public MyTask(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            InputStream in = null;
            try {
                in = this.socket.getInputStream();
                BufferedReader reader = new BufferedReader(new InputStreamReader(in));
                while (true) {
                    String msg = reader.readLine();
                    // TODO: 2022/5/21 业务逻辑
                }
            } catch (Exception ignored) {
            }
        }
    }

    public static void main(String[] args) throws IOException {
        ServerSocket server = new ServerSocket(PORT);
        while (true) {
            Socket socket = server.accept();
            // 封装成任务丢进线程池
            myTP.submit(new MyTask(socket));
        }
    }
}
```

即使是多线程版本，依然存在不容忽视的问题

1. 线程池资源
2. 任务存在阻塞点，单个任务存在执行效率问题
3. 任务一直阻塞的话，引发线程池任务队列容量设置问题，更甚触发线程池拒绝策略或者直接OOM

 

其上，os内核提供的系统调用只支持阻塞式，业务代码再怎么写都是存在着或多或少的问题和弊端，其根本原因就是内核的阻塞调用

换言之，想要改变这样的编码，就得需要内核做出相应的支持

 

## 二 NIO

此时，os内核对某几个方法做出改变

 

### 1 随便查询几个方法的手册

#### 1.1 socket

```bash
man 2 socket
NAME
     socket -- create an endpoint for communication

SYNOPSIS
     #include <sys/socket.h>

     int
     socket(int domain, int type, int protocol);

RETURN VALUES
     A -1 is returned if an error occurs, otherwise the return value is a descriptor referencing the socket.
```

 

#### 1.2 bind

```bash
man 2 bind
NAME
     bind -- bind a name to a socket

SYNOPSIS
     #include <sys/socket.h>

     int
     bind(int socket, const struct sockaddr *address, socklen_t address_len);

DESCRIPTION
     bind() assigns a name to an unnamed socket.  When a socket is created with socket(2) it exists in a name space
     (address family) but has no name assigned.  bind() requests that address be assigned to the socket.

NOTES
     Binding a name in the UNIX domain creates a socket in the file system that must be deleted by the caller when it is no
     longer needed (using unlink(2)).

     The rules used in name binding vary between communication domains.  Consult the manual entries in section 4 for
     detailed information.

RETURN VALUES
     Upon successful completion, a value of 0 is returned.  Otherwise, a value of -1 is returned and the global integer
     variable errno is set to indicate the error.
```

 

#### 1.3 listen

```bash
man 2 listen
LISTEN(2)                   BSD System Calls Manual                  LISTEN(2)

NAME
     listen -- listen for connections on a socket

SYNOPSIS
     #include <sys/socket.h>

     int
     listen(int socket, int backlog);

DESCRIPTION
     Creation of socket-based connections requires several operations.  First, a socket is created with socket(2).  Next, a
     willingness to accept incoming connections and a queue limit for incoming connections are specified with listen().
     Finally, the connections are accepted with accept(2).  The listen() call applies only to sockets of type SOCK_STREAM.

     The backlog parameter defines the maximum length for the queue of pending connections.  If a connection request
     arrives with the queue full, the client may receive an error with an indication of ECONNREFUSED.  Alternatively, if
     the underlying protocol supports retransmission, the request may be ignored so that retries may succeed.

RETURN VALUES
     The listen() function returns the value 0 if successful; otherwise the value -1 is returned and the global variable
     errno is set to indicate the error.
```

 

#### 1.4 accept

```bash
man 2 accept
ACCEPT(2)                   BSD System Calls Manual                  ACCEPT(2)

NAME
     accept -- accept a connection on a socket

SYNOPSIS
     #include <sys/socket.h>

     int
     accept(int socket, struct sockaddr *restrict address, socklen_t *restrict address_len);

DESCRIPTION
     The argument socket is a socket that has been created with socket(2), bound to an address with bind(2), and is listen-
     ing for connections after a listen(2).  accept() extracts the first connection request on the queue of pending connec-
     tions, creates a new socket with the same properties of socket, and allocates a new file descriptor for the socket.
     If no pending connections are present on the queue, and the socket is not marked as non-blocking, accept() blocks the
     caller until a connection is present.  If the socket is marked non-blocking and no pending connections are present on
     the queue, accept() returns an error as described below.  The accepted socket may not be used to accept more connec-
     tions.  The original socket socket, remains open.

     The argument address is a result parameter that is filled in with the address of the connecting entity, as known to
     the communications layer.  The exact format of the address parameter is determined by the domain in which the communi-
     cation is occurring.  The address_len is a value-result parameter; it should initially contain the amount of space
     pointed to by address; on return it will contain the actual length (in bytes) of the address returned.  This call is
     used with connection-based socket types, currently with SOCK_STREAM.

     It is possible to select(2) a socket for the purposes of doing an accept() by selecting it for read.

     For certain protocols which require an explicit confirmation, such as ISO or DATAKIT, accept() can be thought of as
     merely dequeuing the next connection request and not implying confirmation.  Confirmation can be implied by a normal
     read or write on the new file descriptor, and rejection can be implied by closing the new socket.

     One can obtain user connection request data without confirming the connection by issuing a recvmsg(2) call with an
     msg_iovlen of 0 and a non-zero msg_controllen, or by issuing a getsockopt(2) request.  Similarly, one can provide user
     connection rejection information by issuing a sendmsg(2) call with providing only the control information, or by call-
     ing setsockopt(2).

RETURN VALUES
     The call returns -1 on error and the global variable errno is set to indicate the error.  If it succeeds, it returns a
     non-negative integer that is a descriptor for the accepted socket.
```

 

#### 1.5 recv

```bash
man 2 recv
RECV(2)                     BSD System Calls Manual                    RECV(2)

NAME
     recv, recvfrom, recvmsg -- receive a message from a socket

LIBRARY
     Standard C Library (libc, -lc)

SYNOPSIS
     #include <sys/socket.h>

     ssize_t
     recv(int socket, void *buffer, size_t length, int flags);

     ssize_t
     recvfrom(int socket, void *restrict buffer, size_t length, int flags, struct sockaddr *restrict address,
         socklen_t *restrict address_len);

     ssize_t
     recvmsg(int socket, struct msghdr *message, int flags);
     
RETURN VALUES
     These calls return the number of bytes received, or -1 if an error occurred.

     For TCP sockets, the return value 0 means the peer has closed its half side of the connection.
```

 

#### 1.6 send

```bash
man 2 send
SEND(2)                     BSD System Calls Manual                    SEND(2)

NAME
     send, sendmsg, sendto -- send a message from a socket

SYNOPSIS
     #include <sys/socket.h>

     ssize_t
     send(int socket, const void *buffer, size_t length, int flags);

     ssize_t
     sendmsg(int socket, const struct msghdr *message, int flags);

     ssize_t
     sendto(int socket, const void *buffer, size_t length, int flags, const struct sockaddr *dest_addr,
         socklen_t dest_len);
         
RETURN VALUES
     Upon successful completion, the number of bytes which were sent is returned.  Otherwise, -1 is returned and the global
     variable errno is set to indicate the error.
```

 

凡此种种，也就意味着在java层面的代码最终执行到os sc的时候不需要阻塞等待结果返回，而是一定可以拿到一个有明确语义的返回值，java再根据语义封装成类对象，用户代码根据一定规则判断是否获取到连接对象或者是否可读可写之类的

 

### 2 NIO单路版本

```java
package debug.io.model;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.channels.ServerSocketChannel;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

/**
 * <p>NIO非阻塞下 单路模型</p>
 *
 * <p>逆向理解多路复用 当前连接的获取不存在阻塞 也就是说可以源源不断获取大量的连接 但是连接的读写状态我们并不知道
 * <p>现在有个集合 里面全是socket<ul>
 *     <li>用户层可以轮询挨个向os发送sc 问它这个socket的状态 拿到读写状态后进行操作 这个时候发生了一次系统调用 向知道整个集合的socket状态就得发生N次系统调用</li>
 *     <li>os提供一个函数 入参是集合 我们一次性将所有socket发给os os告诉用户这些连接的读写状态 发生一次系统调用</li>
 * </ul></p>
 *
 * <p>如上的这种方式就叫多路复用 实现三剑客<ul>
 *     <li>select</li>
 *     <li>poll</li>
 *     <li>epoll</li>
 * </ul></p>
 * @since 2022/5/21
 * @author dingrui
 */
public class NIOModelSingle {

    private static final List<Socket> SOCKETS = new ArrayList<>();

    public static void main(String[] args) throws IOException {
        ServerSocketChannel channel = ServerSocketChannel.open();
        // 非阻塞模式
        channel.configureBlocking(false);
        ServerSocket server = channel.socket();
        server.bind(new InetSocketAddress(9090));
        while (true) {
            Socket socket = server.accept();
            if (Objects.isNull(socket)) continue;
            SOCKETS.add(socket);
            for (Socket s : SOCKETS) {
                InputStream in = s.getInputStream();
                BufferedReader reader = new BufferedReader(new InputStreamReader(in));
                String msg = reader.readLine();
                if(Objects.isNull(msg)) continue;
                // TODO: 2022/5/21 业务逻辑
            }
        }
    }
}
```

 

### 3 NIO多路复用版本

```java
package debug.io.model;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Set;

/**
 * <p>NIO非阻塞下 多路复用模型</p>
 *
 * @since 2022/5/21
 * @author dingrui
 */
public class NIOModelMultiple {

    private static final List<Socket> SOCKETS = new ArrayList<>();

    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();
        ServerSocketChannel channel = ServerSocketChannel.open();
        // 非阻塞模式
        channel.configureBlocking(false);
        ServerSocket server = channel.socket();
        server.bind(new InetSocketAddress(9090));
        channel.register(selector, SelectionKey.OP_ACCEPT);
        while (true) {
            selector.select();
            // 多路复用器
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> it = selectionKeys.iterator();
            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove();
                if (key.isAcceptable()) {
                    // TODO: 2022/5/21
                } else if (key.isReadable()) {
                    // TODO: 2022/5/21
                } else if (key.isWritable()) {
                    // TODO: 2022/5/21
                }
            }
        }
    }
}
```

两个版本的对比，有了多路复用的加持，同样的NIO模式，在应用层上的并发显而易见得到了质的提升

 

下面是我的推测，还没研究学习源码，留着以后填坑(todo)

我的理解，多路复用仅仅是一种os提供的一种减少系统调用的方式，想要真正优雅的使用，还需要对此封装一个实现，比如上面这个Selector，对于这样的实现其实就是提供给用户层一个多路复用器

对于os而言，多路复用就是一个调用实现，听说有3种

1. select
2. poll
3. epoll

 

## 三 多路复用

按下不表，还没学习，日后填坑再来记下笔记(todo)