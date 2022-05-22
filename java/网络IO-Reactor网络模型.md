reactor=多路复用+线程池

抽象出组件，符合单一职责设计模式

reactor组件负责亲请求接收和分发



## 1 单reactor单线程模型

特点

* 一个reactor负责接收连接和读写请求，分发他们
* 一个handler线程负责所有读写请求的处理



### 1.1 reactor组件

```java
package debug.io.model.reactor.singlereactorsinglethread;

import java.net.InetSocketAddress;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.util.Iterator;
import java.util.Set;

/**
 * <p>单reactor单线程</p>
 * <p>弊端<ul>
 *     <li>服务端只能同时处理一个客户端的请求</li>
 * </ul></p>
 * @since 2022/5/22
 * @author dingrui
 */
public class Reactor implements Runnable {

    // 服务端socket
    ServerSocketChannel serverSocketChannel;
    // 复用器
    Selector selector;

    public Reactor(int port) {
        try {
            // 创建 非阻塞 绑定 监听
            this.serverSocketChannel = ServerSocketChannel.open();
            this.serverSocketChannel.configureBlocking(false);
            this.serverSocketChannel.bind(new InetSocketAddress(port));
            // 注册复用器
            this.selector = Selector.open();
            // att用来处理连接请求
            this.serverSocketChannel.register(this.selector, SelectionKey.OP_ACCEPT, new Acceptor(this.serverSocketChannel, this.selector));
        } catch (Exception ignored) {
        }
    }

    /**
     * <p>一个reactor收到请求后 不管是连接请求还是读写请求 一股脑处理<ul>
     *     <li>如果请求是连接请求 处理的会很快 交给acceptor处理器</li>
     *     <li>如果请求是读写请求 可能会包含着比较重的业务逻辑 分发给一个个的handler单独处理</li>
     * </ul></p>
     */
    @Override
    public void run() {
        while (true) {
            try {
                // os kqueue 收到的都是连接请求
                this.selector.select();
                Set<SelectionKey> selectionKeys = this.selector.selectedKeys();
                Iterator<SelectionKey> it = selectionKeys.iterator();
                while (it.hasNext()) {
                    SelectionKey sk = it.next();
                    it.remove();
                    // 请求交给dispatch进行派发 此时进来的可能是连接请求 也有可能是读请求
                    this.dispatch(sk);
                }
            } catch (Exception ignored) {
            }
        }
    }

    private void dispatch(SelectionKey sk) {
        // 不管是连接请求还是读请求 他们两个处理器的顶层都是Runnable Acceptor收到的是连接请求 Handler收到的是读请求
        Runnable r = (Runnable) sk.attachment();
        r.run();
    }
}

```



### 1.2 acceptor组件

```java
package debug.io.model.reactor.singlereactorsinglethread;

import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;

/**
 * <p>用于处理{@link Reactor}中的连接请求</p>
 * @since 2022/5/22
 * @author dingrui
 */
public class Acceptor implements Runnable {

    ServerSocketChannel serverSocketChannel;
    Selector selector;

    public Acceptor(ServerSocketChannel serverSocketChannel, Selector selector) {
        this.serverSocketChannel = serverSocketChannel;
        this.selector = selector;
    }

    /**
     * <p>处理连接请求 进来的全部都是连接请求</p>
     */
    @Override
    public void run() {
        try {
            SocketChannel socketChannel = this.serverSocketChannel.accept();
            System.out.println("->服务端 来自" + socketChannel.getRemoteAddress()+"的连接请求");
            // 读写非阻塞 注册到复用器
            socketChannel.configureBlocking(false);
            socketChannel.register(this.selector, SelectionKey.OP_READ, new Handler(socketChannel));
        } catch (Exception ignored) {

        }
    }
}

```



### 1.3 handler组件

```java
package debug.io.model.reactor.singlereactorsinglethread;

import java.net.SocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;

/**
 * <p>接收{@link Reactor}的读写请求</p>
 * @since 2022/5/22
 * @author dingrui
 */
public class Handler implements Runnable {

    SocketChannel socketChannel;

    public Handler(SocketChannel socketChannel) {
        this.socketChannel = socketChannel;
    }

    /**
     * <p>处理读写请求</p>
     */
    @Override
    public void run() {
        ByteBuffer b = ByteBuffer.allocate(1024);
        try {
            // 客户端信息
            SocketAddress remoteAddress = this.socketChannel.getRemoteAddress();
            this.socketChannel.read(b);
            // 收到了读请求的内容
            String msg = new String(b.array(), StandardCharsets.UTF_8);
            // TODO: 2022/5/22 业务处理
            System.out.println("->服务端 来自" + remoteAddress + "的一条消息=" + msg);
            // 回写数据给客户端
            String echo = "->客户端 服务端已经收到了你的信息=" + msg;
            this.socketChannel.write(ByteBuffer.wrap(echo.getBytes(StandardCharsets.UTF_8)));
        } catch (Exception ignored) {
        }
    }
}
```



### 1.4 服务端api

```java
package debug.io.model.reactor.singlereactorsinglethread;

import debug.io.model.reactor.ClientTest;

/**
 *
 * @since 2022/5/22
 * @author dingrui
 */
public class SingleReactorSingleThreadTest {

    public static void main(String[] args) {
        Reactor reactor = new Reactor(ClientTest.PORT);
        reactor.run();
    }
}
```



## 2 单reactor多线程模型

特点

* 一个reactor负责接收连接和读写请求，分发他们
* handler组件为每个读写请求分配单独的线程



### 2.1 reactor组件

```java
package debug.io.model.reactor.singlereactormultiplethread;

import java.net.InetSocketAddress;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.util.Iterator;

/**
 * <p>单reactor多线程</p>
 * <p>相较于{@link debug.io.model.reactor.singlereactorsinglethread.Reactor}版本 仅仅是在处理读写请求的时候加了个线程池</p>
 * @since 2022/5/22
 * @author dingrui
 */
public class Reactor implements Runnable {

    ServerSocketChannel serverSocketChannel;
    Selector selector;

    public Reactor(int port) {
        try {
            this.serverSocketChannel = ServerSocketChannel.open();
            this.serverSocketChannel.bind(new InetSocketAddress(port));
            this.serverSocketChannel.configureBlocking(false);

            this.selector = Selector.open();
            this.serverSocketChannel.register(this.selector, SelectionKey.OP_ACCEPT, new Acceptor(this.serverSocketChannel, this.selector));
        } catch (Exception ignored) {
        }
    }

    @Override
    public void run() {
        while (true) {
            try {
                this.selector.select();
                Iterator<SelectionKey> it = this.selector.selectedKeys().iterator();
                while (it.hasNext()) {
                    SelectionKey sk = it.next();
                    it.remove();
                    this.dispatch(sk);
                }
            } catch (Exception ignored) {
            }
        }
    }

    private void dispatch(SelectionKey sk) {
        Runnable r = (Runnable) sk.attachment();
        r.run();
    }
}

```



### 2.2 acceptor组件

```java
package debug.io.model.reactor.singlereactormultiplethread;

import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;

/**
 *
 * @since 2022/5/22
 * @author dingrui
 */
public class Acceptor implements Runnable {

    ServerSocketChannel ss;
    Selector sl;

    public Acceptor(ServerSocketChannel ss, Selector sl) {
        this.ss = ss;
        this.sl = sl;
    }

    @Override
    public void run() {
        try {
            SocketChannel s = this.ss.accept();
            s.configureBlocking(false);
            s.register(this.sl, SelectionKey.OP_READ, new Handler(s));
        } catch (Exception ignored) {
        }
    }
}

```



### 2.3 handler组件

```java
package debug.io.model.reactor.singlereactormultiplethread;

import java.net.SocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 *
 * @since 2022/5/22
 * @author dingrui
 */
public class Handler implements Runnable {

    private static final ExecutorService EXECUTOR = Executors.newFixedThreadPool(5);

    SocketChannel s;

    public Handler(SocketChannel s) {
        this.s = s;
    }

    @Override
    public void run() {
        try {
            ByteBuffer b = ByteBuffer.allocate(1024);
            this.s.read(b);
            EXECUTOR.execute(new Process(this.s, b));
        } catch (Exception ignored) {
        }
    }

    private static class Process implements Runnable {

        SocketChannel s;
        ByteBuffer b;

        public Process(SocketChannel s, ByteBuffer b) {
            this.s = s;
            this.b = b;
        }

        @Override
        public void run() {
            try {
                SocketAddress remoteAddress = this.s.getRemoteAddress();
                String msg = new String(b.array(), StandardCharsets.UTF_8);
                System.out.println("->服务端 收到来自客户端的消息=" + msg);
                String echo = "->客户端 服务端处理业务的线程=" + Thread.currentThread().getName();
                this.s.write(ByteBuffer.wrap(echo.getBytes(StandardCharsets.UTF_8)));
            } catch (Exception ignored) {
            }
        }
    }
}

```



### 2.4 服务端api

```java
package debug.io.model.reactor.singlereactormultiplethread;

import debug.io.model.reactor.ClientTest;

/**
 *
 * @since 2022/5/22
 * @author dingrui
 */
public class singleReactorMultipleThreadTest {

    public static void main(String[] args) {
        Reactor reactor = new Reactor(ClientTest.PORT);
        reactor.run();
    }
}

```





## 3 主从reactor模型

特点

* 多个reactor负责请求的接收和派发
  * main 主负责连接请求的接收和派发给acceptor
  * sub 负责读写请求的接收和派发给handler 可以有一个或多个sub reactor



### 3.1 reactor组件

#### 3.1.1 main

```java
package debug.io.model.reactor.multiplereactormultiplethread;

import java.net.InetSocketAddress;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.util.Iterator;

/**
 *
 * @since 2022/5/22
 * @author dingrui
 */
public class MainReactor implements Runnable {

    ServerSocketChannel ss;
    Selector sl;

    public MainReactor(int port) {
        try {
            this.ss = ServerSocketChannel.open();
            this.sl = Selector.open();
            this.ss.configureBlocking(false);
            this.ss.bind(new InetSocketAddress(port));
            this.ss.register(this.sl, SelectionKey.OP_ACCEPT, new Acceptor(this.ss));
        } catch (Exception ignored) {
        }
    }

    @Override
    public void run() {
        while (true) {
            try {
                this.sl.select();
                Iterator<SelectionKey> it = this.sl.selectedKeys().iterator();
                while (it.hasNext()) {
                    SelectionKey sk = it.next();
                    it.remove();
                    this.dispatch(sk);
                }
            } catch (Exception ignored) {
            }
        }
    }

    private void dispatch(SelectionKey sk) {
        Runnable r = (Runnable) sk.attachment();
        r.run();
    }
}

```



#### 3.1.2 sub

```java
package debug.io.model.reactor.multiplereactormultiplethread;

import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.util.Iterator;

/**
 *
 * @since 2022/5/22
 * @author dingrui
 */
public class SubReactor implements Runnable {

    // sub编号
    int id;
    Selector sl;

    public SubReactor(int id, Selector sl) {
        this.id = id;
        this.sl = sl;
    }

    @Override
    public void run() {
        while (true) {
            try {
                this.sl.select();
                Iterator<SelectionKey> it = this.sl.selectedKeys().iterator();
                while (it.hasNext()) {
                    SelectionKey sk = it.next();
                    it.remove();
                    this.dispatch(sk);
                }
            } catch (Exception ignored) {
            }
        }
    }

    private void dispatch(SelectionKey sk) {
        Runnable r = (Runnable) sk.attachment();
        r.run();
    }
}

```



### 3.2 acceptor组件

```java
package debug.io.model.reactor.multiplereactormultiplethread;

import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;

/**
 *
 * @since 2022/5/22
 * @author dingrui
 */
public class Acceptor implements Runnable {

    ServerSocketChannel ss;

    private int idx;

    private final int SUB_REACTOR_CNT = 5;
    private SubReactor[] subReactors = new SubReactor[SUB_REACTOR_CNT];
    private Thread[] threads = new Thread[SUB_REACTOR_CNT];
    private final Selector[] selectors = new Selector[SUB_REACTOR_CNT];

    public Acceptor(ServerSocketChannel ss) {
        this.ss = ss;

        for (int i = 0; i < SUB_REACTOR_CNT; i++) {
            try {
                this.selectors[i] = Selector.open();
            } catch (Exception e) {
                e.printStackTrace();
            }
            this.subReactors[i] = new SubReactor(i, this.selectors[i]);
            this.threads[i] = new Thread(this.subReactors[i]);
            this.threads[i].start();
        }
    }

    /**
     * mainReactor将连接请求都分发到acceptor中
     */
    @Override
    public void run() {
        try {
            SocketChannel s = this.ss.accept();
            s.configureBlocking(false);
            this.selectors[this.idx].wakeup();
            // 让subReactor中携带的复用器关注读请求
            s.register(this.selectors[this.idx], SelectionKey.OP_READ, new Handler(s));
            if (++this.idx == SUB_REACTOR_CNT) this.idx = 0;
        } catch (Exception ignored) {
        }
    }
}

```



### 3.3 handler组件

```java
package debug.io.model.reactor.multiplereactormultiplethread;

import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 *
 * @since 2022/5/22
 * @author dingrui
 */
public class Handler implements Runnable {

    SocketChannel s;

    private static final ExecutorService EXECUTOR = Executors.newFixedThreadPool(3);

    public Handler(SocketChannel s) {
        this.s = s;
    }

    @Override
    public void run() {
        try {
            // 读
            ByteBuffer b = ByteBuffer.allocate(1024);
            this.s.read(b);
            // 封装任务提交线程池
            EXECUTOR.execute(new Process(this.s, b));
        } catch (Exception ignored) {
        }
    }

    private static class Process implements Runnable {
        SocketChannel s;
        ByteBuffer b;

        public Process(SocketChannel s, ByteBuffer b) {
            this.s = s;
            this.b = b;
        }

        @Override
        public void run() {
            try {
                String msg = new String(this.b.array(), StandardCharsets.UTF_8);
                System.out.println("handler-> 收到来自" + this.s.getRemoteAddress() + "的消息=" + msg);
                // 写
                this.s.write(ByteBuffer.wrap("handler-> 来自handler的回写".getBytes(StandardCharsets.UTF_8)));
            } catch (Exception ignored) {
            }
        }
    }
}

```



### 3.4 服务端api

```java
package debug.io.model.reactor.multiplereactormultiplethread;

import debug.io.model.reactor.ClientTest;

/**
 *
 * @since 2022/5/22
 * @author dingrui
 */
public class MultipleReactorMultipleThreadTest {

    public static void main(String[] args) {
        MainReactor mainReactor = new MainReactor(ClientTest.PORT);
        mainReactor.run();
    }
}

```

