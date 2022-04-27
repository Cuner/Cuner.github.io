reverse_order
```
CREATE TABLE `reverse_order` (
  `reverse_order_id` bigint(20) unsigned NOT NULL COMMENT '主键(reverse numer)',
  `trade_order_id` bigint(20) unsigned NOT NULL COMMENT '交易单ID(trade order id)',
  `buyer_id` bigint(20) unsigned NOT NULL COMMENT '买家ID(buyer id)',
  `apply_comment` varchar(512) DEFAULT NULL COMMENT '申请备注（comments）',
  `request_type` tinyint(3) unsigned NOT NULL COMMENT '请求类型,request tpye(1:cancel,2:return,3:replacement)',
  `reverse_number` varchar(64) DEFAULT NULL COMMENT '字符串退款编号(old bob reverse number or new reverse id)',
  `refund_payment_method` varchar(64) DEFAULT NULL COMMENT '退款方式(refund payment method)',
  `payment_method` varchar(64) DEFAULT NULL COMMENT '下单支付方式(trade order payment method)',
  `payment_data` varchar(2048) DEFAULT NULL COMMENT '支付数据(bank account info)',
  `shipping_type` tinyint(3) unsigned NOT NULL DEFAULT '0' COMMENT '退货类型（shipping option mehtod,1:pick up,2:drop off,3:customer send）',
  `create_role` tinyint(4) NOT NULL COMMENT '创建角色,1:buyer,2:seller,3:seller admin,4:customer service,5:system,6:timeout,7:ov risk',
  `features` varchar(2048) NOT NULL COMMENT '属性(features)',
  `site_id` varchar(10) NOT NULL COMMENT '站点（deploy site）',
  `gmt_create` bigint(20) unsigned NOT NULL COMMENT '创建时间(create time)',
  `gmt_modified` bigint(20) unsigned NOT NULL COMMENT '修改时间(modify time)',
  `sync_time` datetime DEFAULT '0000-00-00 00:00:00' COMMENT '老数据库同步时间（old data sync time）',
  `display_status` tinyint(4) NOT NULL DEFAULT '1' COMMENT '1:show,0:hidden',
  `sync_version` int(10) unsigned DEFAULT '0' COMMENT 'just for esdb sync judge,Avoid wrong judgment',
  `buyer_full_name` varchar(256) DEFAULT NULL COMMENT '买家全名(buyer full name)',
  `features_cc` int(11) NOT NULL COMMENT '乐观锁（optimise lock）',
  `trade_order_gmt_create` bigint(20) unsigned DEFAULT NULL COMMENT 'trade order create time',
  `bob_id_reverse_order` bigint(20) unsigned DEFAULT NULL COMMENT 'bob trade order id',
  `bob_order_nr` bigint(20) unsigned DEFAULT NULL COMMENT 'bob order number',
  `is_deleted` tinyint(4) DEFAULT NULL COMMENT '是否删除',
  PRIMARY KEY (`reverse_order_id`),
  KEY `idx_buyer_id` (`buyer_id`),
  KEY `idx_trade_order_id` (`trade_order_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='逆向单'
;
```

reverse_order_line

```
CREATE TABLE `reverse_order_line` (
  `reverse_order_line_id` bigint(20) unsigned NOT NULL COMMENT '主键（reverse item id）',
  `reverse_order_id` bigint(20) unsigned NOT NULL COMMENT '逆向主订单ID(reverse oder id)',
  `trade_order_id` bigint(20) unsigned NOT NULL COMMENT '交易单ID（trade order id）',
  `trade_order_line_id` bigint(20) unsigned NOT NULL COMMENT '交易商品单ID(trade order item id)',
  `seller_id` bigint(20) unsigned NOT NULL COMMENT '卖家ID(seller id)',
  `buyer_id` bigint(20) unsigned NOT NULL COMMENT '买家ID(buyer id)',
  `refund_amount` bigint(20) unsigned NOT NULL COMMENT '退款金额(refund amount)',
  `paid_amount` bigint(20) unsigned DEFAULT NULL COMMENT '支付金额(trade paid amount)',
  `reverse_status` tinyint(3) unsigned NOT NULL COMMENT '逆向状态(1:REQUEST_INITIATE,2:REQUEST_REJECT,3:REQUEST_CANCEL,4:CANCEL_SUCCESS,5:REFUND_PENDING,6:REFUND_SUCCESS,7:REFUND_REJECT,8:REPLACE_PENDING,9:REPLACE_SUCCESS,10:REFUND_AUTHORIZED)',
  `goods_status` tinyint(3) unsigned NOT NULL COMMENT '货物状态(1:UNSHIP,SHIPPED,3:DELIVERED,4:PICKUP_SCHEDULED,5:RECEIVED,6:INBOUNDED,7:SHIPPED_BACK,8:RETURN_DELIVERED,9:LOST)',
  `reason_id` int(11) NOT NULL COMMENT '退款原因ID(reason id)',
  `reason_text` varchar(256) DEFAULT NULL COMMENT '退款原因文案(reaosn text)',
  `item_id` bigint(20) unsigned NOT NULL COMMENT '商品ID(item id)',
  `item_title` varchar(512) NOT NULL COMMENT '商品标题(item title)',
  `item_pic_url` varchar(512) DEFAULT NULL COMMENT '商品图片地址(item pic url)',
  `item_unit_price` bigint(20) unsigned DEFAULT NULL COMMENT '商品单价(item unit price)',
  `sku_id` varchar(64) DEFAULT NULL COMMENT '商家商品SKU(item sku id)',
  `payment_method` varchar(64) DEFAULT NULL COMMENT '支付方式(trade payment method)',
  `refund_payment_method` varchar(64) DEFAULT NULL COMMENT '逆向退款方式(refund payment method)',
  `request_type` tinyint(3) unsigned NOT NULL COMMENT '请求类型(1:cancel,2:return,3:replacement)',
  `warehouse_qc` tinyint(4) NOT NULL DEFAULT '0' COMMENT '0:none,1:failed,2:pass',
  `features` varchar(4000) NOT NULL COMMENT '属性(features)',
  `features_cc` int(11) NOT NULL COMMENT '乐观锁(optimise lock)',
  `currency_code` varchar(10) NOT NULL COMMENT '货币(currency)',
  `occurred_epoch_mills` bigint(20) unsigned NOT NULL COMMENT 'currency time',
  `site_id` varchar(10) NOT NULL COMMENT '站点(deploy site)',
  `gmt_create` bigint(20) unsigned NOT NULL COMMENT '创建时间(create time)',
  `gmt_modified` bigint(20) unsigned NOT NULL COMMENT '修改时间(modify time)',
  `sync_time` datetime DEFAULT '0000-00-00 00:00:00' COMMENT '老数据库同步时间(old data sync time)',
  `display_status` tinyint(3) unsigned NOT NULL DEFAULT '1' COMMENT '1:show,0:hidden',
  `buyer_full_name` varchar(256) DEFAULT NULL COMMENT '买家全名(buyer full name)',
  `seller_full_name` varchar(256) DEFAULT NULL COMMENT '卖家全名(seller full name)',
  `remaining_amount` bigint(20) unsigned NOT NULL COMMENT '剩余可退金额(remaining amount)',
  `sync_version` int(10) unsigned DEFAULT '0' COMMENT 'just for esdb sync judge,Avoid wrong judgment',
  `trade_order_gmt_create` bigint(20) unsigned DEFAULT NULL COMMENT 'trade order create time',
  `bob_id_reverse_order_line` bigint(20) unsigned DEFAULT NULL COMMENT 'bob trade order item id',
  `biz_code` varchar(64) DEFAULT NULL COMMENT 'reverse biz code',
  `dispute_status` int(11) DEFAULT NULL COMMENT 'dispute状态',
  `dispute_features` varchar(1024) DEFAULT NULL COMMENT 'dispute标',
  `timeout_type` varchar(64) DEFAULT NULL COMMENT '超时类型',
  `timeout_trigger` bigint(20) unsigned DEFAULT NULL COMMENT '超时触发时间',
  `reject_reason_id` bigint(20) unsigned DEFAULT NULL COMMENT '卖家拒绝原因ID',
  `fund_features` varchar(1024) DEFAULT NULL COMMENT '资金明细',
  `order_from` tinyint(4) NOT NULL DEFAULT '1' COMMENT '订单来源',
  `clawback_features` varchar(1024) DEFAULT NULL COMMENT 'clawback资金分摊',
  `fund_detail` varchar(2048) DEFAULT NULL COMMENT '新资金明细',
  `is_deleted` tinyint(4) DEFAULT NULL COMMENT '是否删除',
  `ofc_status` varchar(2048) DEFAULT NULL COMMENT 'ofc详细状态',
  PRIMARY KEY (`reverse_order_line_id`),
  KEY `idx_buyer_id` (`buyer_id`),
  KEY `idx_trade_order_id` (`trade_order_id`),
  KEY `idx_trade_order_line_id` (`trade_order_line_id`),
  KEY `idx_reverse_status` (`reverse_status`),
  KEY `idx_request_type` (`request_type`),
  KEY `idx_gmt_create` (`gmt_create`),
  KEY `idx_order_from` (`order_from`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='逆向单商品行'
;
```

reverse_order_logistic

```
CREATE TABLE `reverse_order_logistic` (
  `reverse_order_logistic_id` bigint(20) unsigned NOT NULL COMMENT '主键(reverse order logisitc id)',
  `buyer_id` bigint(20) unsigned NOT NULL COMMENT '买家ID(buyer id)',
  `reverse_order_id` bigint(20) unsigned NOT NULL COMMENT '逆向单ID(reverse order id)',
  `receiver_full_name` varchar(512) DEFAULT NULL COMMENT '收货人全名(receiver full name)',
  `receiver_contract_no` varchar(36) DEFAULT NULL COMMENT '收货人联系电话(receiver phone)',
  `receiver_post_code` varchar(24) DEFAULT NULL COMMENT '收货人邮政编码(receiver post code)',
  `logistic_type` varchar(24) DEFAULT NULL COMMENT '物流类型(logistic type)',
  `logistic_company` varchar(128) DEFAULT NULL COMMENT '物流公司名称(shipping provider name)',
  `logistic_company_code` varchar(64) DEFAULT NULL COMMENT '物流公司编码(shipping provider code)',
  `tracking_number` varchar(128) DEFAULT NULL COMMENT '寄货物流编号(tracking number)',
  `shipping_full_address` varchar(512) DEFAULT NULL COMMENT '买家寄货地址(shipping full address)',
  `shipping_type` tinyint(4) DEFAULT NULL COMMENT '寄货类型(1:pick-up,2:drop-off,3:customer-send)',
  `shipping_time` bigint(20) unsigned DEFAULT NULL COMMENT '寄货时间(shipping time)',
  `features` varchar(2048) DEFAULT NULL COMMENT '属性(features)',
  `site_id` varchar(10) NOT NULL COMMENT '站点(deploy site)',
  `gmt_modified` bigint(20) unsigned NOT NULL COMMENT '修改时间(modify time)',
  `gmt_create` bigint(20) unsigned NOT NULL COMMENT '创建时间(create time)',
  `sync_time` datetime DEFAULT NULL COMMENT '老数据库同步时间(old data sync time)',
  `trade_order_id` bigint(20) unsigned DEFAULT NULL COMMENT '交易订单号(trade order id)',
  `shipping_address_id` varchar(96) DEFAULT NULL COMMENT '寄货地址ID(shipping address id)',
  `receiver_address_id` varchar(64) DEFAULT NULL COMMENT '收货人地址ID(recevier address id)',
  `receiver_full_address` varchar(512) DEFAULT NULL COMMENT '收货地址(receiver full address)',
  `features_cc` int(11) DEFAULT NULL COMMENT '乐观锁(optimise lock)',
  `sync_version` int(10) unsigned DEFAULT '0' COMMENT 'just for esdb sync judge,Avoid wrong judgment',
  PRIMARY KEY (`reverse_order_logistic_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='逆向物流表'
;
```

reverse_attachment

```
CREATE TABLE `reverse_attachment` (
  `id` bigint(20) unsigned NOT NULL COMMENT '主键',
  `gmt_create` bigint(20) unsigned NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) unsigned NOT NULL COMMENT '修改时间',
  `reverse_order_line_id` bigint(20) unsigned DEFAULT NULL COMMENT '逆向子单id',
  `buyer_id` bigint(20) unsigned DEFAULT NULL COMMENT '买家id',
  `attachment_type` varchar(64) DEFAULT NULL COMMENT '附件类型（picture、video）',
  `status` tinyint(4) DEFAULT NULL COMMENT '附件状态',
  `file_key` varchar(512) DEFAULT NULL COMMENT '文件fileserver2的key',
  `file_url` varchar(512) DEFAULT NULL COMMENT '文件url（优先使用file_key计算）',
  `is_deleted` tinyint(4) DEFAULT NULL COMMENT '是否删除',
  `features` varchar(4000) DEFAULT NULL COMMENT '属性(features)',
  `site_id` varchar(10) DEFAULT NULL COMMENT '站点id',
  `operator_id` varchar(64) DEFAULT NULL COMMENT '操作id',
  `operator_role` int(11) DEFAULT NULL COMMENT '操作人角色',
  `trade_order_line_id` bigint(20) unsigned DEFAULT NULL COMMENT '正向子单id',
  `reverse_record_id` bigint(20) unsigned DEFAULT NULL COMMENT '操作记录id',
  PRIMARY KEY (`id`),
  KEY `idx_reverse_order_line_id` (`reverse_order_line_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='逆向单附件（图片、视频）'
;
```

reverse_record
```
CREATE TABLE `reverse_record` (
  `reverse_record_id` bigint(20) unsigned NOT NULL COMMENT '主键(record id)',
  `out_id` varchar(256) DEFAULT NULL COMMENT 'out ID(out id)',
  `reverse_id` bigint(20) unsigned NOT NULL COMMENT '逆向订单ID(reverse order id or reverse order line id)',
  `reverse_level` tinyint(4) NOT NULL COMMENT '逆向单级别(主订单基本 or 子订单)(reverse item level or order level)',
  `buyer_id` bigint(20) unsigned NOT NULL COMMENT '买家ID(buyer id)',
  `operator_id` varchar(64) DEFAULT NULL COMMENT '操作者ID(operator id)',
  `operator_fullname` varchar(96) DEFAULT NULL COMMENT '操作者fullName(operator full name)',
  `operator_role` tinyint(4) NOT NULL COMMENT '操作角色(operator role)',
  `record_type` tinyint(4) NOT NULL COMMENT '记录类型(COMMENT:1,ACTION:2,STATUS:3)',
  `display_status` tinyint(4) NOT NULL COMMENT '显示状态(is display，111 binary)',
  `pre_reverse_status` tinyint(4) DEFAULT NULL COMMENT '上一次逆向单状态(pre reverse status)',
  `reverse_status` tinyint(4) NOT NULL COMMENT '逆向单当前状态(current reverse status)',
  `contents` varchar(4096) DEFAULT NULL COMMENT '内容，k-v形式存储(record content)',
  `site_id` varchar(10) DEFAULT NULL COMMENT '站点ID(deploy site)',
  `gmt_create` bigint(20) unsigned NOT NULL COMMENT '创建时间(create time)',
  `gmt_modified` bigint(20) unsigned NOT NULL COMMENT '修改时间(modify time)',
  `sync_time` datetime DEFAULT NULL COMMENT 'data migrate time',
  `trade_order_line_id` bigint(20) unsigned DEFAULT NULL COMMENT 'trade order line id',
  `is_deleted` tinyint(4) DEFAULT NULL COMMENT '是否删除',
  `features` varchar(4000) DEFAULT NULL COMMENT 'k-v 形式',
  PRIMARY KEY (`reverse_record_id`),
  UNIQUE KEY `uk_outer_id` (`out_id`),
  KEY `idx_record` (`reverse_id`,`record_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='逆向订单记录表'
;
```

reverse_reason
单表, 建在0000库
```
CREATE TABLE `reverse_reason` (
  `reverse_reason_id` bigint(20) NOT NULL COMMENT '主键',
  `gmt_create` bigint(20) unsigned NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) unsigned NOT NULL COMMENT '修改时间',
  `display_text` varchar(512) NOT NULL COMMENT '原因文本内容',
  `reason_status` tinyint(4) NOT NULL COMMENT '状态',
  `reason_type` tinyint(4) NOT NULL COMMENT '原因类型(申请 or 拒绝)',
  `request_type` tinyint(4) DEFAULT NULL COMMENT '逆向请求(换货 or cancel)',
  `item_category` bigint(20) unsigned DEFAULT NULL COMMENT '商品类目',
  `features` varchar(1024) DEFAULT NULL COMMENT '扩展字段',
  `display_order` int(11) DEFAULT NULL COMMENT '显示顺序',
  `site_id` varchar(10) NOT NULL COMMENT '站点ID',
  `reason_source` int(11) NOT NULL COMMENT '原因操作来源',
  `return_policy_id` varchar(512) DEFAULT NULL COMMENT '退货政策ID的列表,英文逗号分隔',
  `editor_id` bigint(20) unsigned NOT NULL COMMENT '最后修改人ID',
  `editor_name` varchar(128) NOT NULL COMMENT '最后修改人姓名',
  `mandatory` tinyint(4) DEFAULT NULL COMMENT '选择原因后，规则是否必填',
  `oos` tinyint(4) DEFAULT NULL COMMENT '退款选择该原因后，置商品为下架状态',
  PRIMARY KEY (`reverse_reason_id`),
  KEY `idx_reason` (`reason_source`,`reason_status`,`reason_type`,`request_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='逆向订单原因表'
;
```

reverse_origin_reason
单表, 建在0000库
```
CREATE TABLE `reverse_origin_reason` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `gmt_create` bigint(20) unsigned NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) unsigned NOT NULL COMMENT '修改时间',
  `parent_id` bigint(20) unsigned NOT NULL DEFAULT '0' COMMENT '父ID',
  `reason_type` tinyint(4) NOT NULL COMMENT '1:申请,2:拒绝',
  `default_text` varchar(1024) DEFAULT NULL COMMENT '默认原因',
  `duty` tinyint(4) NOT NULL COMMENT '1:买家,2:卖家,3:物流',
  `sync_version` int(10) unsigned NOT NULL DEFAULT '1' COMMENT '乐观锁',
  `reason_object` int(11) NOT NULL COMMENT '1:买家,2:卖家,4系统',
  `features` varchar(4096) DEFAULT NULL COMMENT 'features',
  `site_id` varchar(128) NOT NULL COMMENT '站点',
  `is_deleted` tinyint(4) DEFAULT NULL COMMENT '是否已删除',
  PRIMARY KEY (`id`),
  KEY `idx_parent_id` (`parent_id`),
  KEY `idx_query` (`reason_type`,`reason_object`,`duty`)
) ENGINE=InnoDB AUTO_INCREMENT=20000001 DEFAULT CHARSET=utf8 COMMENT='原因基础信息表'
;
```

reverse_apply_reason_relative
单表, 建在0000库
```
CREATE TABLE `reverse_apply_reason_relative` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `gmt_create` bigint(20) unsigned NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) unsigned NOT NULL COMMENT '修改时间',
  `origin_reason_id` bigint(20) unsigned NOT NULL COMMENT '元数据ID',
  `default_text` varchar(1024) DEFAULT NULL COMMENT '默认原因文案',
  `multi_lang_key` varchar(512) NOT NULL COMMENT 'mcms key',
  `category_id` bigint(20) unsigned NOT NULL COMMENT '类目',
  `site_id` varchar(128) NOT NULL COMMENT '站点',
  `reverse_type` tinyint(4) NOT NULL COMMENT '同逆向子单上的reveres_type',
  `status` tinyint(4) NOT NULL DEFAULT '0' COMMENT '0:不可用,1:隔离,2:发布',
  `is_received` tinyint(4) NOT NULL COMMENT '0:未收到,1:已收到',
  `features` varchar(4096) DEFAULT NULL COMMENT 'feature',
  `sync_version` int(10) unsigned NOT NULL DEFAULT '1' COMMENT '同步锁',
  PRIMARY KEY (`id`),
  KEY `idx_query` (`site_id`,`status`,`reverse_type`,`is_received`,`origin_reason_id`,`category_id`)
) ENGINE=InnoDB AUTO_INCREMENT=20000002 DEFAULT CHARSET=utf8 COMMENT='申请原因关系表'
;
```

reverse_rejective_reason_relative
单表, 建在0000库
```
CREATE TABLE `reverse_rejective_reason_relative` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `gmt_create` bigint(20) unsigned NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) unsigned NOT NULL COMMENT '修改时间',
  `origin_reason_id` bigint(20) unsigned NOT NULL COMMENT '元数据ID',
  `relative_apply_reason_id` bigint(20) unsigned NOT NULL COMMENT '关联的申请原因ID',
  `default_text` varchar(1024) DEFAULT NULL COMMENT '默认文案',
  `multi_lang_key` varchar(128) NOT NULL COMMENT '多语言',
  `status` tinyint(4) NOT NULL COMMENT '0:不可用,1:隔离,2:发布',
  `site_id` varchar(128) NOT NULL COMMENT '站点',
  `features` varchar(4096) NOT NULL COMMENT 'feature',
  `sync_version` int(11) NOT NULL DEFAULT '1' COMMENT '乐观锁',
  PRIMARY KEY (`id`),
  KEY `idx_query` (`site_id`,`status`,`relative_apply_reason_id`)
) ENGINE=InnoDB AUTO_INCREMENT=10000200 DEFAULT CHARSET=utf8 COMMENT='拒绝原因关系表'
;
```

reverse_dict_config
单表, 建在0000库
```
CREATE TABLE `reverse_dict_config` (
  `reverse_dict_id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `gmt_create` bigint(20) unsigned NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) unsigned NOT NULL COMMENT '修改时间',
  `display_text` varchar(256) DEFAULT NULL COMMENT '显示文本',
  `dict_value` varchar(512) NOT NULL COMMENT '配置的值',
  `dict_type` varchar(32) DEFAULT NULL COMMENT '字典分类(Return policy/Cancel policy/Return method)',
  `dict_status` tinyint(4) NOT NULL COMMENT '状态',
  `dict_code` varchar(36) NOT NULL COMMENT '编码',
  `site_id` varchar(10) NOT NULL COMMENT '站点ID',
  `country_code` varchar(10) DEFAULT NULL COMMENT '国家编码',
  `sort_id` tinyint(4) DEFAULT NULL COMMENT '排序',
  `dict_sub_type` varchar(32) DEFAULT NULL COMMENT '字典子分类',
  `operator_id` varchar(64) DEFAULT NULL COMMENT '操作者ID',
  `operator_name` varchar(128) DEFAULT NULL COMMENT '操作者名称',
  PRIMARY KEY (`reverse_dict_id`),
  KEY `idx_dict_code_type_subtype` (`dict_code`,`dict_type`,`dict_sub_type`),
  KEY `idx_dict_type_subtype` (`dict_sub_type`,`dict_type`)
) ENGINE=InnoDB AUTO_INCREMENT=71 DEFAULT CHARSET=utf8mb4 COMMENT='逆向字典配置表'
;
```

reverse_rules
单表, 建在0000库
```
CREATE TABLE `reverse_rules` (
  `reverse_rules_id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `gmt_create` bigint(20) unsigned NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) unsigned NOT NULL COMMENT '修改时间',
  `scenario_code` varchar(256) NOT NULL COMMENT '场景编码',
  `display_text` varchar(256) DEFAULT NULL COMMENT '备注,显示信息',
  `rule_code` varchar(64) NOT NULL COMMENT '规则项ID',
  `rule_value` varchar(256) NOT NULL COMMENT '规则对应的值',
  `site_id` varchar(10) NOT NULL COMMENT '站点ID',
  `rule_status` tinyint(4) NOT NULL COMMENT '状态',
  `features` varchar(2048) DEFAULT NULL COMMENT '扩展字段',
  `target_value` varchar(256) NOT NULL COMMENT '目标对象类型的值(具体卖家类型 or 类目ID)',
  `target_type` varchar(64) NOT NULL COMMENT '目标对象类型(代表是类目 or 卖家类型)',
  `editor_id` bigint(20) unsigned NOT NULL COMMENT '最后修改人ID',
  `editor_name` varchar(128) NOT NULL COMMENT '最后修改人姓名',
  PRIMARY KEY (`reverse_rules_id`),
  KEY `idx_rules` (`scenario_code`,`target_type`,`rule_status`)
) ENGINE=InnoDB AUTO_INCREMENT=116 DEFAULT CHARSET=utf8mb4 COMMENT='规则组和规则对应表'
;
```

sequence
单表, 建在0000库和0008库
```
CREATE TABLE `sequence` (
  `name` varchar(64) NOT NULL COMMENT 'Sequence Name',
  `value` bigint(20) NOT NULL COMMENT 'Sequence Value',
  `gmt_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Modified Time',
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT 'Primary key',
  `gmt_create` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'create time',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_unique_name` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=18 DEFAULT CHARSET=utf8mb4 COMMENT='Global Sequence Table'
;
```