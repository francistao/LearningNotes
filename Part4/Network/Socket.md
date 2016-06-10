#Socket
---
###使用TCP
---
客户端

```
Socket socket = new Socket("ip", 端口);

InputStream is = socket.getInputStream();
DataInputStream dis = new DataInputStream(is);

OutputStream os = socket.getOutputStream();
DataInputStream dos = new DataOutputStream(os);
```

服务器端

```
ServerSocket serverSocket = new ServerSocket(端口);
Socket socket = serverSocket.accept();
//获取流的方式与客户端一样
```

读取输入流

```
byte[] buffer = new byte[1024]; 
do{ 
	int count = is.read(buffer); 
	if(count <= 0){ break; }
	else{ 
	// 对buffer保存或者做些其他操作 
		} 
	}
while(true);


```


使用UDP
---
客户端和服务器端一样的

```
DatagramSocket socket = new DatagramSocket(端口);
InetAddress serverAddress = InetAddress.getbyName("ip");
//发送
DatagramPackage packet = new DatagramPacket(buffer, length, host, port);
socket.send(packet);
//接收
byte[] buf = new byte[1024];
DatagramPacket packet = new DatagramPacket(buf, 1024);
Socket.receive(packet);
```