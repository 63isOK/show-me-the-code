# Chain an asynchronous operation

## 目录

<!-- vim-markdown-toc GFM -->

- [spec](#spec)
- [pion/webrtc@v1.2.0的举例](#pionwebrtcv120的举例)

<!-- vim-markdown-toc -->

[链式异步操作](https://www.w3.org/TR/webrtc/#chain-an-asynchronous-operation)

前面在讲pion/webrtc@v1.2.0的连接构造时提到了，
pion没有operations，也就是没有链式异步操作的哪些字段，
她的链式异步操作都是靠调用方来保证的，所以实现并不像spec中规定的那样。

这里并不是说pion不支持异步操作，pion也是支持的，不过比较简化，实现的目的是一样的。

## spec

异步操作，好理解，就是两个操作并没有相互阻塞的，谁先谁后都行，执行的时候，
可能是并行操作。链式异步操作，其中的链式就是一个队列化的过程，
将本来并发的操作，改为一次只做一个操作。一般在服务端用的比较多，
类似"信令队列"都是一个意思。

spec是以jsep看齐，所以利用了js上的promise异步操作来描述，
不过用其他语言都是一样的。

流程如下：

- 设置连接对象RTCPeerConnection对象
- 连接对象的IsClosed是true，那么直接拒绝，因为连接已经关闭，其他操作都没有意义
- 将新的操作请求添加到类似队列的字段(就是pion没有的operations字段)，暂且称为队列
- 如果当前队列只有一个操作，就执行
- 在每次拒绝或执行了某个操作之后，还要做以下事情：
  - 如果连接的IsClosed是true，丢弃所有的步骤，退出
  - 不管是执行或拒绝，都简爱嗯结果返回给调用者
  - 返回给调用者之后，执行以下操作：
    - 如果连接的IsClosed是true，丢弃所有的步骤，退出
    - 移除操作队列的第一个操作
    - 如果操作队列非空，执行第一个操作

好吧，这就是一个消息队列的基本处理过程，没有什么新的东西

## pion/webrtc@v1.2.0的举例

    func (pc *RTCPeerConnection) Close() error {
      if pc.isClosed {
        return nil
      }

      err := pc.networkManager.Close()

      pc.isClosed = true
      pc.SignalingState = RTCSignalingStateClosed
      pc.IceConnectionState = ice.ConnectionStateClosed // FIXME REMOVE
      pc.ConnectionState = RTCPeerConnectionStateClosed

      return err
    }

上面只是随意截取的一段代码，很多类似可以异步的调用中，
都会有一个pc.isClosed的判断，如果已经关闭，就按xxx处理。
