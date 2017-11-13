---
title: Windows Credential Guard 简介
date: 2017-11-13 18:58:38
tags:
  - Windows
  - credential guard
  - security
---


## 概念
+ Credential Guard (CG): 微软提出的保护系统 credential 的机制
+ TCB (Trusted Computing Base): 一小部分软件, 系统安全的基石, 一旦有问题整个系统安全性完蛋; 
  其他的软件都可以 misbehave, TCB 不可以. 


## 解决安全问题: 攻击者通过读取 RAM 进行 credential theft
"Pass-the-Hash 攻击" 是十分典型的 credential theft.
+ 从 OS RAM 中读取 "hash 过的密码/hash 过的 identity credential" (如 kerberos 的 TGT), 
  并加以利用 (OS 保存 "hash 过的密码" 是为了实现 SSO (Single Sign-On) 功能)
+ 通过 "lateral traversal" (用已经窃取到的身份, 在 network 上不同的机器上登录, 重复同样的攻击) 
  拿到更多的 credential 和权限
+ 已经窃取的 credential 在 network 的部分机器上可能是 admin. 
+ 攻击者进一步利用已有的 user credential, 去读取 network 上所有能读取 RAM 的机器的 RAM. 
  拿到更多的 credential.


## CG (Credential Guard) 如何解决 credential theft
+ do not trust admin
+ do not trust kernel
  kernel 运行在 ring 0. 现在引入新的 hypervisor 机制,
  运行的级别比 ring 0 更低 (ring -1, 硬件保证); 
  hypervisor 允许 TCB 运行在 "ring -1", 从 kernel 隔离开.
+ 硬件级别的保护


## 新机制 VTL (Virtual Trust Levels) 
除了新的 hypervisor, 原本的 kernel space, user space 都可以认为跑在 VTL-0 上.

现在, kernel space 和 user space 都多出了一个 VTL-1 层, only selectively accessible to VTL-0.
也就是说: [新 kernel space] = [旧 kernel space (kernel space VTL-0)] + [kernel space VTL-1]
user space 同理.

VTL-1 被认为是 "secure world", VTL-1 kernel mode 也叫做 "secure kernel mode" (user mode 同理).
VTL-0 VTL-1 之间的 communication 通过 secure channel 进行. 
VTL 利用了新 CPU 的 SLAT (second level address translation) 技术 
(SLAT 原本用于 "虚拟机 physical/virtual page, 物理机 physical page 三者之间的 mapping")

trustlets: 在 VTL-1 的 user mode app 被成为 trustlets.


## 一些重要的事实
+ CG 带来的性能差别: typically less than 5%
+ CG 的价值是不可替代的, 但它本身可以被 disable; 
  故必须配合 win server 的 authen policy/armoring 使用.
+ Win10 有 CG 机制
+ CG 对于 kerberos 来说: CG 保护了 TGT; 
  这也导致了开启了 win10 的机器上, 部分跨平台的 kerberos 如 heimdal kerberos 的
  "读取保存 TGT" 的操作都失效. 在开启 CG 的机器上, 最好都改用 Windows 内置的 kerberos API


## reference
+ Defeating Pass-the-Hash-Separation-Of-Powers, win credential guards.pdf
  幻灯片形式, 良心教程

