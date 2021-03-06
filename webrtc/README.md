# pion/webrtc

## 目录

<!-- vim-markdown-toc GFM -->

- [分析过程](#分析过程)
- [webrtc标准熟悉](#webrtc标准熟悉)
- [release](#release)
  - [v1.0.0](#v100)
  - [v1.1.0](#v110)
  - [v1.2.0](#v120)
  - [v2.0.0](#v200)
  - [v2.2.0](#v220)

<!-- vim-markdown-toc -->

首先是一个纯Go实现的webrtc api，也就是说脱离浏览器也可以运行，
估计还是没有浏览器那么多优化，不过也算是对webrtc协议的一个Go实现。
毕竟有时候只需要一个Go的实现，而不是整个浏览器。
这也是不通过swig/cgo方式，实现的第一个Go版本的开源库，
她的目标是做成一个社区，而不是为了商业或创建公司来把握pion，真正的开源。

pion/webrtc从2018/05开始创建，虽然年轻，但也发布了几十个release，一切都在变好。

我从2019/02开始入坑Go，5月开始接触pion，初始还分析过前面一段源码，
现在(2019/12)算是以另一种方式重新分析。

## 分析过程

- 2018/05,项目创建
- 2018/07,发布v1.0.0,[发布说明](#v1.0.0)
  - 了解v1.0.0的[官方文档](/webrtc/v1.0.0-001.md)
  - 了解v1.0.0的[pkg/errors](/webrtc/v1.0.0-002.md)
  - 了解v1.0.0的[ice](/webrtc/v1.0.0-003.md),与其说是库，还不如说辅助功能
  - 了解v1.0.0的[rtp](/webrtc/v1.0.0-004.md),提供了将字节流打包整rtp包的能力
  - 了解v1.0.0的[rtp/codecs](/webrtc/v1.0.0-005.md),提供opus/vp8切片和提取包的能力
  - 了解v1.0.0的[sdp](/webrtc/v1.0.0-006.md),粗略解析sdp，颗粒度较大
  - 了解v1.0.0的[util](/webrtc/v1.0.0-007.md), 纯工具包
  - 了解v1.0.0的[dtls](/webrtc/v1.0.0-008.md),c实现的dtls，用Go封装了一下
  - 了解v1.0.0的[network](/webrtc/v1.0.0-009.md),udp监听的封装，处理了网络的收发
  - 了解v1.0.0的[webrtc](/webrtc/v1.0.0-010.md),对外暴露了RTCPeerConnection
  - 了解v1.0.0的[demo](/webrtc/v1.0.0-011.md),因为dtls不是纯Go，就部分细demo了
- 2018/11,发布v1.1.0,[发布说明](#v1.1.0)
- 2018/12,发布v1.2.0,[发布说明](#v1.2.0)
  - 了解v1.2.0的[官方文档](/webrtc/v1.2.0-000.md)
  - 了解v1.2.0的[rtcerr](/webrtc/v1.2.0-001.md), 封装了webrtc的错误
  - 了解v1.2.0的[ice](/webrtc/v1.2.0-002.md),完整的ice连接协商过程，让调用方只关注实际数据的收发即可
  - 了解v1.2.0的[rtp](/webrtc/v1.2.0-003.md),提供rtp报文和字节流之间的转换能能力
  - 了解v1.2.0的[rtp/codecs](/webrtc/v1.2.0-004.md),opus/vp8没啥变化，新增了h264/g722的切片逻辑
  - 了解v1.2.0的[null](/webrtc/v1.2.0-005.md),附加了一个固定的校验零值的方法
  - 了解v1.2.0的[media/samplebuilder](/webrtc/v1.2.0-006.md),封装底层的rtp相关操作，对外暴露媒体编码之后的接口
  - 了解v1.2.0的[media/ivfwriter](/webrtc/v1.2.0-007.md),写磁盘
  - 了解v1.2.0的[datachannel](/webrtc/v1.2.0-008.md),定义两种datachannel支持的数据格式
  - 了解v1.2.0的[util](/webrtc/v1.2.0-009.md)
  - 了解v1.2.0的[sdp](/webrtc/v1.2.0-010.md),重构了sdp的实现,扩展了ice和字符串的转换
  - 了解v1.2.0的[srtp](/webrtc/v1.2.0-011.md),对srtp的一个实现，对rtp/rtcp提供了加解密功能
  - 了解v1.2.0的[sctp](/webrtc/v1.2.0-012.md),sctp的一个实现，用于datachannel传数据
  - 了解v1.2.0的[internal/datachannel](/webrtc/v1.2.0-013.md),暴露DataChannel的接口
  - 了解v1.2.0的[mux](/webrtc/v1.2.0-014.md), webrtc协议中接收socket的多路复用
  - 了解v1.2.0的[rtcp](/webrtc/v1.2.0-015.md),rtcp多种格式的实现
  - 了解v1.2.0的[network](/webrtc/v1.2.0-016.md),对底层网络连接的封装
  - 了解v1.2.0的[webrtc](/webrtc/v1.2.0-017.md),配置和连接的构造
    - webrtc的4大状态
      - [信令状态 RTCSignalingState](/webrtc/v1.2.0-018.md), 检查jsep状态机
      - [ice收集状态 RTCIceGatheringState](/webrtc/v1.2.0-019.md)，定义了3种
      - [p2p连接状态 RTCPeerConnectionState](/webrtc/v1.2.0-020.md),定义了6种
      - [ice连接状态](/webrtc/v1.2.0-021.md),定义了7种
    - webrtc sdp 模型
      - [RTCSessionDescription](/webrtc/v1.2.0-041.md),封装了sdp类型，用在连接对象中
    - webrtc 零碎的知识点
      - [注册一个编码类型](/webrtc/v1.2.0-032.md)
      - [ice服务地址的表示](/webrtc/v1.2.0-035.md)
      - [dtls证书的表示](/webrtc/v1.2.0-036.md)
      - [构造连接的参数：配置](/webrtc/v1.2.0-037.md)
      - [agent支持的编码格式：媒体引擎](/webrtc/v1.2.0-038.md)
      - [轨道](/webrtc/v1.2.0-039.md)
      - [基于轨道的传输通道封装](/webrtc/v1.2.0-040.md)
    - webrtc 核心业务逻辑
      - [创建webrtc底层网络连接](/webrtc/v1.2.0-033.md)
      - [创建datachannel底层网络连接](/webrtc/v1.2.0-034.md)
      - [RTCPeerConnection分析](/webrtc/v1.2.0-080.md)
  - 了解v1.2.0的[demo的整个框架](/webrtc/v1.2.0-024.md)
    - ~~[gs发送到浏览器](/webrtc/v1.2.0-025.md)~~
    - ~~[gs从浏览器接收](/webrtc/v1.2.0-026.md)~~
    - [接收摄像头数据并录制](/webrtc/v1.2.0-027.md)
    - [sfu分析](/webrtc/v1.2.0-028.md)
    - [中继](/webrtc/v1.2.0-029.md)
    - [datachannel传输](/webrtc/v1.2.0-030.md)
    - [datachannel和浏览器之间的传输](/webrtc/v1.2.0-031.md)
- 2020/02,发布v2.2.0,[发布说明](#v2.2.0)
  - 了解v2.2.0的[官方文档](/webrtc/v2.2.0-081.md)
  - 了解v2.2.0的[源码结构](/webrtc/v2.2.0-082.md)

## webrtc标准熟悉

[标准的链接](https://www.w3.org/TR/webrtc/)

- 1 介绍
- 2 符合标准
- 3 术语
- 4 p2p连接
  - 4.1 [p2p连接的介绍](/webrtc/spec/p2p-connection-introduction.md)
  - 4.2 配置
    - 4.2.7 [offer/answer选项](/webrtc/spec/offer-answer-options.md)
  - 4.4 [RTCPeerConnection接口](/webrtc/spec/RTCPeerConnection-interface.md)
    - 4.4.1 [操作流程](/webrtc/spec/RTCPeerConnection-operation.md)
      - 4.4.1.1 [RTCPeerConnection构造](/webrtc/spec/RTCPeerConnection-constructor.md)
      - 4.4.1.2 [链式异步操作](/webrtc/spec/RTCPeerConnection-chain-asynchronous-operation.md)
      - 4.4.1.3 [p2p连接状态的更新](/webrtc/spec/RTCPeerConnection-update-connection-state.md)
      - 4.4.1.4 [ice候选收集状态的更新](/webrtc/spec/RTCPeerConnection-update-ice-gathering-state.md)
      - 4.4.1.5 [设置sdp](/webrtc/spec/RTCPeerConnection-set-sdp.md)
      - 4.4.1.6 [设置配置](/webrtc/spec/RTCPeerConnection-set-configuration-flow.md)
    - 4.4.2 接口定义
      - [createOffer](/webrtc/spec/RTCPeerConnection-create-offer.md)
      - [createAnswer](/webrtc/spec/RTCPeerConnection-create-answer.md)
      - [setConfiguration](/webrtc/spec/RTCPeerConnection-set-configuration.md)
  - 4.6 sdp模型
    - 4.6.1[sdp类型 RTCSdpType](/webrtc/v1.2.0-022.md),对offer/answer做了扩展：临时answer
    - 4.6.2[RTCSessionDescription](/webrtc/v1.2.0-023.md)
  - 4.7 会话协商模型(v1.2.0未实现，暂不分析)
    - 4.7.3[会话协商模型的更新](/webrtc/v1.2.0-056.md)
  - 4.8 ice接口
    - 4.8.1[RTCIceCandidate接口](/webrtc/v1.2.0-057.md)
    - 4.8.2[RTCPeerConnectionIceEvent事件](/webrtc/v1.2.0-058.md)
    - 4.8.3[RTCPeerConnectionIceErrorEvent事件](/webrtc/v1.2.0-059.md)
  - 4.9 证书管理
- 5 rtp媒体接口
  - [基础知识补充](/webrtc/v1.2.0-042.md)
  - 5.1 [对RTCPeerConnection接口的扩展](/webrtc/v1.2.0-043.md)
    - 5.1.1 [处理远端MediaStreamTracks](/webrtc/v1.2.0-044.md)
  - 5.2 [RTCRtpSender接口](/webrtc/v1.2.0-045.md)
    - 5.2.1-5.2.12 [RTCRtpSender接口涉及到的数据结构](/webrtc/v1.2.0-047.md)
  - 5.3 [RTCRtpReceiver接口](/webrtc/v1.2.0-048.md)
  - 5.4 [RTCRtpTransceiver接口](/webrtc/v1.2.0-049.md)
    - 5.4.1 [simulcast联播功能](/webrtc/v1.2.0-050.md)
  - 5.5 [RTCDtlsTransport接口](/webrtc/v1.2.0-051.md)
    - 5.5.1 [RTCDtlsFingerprint数据结构](/webrtc/v1.2.0-052.md)
  - 5.6 [RTCIceTransport接口](/webrtc/v1.2.0-053.md)
    - 5.6.1-5.6.6 [RTCIceTransport接口涉及到的数据结构](/webrtc/v1.2.0-054.md)
  - 5.7 [RTCTrackEvent接口](/webrtc/v1.2.0-055.md)
- 6 p2p数据接口
  - 6.1 [对RTCPeerConnection接口的扩展](/webrtc/v1.2.0-063.md)
    - 6.1.1 [RTCSctpTransport接口](/webrtc/v1.2.0-064.md)
    - 6.1.2 [RTCSctpTransportState枚举](/webrtc/v1.2.0-065.md)
  - 6.2 [RTCDataChannel接口](/webrtc/v1.2.0-060.md)
  - 6.3 [RTCDataChannelEvent接口](/webrtc/v1.2.0-061.md)
  - 6.4 [gc，垃圾回收](/webrtc/v1.2.0-062.md)
- 7 p2p的DTMF(双音多频)
- 8 统计模型
  - 8.1 [统计模型的介绍](/webrtc/v1.2.0-066.md)
  - 8.2 [对RTCPeerConnection接口的扩展](/webrtc/v1.2.0-067.md)
  - 8.3 [RTCStatsReport接口](/webrtc/v1.2.0-068.md)
  - 8.4 [RTCStats接口](/webrtc/v1.2.0-068.md)
  - 8.5 [状态收集算法](/webrtc/v1.2.0-046.md)
  - 8.6 [统计的实现](/webrtc/v1.2.0-069.md)
- 9 [网络使用中，媒体流api的扩展](/webrtc/v1.2.0-070.md)
- 10 例子和调用流程
  - 10.1 [简单p2p例子](/webrtc/v1.2.0-071.md)
  - 10.2 [自带"热身"的高级p2p例子](/webrtc/v1.2.0-072.md)
  - 10.3 [联播例子](/webrtc/v1.2.0-073.md)
  - 10.4 [p2p data例子](/webrtc/v1.2.0-074.md)
  - 10.7 [完美协商例子](/webrtc/v1.2.0-075.md)
- 11 错误处理
  - 11.1 [RTCError接口](/webrtc/v1.2.0-076.md)
  - 11.2 [RTCErrorDetailType枚举](/webrtc/v1.2.0-077.md)
  - 11.3 [RTCErrorEvent接口](/webrtc/v1.2.0-078.md)
- 12 [事件汇总](/webrtc/v1.2.0-079.md)
- 13 隐私和安全选项
- 14 辅助功能

## release

### v1.0.0

- 第一个发布版本，仅支持以下特征
  - 音视频的收发
  - srtp库是纯Go
  - dtls是基于openssl(cgo方式集成)
  - 轻量级ice(要么是公网ip，要么是LAN)，后面会继续丰富
- 附带demo，可集成到自己的程序中
  - 通过gs(gstreamer)来收发视音频
  - 录制vp8视频

### v1.1.0

- 第二个发布版本，在第一个版本的基础上新增了如下特征
  - 全功能ice(v1.0.0的就是个辅助)
  - DataChannels支持
  - RTCP支持(有了这个，就可以实现sfu)

### v1.2.0

- 第三个版本，纯Go版本，新增特征如下
  - 支持原始rtp流接入(不是解码，重编码，而是rtp协议级直接支持)
  - 支持Trickle-ice(任何时候都可以添加ice候选)，google/firefox都支持
  - 支持rtcp reception，允许应用程序和rtcp包交互和触发
  - 传输部分重构
  - dtls用Go实现
  - srtp改进，并新增tag认证检查
  - rtcp的go-fuzz支持，(go-fuzz是一个随机测试)

### v2.0.0

- 第四个版本,也是改动非常多的一个版本
  - ortc支持
  - data channel优化
  - 日志和ice调试
  - 实验性的:quic/wasm支持
  - api格式,向spec靠拢
  - sctp的可读性提高
  - ice常规提名

### v2.2.0

- 2020.02.17发布
- 目前最近的一个版本
  - data channel性能提升
  - 为了更好的支持sfu,SettingEngine做了更多的扩展
  - turn支持tcp
  - 支持pcm
  - 轨道的重新协商
  - IVFReader解码器,纯go实现
  - vp9支持
