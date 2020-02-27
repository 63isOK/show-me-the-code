# p2p连接状态的更新

## 目录

<!-- vim-markdown-toc GFM -->

- [spec](#spec)
- [pion/webrtc@v1.2.0中连接状态的更新](#pionwebrtcv120中连接状态的更新)
- [最后](#最后)
- [最后的最后 TODO](#最后的最后-todo)

<!-- vim-markdown-toc -->

## spec

[spec](https://www.w3.org/TR/webrtc/#update-the-connection-state)

RTCPeerConnection中有一个字段表示p2p连接状态，
每当RTCDtlsTransport状态变更或isClosed字段变为true(连接关闭了)，
这个p2p连接状态都会被更新，具体流程如下：

- 获取连接对象
- 将新状态映射到RTCPeerConnectionState枚举中的一个
- 如果新状态和连接对象的状态一致，退出(没有必要进行变更)
- 如果不一致，就将连接对象的状态进行更新
- 并触发一个事件，connectionstatechange的事件

## pion/webrtc@v1.2.0中连接状态的更新

pion对连接状态做的比较简单，对象(RTCPeerConnection)构造时New(),
将连接对象初始化为new，连接关闭时Close()，置为close。

另外底层的udp连接如果出了问题，是如何反馈的？

在构造New中，创建network时，是这样调用的：

    pc.networkManager = network.NewManager(urls, pc.generateChannel, pc.iceStateChange)

第三个参数，就是外部传输的回调函数，当底层网络udp连接异常时，会调用这个回调。

    func (pc *RTCPeerConnection) iceStateChange(newState ice.ConnectionState) {
      pc.Lock()
      pc.IceConnectionState = newState
      pc.Unlock()

      pc.onICEConnectionStateChange(newState)
    }

因为dtls/srtp/ice，都是复用这个连接，所以p2p连接断开，是通过ice连接断开来通知的

    func (pc *RTCPeerConnection) OnICEConnectionStateChange(
      f func(ice.ConnectionState)) {

      pc.Lock()
      defer pc.Unlock()
      pc.onICEConnectionStateChangeHandler = f
    }

    func (pc *RTCPeerConnection) onICEConnectionStateChange
      (cs ice.ConnectionState) (done chan struct{}) {

      pc.RLock()
      hdlr := pc.onICEConnectionStateChangeHandler
      pc.RUnlock()

      done = make(chan struct{})
      if hdlr == nil {
        close(done)
        return
      }

      go func() {
        hdlr(cs)
        close(done)
      }()

      return
    }

可以从上面两个函数看出，显示暴露一个设置回调的接口给外面，当断开事件发生时掉回调，
这就是p2p连接状态的更新。

## 最后

虽然webrtc spec定义了连接状态有6种，pion/webrtc@v1.2.0源码上虽然也定义了这几种，
但是实际使用只有两种：new和closed

最后看一下network库(更进一步是ice库的udp连接)
是如何将连接断开一步步通知给调用者的：

在pion/webrtc@v1.2.0 pkg/ice/ice.go中，定义了ice连接的7种状态，
高能预警：断开连接是6

    type ConnectionState int
    const (
      ConnectionStateNew = iota + 1
      ConnectionStateChecking
      ConnectionStateConnected
      ConnectionStateCompleted
      ConnectionStateFailed
      ConnectionStateDisconnected
      ConnectionStateClosed
    )

agent.go中有一个方法来更新状态

    func (a *Agent) updateConnectionState(newState ConnectionState) {
      if a.connectionState != newState {
        a.connectionState = newState
        if a.notifier != nil {
          // Call handler async since we may be holding the agent lock
          // and the handler may also require it
          go a.notifier(a.connectionState)
        }
      }
    }

调用这个方法的地方有3处

    func (a *Agent) startConnectivityChecks(isControlling bool,
      remoteUfrag, remotePwd string) error {

      if a.haveStarted {
        return errors.Errorf("Attempted to start agent twice")
      } else if remoteUfrag == "" {
        return errors.Errorf("remoteUfrag is empty")
      } else if remotePwd == "" {
        return errors.Errorf("remotePwd is empty")
      }

      return a.run(func(agent *Agent) {
        agent.isControlling = isControlling
        agent.remoteUfrag = remoteUfrag
        agent.remotePwd = remotePwd

        // TODO this should be dynamic, and grow when the connection is stable
        t := time.NewTicker(taskLoopInterval)
        agent.connectivityTicker = t
        agent.connectivityChan = t.C

        agent.updateConnectionState(ConnectionStateChecking)
      })
    }

    func (a *Agent) setValidPair(local, remote *Candidate, selected bool) {
      // TODO: avoid duplicates
      p := newCandidatePair(local, remote)

      if selected {
        a.selectedPair = p
        a.validPairs = nil
        // TODO: only set state to connected on selecting final pair?
        a.updateConnectionState(ConnectionStateConnected)
      } else {
        // keep track of pairs with succesfull bindings since any of them
        // can be used for communication until the final pair is selected:
        // https://tools.ietf.org/html/draft-ietf-ice-rfc5245bis-20#section-12
        a.validPairs = append(a.validPairs, p)
      }

      // Signal connected
      a.onConnectedOnce.Do(func() { close(a.onConnected) })
    }

    func (a *Agent) validateSelectedPair() bool {
      if a.selectedPair == nil {
        // Not valid since not selected
        return false
      }

      if time.Since(a.selectedPair.remote.LastReceived()) > connectionTimeout {
        a.selectedPair = nil
        a.updateConnectionState(ConnectionStateDisconnected)
        return false
      }

      return true
    }

可以看出ice库对webrtc spec的实现，也只实现了4种状态：
构造时的new，连接开始的checking，连接上的connected，断开时的disconnected。

跟着代码走看看这几个状态的设置时机：

- new：构造时的默认状态
- checking：可以看出是收到远端sdp，开始建立p2p连接时触发checking状态
  - ice.Agent.startConnectivityChecks中设置
  - ice.Agent.connect中调用
  - ice.Agent.Dial/Accept
  - network.Manager.startICE
  - network.Manager.Start
  - webrtc.RTCPeerConnection.SetRemoteDescription
- connected：只要有一个候选对匹配上了，就设置为已连接
  - ice.Agent.setValidPair 当ice候选协商通过时设置为connected
- disconnected：udp无心跳返回时(30秒)，认为是连接断开了
  - ice.Agent.validateSelectedPair，只有心跳超时才会认为是断开了
  - 这个在任务队列里，由定时器触发，每2秒一次，检查心跳的的同时，也发送心跳保活

这里，只列出了断开连接的发生(心跳超时)，回到ice.Agent.updateConnectionState,
她是通过调用外部设置的回调函数，来通知的，下面看看这个回调函数的设置：

    // ice库
    func NewAgent(urls []*URL, notifier func(ConnectionState)) *Agent
    // network库
    func NewManager(urls []*ice.URL,
                    btg BufferTransportGenerator, ntf ICENotifier) *Manager
    // webrtc库
    func New(configuration RTCConfiguration) (*RTCPeerConnection, error)
    // 其中中有这么一条
    pc.networkManager = network.NewManager(urls, pc.generateChannel, pc.iceStateChange)

这就是上面分析的内容

最后的彩蛋：
之前预警了断开连接枚举值是6,
在webrtc中，调用外部回调函数时，猜猜断开连接的值是多少？

RTCPeerConnectionState枚举中，断开连接的枚举值是4，为啥没对应，
是不是感觉很吃惊，因为枚举值是更具rfc来的，两个rfc之间并没有强制性对应关系，
所以，回调的签名是这样的，并不是使用连接枚举值：

    onICEConnectionStateChangeHandler func(ice.ConnectionState)

回调的参数，是ice库的枚举值，并不是webrtc库中连接枚举值。

## 最后的最后 TODO

ice库中，7种状态只实现了4种，
初始化是new，表示还未开始进行ice候选的匹配，
开始进行ice候选匹配时，状态是checking，
第一个候选对通过时，状态改为connected，
这个连接断开时，状态改为disconnected，并通知给调用者

问题：多个可用连接是如何抉择的，目前并没有深入去研究，后面需要补上，TODO
