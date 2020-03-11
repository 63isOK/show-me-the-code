# pion/logging 源码分析

从依赖上看,是无第三方依赖的.
这个日志库是基于[标准库log](https://github.com/63isOK/go1.13/blob/master/docs/log/log.md)

## LogLevel

定义了一个枚举类型,方法有读取/转字符串

枚举值如下:

- LogLevelDisabled 不记录日志
- LogLevelError 一般是记录fatal 致命错误,`应该由用户代码来处理`
- LogLevelWarn 一般记录的是不正常(`非致命`)的`库操作信息`
- LogLevelInfo 一般记录`常规的库操作信息`,如`状态改变`等
- LogLevelDebug 一般记录`底层库操作信息`,如`交互操作`等
- LogLevelTrace 一般记录`非常底层的库信息`,如`网络跟踪信息`等

## LeleledLogger

这是一个接口,级别日志器(先这么叫着),定义了5对方法

LoggerFactory接口,定义了NewLogger方法,就是构造级别日志器

    type LoggerFactory interface {
      NewLogger(scope string) LeveledLogger
    }

实现她的是DefaultLoggerFactory

    type DefaultLoggerFactory struct {
      Writer          io.Writer
      DefaultLogLevel LogLevel
      ScopeLevels     map[string]LogLevel
    }

她的构造函数是NewDefaultLoggerFactory,具体逻辑如下:

- 构造一个DefaultLoggerFactory对象
  - 默认日志级别是error
  - 输出是stdout
- 通过环境变量确定默认级别和等级范围

DefaultLoggerFactory实现的NewLogger,具体逻辑如下:

- 再一次确定日志等级
- 构造一个DefaultLeveledLogger对象,通过这个对象的构造函数实现

    type DefaultLeveledLogger struct {
      level  LogLevel
      writer *loggerWriter
      trace  *log.Logger
      debug  *log.Logger
      info   *log.Logger
      warn   *log.Logger
      err    *log.Logger
    }

这是一个非常有意思的结构,有多个标准库提供的日志器,一个日志等级,
一个支持并发读写的io.Writer(用loggerWriter封装了)

她的构造函数在构造简单对象后,调用WithXXXLogger来处理各个日志器.

之后就是调用LeveledLogger的接口方法,也就是调用DefaultLeveledLogger的方法,
最后都会调到DefaultLeveledLogger.logf,最后调用具体的log.Logger来写日志.

在logf中,会有一段代码是按日志等级来过滤的,这是根据枚举值来实现的.

## 总体分析

logger.go 这个文件中除了辅助函数logf,其他都是暴露的,包括两个接口和日志等级,
所以说提供了最大的可定制性(可以不使用提供的构造方法,可以直接实现接口等),
可以说是优点,也可以说是缺点,由于定制性太强,导致会看到一大堆应用场景,
同时需要写额外的代码来工作,默认的工厂对象和默认的级别日志器也不好用.

说一种常用的用法:

- 工厂对象,直接自定义(不使用自带的构造方法)
  - 设置默认级别,也设置例外(例外在ScopeLevels中设置)
- 通过工厂对象调用NewLogger来生成级别日志器对象
  - 如果只需要一个级别日志器,直接使用构造函数也是ok的
- 之后调用LeveledLogger的接口方法即可

如果要将不同级别的日志输出到不同地方,或是更改log.Logger的flag,
就需要自己配置做更精细的定制了.

这个包的具体使用是如何的,就看pion/webrtc中是如何调用的

## 最后

这个库的设计和实现都不是那么一目了然,文档就是shit,
扩展性太强了(设定一个日志级别就使用了多种方式,和Go的处理方式相反),
和log库比起来,各方面都差太多.需要改进.
