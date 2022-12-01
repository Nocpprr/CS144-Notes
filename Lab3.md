Lab Checkpoint 3: the TCP sender

1. Overview

本节实现的是TCPSocket结构里的TCPSender（实验指导书原话是The TCPSender is a tool that translates from an outgoing byte stream to segments that will become the payloads of
unreliable datagrams.  TCPSender是一种工具，它将输出的字节流转换为将成为不可靠数据报的有效负载的段）, 我的理解是它主要负责将上层传来的数据从ByteStream里读取出来并按合适的大小封装在TCPSegment里发送给TCPReceiver
![image](https://github.com/Nocpprr/CS144-Notes/blob/main/BV_3%7DL4I3%5DMW%7BYEAXLT6~S7.png)

2. Getting Started

3. TCP Sender
 - 这是TCP的“发送方”部分，负责从ByteStream(由一些发送方应用程序创建并写入)读取数据，并将流转换为传出的TCP段序列。在远程端，TCP Receiver将这些段(到达的那些段——它们可能都没有到达)转换回原始的字节流，并将确认和窗口广告发送回发送方。
 - TCP发送方写入与实验室2中的TCPReceiver相关的TCPSegment的所有字段:即序列号、SYN标志、有效负载和FIN标志。
 - TCP发送方只读取由接收方写入的段中的字段:确认号和窗口大小。
 TCPSegment:
![image](https://user-images.githubusercontent.com/105581407/205090174-2ec78a29-ecb4-462a-babb-873ca3b8013b.png)

TCP Sender 主要有以下功能:
  - 读取ByteStream的数据，尽可能的填充窗口，不要超出接受方的窗口大小。
  - 将ByteStream的字节流转换为TCP Segment，并且填充Seq, Syn, Fin等。需要注意Syn和FIN也算一个字节
  - 发送方应该一直发送，知道窗口已满或者字节流为空
  - 收到接受方的确认号和窗口后，调整并记录他们
  - 实现重传机制

重传机制：
  - 发送方会保存一个RTO值和重传计时器。
  - 实现一个tick函数，由tick函数来更新重传计时器的时间
  - 若重传计时器超过了RTO值，重传待确认的最老的一个TCP Segment，并且更新RTO值，重置重传计时器时间
  - 关于更新RTO值，由RFC6298第五条规定，每次重传后，将RTO <- RTO * 2 （指数后退）
  - 在收到接收方的确认号后，若有TCP Segment成功确认，将RTO重置为初始值。如果还有段未确认，重新启动重传计时器，否则应关闭计时器

关于这个RFC6298，我也是看了别人的视频才知道的。。。

4. Implement:

需要注意的是，这里的接收方发送的ackno跟三次握手里的ack不同，我一开始也是按接受方需求的下一个序列号（也就是说表示ack - 1的所有序列已经收到）来做，但是我发现这个ackno实际表示的是接受方已经成功接受了0 - ackno的序列号，
这里的_next_seqno也是这样，因为是下标从0开始的，就是说_next_seqno表示已经发送了多少的序列号，下一个要发送的是_next_seqno + 1.

fill_window
```
void TCPSender::fill_window() {
    TCPSegment tcp_segment;
    if(_fin) return;

    if(!_syn)
    {
        tcp_segment.header().syn = true;
        _syn = true;
        send_Tcp_segment(tcp_segment);
        return;
    }

    uint16_t window_size = _receiver_window_size == 0 ? 1  : _receiver_window_size; //没有收到ackno以前，假定接收方窗口为 One Byte.

    if(_stream.eof() && _next_seqno < _ack_seqno + window_size) 
    {
        tcp_segment.header().fin = true;
        send_Tcp_segment(tcp_segment);
        _fin = true;
        return;
    }
    // 注意判断条件，下一次要发送的数据不能超过接收方窗口大小
    // 注意不能等于，因为已经发送的数据序列号不能超过接收方接收到的序列号 + 接受方窗口大小，这里的_next_seqno表示的是已经发送了多少数据
    while(!_stream.buffer_empty() && _next_seqno < _ack_seqno + window_size)
    {
        size_t Maxlen = min(TCPConfig::MAX_PAYLOAD_SIZE, window_size + _ack_seqno - _next_seqno);
        tcp_segment.payload() = _stream.read(Maxlen);
        if(_stream.eof() && tcp_segment.length_in_sequence_space() < window_size) //流为空退出，注意FIN也算一个字节，所以要判断FIN是否会超过接收方窗口
        {
            tcp_segment.header().fin = true;
            _fin = true;
        }
        send_Tcp_segment(tcp_segment);
    }

    
}
```
seng_Tcp_segment:
```
//实现一个发送tcp_segment的函数
void TCPSender::send_Tcp_segment(TCPSegment &tcp_segment)
{
    tcp_segment.header().seqno = wrap(_next_seqno, _isn);
    _next_seqno += tcp_segment.length_in_sequence_space(); // 这里就看出来了，_next_seqno表示的是已经发送了多少字节，因为是下标是从0开始的
    _segments_ack.push(tcp_segment);
    _segments_out.push(tcp_segment);

    if(!_is_retransmission_time_run) // 发送后，应该启动重传计时器
    {
        _is_retransmission_time_run = true;
        _retransmission_timeout = _initial_retransmission_timeout;
        _retransmission_timer = 0;
    }
   
}
```
ack_received:
```
void TCPSender::ack_received(const WrappingInt32 ackno, const uint16_t window_size) {
    uint64_t _abs_ackno = unwrap(ackno, _isn, _next_seqno);
    //接受方已接受的序列号超出已发送的段的序列号
    if(_abs_ackno > _next_seqno) return;

    //更新窗口
    if(_abs_ackno >= _ack_seqno)
    {
        _ack_seqno = _abs_ackno;
        _receiver_window_size = window_size;
        if(window_size == 0) _zero_window = true;
    }
    //删除已被确认的tcp段
    //一个细节：若删除后发送窗口有空闲，则继续尝试发送
    bool flag = false;
    while(!_segments_ack.empty())
    {
        TCPSegment tmp = _segments_ack.front();
        //注意：若接收方已接受序列号大于等于最早待确认的tcp段的序列号+长度，则表示此tcp段已接受
        uint64_t _seqn = unwrap(tmp.header().seqno, _isn, _next_seqno) + tmp.length_in_sequence_space();
        if(_seqn <= _abs_ackno)
        {
            flag = true;
            _segments_ack.pop();
            _is_retransmission_time_run = true;
            _retransmission_timeout = _initial_retransmission_timeout;
            _consecutive_retransmissions = 0;
            _retransmission_timer = 0;
        }
        else break;
    }
    
    if(flag) fill_window();

    //根据_segments_ack是否为空来判断重传计时器是否开启
    if(!_segments_ack.empty())
    {
        _is_retransmission_time_run = true;
    }// 一定要注意关闭重传计时器。。。我忘了关闭重传计时器，有个test一直过不了，还是一步一步debug出来的
    else _is_retransmission_time_run = false;
 }
 ```
 tick:
 ```
 void TCPSender::tick(const size_t ms_since_last_tick) { 
    if(!_is_retransmission_time_run) return;
    
    _retransmission_timer += ms_since_last_tick;
    if(_retransmission_timer >= _retransmission_timeout) //如果计时器超过了当前RTO值
    {
        TCPSegment tmp = _segments_ack.front();
        _segments_out.push(tmp);
        _retransmission_timer = 0;
        //根据RFC6298第五条规则
        //根据Lab3 test #17 修改，若传来的window_size为 0, 不进行RTO翻倍
        //When filling window, treat a '0' window size as equal to '1' but don't back off RTO
        if(!_zero_window) _retransmission_timeout *= 2;
        ++_consecutive_retransmissions;
    }

 }
 ```
 
5. 总结
TCP Sender 算是比较有难度了，参考了很多大佬的视频和代码才慢慢搞出来的，其中有些也是抄的别人的代码。
    - 这节有几个难点，主要就是重传计时器和接受方ackno的问题了。需要注意的就是接收方ackno是接收方已经接收了多少字节，与三次握手里的ack是不一样的，三次握手里的ack表示已经接受了ack-1个字节，下一次想要接收ack序号的数据
    - 在设置重传RTO时，需要参考RFC6298文档里的RTO <- RTO * 2, 也就是指数后退，这也是不知道很难的一个点
    - 还有就是重传已经确认的字段后，如果没有待确定的字段了，一定要注意关闭重传计时器，不然发送方在RTO值后继续重传，但是已经没有段重传了，这时就会报Segment Fault，段错误
    - 本节涉及很多边界判定，比较容易搞混，一定要反复确认，把每个seqno的含义搞清楚
    
