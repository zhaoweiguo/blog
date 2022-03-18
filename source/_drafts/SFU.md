

## 从大的方面可以分为 3 部分:

1. WebRTC 终端
    负责音视频采集、编解码、NAT 穿越、音视频数据传输
2. Signal 服务器
    负责信令处理，如加入房间、离开房间、媒体协商消息的传递等
3. STUN/TURN 服务器
    负责获取 WebRTC 终端在公网的 IP 地址，以及 NAT 穿越失败后的数据中转



多方通信架构无外乎以下三种方案:

1. Mesh 方案，即多个终端之间两两进行连接，形成一个网状结构。
  这种方案对各终端的带宽要求比较高。
2. MCU（Multipoint Conferencing Unit）方案
  该方案由一个服务器和多个终端组成一个星形结构。
  实际上服务器端就是一个音视频混合器，这种方案服务器的压力会非常大。
3. SFU（Selective Forwarding Unit）方案
  该方案也是由一个服务器和多个终端组成，
  但与 MCU 不同的是，SFU 不对音视频进行混流，
  收到某个终端共享的音视频流后，就直接将该音视频流转发给房间内的其他终端。
  它实际上就是一个音视频路由转发器。


SFU 方案时，包括 Simulcast 模式和 SVC 模式
SFU 方案开源项目包括 Licode、Janus-gateway、MediaSoup、Medooze








