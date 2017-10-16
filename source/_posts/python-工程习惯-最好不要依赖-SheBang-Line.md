---
title: 'python 工程习惯: 最好不要依赖 SheBang Line  '
date: 2017-10-16 17:39:00
tags: 
  - python
  - best practice
---

真实的案例. 
一个在大部分机器工作得很好的 code, 首行是 `#!/usr/bin/python`.
在一台 server 上, 这个路径对应的 python 是有问题的, 于是跪了.

单抽出来说很简单, 但实际表现的症状比较莫名其妙, 夹杂在其他问题中, 就不容易找到了.
例如, 操作的人在运行 py 文件之前可能已经通过其他命令行工具临时修改了 `$PATH`, 
但是 hard code 的路径是无视 `$PATH` 的.

更好的写法是: `#!/usr/bin/env python`, 这样会调用 `$PATH` 中对应的 python;
这相当于给用户一个主动决定使用的 python 的版本的机会.

不过我觉得其实倒不如明确地注明 dependency 的版本, 不写 SheBang, 
让用户显式地用 python 可执行文件来执行 code, 这样出问题时调试代价要低得多;
其实大部分时候代码并没有那么 portable.
