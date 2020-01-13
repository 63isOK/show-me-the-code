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
        - 如果sdp类型是answer/pranswer，且transceiver.Sender.SendEncodings多于一个(启动了联播)
          - 如果sdp中不支持联播
            - transceiver.Sender.SendEncodings只保留第一个，其他删除
            - 跳过剩下的子步骤
          - 如果sdp不支持分层(联播中每一层表示一个质量)
            - transceiver.Sender.SendEncodings中删除和分层相关的编码格式
          - 对于联播中的每层
            - 设置active属性为true/false，可以暂停/恢复某一层的流
        - transceiver.mid的值设置为sdp中的值
          - 如果sdp中没有mid，就随机生成一个
        - direction设置成sdp中的值
          - 数据发送者设置为receive
          - 数据接收者设置为send
          - 两者是颠倒的
        - 如果direction是sendrecv/recvonly
          - msids设置为transceiver.Receiver.ReceiverTranck中msids的列表
          - direction是其他值时，msids设置为空
        - 传入transceiver.Receiver/msids/addList/removeList 来"设置相关的远端流"
        - 如果direction是sendrecv/recvonly
          - 如果transceiver.FiredDirection不是sendrecv/recvonly
            - 如果上一步(设置相关远端流)还添加了元素到addList
              - 传入transceiver/trackEventInits 来"删除媒体级描述中的远端track"
        - 如果direction是sendonly/inactive
          - transceiver.Receptive = false
        - 如果direction是sendonly/inactive
          - 如果transceiver.FiredDirection是sendrecv/recvonly
            - 传入transceiver/muteTracks 来"删除媒体级描述中的远端track"
        - transceiver.FiredDirection = direction
        - transceiver.Receiver.ReceiveCodecs = sdp媒体级中准备接收的编码格式
        - 如果sdp类型是answer/pranswer
          - transceiver.Sender.SendCodecs = sdp媒体级中准备发送的编码格式
          - transceiver.CurrentDirection = direction
          - transceiver.Direction = direction
          - 依据bundle策略，将transport设置为rtp/rtcp复用的那个RTCDtlsTransport实例
          - transceiver.Sender.SenderTransport = transport
          - transceiver.Receiver.ReceiverTransport = transport
          - 设置transport的ice角色
            - 如果ice角色是unknow，就修改transport.IceRole
            - 如果sdp是本地offer，transport.IceRole = controlling
            - 如果sdp是远端offer
              - 如果sdp中还包含了"a=ice-lite"
                - transport.IceRole = controlling
              - sdp不包含"a=ice-lite"
                - transport.IceRole = controlled
        - 如果这个媒体级未协商成功(被拒绝了)，且transceiver.Stopped 是false
          - 关闭这个RTCRtpTransceiver
    - 如果sdp类型是rollback
      - 对于RTCPeerConnection.transceivers列表中的每个transceiver：
        - 在设置RTCSessionDescription之前，如果transceiver并未和媒体级sdp关联
          - 回滚就很简单，不关联，且将transceiver.mid 设置为null
        - transceiver.Sender.SenderTransport = transceiver.Sender.LastStableStateSenderTransport
        - transceiver.Receiver.ReceiverTransport = transceiver.Receiver.LastStableStateReceiverTransport
        - 传入transceiver.Receiver/transceiver.Receiver.LastStableStateAssociatedRemoteMediaStreams/
addList/removeList 来"设置相关的远端流"
        - transceiver.Receiver.ReceiveCodecs = transceiver.Receiver.LastStableStateReceiveCodecs
        - 如果要回滚通过RTCSessionDescription创建transceiver，且还没有调用addTrack来添加轨道
          - 如果transceiver.FiredDirection是sendrecv/recvonly
            - 传入transceiver/muteTracks 来"删除媒体级描述中的远端track"
            - transceiver.FiredDirection = inactive
          - 停止这个transceiver
          - 从RTCPeerConnection.transceivers中移除这个transceiver
        - 其他情况(不像上面那几种简单情况)
          - 如果transceiver.FiredDirection 是sendonly/inactive
            - 如果transceiver.CurrentDirection 是sendrecv/recvonly，
或是上一步的"删除媒体级描述中的远端track"导致addList增加了元素
              - 传入transceiver/trackEventInits 来"删除媒体级描述中的远端track"
              - transceiver.Receptive = true
          - 如果transceiver.FiredDirection 是sendrecv/recvonly
            - 如果transceiver.CurrentDirection 是sendonly/inactive/null
              - 传入transceiver/muteTracks 来"删除媒体级描述中的远端track"
              - transceiver.Receptive = false
          - transceiver.FiredDirection = transceiver.CurrentDirection
      - RTCPeerConnection.PendingLocalDescription = null
      - RTCPeerConnection.PendingRemoteDescription = null
      - RTCPeerConnection的信令状态设置为stable
    - 如果sdp是answer类型
      - 对于RTCPeerConnection.transceivers列表中的每个transceiver：
        - 如果transceiver已经stopped，或是她对应的m=媒体级被拒绝了
          - 从RTCPeerConnection.transceivers列表中删除
    - 如果RTCPeerConnection的信令状态是stable
      - "更新协商标记"
      - 如果更新操作执行前后，RTCPeerConnection.NegotiationNeeded都是true
        - 那么将下面几步作为一个异步操作添加到任务队列中
          - 如果RTCPeerConnection.IsClosed是true，跳过之后的步骤
          - 如果RTCPeerConnection.NegotiationNeeded 是false，跳过之后的步骤
          - 触发一个叫negotiationneeded的事件
    - 上一步中，如果信令状态改变了，触发一个叫signalingstatechange的事件
    - 对于每个在errorList中的datachannel
      - 使用RTCErrorEvent接口触发一个叫error的事件，接口的errorDetail属性设置为"打他-channel-failure"
    - 遍历muteTracks，将每个track的静音状态改为true
    - 遍历removeList，将指定的track从stream中删除
    - 遍历addList，添加track到stream中
    - 对于每个trackEventInits实例，使用RTCTrackEvent接口触发一个叫track的事件
      - 接口的receiver属性初始化为每个实例的receiver
      - 接口的track属性初始化为每个实例的track
      - 接口的streams属性初始化为每个实例的streams
      - 接口的transceiver属性初始化为每个实例的transceiver
    - 将这个设置sdp的异步操作解析为undefined
- 返回这个异步操作

## pion/webrtc@v1.2.0 对设置sdp的处理

### RTCPeerConnection中的4个sdp指针

- CurrentLocalDescription
  - 一个本地sdp，已经协商成功了，信令状态已经是stable了
  - 这个还包含了ice agent创建offer/answer之后，所生成的本地ice候选
- CurrentRomoteDescription
  - 一个远端sdp，已经协商成功了，信令状态已经是stable了
  - 这个还包含了offer/answer创建之后，任何通过AddIceCandidate()添加的远端ice候选
- PendingLocalDescription
  - 一个本地sdp，正在做协商
  - 这个还包含了ice agent创建offer/answer之后，所生成的本地ice候选
  - 信令状态转变为stable之后，这个值会置空，也就是nil
- PendingRomoteDescription
  - 一个远端sdp，正在做协商
  - 这个还包含了offer/answer创建之后，任何通过AddIceCandidate()添加的远端ice候选
  - 信令状态转变为stable之后，这个值会置空，也就是nil

### RTCPeerConnection中和sdp相关的处理方法

- offer/answer的生成
  - CreateOffer
  - CreateAnswer
- sdp的设置
  - SetLocalDescription
  - SetRemoteDescription
- 读取sdp信息
  - LocalDescription
  - RemoteDescription

下面依次来读这几个方法。

#### 生成offer

rtcpeerconnection.go

    func (pc *RTCPeerConnection) CreateOffer(
      options *RTCOfferOptions) (RTCSessionDescription, error)

    type RTCOfferAnswerOptions struct {
      // VoiceActivityDetection allows the application to provide information
      // about whether it wishes voice detection feature to be enabled or disabled.
      VoiceActivityDetection bool
    }

    // RTCAnswerOptions structure describes the options used to control the answer
    // creation process.
    type RTCAnswerOptions struct {
      RTCOfferAnswerOptions
    }

    // RTCOfferOptions structure describes the options used to control the offer
    // creation process
    type RTCOfferOptions struct {
      RTCOfferAnswerOptions

      // IceRestart forces the underlying ice gathering process to be restarted.
      // When this value is true, the generated description will have ICE
      // credentials that are different from the current credentials
      IceRestart bool
    }

在分析函数内部之前，先看下参数

spec中定义是这样的：

    Promise<RTCSessionDescriptionInit> 
      createOffer(optional RTCOfferOptions options = {});

可以看spec的4.2.7对应的分析。

回到创建offer创建的函数, 好吧，spec也有定义，所以分单独的篇章来分析4.4.2

createOffer/createAnswer 都可以看4.4.2
