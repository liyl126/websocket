# websocket
C++解析websocket协议
转自 https://www.cnblogs.com/jice1990/p/5436532.html

WebSocket的C++服务器端实现
　　由于需要在项目中增加Websocket协议，与客户端进行通信，不想使用开源的库，比如WebSocketPP，就自己根据WebSocket协议实现一套函数，完全使用C++实现。

代码已经实现，放在个人github上面，地址：https://github.com/jice1001/websocket.git。下面进行解释说明：

一、原理

　　Websocket协议解析，已经在前面博客里面详细讲解过，可以参考博客http://www.cnblogs.com/jice1990/p/5435419.html，这里就不详细细说。

服务器端实现就是使用TCP协议，使用传统的socket流程进行绑定监听，使用epoll控制多路并发，收到Websocket握手包时候进行握手处理，握手成功便可进行数据收发。

二、实现

　　1、服务器监听

　　该部分使用的是TCP socket流程，首先是通过socket函数建立socket,通过bind函数绑定到某个端口，本例使用的是9000，然后通过listen函数开启监听，代码如下：

复制代码
    listenfd_ = socket(AF_INET, SOCK_STREAM, 0);
    if(listenfd_ == -1){
        DEBUG_LOG("创建套接字失败!");
        return -1;
    }
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(sockaddr_in));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(PORT);
    if(-1 == bind(listenfd_, (struct sockaddr *)(&server_addr), sizeof(server_addr))){
        DEBUG_LOG("绑定套接字失败!");
        return -1;
    }
    if(-1 == listen(listenfd_, 5)){
        DEBUG_LOG("监听失败!");
        return -1;
    }
复制代码
　　2、epoll控制多路并发

　　该部分使用的是epoll流程，首先在初始化时候使用epoll_create创建epoll句柄

epollfd_ = epoll_create(1024);
　　然后通过epoll_wait等待fd事件来临，当监听到是listenfd事件时候，说明是客户端连接服务器，就使用accept接受连接，然后注册该连接EPOLLIN事件，当epoll监听到EPOLLIN事件时候，即可进行握手和数据读取。代码如下：

复制代码
void ctl_event(int fd, bool flag){
    struct epoll_event ev;
    ev.data.fd = fd;
    ev.events = flag ? EPOLLIN : 0;
    epoll_ctl(epollfd_, flag ? EPOLL_CTL_ADD : EPOLL_CTL_DEL, fd, &ev);
    if(flag){
        set_noblock(fd);
        websocket_handler_map_[fd] = new Websocket_Handler(fd);
        if(fd != listenfd_)
            DEBUG_LOG("fd: %d 加入epoll循环", fd);
    }
    else{
        close(fd);
        delete websocket_handler_map_[fd];
        websocket_handler_map_.erase(fd);
        DEBUG_LOG("fd: %d 退出epoll循环", fd);
    }
}
复制代码
复制代码
int epoll_loop(){
    struct sockaddr_in client_addr;
    socklen_t clilen;
    int nfds = 0;
    int fd = 0;
    int bufflen = 0;
    struct epoll_event events[MAXEVENTSSIZE];
    while(true){
        nfds = epoll_wait(epollfd_, events, MAXEVENTSSIZE, TIMEWAIT);
        for(int i = 0; i < nfds; i++){
            if(events[i].data.fd == listenfd_){
                fd = accept(listenfd_, (struct sockaddr *)&client_addr, &clilen);
                ctl_event(fd, true);
            }
            else if(events[i].events & EPOLLIN){
                if((fd = events[i].data.fd) < 0)
                    continue;
                Websocket_Handler *handler = websocket_handler_map_[fd];
                if(handler == NULL)
                    continue;
                if((bufflen = read(fd, handler->getbuff(), BUFFLEN)) <= 0){
                    ctl_event(fd, false);
                }
                else{
                    handler->process();
                }
            }
        }
    }

    return 0;
}
复制代码
　　3、Websocket握手连接

　　握手部分主要是根据Websocket握手包进行解析，然后根据Sec-WebSocket-Key进行SHA1哈希，生成相应的key,返回给客户端，与客户端进行握手。代码如下：

复制代码
//该函数是获取websocket握手包的信息，按照分割字符进行解析
int fetch_http_info(){
    std::istringstream s(buff_);
    std::string request;

    std::getline(s, request);
    if (request[request.size()-1] == '\r') {
        request.erase(request.end()-1);
    } else {
        return -1;
    }

    std::string header;
    std::string::size_type end;

    while (std::getline(s, header) && header != "\r") {
        if (header[header.size()-1] != '\r') {
            continue; //end
        } else {
            header.erase(header.end()-1);    //remove last char
        }

        end = header.find(": ",0);
        if (end != std::string::npos) {
            std::string key = header.substr(0,end);
            std::string value = header.substr(end+2);
            header_map_[key] = value;
        }
    }

    return 0;
}
复制代码
复制代码
//该函数是根据websocket返回包的格式拼接相应的返回包
void parse_str(char *request){  
    strcat(request, "HTTP/1.1 101 Switching Protocols\r\n");
    strcat(request, "Connection: upgrade\r\n");
    strcat(request, "Sec-WebSocket-Accept: ");
    std::string server_key = header_map_["Sec-WebSocket-Key"];
    server_key += MAGIC_KEY;

    SHA1 sha;
    unsigned int message_digest[5];
    sha.Reset();
    sha << server_key.c_str();

    sha.Result(message_digest);
    for (int i = 0; i < 5; i++) {
        message_digest[i] = htonl(message_digest[i]);
    }
    server_key = base64_encode(reinterpret_cast<const unsigned char*>(message_digest),20);
    server_key += "\r\n";
    strcat(request, server_key.c_str());
    strcat(request, "Upgrade: websocket\r\n\r\n");
}
复制代码
　　4、数据读取

　　当服务器与客户端握手成功后，就可以进行正常的通信，读取数据了。使用的是TCP协议的方法，解析Websocket包根据协议格式，在前面博客里面有详细分析，这里只把实现代码贴出来。

复制代码
int fetch_websocket_info(char *msg){
    int pos = 0;
    fetch_fin(msg, pos);
    fetch_opcode(msg, pos);
    fetch_mask(msg, pos);
    fetch_payload_length(msg, pos);
    fetch_masking_key(msg, pos);
    return fetch_payload(msg, pos);
}

int fetch_fin(char *msg, int &pos){
    fin_ = (unsigned char)msg[pos] >> 7;
    return 0;
}

int fetch_opcode(char *msg, int &pos){
    opcode_ = msg[pos] & 0x0f;
    pos++;
    return 0;
}

int fetch_mask(char *msg, int &pos){
    mask_ = (unsigned char)msg[pos] >> 7;
    return 0;
}

int fetch_masking_key(char *msg, int &pos){
    if(mask_ != 1)
        return 0;
    for(int i = 0; i < 4; i++)
        masking_key_[i] = msg[pos + i];
    pos += 4;
    return 0;
}

int fetch_payload_length(char *msg, int &pos){
    payload_length_ = msg[pos] & 0x7f;
    pos++;
    if(payload_length_ == 126){
        uint16_t length = 0;
        memcpy(&length, msg + pos, 2);
        pos += 2;
        payload_length_ = ntohs(length);
    }
    else if(payload_length_ == 127){
        uint32_t length = 0;
        memcpy(&length, msg + pos, 4);
        pos += 4;
        payload_length_ = ntohl(length);
    }
    return 0;
}

int fetch_payload(char *msg, int &pos){
    memset(payload_, 0, sizeof(payload_));
    if(mask_ != 1){
        memcpy(payload_, msg + pos, payload_length_);
    }
    else {
        for(uint i = 0; i < payload_length_; i++){
            int j = i % 4;
            payload_[i] = msg[pos + i] ^ masking_key_[j];
        }
    }
    pos += payload_length_;
    return 0;
}
复制代码
　　5、总结

　　到此为止，完整实现了使用C++对Websocket协议进行解析，握手，数据收发，不借助开源库就实现了websocket相关功能，最大程度的与项目保存兼容。
