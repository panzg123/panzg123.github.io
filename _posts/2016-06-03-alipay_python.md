---
layout: post
title: 支付宝即时到账接口
categories: 项目实战
---

支付宝网页即时到账功能，可让用户在线向开发者的支付宝账号支付资金，交易资金即时到账，帮助开发者快速回笼资金。

官方文档中，未提供`Python`接口，因此对自己的使用过程做个笔记。

### 准备工作

1. 目前来说，需要商家账户才能签约即时到账，不支持个人账户，如有需求只能走第三方服务，知乎有相关问答。
2. 需要到后台配置ID、密钥、公钥，具体流程参考[支付宝开放平台](https://doc.open.alipay.com/doc2/detail.htm?spm=a219a.7629140.0.0.yUnjSg&treeId=62&articleId=104739&docType=1 )

### 构造支付请求

 构造请求参数，需要了解各参数的含义，查看支付宝文档，几个重要的参数如下：

- service：接口名称，PC网站支付为`create_direct_pay_by_user`,wap手机网站支付为`alipay.wap.create.direct.pay.by.user`
- `partner`，合作商ID
- `sign_type`，签名方式
- `sign`，签名
- `notify_url`，支付回调通知地址
- `return_url`,支付跳转地址

然后就是一些业务参数，比如

- out_trade_no,订单号
- subject，商品名称
- seller_email,卖家账号
- buyer_email，卖家价格
- price,价格
- body,商品描述

一个即时到账支付请求的例子：
> `https:``//mapi.alipay.com/gateway.do?
body=%C3%C0%B9%FA%D7%A8%D2%B5%BB%A4%CD%F3%CA%F3%B1%EA%B5%E6%2C%CA%E6%BB%BA%CA%BD%C4%FD%BD%BA%C8%ED%B5%E6%C4%A3%C4%E2
%CA%D6%CD%F3%B5%C4%D7%D4%C8%BB%C7%FA%CF%DF%BA%CD%D4%CB%B6%AF%A3%AC%B4%B4%D4%EC%BA%CD%BB%BA%B5%C4GelFlex%CA%E6%CA%CA%B5%D8%B4%F8%21
&subject=%B1%B4%B6%FB%BD%F0%BB%A4%CD%F3%CA%BD&sign_type=MD5&notify_url=http%3A%2F%2Fapi.test.alipay.net&out_trade_no=6741334835157966
&return_url=http%3A%2F%2Fapi.test.alipay.net%2Fatinterface%2Freceive_return.htm&sign=dc3d42f405d7e738ab35344449e2d9f7
&buyer_id=2088002007018955&total_fee=100&service=create_direct_pay_by_user&partner=2088101568338364&seller_id=2088002007018966
&payment_type=1&qr_pay_mode=1&_input_charset=gbk`

### 支付成功跳转地址
支付成功后，会跳转到`return_url`，网站需要根据支付返回的信息，来做相应处理。
主要工作就是对return_url支付宝传递的参数的解析，对订单做校验，改变订单数据库的状态，完成交易。
在return_url中，支付宝会传递主要如下参数：

- sign，签名
- `trade_status`，交易状态，是否支付成功
- `trade_no`,交易号
- subject,商品名称
- buyer_id，卖家账号

一般来说，trade_no是网站后台系统生成的唯一`id`，根据返回的trade_no,trade_status来改变订单数据库的状态。

在return_url的`Handler`中，主要工作是向支付宝校验url参数数据；根据`trade_no,trade_status`来改变订单状态，并在后台系统做相应记录和处理，以及网页的跳转。
如果，后端系统通过异步通知来处理订单，则只需对return_url做相应校验，并跳转即可。

### 异步通知地址
支付宝对商户的请求数据处理完成后，会将处理的结果数据通过`服务器主动通知的方式通知给商户网站`。这些处理结果数据就是服务器异步通知参数。
该方式的特点是，支付宝服务器主动通知商户网站支付处理结果数据,通知地址为`notify_url`

相应参数与return_url基本一致，后端系统再相应的Handler中来处理订单。
不过要注意以下几点：

- 支付宝是用`POST`方式发送通知信息；
- 支付宝主动发起通知，该方式才会被启用，且是服务器之间的交互，前端不可见；
- 商户后台`notify_url`的Handler处理结束后，必须打印输出`success`。如果商户反馈给支付宝的字符不是success这7个字符，支付宝服务器会不断重发通知，直到超过24小时22分钟;
- 重复通知的频率，在一般情况下，25小时以内完成8次通知（通知的间隔频率一般是：4m,10m,10m,1h,2h,6h,15h）；
- `notify_url`中不能进行页面跳转，否则支付宝默认异常，另外调试必须在线上调试，本地不能调试的。
- 该方式的作用主要防止订单丢失，即页面跳转同步return_url通知没有处理订单更新，它则去处理。


由于项目原因，代码不能展示，但可以参考`Reference`中的链接1，该项目提供`即时到账`,`担保记忆`,`自动发货接口`，截图如下：
![接口图](http://7xsvsk.com1.z0.glb.clouddn.com/alipay.png "接口图")


### Reference
1. [支付宝 alipay python接口，支持担保交易，即时到帐和自动发货接口]( https://github.com/fengli/alipay_python)
2. [支付宝开放中心]( https://doc.open.alipay.com/doc2/alipayDocIndex.htm)