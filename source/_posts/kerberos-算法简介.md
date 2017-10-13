---
title: kerberos 算法简介
date: 2017-10-12 19:37:20
tags: 
  - kerberos
  - algorithm
  - authentication
---

## Kerberos 简介
+ 它是 authentication protocol, 只管认证, 不管授权(authorization).
+ 可以实现单点登录 (single sign-on authentication), 一个密码, 认证所有 service.
+ work over untrusted networks.
+ work between trusted systems.
+ 默认使用 UDP port: 88
+ Kerberos 一定不会发送任何密码 (或密码的 hash, 或 key), 即使是加密过的密码 (或加密过的密码 hash) 也绝不会发送;
+ 使用 "对称加密" 保证通信安全;

据说在企业内部系统的 authentication, kerberos 的应用是极其广泛的, 并没有特别值得一题的竞品.


## Kerberos 验证身份的思路 
kerberos 协议中互相通信的实体: 
client, server, AS (kerberos authentication service), TGS (kerberos ticket granting service)  

kerberos 协议的目标: 
+ 使得 client 和 server 之间可以互相确认身份 (authentication); 
  一旦 kerberos 认证成功, client 和 server 都可以确定它们在和可信真实的对象打交道.
  server 可以放心地对 client 提供 application service, 开放敏感数据; client 也不必担心 server 是假冒的
+ client 和 server 之间将会共同得到一个 `app-service session key`, 
  之后它们之间的通信将通过这个 key 和对称加密算法保证安全性, 不被第三方窃听

TGT: 可以认为是一个 "长效证件", 在它失效之前, client 都不必重新找 AS 再次认证.
client key: 一般来说是 "client 密码的 hash", 只有 client 和 AS 知道 (后面的 server key 同理).

client 向 AS 自证身份, 得到 TGT: 
+ client 向 AS 发送: `username` 和 `用 client key 加密的 timestamp`.  
+ AS 在 DB 中找到 client key, 尝试解密收到的密文; 
  如果解密成功, 并且得到有效的时间戳, 则 **client 就成功地向 AS 证明了自己的身份**.
  (这是典型的 kerberos 式的 "通过解密验证身份")
  (加密的内容为什么选择时间戳: 通过时间戳来杜绝 "重放攻击")
+ AS 向已验明正身的 client 发送 TGT. 
  TGT 本身是一串密文, 只有 TGS 才能解密.
  同时也发送 `TGS session key` (用 client key 加密). 
  之后 client 和 TGS 之间的通信, 将用这个 TGS session key 保证通信安全
+ client 收到 TGT, 并通过 client key 解密得到 `TGS session key` 

当 client 需要和 server 建立通信时:
+ client 向 TGS 发送 TGT (英文: client just blindly send TGT to TGS).
  注意: TGT 本身已经是密文, 故 client 不需要对其做处理
+ TGS 收到 TGT, 进行解密 (注意, 前面提到: 只有 TGS 才能解读作为密文的 TGT); 
  如果得到有效信息, **client 就向 TGS 证明了自己的身份** (典型的 kerberos 式的 "通过解密验证身份")
+ TGS 向 client 发送两份加密信息: 
  `用 TGS session key 加密的 app-service session key` 和 `app service ticket (本身是密文, 用 server key 加密过)`
  其中, `app-service session key` 在认证成功之后, 将被用于将来 client 和 server 间通信的加密
+ client 将本身是密文的 `app-service ticket` 发送给 server 
  (英文: blindly send `app-service ticket` to server);
  并且, 通过 `TGS session key` 解开密文, 得到 `app-service session key`
+ server 将尝试用自身的 server key 对作为密文的 `app-service ticket` 解密, 如果解密成功,
  **client 就向 server 证明了自己的身份** (典型的 kerberos 式的 "通过解密验证身份")
+ server 从解密后的 `app service ticket` 得到 `app service session key`, 
  利用它加密 timestamp 发送给 client
+ client 尝试通过 `app service session key` 对收到的密文解密, 如果得到有效的 timestamp, 
  则 **server 向 client 证明了自己的身份**
+ 认证完毕, client 和 server 之后将使用 `app service session key` 加密它们之间的通信

注意:
+ client 向 AS 自证身份的道具, 是用自己的 key 给 timestamp 加密的密文; 因为只有 AS 才也知道 client key
+ client 向 server 自证身份的道具, 用的是 TGS 给的 `app-service ticket`; 
  这个 ticket 是 TGS 帮忙用 server key 加密好的;
+ server 向 client 自证身份的道具, 
  是用自己从 `app-service ticket` 解密出来的 `app service session key` (用它加密一个时间戳)

#### 小细节
实际上, client 在发送 `app-service ticket` 给 server 的时候, 还发送了被加密的 authenticator.
因为 server 需要一个办法知道解密得到的 `app-service ticket` 确实是有效的; 

解决方法:
authenticator 是被 `app-service session key` 加密的, authenticator 中包含时间戳. 
server 用从 `app-service ticket` 中得到的 `app-service session key` 解密 authenticator, 
检查能否得到有效的时间戳, 即可.

不过, 这只是多种验证 `app-service ticket` 方法中的一种, 不是 kerberos 算法思路的核心.
可以发现, 每一步 "验证身份" 的过程都包括了 "发送和验证有效的时间戳", 这样才能杜绝重放攻击.


## Reference
+ http://web.mit.edu/tsitkova/www/build/index.html Kerb 的文档都在这里.
+ Simon Wilkinson, Kerberos Tutorial
+ https://msdn.microsoft.com/en-us/library/bb742516.aspx kerberos explained 重点分明, 把核心介绍得清楚, 略去不少细节



