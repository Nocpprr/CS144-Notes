Lab Checkpoint 1: stitching substrings into a byte stream

1. Overview

2. Getting started
由于我是从自己的github仓库里拉取的lab0的sponge，所以需要
```
git remote add origin2 git@github.com:CS144/sponge.git # 连接到CS144课程仓库
git branch lab1-startercode #创建本地lab1-startercode分支
git checkout lab1-startercode
git pull origin2 lab1-startercode # 拉取课程分支内容
git branch --set-upstream-to=origin2/lab1-starterode # 关联远程分支
git add .
git commit -m "lab1" # 提交, 如果不提交直接merge, 会出现needs merge 错误
git checkout master # 切回master分支
git merge lab1-startercode # 合并分支
```
3. Putting substrings in sequence
![StreamReassembler 的 capacity 示意图]:https://raw.githubusercontent.com/huangrt01/Markdown-Transformer-and-Uploader/mynote/Notes/Computer-Networking-Lab-CS144-Stanford/reassembler.png
 - 需要注意的是 StreamReassembler 的 capacity 和 ByteStream 的 capacity 是共同的
 - 这意味着push_substring读入的数据不能超过当前已读入数据长度和byte_stream剩余容量的总和
 - 假如上层read了数据，那么byte_stream的容量会得到释放，push_substring可读取的数据也会增加
 - first unacceptable = nread + capacity
 - 可以使用hash来记录已读取的数据位置来解决重复数据和过期数据

```
ByteStream _output;  //!< The reassembled in-order byte stream
    size_t _capacity;    //!< The maximum number of bytes
    size_t _unassembled_bytes; // 未发送至_output的大小
    std::string _datas; //存放数据
    std::unordered_set< size_t > _is_write; // 是否已写入
    size_t _next_index; // 下一个要写入的index
    bool _eof = false;
    size_t _eof_index = 0;
```
```
void StreamReassembler::push_substring(const string &data, const size_t index, const bool eof) {

    if(index >= _next_index + _output.remaining_capacity()) # 判断右边界
    {
        return;
    }


    if(eof && index + data.size() <= _next_index + _output.remaining_capacity())
    {
        _eof = true;
        _eof_index = index + data.size();
    }

    if(index + data.size() > _next_index) # 非过期数据
    {
        for(size_t i = (index > _next_index ? index : _next_index); i < _next_index + _output.remaining_capacity()
        && i < index + data.size(); ++i) # 起始位置取大的一个，减少循环次数
        {
            if(_is_write.count(i) == 0)
            {
                // 扩容 _datas
                if(_datas.capacity() <= i)
                {
                    _datas.reserve(i * 2);
                }
                _datas[i] = data[i - index];
                _is_write.insert(i);
                ++_unassembled_bytes;
            }
        }
        std::string _writes;
        while(_is_write.count(_next_index) > 0) #如果数据不是重复数据，采用单个字符形式来判断
        {
            _writes += _datas[_next_index]; 
            ++_next_index;
            --_unassembled_bytes;
        }
        _output.write(_writes); # 写入byte_stream
    }
    
    if(_eof && empty())
    {
        _output.end_input();
    }

}
```





