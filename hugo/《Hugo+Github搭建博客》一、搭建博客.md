---

---
《Hugo+Github搭建博客》一、搭建博客
> 本文介绍如何使用Hugo搭建一个静态博客，并将本地静态博客部署到Github中实现在线博客

## 安装Hugo
直接到hugo的github页去下载即可
<a href="https://github.com/gohugoio/hugo/releases">Hugo下载地址</a>

我下载的是extended版
有些主题必须要使用extended版才可以

下载完毕后直接解压，在环境变量中配置一下hugo文件夹即可

打开CMD，输入
```bash
hugo version
```
查看是否安装成功

## 注册Github
不多说了，github注册教程自己查找...
## 开始搭建
进入想存放博客文件的目录下
输入
```bash
hugo new site [blogname]
```

进入[blogname]目录中

文件结构：
```bash
blogname
├── archetypes
│   └── default.md
├── config.toml
├── content
├── data
├── layouts
├── static
└── themes
```
### 下载主题
在这里要做一个模块化的处理
因为hugoblog是一个项目，下载的主题也是一个项目
所以要将主题设为子模块

我选的是LoveIt，比较简洁，功能也比较全。
后面修改很方便
```bash
cd blogname
git init
git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt
```

### 启动


