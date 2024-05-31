---
layout: post
title: 搭建jekyll blog
date: '2024-05-31 13:54:26 +0800'
categories: [Demo]
tag: [jekyll]
---

#  搭建[Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)主题 [Jekyll](https://jekyllrb.com/).

## 基础环境

[Jekyll基础环境安装](https://jekyllrb.com/docs/installation/)

### 安装Ruby

- [Windows环境](https://jekyllrb.com/docs/installation/windows/)

  安装相当执行完毕后在ridk install弹窗中选择 [3] MSYS2 and MINGW development tool chain，安装Ruby开发工具链 

- [macOS](https://jekyllrb.com/docs/installation/macos/)  使用HomeBrew安装

### 安装jekyll

中国用户建议切换国内镜像源

`````shell
bundle config mirror.https://rubygems.org https://mirrors.tuna.tsinghua.edu.cn/rubygems/
`````

安装jekll

```shell
gem install jekyll
```

### Fork Chirpy并发布到Github

建议使用[Chirpy Starter的方式](https://chirpy.cotes.page/posts/getting-started/#:~:text=is%20not%20recommended.-,Option%201.%20Using%20the%20Chirpy%20Starter,-Sign%20in%20to)创建自己的网站，fork完毕之后，修改 `_config.yml` 文件中的`url` 为自己的github.io地址后，[设置Github Action](https://chirpy.cotes.page/posts/getting-started/#:~:text=Next%2C%20configure%20the%20Pages%20service.) 设置![image-20240531141815015](/assets/img/2024-05-31-搭建jekyll-blog/image-20240531141815015.png)
即可完成发布github io主页

### 在本地运行

#### 编译

将项目 `clone` 到本地之后，在项目根路径下编译，运行

```shell
bundle
```

如果需要查询运行细节可添加 `--verbose` 参数

```shell
bundle --verbose
```

长时间无响应可以尝试上文切换源

#### 启动

```shell
bundle exec jekyll s
```

