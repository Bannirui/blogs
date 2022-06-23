### 一 [代码地址](https://github.com/Bannirui/spider.git)



### 二 先上手册

```c++
int kqueue(void); // 函数

EV_SET(&kev, ident, filter, flags, fflags, data, udata); // 宏

int kevent(int kq, const struct kevent *changelist, int nchanges, struct kevent *eventlist, int nevents, const struct timespec *timeout); // 函数
```

从手册上可以学到几点信息

1 `kqueue()`和`kevent()`都是一次系统调用

2 数据结构kevent是核心

3 kevent的赋值只能通过宏指令实现

4 `kevent()`函数适配不同的实参可以达到两种不同的功能

* 向os内核队列kqueue中注册要监听的fd事件

* os向用户返回事件就绪的kevent集合



### 三 实际使用

1 注册fd关注事件类型

```c++
void MyMulIO::add(const MyHttp &http, int16_t ev_filter)
{
    int fd = http.sock();
    struct kevent change;

    EV_SET(&change, fd, ev_filter, EV_ADD, 0, 0, nullptr);
    // system call 不阻塞
    int ret = kevent(_kqfd, &change, 1, nullptr, 0, nullptr);
    if (-1 == ret)
    {
        std::cout << "kq add error=" << errno << std::endl;
        exit(1);
    } else
    {
        _fd_http_map.insert({fd, http});
    }
}
```



2 获取发生事件的fd

```c++
int MyMulIO::dispatch(int num, int timeout, std::vector<std::string> &storage)
{
    struct kevent events[1024];
    struct timespec ts;
    ts.tv_sec = timeout / 1000;
    // TODO: 2022/6/22 超时怎么正确使用
    // system call 已经就绪的fd数量
    int n = kevent(_kqfd, nullptr, 0, events, 1024, nullptr);
    if (-1 == n)
    {
        std::cerr << "kevent() system call err, errno=" << errno << std::endl;
        return 0;
    }
    int read_ready_cnt = 0;
    for (int i = 0; i < n; i++)
    {
        struct kevent event = events[i];
        int ev_fd = (int) (event.ident); // 绑定的fd
        int ev_filter = (int) (event.filter); // 被关注的事件

        if (ev_filter == EV_ERROR)
        {
            std::cout << "kq event error" << std::endl;
            continue;
        }
        if (_fd_http_map.find(ev_fd) == _fd_http_map.end()) return n;
        if (ev_filter == EVFILT_READ)
        {
            // fd就绪 可读 读完改关注写事件
            std::string out = _fd_http_map[ev_fd]._response;
            _fd_http_map[ev_fd].recv_http_request(&out);
            _fd_http_map[ev_fd]._r->remove(_fd_http_map[ev_fd]);
            storage.push_back(std::move(out));
            read_ready_cnt++;
        }
        if (ev_filter == EVFILT_WRITE)
        {
            // fd就绪 可写 写完改关注读事件
            _fd_http_map[ev_fd].build_htt_request("/");
            _fd_http_map[ev_fd].send_http_request();
            _fd_http_map[ev_fd]._r->modify(_fd_http_map[ev_fd], EVFILT_READ);
        }
    }
    return read_ready_cnt;
}
```

