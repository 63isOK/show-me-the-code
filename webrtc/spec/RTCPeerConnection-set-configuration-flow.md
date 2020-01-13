# 设置配置

[webrtc spec](https://www.w3.org/TR/webrtc/#set-pc-configuration)

## spec

设置配置有如下步骤：

- configuration指向RTCConfiguration
- connection指向RTCPeerConnection
- 如果设置了证书(configuration.certificates)，走以下流程
  - 如果配置中的证书个数和连接中指定的证书个数不一致
    - 就表明证书有修改，并未同步，返回一个InvalidModificationError错误
  - 新建变量index，设置为0
  - 新建变量size，表示配置中的证书个数
  - 遍历配置的证书列表和连接的证书列表，外加上面两个循环的因子和约束，就是一个for循环
    - 如果两个列表对应同一个index的es对象(js术语的一种)不一样，包InvalidModificationError错误
    - index加1
- 下列情况出现时，都会报InvalidModificationError错误
  - 两个列表中的bunlde约束不一致
  - 两个列表的rtcpMux复用策略不一致
  - 两个ice候选池大小不一致
- ice候选收集策略，如果两者不一样，就证明这块有修改
  - 在下次协商之前，老的设置都不会进行修改
  - 如果想立马更新应用，需要进行ice restart
- ice候选池大小的变更，会立马生效
  - 如果是减小，那么可能会丢弃一部分候选
- 如果定义了ice服务器列表，执行以下步骤：
  - server指向当前ice服务列表
  - urls指向server.urls
  - 如果urls是字符串，那将urls改变为一个字符串数组
  - 如果urls为空，包语法错误SyntaxError
  - 对于urls中的每一个url：
    - 第一，解析，失败就报SyntaxError错误
    - 如果不支持，报NotSupportederror错误
    - 按ice协议，解析turn/turns/stun/stuns开头的url
    - 如果是turn服务(不管是turn还是turns)
      - 检查username/credential,如果没有，就报InvalidAccessError错误
    - 如果是stun服务(stun/stuns)，证书类型是password
      - 如果整数不是一个DOM字符串，报InvalidAccessError错误
  - 将检查后的url添加到一个已验证服务url列表中
- ice agent的ice 服务列表，就是上面哪个已验证服务url列表
- 按照jsep的规定，如果ice服务列表有变更，那么在下次协商才开始生效
  - 如果要立马生效，就走ice restart
- 将新配置存储在连接中
