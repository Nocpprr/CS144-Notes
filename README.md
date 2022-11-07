# CS144-Notes
做cs144的一些笔记

Lab0:networking warmup

1.Set up
这节内容主要是热身，包括一些装虚拟机，配环境，调试环境等

2.Networking by hand
2.1 Fetch a Web page
```
telnet cs144.keithw.org http
GET /hello HTTP/1.1 # 发起GET请求，建立HTTP 1.1 持久性连接
Host: cs144.keithw.org # 服务器地址
Connection: close # 告知服务器请求完毕
```
返回值
```
HTTP/1.1 200 OK
Date: Mon, 07 Nov 2022 07:21:24 GMT
Server: Apache
Last-Modified: Thu, 13 Dec 2018 15:45:29 GMT
ETag: "e-57ce93446cb64"
Accept-Ranges: bytes
Content-Length: 14
Connection: close
Content-Type: text/plain

Hello, CS144!
```
使用telnet和netcat实现双端通信
```
netcat -v -l -p 9090 # server
telnet localhost 9090 # client
```

3. Writing a network program using an OS stream socket
3.1 fetching and building
```
git clone https://github.com/cs144/sponge
cd sponge
mkdir build
cd build
cmake .. # cmake 构建项目 ".."表示项目目录在上一级目录
make # 编译项目
```
3.2 Modern C++
3.3 Sponge Documentation
[Sponge](https://cs144.github.io/doc/lab0)

3.4 webget()
```
void get_URL(const string &host, const string &path) {
    TCPSocket sock;
    sock.connect(Address(host, "http"));
    string send = "GET " + path + " HTTP/1.1\r\nHost: " + host + "\r\n" + "Connection: close\r\n\r\n";
    sock.write(send);
    sock.shutdown(SHUT_WR); # 使用shutdown 关闭写端，此时还可以收到对方发来的数据, 比close 更加细粒度，符合TCP四次挥手协议
    while(!sock.eof())
    {
        cout << sock.read();
    }
    sock.close(); # 关闭套接字
}
```
