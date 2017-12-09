---
title: 使用 hexo 建立 github.io 个人 blog
date: 2017-08-14 16:28:01
tags: 
  - hexo
  - github.io blog
---


## 背景 & 准备
把个人 blog 放在 github.io 上已经是现在的主流了.
先根据 [这里](https://pages.github.com/) 的指南建立属于你的 `username.github.io` 主页.

接着, 为什么用 hexo?
+ 第一, 够用, 很强. 
+ 第二, 比起 "官方钦定" 的 jekyll, hexo 是纯基于 node.js 的工具, 不需要 ruby. 相信会成为未来的主流.

生活在墙内的朋友, 先配置好 [cnpm](https://npm.taobao.org). 
之后的命令中, 我都使用了 `cnpm`; 如果是墙外的朋友直接用 `npm` 即可. 


## Reference
+ hexo 官网: https://hexo.io
+ Next 主题官方文档: http://theme-next.iissnan.com


## hexo 安装; 创建 blog project
```sh
sudo cnpm install hexo-cli -g

## 创建 blog project
hexo init my-blog
cd my-blog
cnpm install
```


## 安装配置 next theme
执行:
```sh
git clone https://github.com/iissnan/hexo-theme-next themes/next
```

在 `_config.yml` 中配置:
```yml
theme: next
```

在 `themes/next/_config.yml` 中配置:
```yml
## 选取 next 主题中的 Gemini 风格; 此步可以跳过
scheme: Gemini   ## 在编辑器搜索相关字眼, 并且把原来的配置注释掉
```

注意, 不要混淆 next 主题的配置文件 (`themes/next/_config.yml`) 和 hexo 的配置文件 (直接就在 my-blog 目录下 的`_config.yml`).
更多关于 next 主题参考 [Next 主题官方文档]( http://theme-next.iissnan.com ).

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

配置 blog 的页面, 在  `themes/next/_config.yml` menu 配置项就增加 `tags` 页:
```yml
menu:
  home: /
  archives: /archives
  tags: /tags
  ## about: /about  
    ## hexo 也支持 about 页面; 创建方式和创建 tags 页面相同; 
    ## 不过 about 页面对应的 index.md 里面可以写正文
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


## 结语
+ 推荐将 blog 的项目作为一个 git repo 管理, 参考: https://github.com/zsh-89/zsh-blog-src
+ 由于需要修改 theme/next 下的内容才能对 next 主题进行配置, 不推荐将 next theme 作为 git submodule 进行管理


