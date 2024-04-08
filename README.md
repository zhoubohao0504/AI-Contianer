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



1.难点：对设备的命令的不确定性，由于网络不好，机器不发送开门指令什么
       项目的所有难点都是在设备的一些网络中状态或者是机器的命令的不稳定性
   问题1：服务向机器发送开门指令，机器没有返回值或者长时间未响应。查询一下门的状态，如果没有返回再轮询查几次以后都没有响应 标记订单舱门状态为异常。
   问题2：机器在开门以后 本来订单状态中舱门状态应该是已开门，可能因为网络原因还是其他什么原因 订单状态没有同步，当客户关门时又收到一个关门指令，最开始的时候是判断订单状态是开门中，然后才会去计算重量什么的，这时候由于订单状态是关门，此时的消息就会被丢弃。
         莫名收到一条关门指令的时候，会把改订单标记为异常状态，记录商品详细信息，然后与货柜里面的商品做对比

  问题3：拿一个商品 放入一个同等重量的商品进去
        只能等到补货员补货的时候发现 然后再去排查这个柜子的历史订单，然后根据我们的摄像头记录找到对应人 金额较大报警处理，金额较小通知他付钱，他是在是不付钱，商品标记为正常损耗。

  问题4：从货道4拿一个东西放到货道3中
        先扣钱，订单完成以后如果一个货道的重量增加了，就会产生一条预警信息到运营人员。或者是客户被扣钱以后自己联系客服，提交照片或者根据摄像头观察 是不是真的存在东西放错的情况 再去给他一个退款处理

哪里用到了消息队列
    用户关门以后通过消息队列将订单信息发送给账户服务，告知账户服务接单，如果商品数量小于我们设置的阈值同时也通过消息队列给设备服务发送消息补货通知，然后所有的发送短息通知运营人员都是通过消息队列处理，例如 校准、补过、查验
哪里用到了Redis
    Redis缓存了设备信息，比如Redis的key是我们的设备的一个唯一编号：deviceID,value是对设备序列化后的数据，然后还有最后一次心跳时间，连在哪一个商品服务上。存在一个商品信息，只要是设备相关的会都存在缓存，设备的东西不会经常去修改。
    然后再就是我们的 通信服务 与设备之间的关联关系
  分布式锁：
  下单的时候，同一时刻只允许一个人开门，然后补货、换品改价都是用到这个锁，用的是Redission。
哪里用到了分布式事务：
  强一致性：创建用户的时候，现在APP服务创建用户信息，然后再去账户服务去创建账户信息，必须同时成功才能去买东西，如果用户存在账户不存在那么就会出现无法扣款。用的Seata AT模式，
  最终一致性：设备服务把商品信息算出以后，需要发消息给账户服务、补货服务，这个时候商品已经被拿走了，事务失败不能让用户把商品退回去，发送一个事务性消息给账户服务，账户服务多久结单已经不关设备服务的事了，计算结单失败，重新从RockMq里面重新把消息拿出来再算一遍。发送失败的话先存到数据库，然后通过定时任务一定会把消息发送到账户服务。      

  项目多久：从整个项目搭建到最后上线 用了3 4个月，但是后面一直在优化，比如之前没有对账信息，我们可能会错卖一些商品。比如用户拿了东西又放到其他货道上，这时候扣了他的钱，他会来找我们申述，这时候我们需要把库存调回来，如果他拿了东西没扣钱，一般用户是不回来告知我们的，我们也察觉不到。没有做对账之前只有在补货的时候才发现库存跟货柜里面的东西不一致。


  对账：
  每天晚上12点，或者1点，货柜每90s上传一次重量，我们用最新的重量去 计算一下商品除一下商品的单重量，跟库存数量进行对比。然后根据我们记录的重量更改的信息去发现是哪一个用户拿了东西没给钱，只能打电话通知他能不能把钱补上，不能补上就算我们一个正常的损耗。


  数据量怎么样：2000台设备 每90s发送一次，在开门和关门后的8s内都是300ms发送一次重量，这个重量我们会记录，平均算下来一台机器30s发送一次，1分钟就是2次，1小时就是120次，一天就是2880次，2000台平均下来5000000条记录。我们就只存一个星期的重量在一个表里面，然后上个星期的重量存在一个表里面，我们只存4个星期也就是一个月的重量。以保障我们后期问题的排查。
  JVM调优:JVM分为新生代、老年代，一开始我们的通信服务是100台设备，在eden区，s0、s1，随着设备慢慢的增多，例如增加到600台设备，它会持续的发心跳、发重量然后导致CPU的处理速度变慢，比如我们之前执行一个线程的时间是2s，他现在的一个执行时间需要4s，当我们eden区被填满以后就会触发yong GC，
之前Eden区可能是3s做一次GC，或者4s一次的样子，但是随着我的机器的一个增多，它产生对象的速度非常块，然后每个线程执行需要4秒，但是它3s就做一次GC，那么我们线程还没有执行完，就不是垃圾对象，这时候我就会把Eden区一整块对象复制到S0，它存活的对象比S0区还要大，就会出现放不下的情况，这时候由于空间担保机制，就会把这一整块对象都放到老年代，但是它又不是长期存活的对象，但是它进入老年代了而且还是一批一批的进入，马上就把老年代撑满了，就去Full GC,长期如此就会频繁的触发FULL GC。
  把新生代跟老年代默认比例1：2改为1：1，由于我们的通信服务的一些对象本质上就建立一个通信马上就执行完了，例如发个心跳、发送重量什么的，new一个对象处理完就结束了，所以其实老年代并不需要去存什么大对象，长期存活的对象比较少。
  也可以把s0和s1的比例调大一点比如 由8：1：1改为3：1：1，这样由于s区放的下就不会放到老年代区，再下一次GC的时候就会被回收掉了。


  支付宝退款是同步接口，微信退款是异步接口。
