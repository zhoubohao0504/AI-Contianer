# AI-Contianer
## 设备上线流程
管理员通过后台管理系统配置好货柜、仓门、货道对应关系后，线下去添加设备，设备首次添加会与通信服务建立一个长链接，并且保存设备基本信息到缓存，同时还保存当前货柜与那台通信服务之间建立的链接，更新设备状态为待初始化
非首次添加设备则需要更新设备状态为在线，同时修改缓存中设备基本信息，以及货柜与通信服务的链接关系。

## 设备心跳建立
在设备完成初始化操作以后，设备通过通信服务与设备服务建立一个心跳，该心跳是为了保障设备是处于一个正常的状态。
每30s向设备服务发送一次心跳包，如果距离上次心跳包的时间超过30s就将该设备标记为下线
> question<br>
由于网络抖动导致设备发送的心跳包未能及时到达设备服务，也会把设备标记为下线，但是设备是正常状态的<br>
> resolve<br>
保存最后一次心跳时间到缓存，同时开始定时任务每一分钟去轮询缓存 获取最后一次心跳时间与当前时间做比较 是否超过设置的下线时间阈值<br>
> final resolve<br>
建立心跳表，每次设备发送心跳包以后，心跳计数器+1，更新缓存最后一次收到心跳时间<br>
定时任务查询设备缓存的最后一次心跳时间，如果大于1分钟，去判断当前的设备服务是否是处于一个正常的状态，例如2000台设备1分钟在理想状态下是有4000次心跳发送到设备服务器，针对这个4000次设置一个阈值例如80%就是3200次，那么
在这一分钟下如果心跳包超过3200次，那么说明我的服务是一个正常状态，说明该设备是真的出现故障，将该服务下线

question：<br>
1：建立长链接的目的是什么？<br>
2：定时任务轮询可能会出现 设备量过多 比如2000台设备，轮询到第1000台刚好超过下线时间阈值会导致后续的1000台直接超时<br>
3：心跳检测由设备服务发起？多台设备服务如何区分哪一个心跳检测由某一台服务来完成？就是有没有可能出现重复心跳检测的问题<br>

knowledge points<br>
netty、redis、eureka

## 购买流程
用户扫码以后将当前设备编号与仓门编号通知到APP服务，APP服务区做参数校验、设备校验，以及是否存在未支付订单（基本都是免密支付，存在说明用户一直没钱），生成订单之前先开启一个分布式锁，锁货柜仓门，保障同一时刻只允许一个开门，加锁成功以后会生成我们系统的一个订单号，此时订单状态为预授权状态，仓门状态为未开启，支付状态为未支付<br>
然后向微信或者支付宝去申请授权，更改订单状态为授权中，微信是支付分，支付宝是芝麻信用，这里分为需确认和免去确认（具体选择看实际需求）,同时发送一条延时消息到消息队列，用于判断用户在指定时间内是否授权完成。用户完成授权以后，回调我们的服务告知已经授权成功，更改订单状态为授权成功，发送消息通知设备服务开启仓门，设备服务需要缓存订单信息、货柜信息、仓门信息、货道信息、用户信息等到缓存并且从缓存中获取当前货柜所对应的通信服务是那一台通信服务然后向通信服务发起开门请求，通信服务通知货柜开启仓门，返回设备服务，设备服务标记仓门状态为已开启，同时开启定时任务轮询判断仓门是否已经开启，判断5次以后认定仓门状态为异常，同时取消订单。<br>
当用户关闭仓门以后，需要获取设备的货道重量与开门时保存的重量缓存进行计算得出购买的商品详情及金额，计算完成以后通知账户服务发起扣款请求，并计算是否需要补货，如果需要补货建立补货单，发起补货通知到运营人员

/`
  设备初始化的时候会保存设备信息，仓门信息、货道信息、货道重量到缓存中
  门关着 每90s发送一次重量
  门开着 每300ms发送一次重量
  关门后8s内 每300ms发送一次重量
  发送时有个标记位 表示称是否稳定 0不稳定 1稳定
  重量持续发送 持续保存到数据库
`/

用户关门以后会先获取缓存中开门前记录的各货道的重量，由于用户开门以后设备会每300ms发送一次重量信息到设备服务，设备服务需要保存改重量到数据库和缓存，然后在关门时获取最后一次的重量信息开门前的重量进行计算。<br>
由于发送的重量信息会存在不稳定的数据，不稳定的数据则不会保存，可能会存在缓存里面还是开门前的一个重量，这里记录下最后一次缓存重量的时间，然后在接收到关门指令以后获取系统时间与最后一次重量时间比较，如果最后一次重量时间在关门指令时间前，说明重量不可用<br>
如果一直出现重量不可用，那么等待2s，直到出现可用重量，保障缓存中的重量一定是关门后的重量的。


用户关门时，
*需确认*：每次都需要用户重新确认<br>
*免确认*：授权一次以后都不需要重新授权

knowledge points<br>
分布式锁、消息队列、延时队列

