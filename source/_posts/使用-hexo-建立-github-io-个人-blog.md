---
title: 使用 hexo 建立 github.io 个人 blog
date: 2017-08-14 16:28:01
tags: 
  - hexo
  - github.io 个人 blog
---


## 背景 & 准备
把个人 blog 放在 github.io 上已经是现在的主流了.
先根据 [这里](https://pages.github.com/) 的指南建立属于你的 `username.github.io` 主页.

接着, 为什么用 hexo?
+ 第一, 够用, 很强. 
+ 第二, 比起 "官方钦定" 的 jekyll, hexo 是纯基于 node.js 的工具, 不需要 ruby. 相信会成为未来的主流.

生活在墙内的朋友, 先配置好 [cnpm](https://npm.taobao.org). 
之后的命令中, 我都使用了 `cnpm`; 如果是墙外的朋友直接用 `npm` 即可. 


## hexo 安装; 创建 blog project
```sh
sudo cnpm install hexo-cli -g

## 创建 blog project
hexo init my-blog
cd my-blog
cnpm install
```


## 安装配置 next theme
(更新: 这个 theme 并不稳定, 已弃坑. 用回 hexo 默认的 theme 了)

执行:
```sh
git clone https://github.com/iissnan/hexo-theme-next themes/next
```

在 `_config.yml` 中配置:
```yml
theme: next
scheme: Mist   ## 参考 hexo-theme-next 的文档
```

### 为 next theme 创建 tags 页面
执行: 
```sh
cnpm install hexo-generator-tag --save
hexo new page "tags" 
```

在 `source/tags/index.md` 中填入:
```md
---
title: Tags
date: 2016-12-22 12:39:04
type: "tags"
---
```


## 配置 hexo deploy 的行为: 自动 deploy 到个人 github.io 主页
安装工具: `cnpm install hexo-deployer-git --save`

在 `_config.yml` 中配置:

```yml
deploy:
  type: git
  repo: https://github.com/zsh-89/zsh-89.github.io.git  
    ## 换成你的 github.io repo 地址
  branch: master
```


## hexo 基本命令
```sh
hexo new "My New Post"  ## [Writing](https://hexo.io/docs/writing.html)
hexo server             ## [Server](https://hexo.io/docs/server.html)
                        ## 开启 local http server. 查看效果
hexo generate           ## [Generating](https://hexo.io/docs/generating.html)
hexo deploy             ## [Deployment](https://hexo.io/docs/deployment.html)

## gen + deploy
hexo generate --deploy
hexo gen --deploy
hexo deploy --generate
```


## Next?
+ 推荐将 blog 的项目作为一个 git repo 管理, 参考: https://github.com/zsh-89/zsh-blog-src
+ 考虑用 [git submodule](http://zsh-89.github.io/2017/08/10/git-submodule-%E9%85%8D%E7%BD%AE%E6%9F%A5%E7%9C%8B-submodule-%E7%9A%84%E5%85%B7%E4%BD%93%E7%89%88%E6%9C%AC/) 管理 blog 项目和第三方 theme repo 之间的关系


