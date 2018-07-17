---
title: java aio socket 分析
date: 2018-01-29 17:05:57
tags: [java,aio,socket]
categories: [java]
---

# 说明

本文通过java源代码来分析java aio socket实现原理


<!--more-->

# java aio socket 源代码解析
先上demo，sever端代码如下
```java
/**
 * 接收客户端发送数据，返回
 */
public class AIOEchoServer {
    private final int port;

    private AsynchronousServerSocketChannel server = null;

    public AIOEchoServer(int port) {
        this.port = port;
    }

    public void listen() throws IOException {
        server = AsynchronousServerSocketChannel.open().bind(new InetSocketAddress(port));
        //accept客户端来的连接，生成一个channel，异步回调CompletionHandler
        server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>() {
            @Override
            public void completed(AsynchronousSocketChannel channel, Object attachment) {
                //这里得到一个channel，在这个channel上读取数据
                ByteBuffer readBuff = ByteBuffer.allocate(1024);
                try {
                    //如果没有数据可以读取，这个方法会阻塞
                    channel.read(readBuff, readBuff, new CompletionHandler<Integer, ByteBuffer>() {
                        @Override
                        public void completed(Integer result, ByteBuffer readBuff) {
                            readBuff.flip();
                            if (readBuff.remaining() > 0) {
                                String value = new String(readBuff.array(), 0, readBuff.limit());
                                System.out.println("receive from client: " + value);
                                try {
                                    value = value + " from server.";
                                    System.out.println("write to client: " + value);
                                    //写回客户端，用同步的方式，也可以用异步方式。如果客户端数据一次没有读取完，往channel里面写会抛异常，需要粘包
                                    channel.write(ByteBuffer.wrap((value).getBytes())).get();
                                } catch (Exception e) {
                                    e.printStackTrace();
                                }
                                readBuff.clear();
                                channel.read(readBuff, readBuff, this);
                            }
                        }

                        @Override
                        public void failed(Throwable exc, ByteBuffer attachment) {

                        }
                    });
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    //继续接收其他的客户端连接，如果不加这一行，只能连接一个客户端
                    server.accept(null, this);
                }
            }

            @Override
            public void failed(Throwable exc, Object attachment) {

            }
        });
        System.out.println("Server listen on " + port);
    }

    public static void main(String[] args) throws Exception {
        new AIOEchoServer(8001).listen();
        while (true) {
            try {
                Thread.sleep(100000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

}

```

再看client端：
```java

/**
 * 向服务端发送一条消息，从服务端读取回执
 */
public class AIOClient {
    public static void main(String[] args) throws Exception {
        final AsynchronousSocketChannel clientChannel = AsynchronousSocketChannel.open();
        InetSocketAddress serverAddress = new InetSocketAddress("127.0.0.1", 8001);
        clientChannel.connect(serverAddress).get(1, TimeUnit.SECONDS);

        for (int i = 0; i < 10; i++) {
            ByteBuffer writeBuffer = ByteBuffer.wrap(("Hello " + i).getBytes());
            //这里使用一个CountDownLatch，保证从服务端读取回执后再发送下一条，同时使用一个通道发送，会抛异常
            CountDownLatch countDownLatch = new CountDownLatch(1);
            clientChannel.write(writeBuffer, writeBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                @Override
                public void completed(Integer result, ByteBuffer writeBuffer) {
                    writeBuffer.flip();
                    System.out.println("write success " + new String(writeBuffer.array(), 0, writeBuffer.limit()));
                    ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                    //从服务端读取返回，最多等10秒
                    clientChannel.read(readBuffer, 2, TimeUnit.SECONDS, readBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                        @Override
                        public void completed(Integer result, ByteBuffer readBuffer) {
                            readBuffer.flip();
                            System.out.println("receive: " + new String(readBuffer.array(), 0, readBuffer.limit()));
                            readBuffer.clear();
                            //读完了才countDown，如果服务端发送数据大于缓冲区大小，这里是处理不了的，还需要进行包的拆分和粘连
                            countDownLatch.countDown();
                        }

                        @Override
                        public void failed(Throwable exc, ByteBuffer readBuffer) {
                            //失败了要countDown
                            countDownLatch.countDown();
                        }
                    });
                }

                @Override
                public void failed(Throwable exc, ByteBuffer attachment) {
                    //失败了要countDown
                    countDownLatch.countDown();
                }
            });
            countDownLatch.await();
        }
        clientChannel.close();
        System.out.println("channel close success!");
    }

}
```
相对于NIO，服务端代码没有Selector，SelectionKey，不需要自己去循环，但是异步方式有些地方比较难以理解。客户端代码像javascript的ajax调用。

# windows上实现

## 连接建立
### 先看channel的创建
```
AsynchronousServerSocketChannel.open()
//往下跟
provider.openAsynchronousServerSocketChannel(group);

//windows上，创建一个WindowsAsynchronousServerSocketChannelImpl
    @Override
    public AsynchronousServerSocketChannel openAsynchronousServerSocketChannel(AsynchronousChannelGroup group)
        throws IOException
    {
        return new WindowsAsynchronousServerSocketChannelImpl(toIocp(group));
    }
```
我们来看看WindowsAsynchronousServerSocketChannelImpl，是一个异步ServerSockethannelde的实现，包含几个重要属性
- FileDescriptor fd 文件描述符
- InetSocketAddress localAddress 本地地址
- long handle
- int completionKey
- Iocp iocp 一个具体的异步通道组实现
- PendingIoCache ioCache
- long dataBuffer
- AtomicBoolean accepting
异步通道组的作用是为一组通道提供资源共享，比如异步运行时共享线程池。

### 然后是bind本地地址

```
        try {
            begin();
            synchronized (stateLock) {
                if (localAddress != null)
                    throw new AlreadyBoundException();
                NetHooks.beforeTcpBind(fd, isa.getAddress(), isa.getPort());
                Net.bind(fd, isa.getAddress(), isa.getPort());
                Net.listen(fd, backlog < 1 ? 50 : backlog);
                localAddress = Net.localAddress(fd);
            }
        } finally {
            end();
        }
```
最后调用的是本地方法，将fd绑定到端口上
```
    public static void bind(FileDescriptor fd, InetAddress addr, int port)
        throws IOException
    {
        bind(UNSPEC, fd, addr, port);
    }

    static void bind(ProtocolFamily family, FileDescriptor fd,
                     InetAddress addr, int port) throws IOException
    {
        boolean preferIPv6 = isIPv6Available() &&
            (family != StandardProtocolFamily.INET);
        bind0(fd, preferIPv6, exclusiveBind, addr, port);
    }

    private static native void bind0(FileDescriptor fd, boolean preferIPv6,
                                     boolean useExclBind, InetAddress addr,
                                     int port)
        throws IOException;
```
### 接下来是AsynchronousServerSocketChannel.accept
```
public abstract <A> void accept(A attachment,CompletionHandler<AsynchronousSocketChannel,? super A> handler);
```
accept是一个抽象方法，包含两个参数：
- attachment 附加对象，跟nio中注册到Selector上的附加对象一样
- handler accept成功后回调处理逻辑

accept实现方法：

```
//AsynchronousServerSocketChannelImpl类
    public final <A> void accept(A attachment,  CompletionHandler<AsynchronousSocketChannel,? super A> handler) {
        if (handler == null)
            throw new NullPointerException("'handler' is null");
        implAccept(attachment, (CompletionHandler<AsynchronousSocketChannel,Object>)handler);
    }

//WindowsAsynchronousServerSocketChannelImpl类
    @Override
    Future<AsynchronousSocketChannel> implAccept(Object attachment, final CompletionHandler<AsynchronousSocketChannel,Object> handler) {
。。。
//创建一个被accecped的socket
        WindowsAsynchronousSocketChannelImpl ch = null;
        IOException ioe = null;
        try {
            begin();
            ch = new WindowsAsynchronousSocketChannelImpl(iocp, false);
        } catch (IOException x) {
            ioe = x;
        } finally {
            end();
        }
//出现异常，直接调用handle
        if (ioe != null) {
            if (handler == null)
                return CompletedFuture.withFailure(ioe);
            Invoker.invokeIndirectly(this, handler, attachment, null, ioe);
            return null;
        }
//通过getSecurityManager检查
        AccessControlContext acc = (System.getSecurityManager() == null) ? null : AccessController.getContext();
//PendingFuture实现Future接口，存放this，handler和attachment
        PendingFuture<AsynchronousSocketChannel,Object> result = new PendingFuture<AsynchronousSocketChannel,Object>(this, handler, attachment);
//AcceptTask实现Runnable接口，
        AcceptTask task = new AcceptTask(ch, acc, result);
        result.setContext(task);
//并发控制
        if (!accepting.compareAndSet(false, true))
            throw new AcceptPendingException();

// 检查操作系统是否支持线程无关的I/O，什么意思？
        if (Iocp.supportsThreadAgnosticIo()) {
//我的windows上跑这里
            task.run();
        } else {
            Invoker.invokeOnThreadInThreadPool(this, task);
        }
        return result;
    }
```
AcceptTask的run方法如下
```
        @Override
        public void run() {
            long overlapped = 0L;
。。。
                    synchronized (result) {
                        overlapped = ioCache.add(result);
            //重点是这行，调用native方法，windows上直接返回
                        int n = accept0(handle, channel.handle(), overlapped, dataBuffer);
                        if (n == IOStatus.UNAVAILABLE) {
                            return;
                        }

                        finishAccept();

                        enableAccept();
                //将接收到的channel设置到result中，Future的get方法阻塞就会唤醒
                        result.setResult(channel);
                    }
。。。
            if (result.isCancelled()) {
                closeChildChannel();
            }
            //最后调用handler，这里Invoker是一个帮助类，实际上是根据是否有异常调用CompletionHandler的completed/failed方法
            Invoker.invokeIndirectly(result);
        }
```
调试到这里，accept方法实际上已经返回了，那么，当客户端有连接过来的时候，是如何调用回调函数的呢？我们来看看accept0的native方法，打开WindowsAsynchronousServerSocketChannelImpl.c
```

JNIEXPORT jint JNICALL
Java_sun_nio_ch_WindowsAsynchronousServerSocketChannelImpl_accept0(JNIEnv* env, jclass this,
    jlong listenSocket, jlong acceptSocket, jlong ov, jlong buf)
{
    BOOL res;
    SOCKET s1 = (SOCKET)jlong_to_ptr(listenSocket);
    SOCKET s2 = (SOCKET)jlong_to_ptr(acceptSocket);
    PVOID outputBuffer = (PVOID)jlong_to_ptr(buf);

    DWORD nread = 0;
    OVERLAPPED* lpOverlapped = (OVERLAPPED*)jlong_to_ptr(ov);
    ZeroMemory((PVOID)lpOverlapped, sizeof(OVERLAPPED));
 
    
    
       
        
    
    
    
    
    res = (*AcceptEx_func)(s1,  //一参本地监听Socket 
                           s2, //二参为即将到来的客人准备好的Socket 
                           outputBuffer,//三参接收缓冲区：   一存客人发来的第一份数据、二存Server本地地址、三存Client远端地址 地址包括IP和端口， 
                           0, //四参定三参数据区长度，0表只连不接收、连接到来->请求完成，否则连接到来+任意长数据到来->请求完成 
                           sizeof(SOCKETADDRESS)+16,//五参定三参本地地址区长度，至少sizeof(sockaddr_in) + 16 
                           sizeof(SOCKETADDRESS)+16, //六参定三参远端地址区长度，至少sizeof(sockaddr_in) + 16 
                           &nread,//标识接收到的字节数
                           lpOverlapped);//一个OVERLAPPED结构，传递数据。
    if (res == 0) {
        int error = WSAGetLastError();
        if (error == ERROR_IO_PENDING) {
            return IOS_UNAVAILABLE;
        }
        JNU_ThrowIOExceptionWithLastError(env, "AcceptEx failed");
        return IOS_THROWN;
    }

    return 0;
}
```
看不太明白。。。在看看前面的代码，C语言很多都不记得了，复习了下，搞了个大概
```
//定义一个函数指针，后面会指向windows上的函数AcceptEx
typedef BOOL (*AcceptEx_t)
(
    SOCKET sListenSocket,
    SOCKET sAcceptSocket,
    PVOID lpOutputBuffer,
    DWORD dwReceiveDataLength,
    DWORD dwLocalAddressLength,
    DWORD dwRemoteAddressLength,
    LPDWORD lpdwBytesReceived,
    LPOVERLAPPED lpOverlapped
);

//换个名字
static AcceptEx_t AcceptEx_func;

//native的initIDs方法
JNIEXPORT void JNICALL
Java_sun_nio_ch_WindowsAsynchronousServerSocketChannelImpl_initIDs(JNIEnv* env, jclass this) {
    GUID GuidAcceptEx = WSAID_ACCEPTEX;
    SOCKET s;
    int rv;
    DWORD dwBytes;

    s = socket(AF_INET, SOCK_STREAM, 0);
    if (s == INVALID_SOCKET) {
        JNU_ThrowIOExceptionWithLastError(env, "socket failed");
        return;
    }
    //windows的WSAIoctl函数
    rv = WSAIoctl(s,
                  SIO_GET_EXTENSION_FUNCTION_POINTER,
                  (LPVOID)&GuidAcceptEx,
                  sizeof(GuidAcceptEx),
                  &AcceptEx_func,//给函数指针赋值
                  sizeof(AcceptEx_func),
                  &dwBytes,
                  NULL,
                  NULL);
    if (rv != 0)
        JNU_ThrowIOExceptionWithLastError(env, "WSAIoctl failed");
    closesocket(s);
}
```
这里通过WSAIoctl获取AcceptEx函数指针，供其他地方调用。  
AcceptEx函数的解释：Windows套接字AcceptEx函数接受一个新的连接，返回本地和远程地址，并接收由客户端应用程序发送的第一块数据。关键点就在这个AcceptEx函数，调用这个函数，立即返回了，实际上没有创建连接，创建一个等待连接的数据结构体放在port指定的队列中，如果来了新的连接（客户端连接，触发事件），会将连接信息填写到结构体中，修改状态。这个连接准备好了，肯定有地方去消费这个连接，在哪里呢？  
我们回到AsynchronousServerSocketChannel.open()这个位置，这里打开了一个ServerSocket，内部使用了一个Iocp，前面我们没有解释这个Iocp是什么意思，查阅资料发现，这是windows网路编程提出的一个模型，（I/O Completion Port）,常称I/O完成端口。要深入这个模型，又会比较复杂，我们这里先简单的理解下，AcceptEx被调用时，会在serverSocket上注册一个回调，如果有客户端连上来了，操作系统会来处理。如何处理的呢？ AsynchronousServerSocketChannel.open()的时候，会创建一个iocp java对象，内部调用native方法createIoCompletionPort创建windos的 iocp，然后将与serverSocketChannel关联。

```

    Iocp(AsynchronousChannelProvider provider, ThreadPool pool)
        throws IOException
    {
        super(provider, pool);
        //创建Iocp
        this.port =
          createIoCompletionPort(INVALID_HANDLE_VALUE, 0, 0, fixedThreadCount());
        this.nextCompletionKey = 1;
    }


    /**
     * 与serverSocketChannel关联
     */
    int associate(OverlappedChannel ch, long handle) throws IOException {
        keyToChannelLock.writeLock().lock();

        // generate a completion key (if not shutdown)
        int key;
        try {
            if (isShutdown())
                throw new ShutdownChannelGroupException();

            // generate unique key
            do {
                key = nextCompletionKey++;
            } while ((key == 0) || keyToChannel.containsKey(key));

            // associate with I/O completion port
            if (handle != 0L) {
            //这里是关联，这里有个key，后面通知的时候，会用到这个key
                createIoCompletionPort(handle, port, key, 0);
            }

            // 将key与ch放在hashmao缓存，后面的事件来了，会返回这个key，通过这个key找到channel
            keyToChannel.put(key, ch);
        } finally {
            keyToChannelLock.writeLock().unlock();
        }
        return key;
    }
    
//Iocp.java native方法
    private static native void initIDs();

    private static native long createIoCompletionPort(long handle,
        long existingPort, int completionKey, int concurrency) throws IOException;

    private static native void close0(long handle);

    private static native void getQueuedCompletionStatus(long completionPort,
        CompletionStatus status) throws IOException;

    private static native void postQueuedCompletionStatus(long completionPort,
        int completionKey) throws IOException;

    private static native String getErrorMessage(int error);
```
然后启动一个线程，阻塞在getQueuedCompletionStatus方法处。
```

    Iocp start() {
        startThreads(new EventHandlerTask());
        return this;
    }
    
    private class EventHandlerTask implements Runnable {
        public void run() {
。。。
            try {
                for (;;) {
                    
                    try {
                    //这里会阻塞，有iopc完成时间会唤醒
                        getQueuedCompletionStatus(port, ioResult);
                    } catch (IOException x) {
                        // should not happen
                        x.printStackTrace();
                        return;
                    }
。。。
        }
    }
```
我们用客户端发起一个连接，阻塞的这行就会被唤醒，跳到下一行

```
                    // handle wakeup to execute task or shutdown
                    if (ioResult.completionKey() == 0 &&
                        ioResult.overlapped() == 0L)
                    {
                        Runnable task = pollTask();
                        if (task == null) {
                            // shutdown request
                            return;
                        }

                        // run task
                        // (if error/exception then replace thread)
                        replaceMe = true;
                        task.run();
                        continue;
                    }

                    // map key to channel
                    OverlappedChannel ch = null;
                    keyToChannelLock.readLock().lock();
                    try {
                    //这里根据completionKey找到channel，这个key就是前面我们关联iocp时传入的，现在有时间了，iocp给返回了。
                        ch = keyToChannel.get(ioResult.completionKey());
                        if (ch == null) {
                            checkIfStale(ioResult.overlapped());
                            continue;
                        }
                    } finally {
                        keyToChannelLock.readLock().unlock();
                    }

                    //根据overlapped找到关联的对象。这个overlapped是我们在accept0里面放进去的。
                    PendingFuture<?,?> result = ch.getByOverlapped(ioResult.overlapped());
                    if (result == null) {
                        // we get here if the OVERLAPPED structure is associated
                        // with an I/O operation on a channel that was closed
                        // but the I/O operation event wasn't read in a timely
                        // manner. Alternatively, it may be related to a
                        // tryLock operation as the OVERLAPPED structures for
                        // these operations are not in the I/O cache.
                        checkIfStale(ioResult.overlapped());
                        continue;
                    }

                    // synchronize on result in case I/O completed immediately
                    // and was handled by initiator
                    synchronized (result) {
                        if (result.isDone()) {
                            continue;
                        }
                        // not handled by initiator
                    }

                    // invoke I/O result handler
                    int error = ioResult.error();
                    //获取ResultHandler，实际上获取的是AcceptTask
                    ResultHandler， rh = (ResultHandler)result.getContext();
                    replaceMe = true; // (if error/exception then replace thread)
                    if (error == 0) {
                    //调用AcceptTask的complete方法，里面调用回调函数
                        rh.completed(ioResult.bytesTransferred(), canInvokeDirect);
                    } else {
                        rh.failed(error, translateErrorToIOException(error));
                    }
                }
```
根据以上代码中的注释，我们基本上搞清楚了，如何accept一个请求后如何被唤醒的。总结下，主要是两个key，一个是将serverSocketChannel注册到Iocp中的CompletionKey，另一个是在AccetpEx函数中注册的overlapped。在java内存中用hashmap缓存key与对象关系。然后我们再看看回调函数的执行：
```
        @Override
        public void completed(int bytesTransferred, boolean canInvokeDirect) {
。。。
                //result是一个Future对象，设置result，get的地方会唤醒返回。
                result.setResult(channel);
。。。
            // 调用回调的handler，Invoker是一个工具类，将调用进行了封装
            Invoker.invokeIndirectly(result);
        }
```
Invoker内部代码逻辑比较简单，不再这里描述了。 

到这里，整个accept的过程就分析完了。
---

## 异步读取数据
下面我们继续分析，是如何异步读取数据的。

### 回到demo中的accept的回调函数completed
```
            public void completed(AsynchronousSocketChannel channel, Object attachment) {
                //这里得到一个channel，在这个channel上读取数据
                ByteBuffer readBuff = ByteBuffer.allocate(1024);
                try {
                    //如果没有数据可以读取
                    channel.read(readBuff, readBuff, new CompletionHandler<Integer, ByteBuffer>() {
。。。。。。。
                    });
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    //继续接收其他的客户端连接，如果不加这一行，只能连接一个客户端
                    server.accept(null, this);
                }
            }
```
我们跟到channel.read内部：

```
//AsynchronousSocketChannel
    @Override
    public final <A> void read(ByteBuffer dst,
                               A attachment,
                               CompletionHandler<Integer,? super A> handler)
    {
        read(dst, 0L, TimeUnit.MILLISECONDS, attachment, handler);
    }
    
//AsynchronousSocketChannelImpl    
    @Override
    public final <A> void read(ByteBuffer dst,
                               long timeout,
                               TimeUnit unit,
                               A attachment,
                               CompletionHandler<Integer,? super A> handler)
    {
        if (handler == null)
            throw new NullPointerException("'handler' is null");
        if (dst.isReadOnly())
            throw new IllegalArgumentException("Read-only buffer");
        read(false, dst, null, timeout, unit, attachment, handler);
    }
    
    
    private <V extends Number,A> Future<V> read(boolean isScatteringRead,
                                                ByteBuffer dst,
                                                ByteBuffer[] dsts,
                                                long timeout,
                                                TimeUnit unit,
                                                A att,
                                                CompletionHandler<V,? super A> handler)
    {
。。。
        return implRead(isScatteringRead, dst, dsts, timeout, unit, att, handler);
    }
    
//WindowsAsynchronousSocketChannelImpl    
    @Override
    <V extends Number,A> Future<V> implRead(boolean isScatteringRead,
                                            ByteBuffer dst,
                                            ByteBuffer[] dsts,
                                            long timeout,
                                            TimeUnit unit,
                                            A attachment,
                                            CompletionHandler<V,? super A> handler)
    {
        // setup task
        PendingFuture<V,A> result =
            new PendingFuture<V,A>(this, handler, attachment);
        ByteBuffer[] bufs;
        if (isScatteringRead) {
            bufs = dsts;
        } else {
            bufs = new ByteBuffer[1];
            bufs[0] = dst;
        }
        //新建一个ReadTask
        final ReadTask<V,A> readTask =
                new ReadTask<V,A>(bufs, isScatteringRead, result);
        result.setContext(readTask);

        // schedule timeout
        if (timeout > 0L) {
            Future<?> timeoutTask = iocp.schedule(new Runnable() {
                public void run() {
                    readTask.timeout();
                }
            }, timeout, unit);
            result.setTimeoutTask(timeoutTask);
        }

        // initiate I/O
        if (Iocp.supportsThreadAgnosticIo()) {
        //这里执行这个readTask
            readTask.run();
        } else {
            Invoker.invokeOnThreadInThreadPool(this, readTask);
        }
        return result;
    }
    
//ReadTask执行    
        @Override
        @SuppressWarnings("unchecked")
        public void run() {
            long overlapped = 0L;
            boolean prepared = false;
            boolean pending = false;

            try {
                begin();

                // substitute non-direct buffers
                prepareBuffers();
                prepared = true;

                // 这里又是一个overlapped
                overlapped = ioCache.add(result);

                // 调用native方法执行read，跟前面accept类似，这个方法不阻塞
                int n = read0(handle, numBufs, readBufferArray, overlapped);
                if (n == IOStatus.UNAVAILABLE) {
                    // I/O is pending
                    pending = true;
                    return;
                }
                if (n == IOStatus.EOF) {
                    // input shutdown
                    enableReading();
                    if (scatteringRead) {
                        result.setResult((V)Long.valueOf(-1L));
                    } else {
                        result.setResult((V)Integer.valueOf(-1));
                    }
                } else {
                    throw new InternalError("Read completed immediately");
                }
            } catch (Throwable x) {
                // failed to initiate read
                // reset read flag before releasing waiters
                enableReading();
                if (x instanceof ClosedChannelException)
                    x = new AsynchronousCloseException();
                if (!(x instanceof IOException))
                    x = new IOException(x);
                result.setFailure(x);
            } finally {
                // release resources if I/O not pending
                if (!pending) {
                    if (overlapped != 0L)
                        ioCache.remove(overlapped);
                    if (prepared)
                        releaseBuffers();
                }
                end();
            }

            // invoke completion handler
            Invoker.invoke(result);
        }
```
跟accept的过程很类似，同样是通过iocp模型实现异步的，我们看看native方法read0的实现：
```
JNIEXPORT jint JNICALL
Java_sun_nio_ch_WindowsAsynchronousSocketChannelImpl_read0(JNIEnv* env, jclass this,
    jlong socket, jint count, jlong address, jlong ov)
{
    SOCKET s = (SOCKET) jlong_to_ptr(socket);
    WSABUF* lpWsaBuf = (WSABUF*) jlong_to_ptr(address);
    OVERLAPPED* lpOverlapped = (OVERLAPPED*) jlong_to_ptr(ov);
    BOOL res;
    DWORD flags = 0;

    ZeroMemory((PVOID)lpOverlapped, sizeof(OVERLAPPED));
    res = WSARecv(s,
                  lpWsaBuf,
                  (DWORD)count,
                  NULL,
                  &flags,
                  lpOverlapped,
                  NULL);

    if (res == SOCKET_ERROR) {
        int error = WSAGetLastError();
        if (error == WSA_IO_PENDING) {
            return IOS_UNAVAILABLE;
        }
        if (error == WSAESHUTDOWN) {
            return IOS_EOF;       // input shutdown
        }
        JNU_ThrowIOExceptionWithLastError(env, "WSARecv failed");
        return IOS_THROWN;
    }
    return IOS_UNAVAILABLE;
}
```
这里是调用windows的WSARecv方法负责注册接收数据的实现方式。有数据过来，iocp的getQueuedCompletionStatus会被唤醒，得到ioResult对象，再根据CompletionKey和overlapped得到channel和PendingFuture，然后调用这个PendingFuture中的回调函数。

## wirte 异步
wirte异步就跟accept/read方式一样了，不再重述。


## 附加
根据我们以上分析，这里最核心的地方是iocp，其内部是有一个线程池，负责管理多个Channel，默认使用 ThreadPool.getDefault()得到这个线程池，也可以自定义这个线程池：
```
AsynchronousChannelGroup threadGroup = AsynchronousChannelGroup.withThreadPool((ExecutorService) getExecutor());
serverSock = AsynchronousServerSocketChannel.open(threadGroup);
```

# linux实现
打开linux源代码，找到DefaultAsynchronousChannelProvider.create，继续找到LinuxAsynchronousChannelProvider

```
    @Override
    public AsynchronousServerSocketChannel openAsynchronousServerSocketChannel(AsynchronousChannelGroup group)
        throws IOException
    {
        return new UnixAsynchronousServerSocketChannelImpl(toPort(group));
    }


    private Port toPort(AsynchronousChannelGroup group) throws IOException {
        if (group == null) {
            return defaultEventPort();
        } else {
            if (!(group instanceof EPollPort))
                throw new IllegalChannelGroupException();
            return (Port)group;
        }
    }
    
    private EPollPort defaultEventPort() throws IOException {
        if (defaultPort == null) {
            synchronized (LinuxAsynchronousChannelProvider.class) {
                if (defaultPort == null) {
                    defaultPort = new EPollPort(this, ThreadPool.getDefault()).start();
                }
            }
        }
        return defaultPort;
    }
```
可以看到这里使用的是一个EPollPort，跟windows上的Iocp类似：

```
final class EPollPort extends Port{。。。}
abstract class Port extends AsynchronousChannelGroupImpl{。。。}
```
EPollPort实际上也是一个AsynchronousChannelGroup。我们先看看UnixAsynchronousServerSocketChannelImpl

```
    @Override
    Future<AsynchronousSocketChannel> implAccept(Object att,
        CompletionHandler<AsynchronousSocketChannel,Object> handler)
    {
。。。
        try {
            begin();
            //这里直接调用accept方法
            int n = accept(this.fd, newfd, isaa);
```
调用的是native方法：

```

JNIEXPORT jint JNICALL
Java_sun_nio_ch_UnixAsynchronousServerSocketChannelImpl_accept0(JNIEnv* env,
    jobject this, jobject ssfdo, jobject newfdo, jobjectArray isaa)
{
    return Java_sun_nio_ch_ServerSocketChannelImpl_accept0(env, this,
        ssfdo, newfdo, isaa);
}
```
找到Java_sun_nio_ch_ServerSocketChannelImpl_accept0在ServerSocketChannelImpl.c中：

```

JNIEXPORT jint JNICALL
Java_sun_nio_ch_ServerSocketChannelImpl_accept0(JNIEnv *env, jobject this,
                                                jobject ssfdo, jobject newfdo,
                                                jobjectArray isaa)
{
    jint ssfd = (*env)->GetIntField(env, ssfdo, fd_fdID);
    jint newfd;
    struct sockaddr *sa;
    int alloc_len;
    jobject remote_ia = 0;
    jobject isa;
    jint remote_port;

    NET_AllocSockaddr(&sa, &alloc_len);
    if (sa == NULL) {
        JNU_ThrowOutOfMemoryError(env, NULL);
        return IOS_THROWN;
    }

    /*
     * accept connection but ignore ECONNABORTED indicating that
     * a connection was eagerly accepted but was reset before
     * accept() was called.
     */
    for (;;) {
        socklen_t sa_len = alloc_len;
        newfd = accept(ssfd, sa, &sa_len);
        if (newfd >= 0) {
            break;
        }
        if (errno != ECONNABORTED) {
            break;
        }
        /* ECONNABORTED => restart accept */
    }

    if (newfd < 0) {
        free((void *)sa);
        if (errno == EAGAIN)
            return IOS_UNAVAILABLE;
        if (errno == EINTR)
            return IOS_INTERRUPTED;
        JNU_ThrowIOExceptionWithLastError(env, "Accept failed");
        return IOS_THROWN;
    }

    (*env)->SetIntField(env, newfdo, fd_fdID, newfd);
    remote_ia = NET_SockaddrToInetAddress(env, sa, (int *)&remote_port);
    free((void *)sa);
    CHECK_NULL_RETURN(remote_ia, IOS_THROWN);
    isa = (*env)->NewObject(env, isa_class, isa_ctorID, remote_ia, remote_port);
    CHECK_NULL_RETURN(isa, IOS_THROWN);
    (*env)->SetObjectArrayElement(env, isaa, 0, isa);
    return 1;
}
```
这里有个循环，一直accept，accept到了才跳出去。linux环境没有搭建好，不好调试，先写到这里，下次有机会在linux中调试。




# 总结
- java异步，不高清楚里面的原理，很容易出错。我在编写本文demo时，就是没有对原理搞清楚，遇到好多问题。  
- api看起来很简单，如果完全用异步，那就不简单了。比如客户端发送数据，我要使用一个CountDown才能发下一条，否则容易报错，因为底层一个channel不能同时发多份数据
