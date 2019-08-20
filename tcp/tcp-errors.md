# TCP 错误

## java.net.SocketException: Connection refused: connect

该异常发生在客户端进行 new Socket(ip, port)操作时，该异常发生的原因是或者具有ip地址的机器不能找到（也就是说从当前机器不存在到指定ip路由），或者是该ip存在，但找不到指定的端口进行监听。

## java.net.SocketException: Socket is closed

该异常在客户端和服务器均可能发生。异常的原因是己方主动关闭了连接后（调用了Socket的close方法）再对网络连接进行读写操作。

## java.net.SocketException: Connection reset

该异常在客户端和服务器端均有可能发生. 一端退出，但退出时并未关闭该连接，另一端如果在从连接中读数据则抛出该异常（Connection reset）。

## java.net.SocketException: Connection reset by peer : Socket write error

该异常在客户端和服务器端均有可能发生. 一端的Socket被关闭（或主动关闭或者因为异常退出而引起的关闭），另一端仍发送数据，发送的第一个数据包引发该异常(Connect reset by peer).

## java.net.SocketException: Broken pipe

该异常在客户端和服务器均有可能发生。在抛出 SocketExcepton:Connect reset by peer:Socket write error后，如果再继续写数据则抛出该异常。

[1]: <http://developer.51cto.com/art/201003/189724.htm>
[2]: <https://docs.oracle.com/javase/1.5.0/docs/guide/net/articles/connection_release.html>
