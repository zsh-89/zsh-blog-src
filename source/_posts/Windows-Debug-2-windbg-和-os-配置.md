---
title: 'Windows Debug - 2: windbg 和 os 配置'
date: 2017-12-31 23:45:37
tags:
  - Windows Debugging
  - symbol file
  - windbg
---


## symbol file; windbg 配置 symbol
### symbol 简介
通常就是 `.pdb` 文件。
release 和 debug symbol 要配合对应的 exe 使用, 不可混用.
没有 symbol，就看不到函数名，只能看到 **地址**。

public symbol： 包含基本的玩意 (微软发布的就是 public symbol); 可能只有 public 函数等等;
private symbol：和我们自己用 VS build 出来的一样. 全. 可以用来干坏事.

symchk.exe (windbg 自带) 可以检测 symbol 文件正确性

### windbg 配置 symbol
可以把 symbol string 设置在两个位置之一, 告诉 windbg 'symbol 的位置':
+ 设定在 `_NT_SYMBOL_PATH` 环境变量: `srv*https://msdl.microsoft.com/download/symbols`
+ 通过 UI 设定; 每次设定的值在 windbg 关闭后, 都会被清空.

### symbol string 的语法
```
srv*c:\mss*http://msdl.microsoft.com/download/symbols;D:\Windbg_training\Setup\Projects\assembly\x64\Release

## 分号分割; (第二个目录是 app specific symbol)
## srv*c:\mss*http://msdl.microsoft.com/download/symbols; 设定 local cache 行为,
## cache 目录为 c:\mss  
```

### symbol server 和 symbol 发布
文件系统的共享目录可以作为 symbol server. 
visual studio 可以配置编译后自动发布 symbol 到 symbol server.


## windbg extension; 支持更多的 debug;
存在 for .net extension 等等; 有很多开源的 extension; 
参见: http://windbg.org




