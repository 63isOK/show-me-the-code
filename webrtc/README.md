# pion/webrtc

首先是一个纯Go实现的webrtc api，也就是说脱离浏览器也可以运行，
估计还是没有浏览器那么多优化，不过也算是对webrtc协议的一个Go实现。
毕竟有时候只需要一个Go的实现，而不是整个浏览器。
这也是不通过swig/cgo方式，实现的第一个Go版本的开源库，
她的目标是做成一个社区，而不是为了商业或创建公司来把握pion，真正的开源。

pion/webrtc从2018/05开始创建，虽然年轻，但也发布了几十个release，一切都在变好。

我从2019/02开始入坑Go，5月开始接触pion，初始还分析过前面一段源码，
现在(2019/12)算是以另一种方式重新分析。

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
- 2019/04,发布v1.2.0,[发布说明](#v1.2.0)
  - 了解v1.2.0的[官方文档](/webrtc/v1.2.0-000.md)
  - 了解v1.2.0的[rtcerr](/webrtc/v1.2.0-001.md), 封装了webrtc的错误
  - 了解v1.2.0的[ice](/webrtc/v1.2.0-002.md)
  - 了解v1.2.0的[rtp](/webrtc/v1.2.0-003.md)
  - 了解v1.2.0的[rtp/codecs](/webrtc/v1.2.0-004.md)
  - 了解v1.2.0的[null](/webrtc/v1.2.0-005.md)
  - 了解v1.2.0的[media/samplebuilder](/webrtc/v1.2.0-006.md)
  - 了解v1.2.0的[media/ivfwriter](/webrtc/v1.2.0-007.md)
  - 了解v1.2.0的[datachannel](/webrtc/v1.2.0-008.md)
  - 了解v1.2.0的[util](/webrtc/v1.2.0-009.md)
  - 了解v1.2.0的[sdp](/webrtc/v1.2.0-010.md)
  - 了解v1.2.0的[srtp](/webrtc/v1.2.0-011.md)
  - 了解v1.2.0的[sctp](/webrtc/v1.2.0-012.md)
  - 了解v1.2.0的[internal/datachannel](/webrtc/v1.2.0-013.md)
  - 了解v1.2.0的[mux](/webrtc/v1.2.0-014.md)
  - 了解v1.2.0的[rtcp](/webrtc/v1.2.0-015.md)
  - 了解v1.2.0的[network](/webrtc/v1.2.0-016.md)
  - 了解v1.2.0的[webrtc](/webrtc/v1.2.0-017.md)
  - 了解v1.2.0的[demo](/webrtc/v1.2.0-018.md)

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
