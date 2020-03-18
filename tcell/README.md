# tcell

一个纯Go的库,提供一些api来辅助编程,支持各平台的控制台或终端编程.
说白一点就是cli图形界面编程api的提供.

这个系列中,只分析终端/linux下的一些常规api,部分平台的特定适配不过多关注.

ps: ___本系列分析的起点是 8ec73b6fa6c543d5d067722c0444b07f7607ba2f___

- 这个库的类分析在[这里](https://github.com/63isOK/conference_graph/tree/master/15.puzzle)
- Screen是一个核心概念
  - tScreen是Screen的一个实现,具体分析在[这里](/tcell/001.md)
