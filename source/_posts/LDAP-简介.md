---
title: LDAP 简介
date: 2017-12-31 23:42:04
tags:
  - LDAP
  - Lightweight Directory Access Protocal
---


## ref
+ https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol


## 基本
LDAP 是 Lightweight Directory Access Protocal 的缩写. LDAP 是 "应用层" (Application Level) 的网络协议; 

LDAP用于保存 "Directory Information", 或者说可以对外提供 "Directory services";

LDAP server 通常使用的端口号是:
+ TCP 389, UDP 389; 传输 LDAP traffic
+ TCP 636; 传输 LDAP SSL traffic (LDAP SSL 又被成为 LDAPS, LDAP over SSL)
+ TCP 3268; 传输 LDAP GC (Global Catalog)
+ TCP 3269; 传输 LDAPS GC


## 常见用途
可以认为 LDAP 是 web service 级别的一个 dictionary 数据结构 (正如 python 等高级语言中 dictionary);
LDAP 也可以说是一个轻量级的, 为特定用途优化的数据库; 方便好用, 自带 web service 功能.

常见用途
+ 保存 firmwide web application config 
+ 保存 firmwide system config
+ 保存公司的人员信息


## LDAP 的数据结构; LDAP entry; LDIF (LDAP Data Interchange Format)
什么是 LDAP 中的 Directory 数据结构:
+ Directory 是 entry 的集合
+ An **entry** consists of a set of **attributes**.
+ An attribute has a name (an attribute type or attribute description) and one or more values. 
  The attributes are defined in a schema (see below).
+ Each entry has a unique identifier: its **Distinguished Name (DN)**. 
  This consists of its **Relative Distinguished Name (RDN)**, constructed from 
  some attribute(s) in the entry, followed by the parent entry's DN. 
  Think of the **DN as the full file path** and the **RDN as its relative filename** 
  in its parent folder (e.g. if `/foo/bar/myfile` were the DN, then `myfile` would be the RDN).
  entry 和文件系统中的 '文件' 类似, 对应于文件 '绝对路径' 概念的是 'DN', 对应于 '相对路径' 概念的是 'RDN'.

LDAP 本身是 binary protocal, 传输的信息对于机器友好 (相反, HTTP 则是 plain text protocol);
不过 LDAP 传输的信息可以用 LDIF (LDAP Data Interchange Format) 用人类易读的方式表示.

以下是一个 LDAP entry 的例子 (用 LDIF 的格式写出):
```ldif
dn: cn=John Doe,dc=bigsoft,dc=com
cn: John Doe
givenName: John
sn: Doe
telephoneNumber: +1 888 555 6789
telephoneNumber: +1 888 555 1232
mail: john@bigsoft.com
manager: cn=Barbara Doe,dc=bigsoft,dc=com
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
```
结合例子解读 "directory" 数据结构:
+ 每个 entry 一定包含 dn; dn 就是 DN (Distinguished Name); dn 不算是 attribute (虽然很像);
+ 例子中表示 dn 的是第一行 `dn: cn=John Doe,dc=bigsoft,dc=com`;
  这个字符串中, `cn=John Doe` 给出了 RDN (Relative Distinguished Name) (不过 cn 也是 "Common Name" 的缩写),
  `dc=bigsoft,dc=com` 给出了 parent 的 dn; 字符串中 dc 的含义是 "Domain Component" 
+ 可以知道, 这个 entry 已经是 `bigsoft.com` 这个 domain 下顶层的 entry 了;
  (这个 entry 的 parent 的 dn 字符串中没有 cn, 所以它的 parent 不是一个 entry 而是 domain).
  一个非顶层的 entry 的 dn 的例子是: `dn: cn=MyAppConfig,cn=Config,dc=bigsoft,dc=com`
+ 部分 entry 和 entry 之间存在 hierarchy 关系, 即一个 entry 可能是另一个 entry 的 parent.
+ LDAP entry 的 DN 在 entry 的生命周期中 **可能会改变**; 但是一定是唯一的;
  部分 LDAP entry 的 optional attribute 中保存了 UUID; UUID 是不变的, 也可以唯一确定地一个 entry.
+ 一个 entry 包含若干个 attribute; 
+ attribute 是 name-values pair, 如例子中这个 attribute `telephoneNumber: +1 888 555 6789`, 
  它的 attribute name 是 `telephoneNumber`, attribute values 是 `+1 888 555 6789`;

更多关于 "entry 的 dn" 的例子: 
+ `dn: ou=myGroups,dc=bigsoft,dc=com`, `ou` 的含义是 **Organizational Unit**;
+ `dn: cn=John Smith,ou=myGroups,dc=bigsoft,dc=com`
+ `dn: cn=Alen Smith,ou=mySubGroup,ou=myGroups,dc=bigsoft,dc=com`; ou 在路径中也可以多次出现


## client 端可能执行的操作
+ StartTLS — use the LDAPv3 Transport Layer Security (TLS) extension for a secure connection
+ Bind — authenticate and specify LDAP protocol version
+ Search — search for and/or retrieve directory entries
+ Compare — test if a named entry contains a given attribute value
+ Add a new entry
+ Delete an entry
+ Modify an entry
+ Modify Distinguished Name (DN) — move or rename an entry
+ Abandon — abort a previous request
+ Extended Operation — generic operation used to define other operations
+ Unbind — close the connection (not the inverse of Bind)

