### Reactor模式

#### 定义

reactor模式也称作反应器模式，服务处理程序将一个或多个请求通过多路复用的方式分发给相应的请求处理程序。

#### 关键组件

- Reactor：负责监听和分发IO事件到处理器中
- Handlers：处理Reactor分发过来的请求事件

#### IO处理版本1.0

```
while(true){

    socket = accept();

    handle(socket)

}
```

在循环中处理所有的事件。accept和handle都会阻塞，只有处理完连接后才能接收下一个连接，处理效率太低。

既然处理请求时会阻塞，自然而然的想到了多线程。既然处理阶段阻塞了，将就让其在新线程中处理请求。这样服务端仍旧可以接收新的连接。

#### IO处理版本2.0

```
class BasicModel implements Runnable {
    public void run() {
        try {
            ServerSocket serverSocket = new ServerSocket(8888);
            while (!Thread.interrupted()) {
                //创建新线程来handle
                // or, single-threaded, or a thread pool
                new Thread(new Handler(serverSocket.accept())).start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    static class Handler implements Runnable {
        
        private Socket socket;
        
        public Handler(Socket socket) {
            this.socket = socket; 
        }
        
        public void run() {
            try {
                byte[] input = new byte[1024];
                socket.getInputStream().read(input);
                byte[] output = process(input);
                socket.getOutputStream().write(output);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        
        private byte[] process(byte[] input) {
            byte[] output=null;
            
            //业务逻辑处理
            
            return output;
        }
    }
}
```

经典的处理IO的做法：per connection per thread。

缺点：在连接数少的情况下可以满足实际需求。当连接数多的请求下（C10k问题）这意味着需要大量的线程被创建处来处理连接，而创建线程和销毁线程都需要消耗大量的系统资源，容易引发系统崩溃的风险。

既然大量的请求需要在线程中处理，而线程的数量又受到系统资源的限制，然后产生了一个想法：一个线程处理多个请求连接。

然后就诞生了基于事件驱动，可以在在单个线程中处理多个请求的Reactor模式。

#### IO处理版本3.0

Reactor模式主要有三种：

- 单Reactor单线程
- 单Reactor多线程
- 主从Reactor多线程

##### 单Reactor单线程

![1638697743769](D:\学习\redis学习\assets\1638697743769.png)



```java
public class Handler implements Runnable {
    
    private SocketChannel channel;

    private SelectionKey selectionKey;
    
    ByteBuffer input = ByteBuffer.allocate(1024);
    ByteBuffer output = ByteBuffer.allocate(1024);
    static final int READING = 0, SENDING = 1;
    int state = READING;

    public Handler(Selector selector, SocketChannel c) throws IOException {
        this.channel = c;
        c.configureBlocking(false);
        // Optionally try first read now
        this.selectionKey = channel.register(selector, 0);

        //将Handler作为callback对象
        this.selectionKey.attach(this);

        //第二步,注册Read就绪事件
        this.selectionKey.interestOps(SelectionKey.OP_READ);
        selector.wakeup();
    }

    private boolean inputIsComplete() {
        /* ... */
        return false;
    }

    private boolean outputIsComplete() {
        /* ... */
        return false;
    }

    private void process() {
        /* ... */
        return;
    }

    public void run() {
        try {
            if (state == READING) {
                read();
            } else if (state == SENDING) {
                send();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void read() throws IOException {
        channel.read(input);
        if (inputIsComplete()) {
            process();

            state = SENDING;
            // Normally also do first write now
            //第三步,接收write就绪事件
            selectionKey.interestOps(SelectionKey.OP_WRITE);
        }
    }

    private void send() throws IOException {
        channel.write(output);

        //write完就结束了, 关闭select key
        if (outputIsComplete()) {
            selectionKey.cancel();
        }
    }
}
```

优点：

- accept、write和read事件都在一个线程中处理。模型简单，没有多线程带来的安全问题

缺点：

- 无法利用多核cpu的优势
- 任意的事件处理被阻塞了造成其它的请求也不能处理

##### 单Reactor多线程

redis中使用的时单Reactor单线程，其它的等稍后研究NIO时再深入研究。

##### 主从Reactor多线程



#### 参考文献

- [Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf) 

- [Java NIO——Reactor模式](https://www.jianshu.com/p/2759a2374ed4)