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
4. Writing a pipe
```
ByteStream::ByteStream(const size_t capacity) 
{
    DUMMY_CODE(capacity); 
    _ncapacity = capacity;
    _buf.resize(capacity);
}

size_t ByteStream::write(const string &data) {
    DUMMY_CODE(data);
    size_t len = 0;
    for(const auto &e : data)
    {
        if(_nwrite >= _nread + _ncapacity)
        {
            break;
        }
        _buf[_nwrite % _ncapacity] = e; # 此处采用xv6 中 pipe 的读写方式，使用取模运算循环使用数组
        ++_nwrite;
        ++len;
    }
    return len;
}

//! \param[in] len bytes will be copied from the output side of the buffer
string ByteStream::peek_output(const size_t len) const {
    DUMMY_CODE(len);
    string res;
    for(size_t i = _nread; i < (_nread + len) && i < _nwrite; ++i)
    {
        res.push_back(_buf[i % _ncapacity]);
    }    
    return res;
}

//! \param[in] len bytes will be removed from the output side of the buffer
void ByteStream::pop_output(const size_t len) { 
    DUMMY_CODE(len);
    for(size_t i = 0; i < len; ++i)
    {
        if(_nread < _nwrite)
        {
            ++_nread;
        }
        else
        {
            return;
        }
    } 
}

//! Read (i.e., copy and then pop) the next "len" bytes of the stream
//! \param[in] len bytes will be popped and returned
//! \returns a string
std::string ByteStream::read(const size_t len) {
    DUMMY_CODE(len);
    size_t length = len;
    if(length > _nwrite)
    {
        length = _nwrite;
    }
    std::string ans = peek_output(length);
    pop_output(length);
    return ans;
}

void ByteStream::end_input() { input_ended_flag = true; }

bool ByteStream::input_ended() const { return input_ended_flag; }

size_t ByteStream::buffer_size() const { return _nwrite - _nread; }

bool ByteStream::buffer_empty() const { return buffer_size() == 0; }

bool ByteStream::eof() const { return input_ended_flag && buffer_empty(); }

size_t ByteStream::bytes_written() const { return _nwrite; }

size_t ByteStream::bytes_read() const { return _nread; }

size_t ByteStream::remaining_capacity() const { return _ncapacity - (_nwrite - _nread); }
```
使用一下命令检查正确性
```
make check_lab0
```
提交了多次，发现每次都在同几个错误上
```
The following tests FAILED:
         26 - t_byte_stream_construction (Failed)
         27 - t_byte_stream_one_write (Failed)
         28 - t_byte_stream_two_writes (Failed)
         29 - t_byte_stream_capacity (Failed)
         30 - t_byte_stream_many_writes (Timeout)
Errors while running CTest
make[3]: *** [CMakeFiles/check_lab0.dir/build.make:58：CMakeFiles/check_lab0] 错误 8
make[2]: *** [CMakeFiles/Makefile2:371：CMakeFiles/check_lab0.dir/all] 错误 2
make[1]: *** [CMakeFiles/Makefile2:378：CMakeFiles/check_lab0.dir/rule] 错误 2
make: *** [Makefile:194：check_lab0] 错误 2
```
一开始以为是创建vector时初始化了值，导致输出buffer_empty() 时报错，改动了 vector 的初始化方式和buffer_empty() 判空条件后依然报错
后来发现在 build 目录下使用 make 后会一个错误
```
/sponge/build$ make
Scanning dependencies of target sponge
make[2]: 警告：文件“libsponge/CMakeFiles/sponge.dir/depend.make”的修改时间在未来 8.7 秒后
[  3%] Building CXX object libsponge/CMakeFiles/sponge.dir/byte_stream.cc.o
/home/feng/SSH/Project_Linux_TCPIP/sponge/libsponge/byte_stream.cc: In constructor ‘ByteStream::ByteStream(size_t)’:
/home/feng/SSH/Project_Linux_TCPIP/sponge/libsponge/byte_stream.cc:15:1: error: ‘ByteStream::_buf’ should be initialized in the member initialization list [-Werror=effc++]
   15 | ByteStream::ByteStream(const size_t capacity)
      | ^~~~~~~~~~
cc1plus: all warnings being treated as errors
make[2]: *** [libsponge/CMakeFiles/sponge.dir/build.make:63：libsponge/CMakeFiles/sponge.dir/byte_stream.cc.o] 错误 1
make[1]: *** [CMakeFiles/Makefile2:2041：libsponge/CMakeFiles/sponge.dir/all] 错误 2
make: *** [Makefile:95：all] 错误 2
```
错误显示在构造函数初始化时没有按照成员初始化列表来初始化，这是C++11 的新特性，改了之后运行成功
```
ByteStream::ByteStream(const size_t capacity) : _nwrite(0), _nread(0), _ncapacity(capacity), _buf(std::vector<char>(capacity))
{
    DUMMY_CODE(capacity); 
}
```
```
/sponge/build$ make check_lab0
[100%] Testing Lab 0...
Test project /home/feng/SSH/Project_Linux_TCPIP/sponge/build
    Start 26: t_byte_stream_construction
1/9 Test #26: t_byte_stream_construction .......   Passed    0.01 sec
    Start 27: t_byte_stream_one_write
2/9 Test #27: t_byte_stream_one_write ..........   Passed    0.01 sec
    Start 28: t_byte_stream_two_writes
3/9 Test #28: t_byte_stream_two_writes .........   Passed    0.02 sec
    Start 29: t_byte_stream_capacity
4/9 Test #29: t_byte_stream_capacity ...........   Passed    0.40 sec
    Start 30: t_byte_stream_many_writes
5/9 Test #30: t_byte_stream_many_writes ........   Passed    0.02 sec
    Start 31: t_webget
6/9 Test #31: t_webget .........................   Passed    1.50 sec
    Start 53: t_address_dt
7/9 Test #53: t_address_dt .....................   Passed    0.03 sec
    Start 54: t_parser_dt
8/9 Test #54: t_parser_dt ......................   Passed    0.02 sec
    Start 55: t_socket_dt
9/9 Test #55: t_socket_dt ......................   Passed    0.02 sec

100% tests passed, 0 tests failed out of 9

Total Test time (real) =   2.09 sec
[100%] Built target check_lab0
```

总的来说，Lab0 几乎没有难度，更多的是集中在C++工程项目的代码规范性上
