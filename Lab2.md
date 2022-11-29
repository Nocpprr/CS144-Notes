Lab Checkpoint 2: the TCP receiver

![TCP结构图](https://github.com/Nocpprr/CS144-Notes/blob/main/BV_3%7DL4I3%5DMW%7BYEAXLT6~S7.png)
本节实现的是TCP结构的TCPReceiver

2 Getting started
```
git merge origin/lab2-startercode
```
3 Lab 2: The TCP Receiver

计算TCP序列号和绝对序列号
注意在计算abs_seqn时，需要与checkpoint做对比, 选择与checkpoint相近的一个，这是为了保证TCP字节的连续性。
这节在make时还遇到一个错误
![如图](https://user-images.githubusercontent.com/105581407/202733810-5aa26f56-01ac-4487-a976-b1b15389f058.png)
在tcp_parser里找了下
![如图](https://user-images.githubusercontent.com/105581407/202734142-c4161645-bbc6-443e-8838-a08c7534c91f.png)
应该是这个包找不到
运行
```
sudo apt-get install libpcap_dev
```
主要代码:
```
WrappingInt32 wrap(uint64_t n, WrappingInt32 isn) {
    DUMMY_CODE(n, isn);
    WrappingInt32 seq(isn.raw_value() + static_cast<uint32_t>(n));
    return seq;
}

uint64_t unwrap(WrappingInt32 n, WrappingInt32 isn, uint64_t checkpoint) {
    DUMMY_CODE(n, isn, checkpoint);
    // 找到n和checkpoint之间的最小步数
    int32_t min_step = n - wrap(checkpoint, isn);
    // 将步数加到checkpoint上
    int64_t ret = checkpoint + min_step;
    // 如果反着走的话要加2^32
    return ret >= 0 ? static_cast<uint64_t>(ret) : ret + (1ul << 32);
}
```
3.2 Implementing the TCP receiver
这节代码不复杂，但是有很多小细节
segment received() 这是主要的函数
  - 主要对发送方发来的TCP数据进行处理，初始化一个序列号，作为三次握手的初始序列号
  - 将数据传入上一节实现的reassembler，调用push_substring方法
  - 如果TCP数据报头部FIN标志位为1，则这意味着有效负载的最后一个字节是整个流的最后一个字节
  - 注意三次握手中只有建立连接时(前两次握手)SYN才为1
 ```
 void TCPReceiver::segment_received(const TCPSegment &seg) {
    DUMMY_CODE(seg);    
    // TCP 三次握手中，只有建立连接时SYN == 1
    TCPHeader _header = seg.header();
    
    uint64_t abs_n = 0; // reassembler 的 index
    if(!_is_first)
    {
        if(_header.syn)
        {
            _isn = _header.seqno;
            _is_first = true;
            _reassembler.push_substring(seg.payload().copy(), 0, _header.fin);
        }
        return;
    }
    abs_n = unwrap(_header.seqno, _isn, _reassembler.stream_out().bytes_written());  
    _reassembler.push_substring(seg.payload().copy(), abs_n - 1, _header.fin);
}
```
ackno()，返回一个可选的，其中包含接收方不知道的第一个字节的序列号。这是窗口的左边缘:接收器感兴趣接收的第一个字节。
我一开始还以为这是接收方发送数据报的ack, 根据三次握手，ack = 发送方seq + 1，但是始终不对
![lab1的图](https://github.com/Nocpprr/CS144-Notes/blob/main/image-20211107124153476.png)
  - 根据这个图显示，接收方不知道的第一个字节应该是 first_unassembled + 1
  - 需要注意的是，接受的TCP数据包中包含SYN和FIN标志位，若FIN标志位为1，则ack应该+1 (本次实验中不考虑其他标志位)
```
optional<WrappingInt32> TCPReceiver::ackno() const {
    if(!_is_first) return nullopt;
    uint64_t _written = _reassembler.stream_out().bytes_written() + 1; // +1 是因为建立连接时发送的第一个数据SYN为1
    if(_reassembler.stream_out().input_ended() ) // fin 算一个 byte
    {
        ++_written;
    }
    return wrap(_written, _isn);
}
```
window size()，这是滑动窗口剩余的大小，相当于fisrt_unaccepted - first_unassembled
```
size_t TCPReceiver::window_size() const { return _capacity - _reassembler.stream_out().buffer_size(); }
```
在make check_lab2 时，我仍然遇到了问题
![image](https://user-images.githubusercontent.com/105581407/202742383-28b6e59a-ca8d-4f9e-8b4e-a00efe0129de.png)
Test 6 无法通过，报的错误很奇怪，不知道怎么修改，debug也无法找到原因,我仍在修改

本节实验主要是对TCP接收方的一些处理，需要理解的就是TCP接收方序列号的变化和滑动窗口的变化。
一个跟lab1里很相似的细节，有些不连贯的数据发过来后，进入了滑动窗口但是并不被reassembler转发到byte_stream, 所以滑动窗口的左右边界并不变化
还有就是注意SYN和FIN标志位

-----------------------------------------------------------------------------------------------------

更新一下，原来test6的报错是因为数据量太大，代码时间复杂度高了没过，debug后发现是lab1的reassembler有点问题。
原先是使用的string + map来记录分散的数据段并且去重，但是这样的时间复杂度是nlogn的，甚至由于随时扩容string的原因，时间复杂度会更高
因此这里采用和ByteStream里相同的方式，用两个环形vector来记录数据，一个用来存数据，一个用来记录数据是否已写入，将复杂度降到On;
![image](https://user-images.githubusercontent.com/105581407/204532715-1fe5566e-a49b-40ff-af90-74bf13409da1.png)
![image](https://user-images.githubusercontent.com/105581407/204532856-ea3aae6f-1c35-4046-b769-86499ab8136f.png)
更改后，test6通过

