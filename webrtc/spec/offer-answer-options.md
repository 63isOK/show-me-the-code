# offer/answer 选项

[spec](https://www.w3.org/TR/webrtc/#offer-answer-options)

## spec

spec中定义了一个RTCOfferAnswerOptions结构，
派生了两个结构：RTCOfferOptions和RTCAnswerOptions。

这些结构(可选项)只影响offer/answer的创建流程。

RTCOfferOptions对父类做了一个扩展：添加了一个iceRestart成员，布尔型，默认false。

这个ice restart，是rfc5245(ice)中9.1.1.1定义的，就是offer/answer模型的扩展。
rfc5245 9.1.1.1对ice restart是这样描述的：

对于一个已存在的媒体流来说，agent可以重新执行ice处理，
(ice处理就是找出一个合理的连接通道)，重新执行ice处理(也就是ice restart)，
会导致ice处理以前的状态全部刷新，重新开始检查。
restart(媒体流已存在，重新开始检查)和start(一个全新的检查，为媒体流服务)的区别：
restart过程中，已存在的媒体流还是可以继续用之前的协程通道进行传输数据。

下面两种场景是必须要进行restart的：

- offer是为了更改媒体流
  - 此时agent生成一个变更后的offer，可以为媒体组件带来新的价值
- agent替换了另一个实现
  - 只会出现在第三方呼叫场景，第三方修改了媒体会话，底层ice库的实现都换了
  - 此时媒体会话id(mid)都变了，restart是最好的选择

如何触发restart：

sdp中的c=，是表示连接的，只需要将c=中的ip换成0.0.0.0,就会触发restart。
rfc还规定了，"呼叫保持call hold"不能利用restart来实现，而是要用
a=inactive/a=sendonly来实现。

除此之外，新offer中的ice-pwd/ice-ufrag都是需要改变的。
ps：ice-pwd/ice-ufrag可是是会话级，但后续媒体级的ice-pwd/ice-ufrag要保持一致。

回到webrtc spec中对restart的用法描述：

- iceRestart是true，或RTCPeerConnection.LocalIceCredentialsToReplace非空
  - 那么生成sdp时，证书(新sdp的)和当前sdp证书(已协商好的sdp的证书)是不同的
  - 而且会导致ice restart
- iceRestart是false，且RTCPeerConnection.LocalIceCredentialsToReplace为空
  - 那么生成sdp时，证书和当前sdp证书是一致的

webrtc spec还提到了，当ice连接状态是failed时，推荐进行ice restart。
应用程序可以监听ice连接状态的disconnected，然后利用getStats(获取但路媒体流信息)，
来决定ice restart是否可行。如果媒体流都断了，start是最好的;如果媒体流没断，
restart是最好的选择。

至于AnswerOption，webrtc spec中并没有进一步描述，等会看源码就行。

## pion/webrtc@v1.2.0

    type RTCOfferAnswerOptions struct {

      // webrtc 后续spec中会规划的参数
      // 按文档意思，是音频检测功能是否开启
      VoiceActivityDetection bool
    }

    type RTCAnswerOptions struct {
      RTCOfferAnswerOptions
    }

    type RTCOfferOptions struct {
      RTCOfferAnswerOptions

      // 为true
      // 表示证书是不一样的了，ice候选收集流程又重新开始了
      IceRestart bool
    }

源码中很简单，只是定义了几个结构，具体的用法可以看生成offer/answer的其他文章
