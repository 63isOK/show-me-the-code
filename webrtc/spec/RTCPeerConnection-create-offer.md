# 创建offer

RTCPeerConnection核心的部分是由几个异步操作组成，也就是对外暴露的几个异步操作。
RTP媒体流的收发，现在由MediaStreamTrack对象来提供。当然这属于扩展接口。

第一个异步操作是[createOffer](https://www.w3.org/TR/webrtc/#methods)

## spec

createOffer()最后会创建一个sdp，一个符合rfc3264的sdp，
这个sdp里中会包含RTCPeerConnection中MediaStreamTrack的描述;
包含了webrtc实现(eg：pion)支持的编码/rtp/rtcp能力;
ice agent/dtls参数。

这个方法会带一个参数 offerOption，
这个参数会包含一些附加的控制信息(ice restart，音频检测，未来还会扩展的)。

有时有些系统会限制资源的使用(eg：解码器的个数)，createOffer()返回的sdp中，
也需要体现这点，能获取资源，setLocalDescription调用就会成功。
要保证setLocalDescription()不出错，至少在fulfillment回调完成之前不能出错。

sdp的创建要符合jsep协议，除非用户端将transceiver的停止理解为已停止

作为一个offer，应该包含了会话支持或系统能支持的所有codec/rtp/rtcp能力，
而answer正好相反，她只会包含要使用一个协商子集

一般都是先建立会话，后调用createOffer，这样生成的sdp才会兼容这个会话，
才会合并自最后一次offer-answer交换之后的变更，这些变更可能是增删track。

按ice协议第14节的描述，生成的sdp还需要包含ice agent的usernameFragment/password,
也就是我们提到过多次的ice-ufrag/ice-pwd。
除此之外，sdp还可能包含ice agent生成的本地ice候选。

RTCPeerConnection的构造会用到一个配置，这个配置里包含了一些证书。
这些证书和其他默认的证书，会用于生成证书指纹(certificate fingerprints),
这些证书指纹会用到sdp生成的过程中。

sdp包含很多媒体级的信息，这些信息都是当前系统的能力表现，
这些信息都是设备提供的，且是固定的，跨域的。在一些隐私敏感的场景，
浏览器可能会考虑只生成一些可暴露的能力对应的媒体级信息。

用户端调用createOffer()时的步骤：

- 用connection变量来存放RTCPeerConnection对象，就是这个连接对象要调用创建offer
- 如果connection.IsClosed 是true，拒绝这个异步操作，返回一个InvalidStateError错误
- 返回createOffer这个链式异步操作的结果

connection创建一个offer的步骤：

- 如果connection的信令状态不是stable/have-local-offer，返回一个InvalidStateError错误
- 创建一个异步操作p
- 并发执行"并发创建offer"
- 返回这个异步操作p

用connection和异步操作p，并发创建offer的步骤：

- 如果connection构造时未带证书，等待证书的生成
- 检查当前系统，确定生成offer时的可用资源
- 如果系统检查失败了，不管什么原因，返回一个OperationError错误，并拒绝异步操作p
- 查询任务队列，执行"创建offer的最后几步"

用connection和异步操作p，创建offer的最后几步：

- 如果connection.IsClosed是true，跳过下面的步骤
- 如果connection被改变了
  - 那需要进行offer生成的系统进行检查，并执行"并发创建offer",并跳过后面的步骤
- 根据上一步的检查信息/当前连接状态/还有RTCRtpTransceivers，生成sdp
  - 如果有启动bundle策略，那么每个m=媒体级都会关联到指定的bundle group

上面这段不太好理解，看下rfc对sdp中bundle复用协商的[描述](https://tools.ietf.org/html/draft-ietf-mmusic-sdp-bundle-negotiation-54#section-7)

整理完下面这节后,继续来看

用connection和异步操作p，创建offer的最后几步：

- 如果connection.IsClosed是true，跳过下面的步骤
- 如果connection被改变了
  - 那需要进行offer生成的系统进行检查，并执行"并发创建offer",并跳过后面的步骤
- 根据上一步的检查信息/当前连接状态/还有RTCRtpTransceivers，生成sdp
  - 如果有启动bundle策略，呼叫者必须标记一些媒体级用于协商，也就是推荐的过程
    - 推荐的媒体级应该和第一个未关闭的transceiver关联
    - 关联的好处是让远端不解析sdp就可以预测offer标记媒体级关联的transceiver是哪个
  - 这个媒体级关联的transceiver指定的优先编码格式，主要看RTCRtpTranceiver.PreferredCodecs的规则
    - 如果RTCRtpTranceiver.PreferredCodecs为空，那sdp中就不设置编码格式
    - transceiver.direction是sendrecv
      - 那排除不在下列列表中的所有编码格式
        - RTCRtpSender.getCapabilities(kind).codecs
        - RTCRtpReceiver.getCapabilities(kind).codecs
    - transceiver.direction是sendonly
      - 那排除不在下列列表中的所有编码格式
        - RTCRtpSender.getCapabilities(kind).codecs
    - transceiver.direction是recvonly
      - 那排除不在下列列表中的所有编码格式
        - RTCRtpReceiver.getCapabilities(kind).codecs
    - 不管如何，上面3个规则并不影响编码格式的优先级
  - RTCRtpSender.SendEncodings里的成员大于1，
    - 对于SendEncodings中的每中编码格式，给相应的媒体级添加一个a=rid send行
    - 然后在rid下添加一行 a=simulcast:send
    - rid 不能冲突
- 创建一个RTCSessionDescriptionInit对象，这就是生成的offer对象
  - type初始化成"offer"
  - sdp初始化为刚刚生成的sdp字符串
- LastCreatedOffer设置为sdp字符串
- 返回异步操作p和offer对象

## sdp offer/answer 处理(bundle复用后的协商处理)

这个目前还是草稿协议，是对sdp协议(rfc3264)的一个扩展。

[rfc](https://tools.ietf.org/html/draft-ietf-mmusic-sdp-bundle-negotiation-54)

1.2 bundle机制

sdp包含了多个媒体级信息，每个媒体级(m=)都会包含自己的ip:port,
这样会造成每个媒体级都有一个自己的transport，
bundle的目的就是使用一个transport，来传输sdp中的多个媒体级对应的媒体数据，
说白了也是复用的一种。
eg：以前有音频/视频/程序3路udp传输，现在只建立一个udp传输来传输各种媒体流，
这就是复用。底层的rtp/rtcp也有一个复用规则，之前的源码分析也提到过了，
这些复用的目的只有一个：提高性能，减少资源的使用。

提高性能：每个transport都会进行一次协商，光收集本地ice候选，这个重复的操作，
也随着协商次数的增加而重复着，特别是rtc环境，收集候选，协商都是浪费。

减少资源的使用：服务端的端口是非常紧缺的，复用会减少资源浪费

到目前分析pion为止，接触到了两类复用：rtp/rtcp复用;bundle复用。

启用bundle之后，transport就变成了bundle的ip:port，而这一属性集，
会适用到所有同一bundle group的所有媒体级(m=),这一机制适用于rtp媒体。

什么叫bundle group：

多个媒体级共用一个bundle transport，她们就组成一组，一个sdp可定义多个组，
每个组都有一个bundle transport来承载组内媒体级对应的数据传输。

rfc5888中定义了一个新的sdp组框架，叫BUNDLE，sdp如下：

    a=group:BUNDLE 0 1

有了这个BUNDLE组框架后，配合sdp的上层使用框架 offer/answer，
就可以在协商时知道哪些sdp媒体级信息属于BUNDLE组。

rfc3264(spd的offer/answer框架)也定义了：
offer/answer协商的bundle transport(ip:port)和bundle属性，
都会应用到同一个BUNDLE组里的其他媒体级信息中。

rfc8445(ice)中协商的传输通道，也可以作为bundle transport，那就非常好了。

一个bundle的ip:port只能与一个BUNDLE group关联，因为会存在多个组。
一个bundle group里可能有多个不同类型的媒体流，但底层对应的rtp会话，
只有一个。

向后兼容，bundle还要支持补齐用bundle策略的offer/answer。
所以在检测到一端不支持bundle时，提供了3中处理结果：纯bundle/最大兼容/平衡。
这是前面文章提到过的。

7 offer/answer处理：

这个草稿对sdp协议做了如下扩展：

- bundle group的协商
- 建议并选择带标签的m=部分
- 将m=部分添加到bundle group
- 从bundle group中移除一个m=部分
- 在bundle group中，disable一个m=部分

如果一个offer被拒绝了，
那么就会使用前一次协商好的ip:port/sdp参数/字符集/bundle属性。
所以，如果offer要协商一个bundle group，被拒绝了，那么这个bundle group就不会创建。

offer/answer可以包含多个bundle groups。

7.1 一般sdp注意事项

7.1.1 连接属性 c=

    c=<nettype> <addrtype> <connection-address>

如果要启动bundle，nettype必须是IN，表示internet，
addrtype必须是IP4/IP6,并且与m=中的地址类型保持一致。

要bundle机制支持其他网络或地址类型，就要等这个草稿协议进一步扩展。

7.1.2 带宽属性 b=

还要符合另一个草稿[协议](https://tools.ietf.org/html/draft-ietf-mmusic-sdp-mux-attributes-16)

7.1.3 扩展属性 a=

不管是offer还是answer，如果某个m=启用了bundle策略，那么都需要附加一些bundle的属性，
这些属性的处理过程符合sdp rfc标准，这个草稿协议扩展的属性如下：

- 在 initial offer(sdp会话的第一个offer，她表示bundle group协商的开始)中
  - 指定复用类别IDENTICAL and TRANSPORT
    - 如果媒体级(m=)是bundle-only，那么不需要指定
    - 如果媒体级不是bundle-only(要么是没启用bundle，要么不是第一个媒体级)，需要指定
  - 开启bundle之后，每个媒体级(m=)都可以设置不同的bundle属性集
  - 复用类型的定义在另一个[草稿协议](https://tools.ietf.org/html/draft-ietf-mmusic-sdp-mux-attributes-16)
  - 在每个m=(每个媒体级信息中)，都会包含bundle属性和属性值
  - 一旦某个媒体级(m=)被选中(协商通过)，那么她包含的bundle属性会应用到同一个BUNDLE group中的其他媒体级(m=)
- 在一个subsequent offer/answer(上次协商的后续)中
  - 分组的第一个媒体级需要指定IDENTICAL and TRANSPORT复用属性
  - 不是第一个的，不需要设置
  - 分组的第一个媒体级会将bundle属性应用到同组其他媒体级(m=)上
- 在sdp中，不管是offer还是answer，不管是第一个还是后续的
  - 如果媒体级指定了复用类别，那就不使用IDENTICAL and TRANSPORT
  - 媒体级的bundle属性集，跟着媒体级走

上面的结论是IDENTICAL and TRANSPORT复用类别的sdp属性，只适用于以下的媒体级：

- 首先必须是分组的第一个媒体级
- 其次要这个媒体级还没指定复用类别

7.2 生成一个初始的sdp offer

当呼叫者想通过BUNDLE group来协商，那么就可以通过第一个offer来协商，
这第一个offer可能是初始offer，也可以是上次协商的offer(rfc称为subsequent offer)。

为了进行bundle group协商，创建一个初始offer，下面是流程：

- 每个媒体级(m-)都需要带一个唯一的ip:port，前提是这个媒体级不是bundle-only
- 选一个媒体级，标记为建议，这个过程在 7.2.1中介绍
- 按 7.1.3 中的规则，为媒体级添加一些属性
- 会话级添加 group:BUNDLE属性
- 将每个媒体级的分组tag添加到group:BUNDLE分组tag列表

注意：呼叫者会为每个媒体级分配唯一的ip:port，准备从这些transport收数据，
直到收到的answer，找到了被呼叫方通过answer选择的tagged媒体级。

某个媒体级含有bundle-only属性，端口设置为0。仅当当前协商好的BUNDLE group
还有这个媒体级，此时，呼叫方才能请求被叫方接收这个媒体级。具体还是看下的说法。

如果媒体级端口是0,但并没有bundle-only，说明这个媒体级是要disable的

7.2.1 某一个媒体，被建议标记

在初始offer中，有bundle tag的都是建议标记的，也就是每个分组的第一个媒体级，
里面带有a=mid:foo，这个foo就是bundle tag。

这个被标记的媒体级，会有一个ip:port,如果被呼叫方选择了这个媒体级，
就使用这个作为收发数据的bundle transport。且这个媒体级的bundle属性就会应用到
组内其他的媒体级。

带bundle-only属性的媒体级不能推荐。

一般推荐的媒体级，不会被被呼叫者拒绝，或是被移除出bundle group。

7.2.2 例子分析

    SDP Offer

    v=0
    o=alice 2890844526 2890844526 IN IP6 2001:db8::3
    s=
    c=IN IP6 2001:db8::3
    t=0 0
    a=group:BUNDLE foo bar

    m=audio 10000 RTP/AVP 0 8 97
    b=AS:200
    a=mid:foo
    a=rtcp-mux
    a=rtpmap:0 PCMU/8000
    a=rtpmap:8 PCMA/8000
    a=rtpmap:97 iLBC/8000
    a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid

    m=video 10002 RTP/AVP 31 32
    b=AS:1000
    a=mid:bar
    a=rtcp-mux
    a=rtpmap:31 H261/90000
    a=rtpmap:32 MPV/90000
    a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid

- 这是一个启用了bundle的初始offer
- 有两个媒体级(m=)，一个音频，一个视频
- 这两个媒体都被tag了，就是说她们两个媒体级都在一个bundle group中
- 两个媒体级都被推荐了
- 推荐的优先级按bundle tag列表顺序来，所以音频媒体级是最推荐的，因为列表位置是第一个

7.3 生成一个answer

流程如下：

- 如果呼叫方在初始offer中请求创建一个BUNDLE group
  - 那么被呼叫者只能在初始answer中包含BUNDLE group
- 如果subsequent offer包含前次协商的BUNDLE group
  - 那么被呼叫者只能在subsequent answer中包含BUNDLE group
- offer中包含了某个媒体级
  - answer才能包含这个媒体级
- offer中包含了某个m=这行
  - answer才能在同一个BUNDLE group中包含这个媒体级

另外，answer中如果想包含BUNDLE group，要走以下流程：

- 如果是初始answer，用 7.3.1来处理offer中tagged的媒体级
- 如果是subsequent answer，不能修改subsequent offer中的媒体级
- 按 7.3.1 选择一个answer中的tagged 媒体级
- 将被叫方的 bundle ip:port 赋值给 answer中的tagged 媒体级
- 将同一个group总的其他媒体级，设置0端口，和bundle-only属性
- 按 7.1.3 中的规则，设置sdp属性
- answer的会话级中添加group:BUNDLE属性
- 把answer中，tagged媒体级的bundlt tag放到会话级属性的tag列表中

在answer中，如果不希望有某个媒体级，要么移除(7.3.2)，要么拒绝(7.3.3)

被呼叫方可以在subsequent answer中修改bundle ip:port,
或是增删sdp属性，或是修改spd属性的值

offer中如果媒体级端口是0,但并没有bundle-only，说明这个媒体级是要disable的

7.3.1 被呼叫方选择一个个tagged的媒体级

选择第一步是检查offer中推荐的tagged媒体级，以下条件是要满足的：

- answer是不会将媒体级移除出bundle group的
- 被呼叫方按 7.3.3 不会拒绝这个媒体级
- 媒体级的端口不能为0(为0表示要被disable)

所有条件满足后，那么被呼叫方会选中这个媒体级，然后创建一个相应的媒体级，并tagged。

如果有部分条件不满足，就选择offer推荐列表中的下一个媒体级进行检查。
最后，如果answer中的bundle tag列表为空，那就不能创建BUNDLE group了。
除非拒绝整个offer，不然被呼叫者必须在在bundle group中移除一个媒体级，或是拒绝一个。

拒绝一个媒体级，answer中只要将媒体级的端口设为0,不需要添加bundle-only,即可
