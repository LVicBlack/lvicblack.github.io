---
title: hexo 搭建
date: 2021-08-31 23:31:21
categories: 
- hexo
tags:
- hexo
---



### 1.安装与初始化
```
npm install -g hexo-cli

hexo init myblog

cd myblog //进入这个myblog文件夹
npm install
```
新建完成后，指定文件夹目录下有：

- node_modules: 依赖包
- public：存放生成的页面
- scaffolds：生成文章的一些模板
- source：用来存放你的文章
- themes：主题
- _config.yml: 博客的配置文件
```
hexo g
hexo server
```

打开hexo的服务，在浏览器输入localhost:4000就可以看到你生成的博客了。

### 2.将hexo部署到GitHub的准备工作
- 配置Deployment
同样在_config.yml文件中，找到Deployment，然后按照如下修改：

```
deploy:
  type: git
  repo: git@github.com:yourname/yourname.github.io.git
  branch: master
```

- 安装deploy-git

这个时候需要先安装deploy-git ，也就是部署的命令,这样你才能用命令部署到GitHub。

```
npm install hexo-deployer-git --save
```

### 3.写博客、发布文章
新建一篇博客，执行下面的命令：

```
hexo new post "article title"
```

此时在source\ _posts 目录下将会看到 article title.md 文件。

用MarDown编辑器打开就可以编辑文章了。文章编辑好之后，运行生成、部署命令：

```
hexo g   // 生成
hexo d   // 部署
```
当然你也可以执行下面的命令，相当于上面两条命令的效果

```
hexo d -g #在部署前先生成
```

如果没更新，可以用hexo clean && hexo g来先清除之前的再生成。直接部署可以使用

```
//等于一次性执行了，清空、刷新、部署三个命令
hexo clean && hexo g && hexo d
```

部署成功后，就可以在http://yourname.github.io 这个网站看到博客了！！


### 4.绑定域名

虽然在Internet上可以访问我们的网站，但是网址是GitHub提供的:http://xxxx.github.io (知乎排版可能会出现"http://"字样) 而我们想使用我们自己的个性化域名，这就需要绑定我们自己的域名。这里演示的是在阿里云万网的域名绑定，在国内主流的域名代理厂商也就阿里云和腾讯云。登录到阿里云，进入管理控制台的域名列表，找到你的个性化域名，进入解析

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/image-20210831235715373.png)


然后添加解析

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/image-20210831235749184.png)



包括添加三条解析记录，192.30.252.153是GitHub的地址，你也可以ping你的 http://xxxx.github.io 的ip地址，填入进去。第三个记录类型是CNAME，CNAME的记录值是：你的用户名.http://github.io 这里千万别弄错了。第二步，登录GitHub，进入之前创建的仓库，点击settings，设置Custom domain，输入你的域名

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/image-20210901000235693.png)


点击save保存。第三步，进入本地博客文件夹 ，进入blog/source目录下，创建一个记事本文件，输入你的域名，对，只要写进你自己的域名即可。如果带有www，那么以后访问的时候必须带有www完整的域名才可以访问，但如果不带有www，以后访问的时候带不带www都可以访问。所以建议，不要带有www。这里我还是写了www(不建议带有www):

```
vagrantpoet.site
```

保存，命名为CNAME ，注意保存成所有文件而不是txt文件。

完成这三步，进入blog目录中，按住shift键右击打开命令行，依次输入：

```
hexo clean && hexo g && hexo d
```

这时候打开浏览器在地址栏输入你的个性化域名将会直接进入你自己搭建的网站。

### 5 搭建图床

```
https://blog.csdn.net/prefectjava/article/details/111192741
```

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/20210901102004.png)