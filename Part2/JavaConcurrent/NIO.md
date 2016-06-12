#NIO

---

Java NIO(New IO)是一个可以替代标准Java IO API的IO API(从Java1.4开始)，Java NIO提供了与标准IO不同的IO工作方式。

###Java NIO: Channels and Buffers（通道和缓冲区）

标准的俄IO基于字节流和字符流进行操作的，而NIO是基于通道(Channel)和缓冲区(Buffer)进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入通道也类似。

###Java NIO: Non-blocking IO（非阻塞IO）

Java NIO可以让你非阻塞的使用IO，例如：当线程从通道读取数据到缓冲区时，线程还是进行其他事情。当数据被写入到缓冲区时，线程可以继续处理它。从缓冲区写入通道也类似。

###Java NIO: Selectors(选择器)

Java NIO引入了选择器的概念，选择器用于监听多个通道的事件(比如：连接打开，数据到达)。因此，单个的线程可以监听多个数据通道。

NIO由以下核心部分组成：

* Channels
* Buffers
* Selectors

**Channel和Buffer**

基本上，所有的IO和NIO都从一个Channel开始。Channel有点像流。数据可以从Channel读到Buffer中，也可以从Buffer写到Channel中

![](http://ifeve.com/wp-content/uploads/2013/06/overview-channels-buffers1.png)

Channel的实现

* FileChannel
* DatagramChannel
* SocketChannel
* ServerSocketChannel

这些通道涵盖了UDP和TCP网络IO，以及文件IO。

以下是Java NIO里关键的Buffer实现

* ByteBuffer
* CharBuffer
* DoubleBuffer
* FloatBuffer
* IntBuffer
* LongBuffer
* ShortBuffer

这些Buffer覆盖了你能通过IO发送的基本数据类型：byte,short,int,long,float,double和char

Java NIO还有个MappedByteBuffer，用于表示内存映射文件。

**Selextor**

Selector允许单线程处理多个Channel。如果你的应用打开了多个连接(通道)，但每一个连接的流量都很低，使用Selector就会很方便。

例如，在一个聊天服务器中

这是在一个单线程中使用一个Selector处理3个Channel的图示：

![](http://ifeve.com/wp-content/uploads/2013/06/overview-selectors.png)

要使用Selector，得向Selector注册Channel，然后调用它的select()方法。这个方法会一直阻塞到某个注册的通道有事件就绪。一旦这个方法返回，线程就可以处理这些事件，事件的例子有如新连接进来，数据接送等。


##Channel
---

Java NIO的通道类似流，但又有些不同：

* 既可以从通道中读取数据，又可以写数据到通道。但流的读写通常是单向的。
* 通道可以异步的读写。
* 通道的数据总是要先读到一个Buffer，或者总要从一个Buffer中写入。

正如上面所说，从通道读取数据到缓冲区，从缓冲区写入数据到通道。

**Channel的实现**

* FileChannel  从文件中读取数据
* DataChannel  能通过UDP读写网络中的数据
* SocketChannel   能通过TCP读写网络中的数据
* ServerSocketChannel   可以监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个SocketChannel


基本的Channel示例

下面是一个使用FileChannel读取数据到Buffer中的示例

```
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");
FileChannel inChannel = aFile.getChannel();
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = inChannel.read(buf);
while (bytesRead != -1) {
System.out.println("Read " + bytesRead);
buf.flip();
while(buf.hasRemaining()){
System.out.print((char) buf.get());
}
buf.clear();
bytesRead = inChannel.read(buf);
}
aFile.close();

```
注意 buf.flip() 的调用，首先读取数据到Buffer，然后反转Buffer,接着再从Buffer中读取数据。


##Buffer

Java NIO中的Buffer用于和NIO通道进行交互。如你所知，数据是从通道读入缓冲区，从缓冲区写入到通道中的。

缓冲区本质是一块可以写入数据，然后可以从中读取数据的内存。这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问这块内存。

**Buffer的基本用法**

使用Buffer读写数据一般遵循以下四个步骤：

1. 写入数据到Buffer
2. 调用flip()方法
3. 从Buffer中读取数据
4. 调用clear()方法或者compact()方法





