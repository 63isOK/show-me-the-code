# 设置sdp

[spec](https://www.w3.org/TR/webrtc/#set-the-rtcsessiondescription)

## spec

设置sdp，可处理两种，一种是本地sdp，一种是远端sdp，
spec规定，应该有一个设置函数，入参应该有个boolean表示是远端sdp还是本地sdp。

整个处理流程应该如下：

- 如果sdp的类型是rollback
  - 如果信令状态是stable/have-local-pranswer/have-remote-pranswer
  - 拒绝这次请求，并返回一个InvalidStateError错误
- 将此次操作，标记为一个异步操作
- 并行执行设置sdp的流程，其中有个限制:
如果这个sdp会导致收发器transceiver的修改,
并且这个收发器transceiver.Sender.SendEncodings非空,
并且，发送编码和sdp中的还不一致,
那么设置sdp会失败,
这个spec规定了，并不允许重新进行RID协商
  - 如果设置sdp的过程出错了，要走以下流程
    - RTCPeerConnection.IsClosed是true，退出
    - 结合当前信令状态，如果sdp类型是无效的，就返回一个InvalidStateError错误
    - 如果sdp内容并不是一个有效的sdp格式，就返回一个errorDetail错误
      - 这个错误中还要设置sdp-syntax-error，sdp哪一行出错信息
    - 如果是远程sdp，而且连接的RTCRtcpMuxPolicy是require(启用了rtcp/rtp复用)
      - 如果sdp中并未启用rtcp复用，那么返回一个InvalidAccessError错误
    - 如果sdp试图重新协商RID，返回InvalidAccessError错误
    - 如果sdp内容是无效的，返回InvalidAccessError错误
    - 如果是其他错误，返回OperationError错误
  - 如果sdp设置成功了，还要做以下流程
    - RTCPeerConnection.IsClosed是true，退出
    - 如果sdp是offer，信令状态是stable，那对连接中的每个transceiver做以下流程
      - transceiver.Sender.LastStableStateSenderTransport = transceiver.Sender.SenderTransport
      - transceiver.Receiver.LastStableStateReceiverTransport = transceiver.Receiver.ReceiverTransport
      - transceiver.Receiver.LastStableStateAssociatedRemoteMediaStreams = transceiver.Receiver.AssociatedRemoteMediaStreams
      - transceiver.Receiver.LastStableStateReceiveCodecs = transceiver.Receiver.ReceiveCodecs
    - 如果是本地sdp，执行以下流程
      - 如果是offer
        - RTCPeerConnection.PendingLocalDescription = 这个sdp构造的RTCSessionDescription
        - 信令状态改为have-local-offer
        - 释放早期ice候选
      - 如果是answer
        - 这表示offer/answer协商的完结
        - RTCPeerConnection.CurrentLocalDescription = 这个sdp构造的RTCSessionDescription
        - RTCPeerConnection.CurrentRemoteDescription = RTCPeerConnection.PendingRemoteDescription
        - RTCPeerConnection.PendingRemoteDescription = null
        - RTCPeerConnection.PendingLocalDescription = null
        - RTCPeerConnection.LastCreatedOffer = ""
        - RTCPeerConnection.LastCreatedAnswer = ""
        - 信令状态设置为stable
        - 释放早期ice候选
        - 如果sdp中并没有ice证书，则RTCPeerConnection.LocalIceCredentialsToReplace设置为空
      - 如果是pranswer
        - RTCPeerConnection.PendingLocalDescription = 这个sdp构造的RTCSessionDescription
        - 信令状态改为have-local-pranswer
        - 释放早期ice候选
    - 如果远端sdp，执行以下流程
      - 如果是offer
        - RTCPeerConnection.PendingRemoteDescription = 这个sdp构造的RTCSessionDescription
        - 信令状态改为have-remote-offer
      - 如果是answer
        - RTCPeerConnection.CurrentRemoteDescription = 这个sdp构造的RTCSessionDescription
        - RTCPeerConnection.CurrentLocalDescription = RTCPeerConnection.PendingLocalDescription
        - RTCPeerConnection.PendingRemoteDescription = null
        - RTCPeerConnection.PendingLocalDescription = null
        - RTCPeerConnection.LastCreatedOffer = ""
        - RTCPeerConnection.LastCreatedAnswer = ""
        - 信令状态设置为stable
        - 如果sdp中并没有ice证书，则RTCPeerConnection.LocalIceCredentialsToReplace设置为空
      - 如果是pranswer
        - RTCPeerConnection.PendingRemoteDescription = 这个sdp构造的RTCSessionDescription
        - 信令状态改为have-remote-pranswer
    - 如果sdp是answer，会关闭已存在的sctp连接
      - RTCPeerConnection.SctpTransport = null
    - 清空以下列表
      - trackEventInits
      - muteTracks
      - addList
      - removeList
      - errorList
    - 如果sdp类型是answer或pranswer，执行以下步骤
      - 如果sdp中定义了要开启一个sctp连接
        - 创建一个RTCSctpTransport，并初始化状态为connecting
        - RTCPeerConnection.SctpTransport = 上面创建的对象
        - 如果sctp连接已经建立，只是更新了max-message-size属性
          - 更新RTCPeerConnection.SctpTransport的最大消息数
      - 如果sdp中带了sctp连接的dtls属性，对于每个带null id的RTCDataChannel：
        - 依据标准，生成一个新的id
        - 如果id生成失败
          - RTCDataChannelReadyState = closed
          - 将这个channel添加到errorList列表中
    - 清空以下列表
      - trackEventInits
      - muteTracks
      - addList
      - removeList
    - 如果sdp不是rollback类型，执行以下步骤
      - 如果是本地sdp，对于sdp中媒体级信息，做如下步骤
        - 如果这个媒体级信息还未赋值给RTCRtpTransceiver对象，执行以下步骤
          - 如果媒体级信息还未赋值到RTCRtpTransceiver对象，执行以下步骤
            - 为媒体级信息创建一个RTCRtpTransceiver实例
            - RTCRtpTransceiver.mid 设置为媒体级中的值
            - 如果RTCRtpTransceiver.Stopped 为true，退出
            - 如果媒体级涉及到bundle约束
              - 将其对应的transport设置为rtp/rtcp复用的RTCDtlsTransport
            - 如果媒体级不启用bundle约束
              - 创建一个新的RTCDtlsTransport实例，底层包含一个RTCIceTransport
              - transport = 上面创建的实例
            - transceiver.Sender.SenderTransport = 上面创建的RTCRtpTransceiver实例
            - transceiver.Receiver.ReceiverTransport = 上面创建的RTCRtpTransceiver实例
        - 如果这个媒体级信息已经赋值给某个RTCRtpTransceiver对象
          - 赋值给transceiver变量
        - 如果transceiver.Stopped是true，结束下面的子步骤，接着执行上层步骤
        - transceiver.direction = 媒体级中的值
        - 如果transceiver.direction是sendrecv/recvonly
          - transceiver.Reception = true
          - 其他情况设置为false
        - transceiver.Receiver.ReceiveCodecs = 协商的编码格式
          - 并准备好接收
          - 如果方向是sendonly/inactive，就不是接收，这个列表也会置空
        - 如果sdp类型是answer/pranswer，执行以下步骤
          - transceiver.Sender.SendCodecs = 协商中发送的编码格式
          - transceiver.Sender.LastReturnedParameters = null
          - 如果方向是sendonly/inactive且transceiver.FiredDirection是sendrecv/recvonly
            - 传入transceiver.Receiver/空列表/空列表/removeList 来"设置相关的远端流"
            - 传入transceiver/muteTracks 来"删除媒体级描述中的远端track"
          - transceiver.CurrentDirection = 媒体级中的方向
          - transceiver.FiredDirection = 媒体级中的方向
      - 如果是远端sdp，对每个媒体级描述，执行以下步骤
        - 如果是offer，包含联播属性(simulcast)
          - 利用联播属性中rid的顺序，为每个联播层新建一个RTCRtpEncodingParamters列表实例
          - 每个实例都填充相应的rid,并将这个列表赋值给sendEncodings属性
          - 如果不包含联播属性，那么sendEncodings设置为空
        - 将supportedEncodings尽可能设置为最大值(最多支持多少种编码格式)
          - 如果sendEncodings > supportedEncodings
            - 将sendEncodings列表进行截断，长度是supportedEncodings
        - 查找是否有表示媒体级信息的RTCRtpTransceiver对象，有就赋值给transceiver
        - 如果找到了，且sendEncodings非空(证明启动了联播功能)
          - transceiver.Sender.SendEncodings = sendEncodings
          - transceiver.Sender.LastReturnedParameters = null
        - 如果没有找到
          - 创建一个RTCRtpSender对象表示sender
          - 创建一个RTCRtpReceiver对象表示receiver
          - 利用上面创建的两个对象，创建一个RTCRtpTransceiver
          - RTCRtpTransceiver.RTCRtpTransceiverDirection = recvonly
          - 将新创建的RTCRtpTransceiver添加到RTCPeerConnection的传输通道列表中
