# 大概的样子

```
在不同局域网的两个客户端 A, B 通过 S 协同:
            +------------+
            |  Server(S) |
            +------------+
                   |
        +----------+------------+
        |                       |
    +-------+               +-------+
    | NAT(X)|               | NAT(Y)|
    +-------+               +-------+
        |                       |
    +-------+               +-------+
    |   A   |               |   B   |
    +-------+               +-------+
```

# 资源

所有的资源被拆分成 N 个片段。每个片段大小为 `8K(8192) - 序号 - 指纹`，一个数据包格式为

    [片段序号][md5指纹][数据]

客户端请求服务器某个资源，服务器返回一组资源列表

      外网IP:端口          内网IP:端口         资源片段
    184.23.291.5:68900,192.168.5.78:9981,0-643,655,991
    299.36.97.25:93100,10.10.251.39:7812,0-992

客户端将会自行决定从那几个资源下载片段，最后拼装成完整的资源

# 客户端 A 如何连接 B

站在 *客户端 A* 的角度，如果它想获取 *客户端 B* 的资源，整个过程是:

```
假设:
    A: 184.23.291.5:68900,192.168.5.78:9981
    B: 206.36.97.25:93100,10.10.251.39:7812
    A 要获取 B 的 0-79 片段 和 199-355 片段
-----------------------------------------------------
A 客户端的行为是:
-----------------------------------------------------
1. UDP 监听 9981 端口 （实际上，它一直会监听这个端口）

2. 发起 UDP 连接至 10.10.251.39:7812 (B的内网)
   失败后 NAT(X) 上会保留一条记录，这就是A给B打的洞

3. 发起请求到 S， 让它在 B 下次心跳的时候发起请求给自己传数据

4. 等着 B 发数据到自己的 9981 端口

5. 收到的第一个数据包，是 B 对自己本次行为的描述:
   {
       source   : "206.36.97.25:93100,10.10.251.39:7812"
       resource : "64dac1rcde9ua..",
       range    : "0,79:199,355"
   }
   如果收到了数据包，那么 B 肯定也在 Y 上给 A 打好洞了。
   这个洞是给 A 发送控制信息用的

6. 每次收到数据，做校验，全部收取完毕后
   给 B 的洞发消息，说没你任务了。
   如果收到的数据不对，则给 B 的洞发消息，让它重发某几个片段

-----------------------------------------------------
B 客户端的行为是:
-----------------------------------------------------
1. 当心跳到 S 时，S 告诉它:
   {
       action   : "send2peer"
       target   : "184.23.291.5:68900",
       resource : "64dac1rcde9ua..",
       range    : "0,79:199,355"
   }
2. 监听自己的 7812 端口（实际上，它一直会监听这个端口）

3. 发起 UDP 连接至 192.168.5.78:9981 (A的内网)
   失败后 NAT(Y) 上会保留一条记录，这就是B给A打的洞

4. 建立 UDP 连接至 184.23.291.5:68900 (A的外网)
   找到自己本地有的，且 在 range 之中的数据包，逐次发送
   发一个刷新一下

5. 发完了，或者从洞里得到A的消息让自己停止的时候就停止。
   自己会有一个片段发送队列，如果从洞里知道A要自己重发某几个片段
   则会将这几个片段插到队列最前面
```
