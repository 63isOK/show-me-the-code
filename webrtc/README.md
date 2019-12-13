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
- 2018/07,发布v1.0.0,[发布说明](/release)
  - 了解v1.0.0的[官方文档](/webrtc/v1.0.0-001.md)
  - 了解v1.0.0的[pkg/errors](/webrtc/v1.0.0-002.md)
  - 了解v1.0.0的[ice](/webrtc/v1.0.0-003.md)
  - 了解v1.0.0的[rtp](/webrtc/v1.0.0-004.md)
  - 了解v1.0.0的[rtp/codecs](/webrtc/v1.0.0-005.md)
  - 了解v1.0.0的[sdp](/webrtc/v1.0.0-006.md)
  - 了解v1.0.0的[util](/webrtc/v1.0.0-007.md)
  - 了解v1.0.0的[dtls](/webrtc/v1.0.0-008.md)
  - 了解v1.0.0的[network](/webrtc/v1.0.0-009.md)
  - 了解v1.0.0的[webrtc](/webrtc/v1.0.0-010.md)
  - 了解v1.0.0的[demo](/webrtc/v1.0.0-011.md)

## release

v1.0.0

- 第一个发布版本，仅支持以下特征
  - 音视频的收发
  - srtp库是纯Go
  - dtls是基于openssl(cgo方式集成)
  - 轻量级ice(要么是公网ip，要么是LAN)，后面会继续丰富
- 附带demo，可集成到自己的程序中
  - 通过gs(gstreamer)来收发视音频
  - 录制vp8视频
