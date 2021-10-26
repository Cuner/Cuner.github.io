# 订单渲染阶段
```
@startuml

skinparam componentStyle uml2

package "交易域" {
[buy2]
}

package "商品域" {
 [inventoryplatform]
 [ic]
}

[buy2]--->[inventoryplatform]:库存 1:1
[buy2]--->[ic]: 商品信息 1:1

package "海外运营域" {
[ovs_item_service]
}

[buy2]...>[ovs_item_service]: 禁限售判断 1:N

package "会员域" {
[uic]
}

[buy2]--->[uic]: 查询用户信息 1:3

package "店铺域" {
[shop]
}

[buy2]--->[shop]: 查询店铺信息 1:1

package "税费域" {
[hj-tax-calc]
}

[buy2]--->[hj-tax-calc]:税费计算 1:1

package "服务域" {
[tsp]
}

[buy2]--->[tsp]: 获取服务信息 1:1

package "营销域" {
[ump] -down-> [fp]:获取资产 1:1	
}

[buy2]--->[ump]:获取优惠 1:1
[buy2]--->[ump]:获取资产 1:1

package "支付域" {
[cashier] -down->[pg]:支付咨询
[pg]-->[ipay]:支付咨询
}

[buy2]--->[cashier]:支付咨询

package "物流域" {
[delivery]
[adp]-->[sdp]:物流计算 1:1
[sdp]-->[cngse]:菜鸟物流表达、二段运费计算 1:1.5
}

[buy2]--->[adp]: 计算物流方式&运费，物流拆单 1:2
[buy2]--->[delivery]: 查询集运仓 1:1
[buy2]...>[cngse]: 查询未选中的物流方式信息 1:1


@enduml
```

# 订单提交阶段
```
@startuml

skinparam componentStyle uml2

package "交易域" {
[buy2]-down->[tp3]:订单创建 1:1
}

package "商品域" {
 [inventoryplatform]
 [ic]
}

[buy2]--->[inventoryplatform]:库存 1:1
[buy2]--->[ic]: 商品信息 1:1

package "会员域" {
[uic]
}

[buy2]--->[uic]: 查询用户信息 1:3

package "店铺域" {
[shop]
}

[buy2]--->[shop]: 查询店铺信息 1:1

package "税费域" {
[hj-tax-calc]
}

[buy2]--->[hj-tax-calc]:税费计算 1:1

package "服务域" {
[tsp]
}

[buy2]--->[tsp]: 服务信息确认 1:1

package "营销域" {
[ump] -down-> [fp]:资产确认 1:1	
}

[buy2]--->[ump]:优惠确认 1:1
[buy2]--->[ump]:资产确认 1:1


package "支付域" {
[cashier] -down->[pp]:支付
[pp]-down->[pg]:创单
[pp]->[pg]:支付
[pg]-->[alipay]:创单
[pg]-->[alipay]:支付
[alipay]-down->[ipay]:支付
}

[tp3]--->[pp]:创单
[buy2]--->[cashier]:支付

package "物流域" {
[delivery]
[adp]-->[sdp]:物流确认 1:1
[sdp]-->[cngse]:物流确认 1:1
}

[buy2]--->[adp]: 物流确认 1:1
[buy2]--->[delivery]: 查询集运仓与自提点 1:2

@enduml
```