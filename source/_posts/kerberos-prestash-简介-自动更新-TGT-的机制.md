---
title: 'kerberos pre-stash 简介: 自动更新 TGT 的机制'
date: 2017-10-13 12:19:24
tags: 
  - kerberos
  - pre-stash
  - security
---

## pre-stash; krb5-prestash
参考 http://oskt.secure-endpoints.com/krb5_prestash.1.html 不过链接里说得并不清楚.

解决什么问题:
我们不希望 long running 的 service 因为 kerberos ticket 失效而产生 outage.
所以, 需要一个定期帮助 long running service 获取和刷新 TGT.
当然解决方法有很多种, pre-stash 是其中一种

pre-stashed tickets:
Used by long running client to authenticate itself to kerberized services.

"pre-stashed" 这个动词的含义: 
将 client 需要的 kerberos ticket 预先保存到某个本地的 credential file 中.

具体机制:
+ 一个特定的 long running service 拥有一个称之为 production-id 的特殊的用户; 一一对应
+ 任何 long running service 会用对应的 production-id 的身份启动
+ 一个 linux 定期任务帮助每个活跃的 production-id 定期刷新 TGT, 保存在本地 credential file 中
+ long running service 只需要从 credential file 读取 TGT

这个方法的特点还是集中管理吧, 否则解决这个问题的方法有很多;
通过 keytab file + 用户创建自己的周期任务自动调用 kinit 也可以实现自动更新 TGT, 
如: https://serverfault.com/questions/422778/how-to-automate-kinit-process-to-obtain-tgt-for-kerberos
不过这就是野路子了.

如果要做严格的权限管理, 势必有 production-id 的概念 (保证 service 不会有过多的权限), 
可能会有 production-id 对应的密码的定期更新, 可能每次登录都必须用专门系统生成的随机密码,
所以 pre-stash 的思路还是有可以参考之处.


## 其他
一个相关的概念是 stash file, 参见:
+ https://web.mit.edu/kerberos/krb5-1.12/doc/basic/stash_file_def.html
+ 相关的命令行 https://web.mit.edu/kerberos/krb5-1.5/krb5-1.5.4/doc/krb5-admin/Creating-a-Stash-File.html

由于 stash file 的存在, "把 kerberos TGT 放进某文件" 这个操作常常用动词 'stash' 来描述;
如前文所言, 由于 prestash 机制的存在, 这个操作也常常用 'prestash' 来描述.

