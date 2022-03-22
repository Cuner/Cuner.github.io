# 咨询
```
@startuml

skinparam componentStyle uml2

package "收银台" {
[cashierplatform]
}

package "支付网关" {
 [pg]
 [ipay]
}

[cashierplatform]...>[pg]:传递风控设备信息 1:1
[pg]...>[ipay]:传递风控设备信息 1:1


package "缓存" {
[tair]
}

[cashierplatform]--->[tair]:获取/写入用户维度咨询缓存 1:2
[cashierplatform]--->[tair]:获取/写入收银单缓存 1:2
[cashierplatform]--->[pg]:收银咨询 1:1
[pg]--->[ipay]:可用支付方式咨询 1:1

package "金融会员" {
[finance_member]
}
[cashierplatform]--->[finance_member]:静态化开启时获取用户可用资产 1:1

package "营销" {
[ump]
}

[cashierplatform]--->[ump]:用户支付优惠查询 1:1

@enduml
```

# 首次支付
```
@startuml

skinparam componentStyle uml2

package "收银台" {
[cashierplatform]
}

package "支付中台" {
 [paymentplatform]
}

[cashierplatform] ...>[paymentplatform]:创建支付单及支付指令\n(同时有同步及异步流程)\n1:1


package "营销" {
[ump]
}

package "资产平台" {
[fp]
}

[paymentplatform]--->[ump]:检查并冻结优惠 1:1
[paymentplatform]--->[fp]:资金冻结 1:1

package "支付网关" {
 [pg]
 [alipay]
 [ipay]
}


'异步流程
[paymentplatform]...>[pg]:发起支付宝主站协议支付 1:1
[pg]...>[alipay]:协议支付 1:1
[alipay]...>[ipay]:创建ipay收单 1:1
[paymentplatform]...>[pg]:发起支付 1:1
[pg]...>[ipay]:ipay支付 1:1

[cashierplatform] --->[paymentplatform]:支付结果轮询 1:1
'pg是否调用了ipay?
[paymentplatform] --->[pg]:查询支付结果 1:1 

@enduml
```

# 二次支付
```
@startuml

skinparam componentStyle uml2

package "收银台" {
[cashierplatform]
}

package "支付中台" {
 [paymentplatform]
}

[cashierplatform] ...>[paymentplatform]:取消上次支付 1:1


package "营销" {
[ump]
}

package "资产平台" {
[fp]
}

[paymentplatform]--->[ump]:优惠解冻 1:1
[paymentplatform]--->[fp]:资金解冻 1:1

[cashierplatform] ...>[paymentplatform]:再次支付 1:1

package "支付网关" {
 [pg]
 [alipay]
 [ipay]
}


'异步流程
[paymentplatform]...>[ump]:优惠冻结 1:1
[paymentplatform]...>[fp]:资金冻结 1:1
[paymentplatform]...>[pg]:发起支付宝主站协议支付 1:1
[pg]...>[alipay]:协议支付 1:1
[alipay]...>[ipay]:创建ipay收单 1:1
[paymentplatform]...>[pg]:发起支付 1:1
[pg]...>[ipay]:ipay支付 1:1

[cashierplatform] --->[paymentplatform]:支付结果轮询 1:1
'pg是否调用了ipay?
[paymentplatform] --->[pg]:查询支付结果 1:1 

@enduml
```