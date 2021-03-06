# AndroidSocketClient
socket client server简易封装

## Import
[JitPack](https://jitpack.io/)

Add it in your project's build.gradle at the end of repositories:

```gradle
repositories {
  // ...
  maven { url "https://jitpack.io" }
}
```

Step 2. Add the dependency in the form

```gradle
dependencies {
  compile 'com.github.vilyever:AndroidSocketClient:2.0.3'
}
```

## Updates
* 2.0.0
</br>
重构，移除了一些方法，解耦一些设置，变更比较大
</br>
增加发送消息时自动发送包头包尾信息用于粘包和分包
</br>
增加接收消息时自动依据设置的包头包尾分割消息
</br>
增加发送消息状态回调，增加分段发送设置，可查看发送单个包的发送进度

* 1.5.0
</br>
重构
</br>
抽离心跳包设置到HeartBeatHelper类中
</br>
抽离包尾判断到SocketPacketHelper类中

* 1.4.1
</br>
增加禁用心跳包和超时自动断开
</br>
SocketClient.disableHeartBeat();(设置heartBeatMessage为null效果相同）
</br>
SocketClient.disableRemoteNoReplyAliveTimeout();

* 1.4.0
</br>
将发送和接收时对String和byte数组的转换移到后台线程进行

* 1.3.4
</br>
修复SocketServer停止监听时未将已连接的client断开问题
</br>
心跳包修改为可发送byte数组
</br>
自动应答集成一个帮助类

* 1.3.3
</br>
修复SocketServer启动监听回调时机问题

* 1.3.0
</br>
修改消息收发机制
</br>
接收消息回调参数由String改为SocketResponsePacket，提供byte[]数据
</br>
获取String消息调用SocketResponsePacket.getMessage()实时获取

## Usage
```java

SocketClient socketClient = new SocketClient(new SocketClientAddress("192.168.1.1", 80, 15 * 1000)); // 设置ip端口，连接超时时长
socketClient.registerSocketClientDelegate(new SocketClientDelegate() {
     @Override
     public void onConnected(SocketClient client) {
         SocketPacket packet = socketClient.sendData(new byte[]{0x02}); // 发送消息
         packet = socketClient.sendString("sy hi!"); // 发送消息

         socketClient.cancelSend(packet); // 取消发送，若在等待发送队列中则从队列中移除，若已发送完成则无法取消，若正在发送且分段发送则取消发送剩余数据（此时若没有设置包头用于远程端判断下一个信息包的开始可能出现粘包或数据不全问题）
     }

     @Override
     public void onDisconnected(SocketClient client) {
        // 断开连接回调，可在此实现自动重连
        socketClient.connect();
     }

     @Override
     public void onResponse(SocketClient client, @NonNull SocketResponsePacket responsePacket) {
        byte[] data = responsePacket.getData(); // 获取接收的byte数组，不为null
        String message = responsePacket.getMessage(); // 获取按默认设置的编码转化的String，可能为null
     }
 });

socketClient.getAddress().setRemoteIP("192.168.1.123"); // 设置IP
socketClient.getAddress().setRemotePort(8080); // 设置端口
socketClient.getAddress().setConnectionTimeout(30 * 1000); // 设置连接超时时长

socketClient.setCharsetName("UTF-8"); // 设置发送和接收String消息的默认编码

socketClient.getHeartBeatHelper().setHeartBeatInterval(1000 * 30); // 设置自动发送心跳包的时间间隔，若值小于0则不发送心跳包
socketClient.getHeartBeatHelper().setRemoteNoReplyAliveTimeout(1000 * 60); // 设置远程端多长时间内没有消息发送到本地就自动断开连接，若值小于0则不自动断开

socketClient.getHeartBeatHelper().setSendString("$HB$"); // 设置自动发送心跳包的字符串，若为null则不发送心跳包
socketClient.getHeartBeatHelper().setSendData("$HB$".getBytes()); // 同上

socketClient.getHeartBeatHelper().setReceiveString("$HB$"); // 设置从远程端接收的心跳包字符串，onResponse回调将过滤此信息，若为null则不过滤
socketClient.getHeartBeatHelper().setReceiveData("$HB$".getBytes()); // 同上

socketClient.getSocketPacketHelper().setSendHeaderData(new byte[]{0x02}); // 设置发送消息时自动在消息头部添加的信息，远程端收到此信息后表示一条消息开始，用于解决粘包分包问题，若为null则不添加头部信息
socketClient.getSocketPacketHelper().setSendTrailerString("\r\n"); // 设置发送消息时自动在消息尾部添加的信息，远程端收到此信息后表示一条消息结束，用于解决粘包分包问题，若为null则不添加尾部信息
socketClient.getSocketPacketHelper().setReceiveHeaderData(new byte[]{0x02}); // 设置接收消息时判断消息开始的头部信息，用于解决粘包分包问题，若为null则不判断
socketClient.getSocketPacketHelper().setReceiveTrailerString("\r\n"); // 设置接收消息时判断消息结束的尾部信息，用于解决粘包分包问题，若为null则每次读取InputStream直到其为空，可能出现粘包问题

socketClient.getSocketPacketHelper().setSegmentLength(4 * 1024); // 设置每次发送消息的分段长度，即20KB的数据将分段为4KB分5次发送，每次发送完一段都会回调发送的进度

socketClient.registerSocketClientSendingDelegate(new SocketClientSendingDelegate() {
    @Override
    public void onSendPacketBegin(SocketClient client, SocketPacket packet) {
        // 数据包发送开始，packet为调用send方法的返回值
    }

    @Override
    public void onSendPacketCancel(SocketClient client, SocketPacket packet) {
        // 数据包发送撤销，packet为调用send方法的返回值，此回调表示撤销成功
    }

    @Override
    public void onSendPacketEnd(SocketClient client, SocketPacket packet) {
        // 数据包发送结束，packet为调用send方法的返回值
    }

    @Override
    public void onSendPacketProgress(SocketClient client, SocketPacket packet, float progress) {
        // 数据包发送进度，packet为调用send方法的返回值，progress值为0-1
    }
});

socketClient.connect(); // 连接，异步进行

socketClient.disconnect(); // 断开连接

socketClient.getState(); // 获取当前状态，Connecting, Connected, Disconnected

```

## License
[Apache License Version 2.0](http://www.apache.org/licenses/LICENSE-2.0.txt)
