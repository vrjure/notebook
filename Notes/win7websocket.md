# clientwebsocket在win7上报PlatformNotSupportedException

`ClientWebSocket`在win7上报了`PlatformNotSupportedException`，而在自己的win10上是正常的。  
官方有一段解释：

> Some of the classes and class elements in the System.Net.WebSockets namespace are supported on Windows 7, Windows Vista SP2, and Windows Server 2008. However, the only public implementations of client and server WebSockets are supported on Windows 8 and Windows Server 2012. The class elements in the System.Net.WebSockets namespace that are supported on Windows 7, Windows Vista SP2, and Windows Server 2008 are abstract class elements. This allows an application developer to inherit and extend these abstract class classes and class elements with an actual implementation of client WebSockets.

大致意思是命名空间`System.Net.WebSockets`在所有平台上都是可用的，但是websocket的client和server的实现只在win8和win server 2012上支持

不明白为啥要这样设计。

幸运的是在github上找了个实现[System.Net.WebSockets.Client.Managed
](https://github.com/PingmanTools/System.Net.WebSockets.Client.Managed/)

明明是可以实现的，是因为win7不升级了？
