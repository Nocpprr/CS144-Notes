Lab Checkpoint 4: the summit (TCP in full)

1. 本节实验了一个TCP Connection类，也是TCP最重要的功能，与其他的对端通信

2. 首先还是看这个图
TCPSocket的结构图
![image](https://github.com/Nocpprr/CS144-Notes/blob/main/BV_3%7DL4I3%5DMW%7BYEAXLT6~S7.png)

从图里看得出来，TCPConnection是TCPSocket的核心，其中包含了Lab2和Lab3实现的Receiver和Sender。
在TCP的连接和断开过程中，Socket收到网络传来的信息后，将TCPSegment传给Connection，Connection根据Segment的信息来更新Receiver和Sender的状态，然后做出相应的回应
要了解这些状态，就要明白三次握手和四次挥手过程中TCP及其Receiver和Sender分别的状态转换

3. 连接和断开途中具体状态转移

TCP三次握手
![image](https://user-images.githubusercontent.com/105581407/210364854-54ad2b47-d80f-45e2-9c35-1a658ed6772e.png)

TCP四次挥手
![image](https://user-images.githubusercontent.com/105581407/210365634-6d7a7041-13b4-4e4c-a60b-780d18b41f46.png)

Lab2中TCPReceiver的状态转移图
![image](https://user-images.githubusercontent.com/105581407/210366797-8ed294a7-86c8-401c-9217-62ee343c884c.png)

Lab3中TCPSender的状态转移图
![image](https://user-images.githubusercontent.com/105581407/210367296-eab6494c-7c45-40a2-87bc-7a1a921505c5.png)

然后再看CS144给出的tcp_state.cc中的各个状态下Receiver和Sender的状态
```
TCPState::TCPState(const TCPState::State state) {
    switch (state) {
        case TCPState::State::LISTEN:
            _receiver = TCPReceiverStateSummary::LISTEN;
            _sender = TCPSenderStateSummary::CLOSED;
            break;
        case TCPState::State::SYN_RCVD:
            _receiver = TCPReceiverStateSummary::SYN_RECV;
            _sender = TCPSenderStateSummary::SYN_SENT;
            break;
        case TCPState::State::SYN_SENT:
            _receiver = TCPReceiverStateSummary::LISTEN;
            _sender = TCPSenderStateSummary::SYN_SENT;
            break;
        case TCPState::State::ESTABLISHED:
            _receiver = TCPReceiverStateSummary::SYN_RECV;
            _sender = TCPSenderStateSummary::SYN_ACKED;
            break;
        case TCPState::State::CLOSE_WAIT:
            _receiver = TCPReceiverStateSummary::FIN_RECV;
            _sender = TCPSenderStateSummary::SYN_ACKED;
            _linger_after_streams_finish = false;
            break;
        case TCPState::State::LAST_ACK:
            _receiver = TCPReceiverStateSummary::FIN_RECV;
            _sender = TCPSenderStateSummary::FIN_SENT;
            _linger_after_streams_finish = false;
            break;
        case TCPState::State::CLOSING:
            _receiver = TCPReceiverStateSummary::FIN_RECV;
            _sender = TCPSenderStateSummary::FIN_SENT;
            break;
        case TCPState::State::FIN_WAIT_1:
            _receiver = TCPReceiverStateSummary::SYN_RECV;
            _sender = TCPSenderStateSummary::FIN_SENT;
            break;
        case TCPState::State::FIN_WAIT_2:
            _receiver = TCPReceiverStateSummary::SYN_RECV;
            _sender = TCPSenderStateSummary::FIN_ACKED;
            break;
        case TCPState::State::TIME_WAIT:
            _receiver = TCPReceiverStateSummary::FIN_RECV;
            _sender = TCPSenderStateSummary::FIN_ACKED;
            break;
        case TCPState::State::RESET:
            _receiver = TCPReceiverStateSummary::ERROR;
            _sender = TCPSenderStateSummary::ERROR;
            _linger_after_streams_finish = false;
            _active = false;
            break;
        case TCPState::State::CLOSED:
            _receiver = TCPReceiverStateSummary::FIN_RECV;
            _sender = TCPSenderStateSummary::FIN_ACKED;
            _linger_after_streams_finish = false;
            _active = false;
            break;
    }
}
```
根据以上的state图对照三次握手与四次挥手状态转移图，就可以搞清楚TCPConnection在不同状态下收到TCPSegment的反应
比如TCP的receiver处于Listen状态，sender处于SYN-SENT状态，则TCP处于SYN-SENT状态，此时收到Segment，若包含SYN和ACK, 表明对端主动发起连接，处于三次握手第二阶段，
进入Established状态，我方发送一个空包传递ACK, 进入三次握手最后阶段；若收到了SYN，说明双方同时发起连接，此时处于三次握手第二阶段，进入SYN-REVD状态，此时没有
收到第一个SYN的确认，因此重发一个SYN,并且带上对对方SYN的确认；若无SYN，因为此时处于三次握手阶段，直接无视

特别说明一下TCP双方同时发起连接的情况，两端几乎在同时发送SYN，并进入 SYN_SENT状态。当每一端收到SYN时，状态变为SYN_RCVD，同时它们都再发SYN并对收到的SYN进行
确认。当双方都收到SYN及相应的ACK时，状态都变迁为ESTABLISHED。
![image](https://user-images.githubusercontent.com/105581407/210370083-82332fcc-05bb-4b52-a8d8-ade1071c31cc.png)

4. 实现
segment_received:
```
void TCPConnection::segment_received(const TCPSegment &seg) {

    if(!_is_ative) return; //未启动时不接受segment

    _time_since_last_segment_received = 0;


    if(seg.header().rst)
    {
        //_linger_after_streams_finish = false;
        _receiver.stream_out().set_error();
        _sender.stream_in().set_error();
        _is_ative = false;
    }
    else if(_sender.next_seqno_absolute() == 0)
    {
        // TCP的receiver处于Listen状态，sender处于Closed状态，则TCP处于Listen状态 
        // (由lab2 和 lab3的状态图决定)
        _receiver.segment_received(seg);
        if(seg.header().syn) // 收到SYN, 表明TCP连接由对端发起，进入SYN-REVD状态
        {
            //三次握手第一阶段，没有ACK, sender不需要ack_received
            _receiver.segment_received(seg);
            //回复SYN
            //receiver处于SYN-RECV状态，sender处于SYN-SEND状态
            connect();
        }
    }
    else if(_sender.next_seqno_absolute() == _sender.bytes_in_flight() && !_receiver.ackno().has_value())
    {
        // TCP的receiver处于Listen状态，sender处于SYN-SENT状态，则TCP处于SYN-SENT状态 
        if(seg.header().syn && seg.header().ack)
        {
            // 收到SYN和ACK, 表明对端主动发起连接，处于三次握手第二阶段，进入Established状态
            // 发送一个空包传递ACK, 进入三次握手最后阶段
            _sender.ack_received(seg.header().ackno, seg.header().win);
            _receiver.segment_received(seg);
            _sender.send_empty_segment();
            send_segment(); //发送segment
        }
        else if(seg.header().syn && !seg.header().ack)
        {
            // 只收到了SYN，说明双方同时发起连接，此时处于三次握手第二阶段，进入SYN-REVD状态
            // 此时没有收到第一个SYN的确认，因此重发一个SYN,并且带上对对方SYN的确认
            _receiver.segment_received(seg);
            _sender.send_empty_segment();
            send_segment();
        }
        
    }
    else if(_sender.next_seqno_absolute() == _sender.bytes_in_flight() 
    && _receiver.ackno().has_value() && !_receiver.stream_out().input_ended())
    {
        // TCP的receiver处于SYN-RECV状态，sender处于SYN-SENT状态，则TCP处于SYN-REVD状态 
        // 接收ACK，进入Established状态
        _sender.ack_received(seg.header().ackno, seg.header().win);
        _receiver.segment_received(seg);
    }
    else if(_sender.next_seqno_absolute() > _sender.bytes_in_flight() && !_sender.stream_in().eof())
    {
        //_sender还有数据未确认且上层还有数据未发送完毕，此时TCP处于Established状态
        _sender.ack_received(seg.header().ackno, seg.header().win);
        _receiver.segment_received(seg);
        if(seg.length_in_sequence_space() > 0) //收到数据，更新ACK
        {
            _sender.send_empty_segment();
        }
        _sender.fill_window();
        send_segment();
    }
    else if(_sender.stream_in().eof() && _sender.next_seqno_absolute() == _sender.stream_in().bytes_written() + 2
        && _sender.bytes_in_flight() > 0 && !_receiver.stream_out().input_ended())
    {
        //TCP的receiver处于SYN-RECV状态，sender处于FIN-SENT状态，则TCP处于FIN_WAIT_1状态
        if(seg.header().fin)
        {
            // 由TCP四次挥手可知，我方收到FIN，表明处于四次挥手第三阶段，发送新ACK，进入TIME_WAIT状态
            _sender.ack_received(seg.header().ackno, seg.header().win);
            _receiver.segment_received(seg);
            _sender.send_empty_segment();
            send_segment();
        }
        else if(seg.header().ack)
        {
            // 若收到ACK,则处于四次挥手第二阶段，进入FIN_WAIT_2状态
            _sender.ack_received(seg.header().ackno, seg.header().win);
            _receiver.segment_received(seg);
            send_segment();
        }
    }
    else if(_sender.stream_in().eof() && _sender.next_seqno_absolute() == _sender.stream_in().bytes_written() + 2
        && _sender.bytes_in_flight() == 0 && !_receiver.stream_out().input_ended())
    {
        //TCP的receiver处于SYN-RECV状态，sender处于FIN-ACKED状态，则TCP处于FIN_WAIT_2状态
        //if(seg.header().fin)
        // 无论收到FIN还是收到普通数据, 都发送ACK
        _sender.ack_received(seg.header().ackno, seg.header().win);
        _receiver.segment_received(seg);
        _sender.send_empty_segment();
        send_segment();
    }
    else if(_sender.stream_in().eof() && _sender.next_seqno_absolute() == _sender.stream_in().bytes_written() + 2
        && _sender.bytes_in_flight() == 0 && _receiver.stream_out().input_ended())
    {
        //TCP的receiver处于FIN-RECV状态，sender处于FIN-SENT状态，则TCP处于TIME_WAIT状态
        if(seg.header().fin)
        {
            //由四次挥手得知，TIME_WAIT会持续两个MSL，此时收到FIN，会发送ACK，确保对方成功关闭
            _sender.ack_received(seg.header().ackno, seg.header().win);
            _receiver.segment_received(seg);
            _sender.send_empty_segment();
            send_segment();
        }
    }
    else
    {
        _sender.ack_received(seg.header().ackno, seg.header().win);
        _receiver.segment_received(seg);
        _sender.fill_window();
        send_segment();
    }

}
```
write:
```
size_t TCPConnection::write(const string &data) {
    if(data.size() == 0) return 0;

    size_t _size = _sender.stream_in().write(data);
    _sender.fill_window();
    send_segment();
    return _size;
}
```
tick:
```
void TCPConnection::tick(const size_t ms_since_last_tick) { 

    if(!_is_ative) return;

    _time_since_last_segment_received += ms_since_last_tick;
    _sender.tick(ms_since_last_tick);

    if(_sender.consecutive_retransmissions() > TCPConfig::MAX_RETX_ATTEMPTS)
    {
        send_rst();
        return;
    }
    send_segment();

}
```
send_rst: //收到rst的反应
```
void TCPConnection::send_rst()
{
    //发送RST
    TCPSegment seg;
    seg.header().rst = true;
    _segments_out.push(seg);

    _receiver.stream_out().set_error();
    _sender.stream_in().set_error();
    _is_ative = false;
}
```
send_segment:
```
void TCPConnection::send_segment()
{
    //将sender的数据发送出去
    while(!_sender.segments_out().empty())
    {
        TCPSegment seg = _sender.segments_out().front();
        _sender.segments_out().pop();
        if(_receiver.ackno().has_value())
        {
            //设置segment的ackno和window_size
            seg.header().ack = true;
            seg.header().ackno = _receiver.ackno().value();
            seg.header().win = _receiver.window_size();
        }
        _segments_out.push(seg);
    }

    if(_receiver.stream_out().input_ended())
    {
        if(!_sender.stream_in().eof())
        {
            _linger_after_streams_finish = false;

        }
        else if(_sender.bytes_in_flight() == 0)
        {
            if(!_linger_after_streams_finish || time_since_last_segment_received() >= 10 * _cfg.rt_timeout)
            {
                _is_ative = false;
            }
        }
    }
}
```
connect:
```
void TCPConnection::connect() {
    _sender.fill_window(); // 主动发起连接，发送SYN
    send_segment();
}
```

实现效果
![image](https://user-images.githubusercontent.com/105581407/210371942-75d81f2e-024a-40b8-9eb0-2f942c2ed267.png)

benchmark测试
![image](https://user-images.githubusercontent.com/105581407/210372077-e2220bf5-6bc4-4530-a8f6-d2cc15556803.png)

5.问题汇总

用wireshark抓包的时候，发现一直找不到tun144和tun145两张虚拟网卡，各种百度后发现要先make check_lab4创建出两张网卡才行，然后在我make check_lab4后，一直报一个
../tun.sh: command not found的错误，百度说是权限原因，但是这是在makefile里执行的，更改权限也同样报错，后来在makefile文件里查找发现，文件执行目录是同级，但是
tun.sh文件是在上一级，有可能是我从github上merge时出的错，本身的文件应该是在同一级的，更改后成功执行

本节问题最多的还是在边界处理上，cs144文档中给出了两个关闭连接的情况，一个是不正常关闭，即收到或者接受了RST段；第二个是干净的关闭，满足四个条件
  - 入站流已全部组装完毕。
  - 出站流已经被本地应用程序结束，并完全发送(包括它结束的事实，即一个带有fin的段)到远端对等端。
  - 出流已被远端对等体完全确认。
  - 考虑两军问题，如果我方为主动关闭方，则在发送ack并等待一段时间后（通常为2MSL），我方没有收到对端发送的任何需要确认的内容，则认为连接已经完成，此时应关闭连接。
如果我方为被动关闭方，则考虑TCPConnection的入站流在TCPConnection发送FIN段之前结束，那么在满足前三个条件后，连接可以关闭。





