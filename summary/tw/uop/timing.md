# 时序图【形式不限，图中标示出资损点】

## 发货
含TP链路（1688）、TBC链路、CB链路、Local（不含卖家宅配）链路、Local（卖家宅配）链路

### TP链路（1688）
```
@startuml

participant 1688商家后台 order 1
participant uop order 2
participant 菜鸟 order 3
participant 1688供货中心 order 4
participant gsp order 5
participant guop order 6
participant 交易 order 7


1688商家后台 -> uop:1、商家发货
uop -> 菜鸟:1.1、菜鸟收单

1688供货中心 -->gsp:2、监听到商家发货
destroy 1688供货中心
note left
    1、未监听到商家发货消息
    2、gsp、guop、交易未能更新状态
    3、买家关单，商家钱货两空
end note


gsp ->guop:2.1、发货

guop -> guop:2.1.1、更新履约状态为已发货

guop --> 交易: 2.1.2、发货状态消息
destroy guop
note left
    1、履约消息发送失败或交易处理失败
    2、交易未能更新状态
    3、买家关单，商家钱货两空
end note
交易 -> 交易: 2.1.2.1、更新物流状态为已发货

菜鸟 --[#red]> uop:3、货品到达国际仓
note right
    1、消息未发出
    2、导致guop的LP单号未更新
    3、商家结算失败
end note
destroy 菜鸟
uop->guop:3.1、发货

guop -> guop:3.1.1、更新LP单号

guop --> 交易: 3.1.2、发货状态消息

交易 -> 交易: 3.1.2.1、创建支付单

@enduml
```

### TBC链路
```$xslt
@startuml

participant 供销平台 order 1
participant gsp order 2
participant guop order 3
participant 菜鸟 order 4
participant 交易 order 5

供销平台 --> gsp:1、监听到商家发货

gsp -> guop:1.1、发货

guop -> guop:1.1.1、更新履约单状态为packed

guop -> 菜鸟:1.1.2、申请发货

guop -> guop:1.1.3、更新履约状态为rts_pending

菜鸟 --> guop:2、发货成功消息710
destroy 菜鸟
note left
    1、消息发送失败或未发送
    2、guop、交易未能更新状态（买家关单，商家钱货两空）
end note
guop -> guop:2.1、更新履约单状态为已发货

guop --> 交易: 2.2、发货状态消息

guop --> gsp: 2.2、发货状态消息
@enduml
```

### CB&LOCAL(不含卖家宅配)
```$xslt
@startuml

participant 商家后台 order 1
participant guop order 2
participant 菜鸟 order 3
participant 交易 order 4

group CB&LOCAL(不含卖家宅配)链路
    商家后台 -> guop:1、发货

    guop -> guop:1.1、更新履约单状态为packed

    guop -> 菜鸟:1.2、申请发货

    guop -> guop:1.3、更新履约状态为rts_pending

    菜鸟 --> guop:2、发货成功消息
    destroy 菜鸟
    note left
        1、消息发送失败或未发送
        2、guop、交易未能更新状态（买家关单，商家钱货两空）
    end note
    guop -> guop:2.1、更新履约状态为已发货

    guop --> 交易: 2.2、发货状态消息

    交易 -> 交易: 2.2.1、更新物流状态为已发货
end

@enduml
```

### Local卖家宅配
```$xslt
@startuml

participant 商家后台 order 1
participant guop order 2
participant 交易 order 3
participant 定时任务 order 4

group LOCAL卖家宅配链路
    商家后台 -> guop:1、发货

    guop -> guop:1.1、更新履约单状态为packed

    guop -> guop:1.2、更新履约状态为已发货

    guop --> 交易:1.3、发货成功消息

    交易 -> 交易: 1.3.1、更新物流状态为已发货

    定时任务 -> 定时任务:1.4、提交6天自动妥投定时任务

    定时任务 -> guop:2、6天自动妥投
    destroy 定时任务
    note left
        超过6天未妥投
    end note

    guop -> guop:2.1、更新履约状态为妥投

    guop --> 交易:2.2、妥投状态消息
end

@enduml
```

## 揽收成功
```$xslt
@startuml

participant 菜鸟 order 1
participant uop order 2
participant 1688供货中心 order 3
participant guop order 4
participant gsp order 5
participant 交易 order 6

group 1688链路
    菜鸟 --> uop:1、揽收成功消息

    uop --> 1688供货中心:1.1、揽收成功消息

    1688供货中心 --> gsp:1.1.1、揽收成功消息

    gsp -> gsp:1.1.1.1、确认收货
end
group TBC链路
    菜鸟 --> guop:1、揽收成功消息

    guop -> guop:1.1、更新履约状态为运输中

    guop --> gsp:1.2、运输状态消息

    gsp -> gsp:1.2.1、确认收货

    guop --> 交易:1.3、运输状态消息
end

group CB&LOCAL(不包括卖家宅配)链路
    菜鸟 --> guop:1、揽收成功消息

    guop -> guop:1.1、更新履约状态为运输中

    guop --> 交易:1.2、运输状态消息
end

@enduml
```

## 揽收失败

### TP链路（1688）
```$xslt
@startuml

participant 菜鸟 order 1
participant uop order 2
participant 1688供货中心 order 3
participant gsp order 4
participant guop order 5
participant 逆向 order 6
participant 交易 order 7


菜鸟 --> uop:1、揽收失败消息

uop --> 1688供货中心:1.1、揽收失败消息

1688供货中心 -> 1688供货中心:1.1.1、关单

1688供货中心 --> gsp:1.1.2、揽收失败消息

gsp -> guop:2、关单

guop -> guop:2.1、关闭履约单

guop --> 逆向:取消状态信息
destroy guop
    note left
        履约已关单，交易未关（买家收不到退款）
    end note
逆向 -> 交易:关闭交易单

@enduml
```

### TBC链路
```$xslt
@startuml

    菜鸟 --> guop:1、揽收失败消息

    guop -> guop:1.1、关闭履约单

    guop --> gsp:1.2、取消状态消息

    gsp -> gsp:1.3、关单

    guop --> 逆向:1.3.1、取消状态消息
destroy guop
    note left
        履约已关单，交易未关（买家收不到退款）
    end note	
    逆向 -> 交易:1.3.1.1、关闭交易单
@enduml
```

### CB&LOCAL（不含卖家宅配）
```$xslt
@startuml

菜鸟 --> guop:1、揽收失败消息

guop -> guop:1.1、关闭履约单

guop --> 逆向:1.2、取消状态消息
destroy guop
    note left
        履约已关单，交易未关（买家收不到退款）
    end note
逆向 -> 交易:1.2.1、关单

交易 -> 交易:1.2.1.1、关闭交易单

@enduml
```

## 妥投
```$xslt
@startuml

participant 菜鸟 order 1
participant uop order 2
participant guop order 3
participant 交易 order 4

group 1688链路
    菜鸟 --> uop:1、签收成功消息
    destroy 菜鸟
    note left
        菜鸟妥投未发消息
        履约、交易状态未更新
        商家一直不能收款，产生资损
    end note
    uop -> guop:1.1、同步妥投状态

    guop -> guop:1.1.1、更新履约状态为妥投

    guop --> 交易:1.1.2、妥投状态消息
    destroy guop
    note left
        履约状态更新为妥投
        交易状态未更新
        商家一直不能收款，产生资损
    end note
end 1688链路

group TBC链路、CB、Local(不含卖家宅配)
    菜鸟 --> guop:1、签收成功消息
    destroy 菜鸟
    note left
        菜鸟妥投未发消息
        履约、交易状态未更新
        商家一直不能收款，产生资损
    end note
    guop -> guop:1.1、更新履约状态为妥投

    guop --> 交易:1.2、妥投状态消息
    destroy guop
    note left
        履约状态更新为妥投
        交易状态未更新
        商家一直不能收款，产生资损
    end note
end TBC链路、CB、Local(不含卖家宅配)

@enduml
```

## 妥投失败
```$xslt
@startuml

participant 菜鸟 order 1
participant uop order 2
participant guop order 3
participant 交易 order 4
participant 逆向 order 5

group 1688链路
    菜鸟 --> uop:1、投递失败消息

    uop -> guop:1.1、同步投递失败状态

    guop -> guop:1.1.1、更新履约状态为投递失败

    guop --> 交易:1.1.2、投递失败状态消息

    guop --> 逆向:1.1.3、投递失败状态消息

    逆向 -> 交易:1.1.3.1、关单

    交易 -> 交易:1.1.3.2、交易单关闭
end 1688链路

group TBC链路、CB、Local(不含卖家宅配)
    菜鸟 --> guop:1、投递失败消息

    guop -> guop:1.1、更新履约状态为投递失败

    guop --> 交易:1.2、投递失败状态消息

    guop --> 逆向:1.3、投递失败状态消息

    逆向 -> 交易:1.3.1、关单

    交易 -> 交易:1.3.2、交易单关闭
end TBC链路、CB、Local(不含卖家宅配)

@enduml
```

## 逆向

### 关单
```$xslt
@startuml

participant 逆向 order 1
participant guop order 2
participant gsp order 3
participant 交易 order 4

逆向 -> guop:1、关单

guop -> guop:1.1、关闭履约单

guop --> gsp:1.2、取消状态消息

gsp -> gsp:1.2.1、关单

逆向 -> 交易:2、关单

交易  -> 交易:2.1、关闭交易单

@enduml
```

### 退货退款
```$xslt
@startuml

participant 逆向 order 1
participant guop order 2
participant 菜鸟 order 3

逆向 -> guop:1、查询逆向物流方案

guop -> 菜鸟:1.1、查询逆向物流方案

逆向 -> guop:2、创建逆向履约单

guop -> 菜鸟:2.1、发货

菜鸟 --> guop:3、发货成功

guop -> guop:3.1、更新履约单状态，创建履约包裹


@enduml
```

