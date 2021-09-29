---
title: Jenkins部署React项目build失败
date: 2021-09-29 14:06:00
categories: 
- Jenkins
tags:
- Jenkins
- React
---



### React打包脚本

```
yarn install --pure-lockfile 
yarn run build
......
```

脚本在服务器上可以正常打包，但是会输出warning，不影响打包结果。
但是在Jenkins 进行build时却无法通过。

### 服务器打印输出

```
......

Compiled with warnings.

......
```

### Jenkins 控制台输出

```
......

Treating warnings as errors because process.env.CI = true. 
Most CI servers set it automatically.

......
```

### 解决

#### 方法一
Jenkins配置添加全局属性：系统管理 -> 系统配置 -> 全局属性 -> 添加环境变量

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/Jenkins%E9%85%8D%E7%BD%AE%E5%85%A8%E5%B1%80%E5%B1%9E%E6%80%A7.png)

#### 方法二

build前设置该参数值为false

```
yarn install --pure-lockfile 
export CI=false
yarn run build
......
```