# 创建answer

## spec

[rfc](https://www.w3.org/TR/webrtc/#dom-rtcpeerconnection-createanswer)

sdp answer,生成的参数要于offer是兼容的。
sdp中协商的参数都可以用answer做最后的决定。

offer表示了呼叫者机器的状态，answer也是类似，需要表示被呼叫者的状态。

在调用setLocalDescription期间，不能出错，至少fulfillment回调返回前不能出错。

按照jsep对answer的扩展，answer中是需要包含编码信息，ice的usernameFragment/password，
ice可选项，本地ice候选。

RTCPeerConnection的证书是用来生成指纹的，sdp的构造会用到这些指纹

如果是临时answer，那么sdp的类型就是pranswer

生成answer的流程：

- 用connection对象来表示RTCPeerConnection
- 如果connection.IsClosed是true，返回一个InvalidStateError错误
- 创建一个链式异步操作，来执行创建answer的任务

链式异步操作中，创建answer任务的步骤如下：

- 如果信令状态不是have-remote-offer/have-local-pranswer，返回一个InvalidStateError错误
- 创建一个异步操作promise，叫p，用来执行具体的异步动作
- 用p来来并行执行创建answer，这一步是符合js调用的，其他语句的实现可能不同

并行执行创建answer:

- 如果连接对象没有证书，那就等待证书生成，因为sdp需要指纹，而指纹是由证书生成的
- 检查本机状态(也是检查本机可用资源)
- 检查失败就返回一个OperationError的错误，并跳过后面步骤
- 执行具体的创建过程

具体的创建过程：

- connection.IsClosed 是true，跳过后面步骤
- 如果connectin被改变了，那么重新检查系统状态，重新执行具体的创建过程
- 利用检查信息/连接的当前状态信息/RTCRtpTransceiver，生成一个sdp字符串
  - 对于媒体级，如果有偏好编码格式，就在sdp中体现，如果没有就不设置
    - 主要看RTCRtpTranceiver.PreferredCodecs指定的列表
    - 如果direction是sendrecv，不在下列范围的编码格式都会被排除
      - RTCRtpSender.getCapabilities(kind).codecs
      - RTCRtpReceiver.getCapabilities(kind).codecs
    - 如果direction是sendonly，不再下列返回的编码格式都会被排除
      - RTCRtpSender.getCapabilities(kind).codecs
    - 如果direction是recvonly，不再下列返回的编码格式都会被排除
      - RTCRtpReceiver.getCapabilities(kind).codecs
  - 如果启用了联播(SendEncodings列表元素的数量大于1)
    - 对于每个媒体级，每个编码格式，添加一个a=rid send和a=simulcast:send
    - rid要保持不能冲突
- 创建一个RTCSessionDescriptionInit，类型是answer，sdp就是上一步生成的sdp字符串
- 返回p和answer
