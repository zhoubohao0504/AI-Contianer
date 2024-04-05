# AI-Contianer
## 设备上线流程
管理员通过后台管理系统配置好货柜、仓门、货道对应关系后，线下去添加设备，设备首次添加会与通信服务建立一个长链接，并且保存设备基本信息到缓存，同时还保存当前货柜与那台通信服务之间建立的链接，更新设备状态为待初始化
非首次添加设备则需要更新设备状态为在线，同时修改缓存中设备基本信息，以及货柜与通信服务的链接关系。

## 设备心跳建立
在设备完成初始化操作以后，设备通过通信服务与设备服务建立一个心跳，该心跳是为了保障设备是处于一个正常的状态。
每30s向设备服务发送一次心跳包，如果距离上次心跳包的时间超过30s就将该设备标记为下线
| question
由于网络抖动导致设备发送的心跳包未能及时到达设备服务，也会把设备标记为下线，但是设备是正常状态的
> resolve
保存最后一次心跳时间到缓存，同时开始定时任务每一分钟去轮询缓存 获取最后一次心跳时间与当前时间做比较 是否超过设置的下线时间阈值
> final resolve
建立心跳表，每次设备发送心跳包以后，心跳计数器+1，更新缓存最后一次收到心跳时间
定时任务查询设备缓存的最后一次心跳时间，如果大于1分钟，去判断当前的设备服务是否是处于一个正常的状态，例如2000台设备1分钟在理想状态下是有4000次心跳发送到设备服务器，针对这个4000次设置一个阈值例如80%就是3200次，那么
在这一分钟下如果心跳包超过3200次，那么说明我的服务是一个正常状态，说明该设备是真的出现故障，将该服务下线

question：
1：建立长链接的目的是什么？<br>
2：定时任务轮询可能会出现 设备量过多 比如2000台设备，轮询到第1000台刚好超过下线时间阈值会导致后续的1000台直接超时<br>
3：心跳检测由设备服务发起？多台设备服务如何区分哪一个心跳检测由某一台服务来完成？就是有没有可能出现重复心跳检测的问题<br>

knowledge points
netty、redis、eureka


