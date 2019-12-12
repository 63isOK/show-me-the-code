# notedit/go-sdp-transform

说明：对sdp的简单封装，支持解析和写操作。

下面主要分析解析操作, 而且解析也集中在Parse(),
对着源码更容易理解

Parse(string),这个函数主要是接收一个sdp文本，先解析为一个json格式文本，
再将json格式的文本反序列化成Go的数据结构。

    func Parse(sdp string) (sdpStruct *SdpStruct, err error)

Parse的目的是将一个sdp字符串解析，并用Go数据结构存储，
其实就是一个反序列化的过程，不过添加了校验等辅助流程。

## Parse分析

整个解析过程分3步走：

1. 将sdp文本转成中间格式
2. 从中间格式生成json字符串
3. 将json字符串反序列到Go数据结构

[图](https://www.draw.io/?mode=github#H63isOK%2Fconference_graph%2Fmaster%2Fmedia-server-go%2Fsdp)

Jeffail/gabs的包，主要对encoding/json做了一些扩展，
这些扩展集中在对未知json类型或动态json提供了一个层级的访问，
通过层级的概念，可访问并修改任意一个json对应的元素(这个元素当然是Go数据结构的)，
而且gabs提供的ds(data struct)也非常简单 interface{},
对json对象和数组分别提供了对应的api，映射的Go数据结构也非常简单：
map[string]interface{} 和 []interface{}。

gabs提供的数据结构无法直接转换成实际场景中的最终Go数据结构(这个是我们业务上的)，
但间接方式是有的：gabs提供了导出为json字符串的功能，再加上标准库的反序列化，
就可以将gabs数据结构间接转换成我们最终的Go数据结构了。正好对应着Parse流程的23步。

所以下面只剩下一个问题，就是如何将文本数据用gabs的数据结构表示(步骤1)

在Parse()中，for循环就是来逐行读取sdp文本，并解析成gabs数据，
从源码上看，执行了以下几步：

1. 去掉行尾控制字符
2. 使用正则检查单行数据的格式
3. 从单行数据分离出类型和信息数据
4. 通过类型找信息数据的匹配规则
5. 用匹配规则来提取信息数据中的有效数据
6. 用gabs来存储提取的数据

1236步都是一些简单的逻辑处理，45步涉及到一个匹配规则，下面来看一下

    var rulesMap map[byte][]*Rule = map[byte][]*Rule{...}

在grammer.go中，rulesMap里面是一些规则集，key是单行数据分离出的类型，
每一个类型都可能对应多个规则，实际上a=的规则是多个，其他都是单个，
因为a=比较特殊。a=中依据信息数据的不同，对应的规则也不一样，
但一个sdp单行数据最终只会匹配一个具体的规则。

这种设计方式的好处：以后有新的规则要添加，是非常方便的。

找到具体规则后，会调用parseReg(rule, location, content)来提取信息,
最后具体执行会调用attachProperties来实现

其中需要说一下规则对应的数据结构

    type Rule struct {
      Name       string
      Push       string
      Reg        *regexp.Regexp
      Names      []string
      Types      []rune
      Format     string
      FormatFunc func(obj *gabs.Container) string
    }

其中Name和Names分别对应sdp rfc中的单行数据名和数据的子段，
Push标记着是否是可重复数据，对应到Go中就是数组，
Types是表明Names字段中数据的类型，s和d分别表示字符串和数值，
至于Format和FormatFunc是写sdp时用到的，解析用不上，
Reg就是正则，用于提取数据。

不得不说作者的设计思路非常优雅和巧妙，还有一个巧妙的地方

for循环在分离出单行的类型和信息数据后，有个操作，
如果当前类型是m=，也就是一个新的媒体级信息，此时会修改location，
何时会修改回去？直到遇到另一个新的m=

这点正好符合sdp rfc中的定义：会话级 媒体级 媒体级 ...

为啥这里面透出了mvp的味道。

作为一个库，sdp-tranform最后用SdpStruct数据结构来保持反序列化的数据，
实际上的调用者，可以从中截取一小部分使用即可，非常照顾调用者。

## 最后

我是在分析media-server-go时遇到了这个解析sdp的库，
期间还分析了encoding/json和Jeffail/gabs之后才进一步分析了这个库，
真的想说，这个库真的非常优雅，扩展性也非常强
