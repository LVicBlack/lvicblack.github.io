---
title: 解决Hexo在使用高版本NodeJs报错
date: 2021-09-13 11:06:00
categories: 
- hexo
tags:
- hexo
---

今天把博客迁移到了新域名，顺便把 [node.js](https://github.com/nodejs/node)、[Hexo](https://hexo.io/zh-cn/) 和主题都升了下级。

当习惯的运行 `hexo s` 命令时，发现多了些 warnings，如下：

```
$ hexo -s                              
(node:87224) Warning: Accessing non-existent property 'lineno' of module exports inside circular dependency
(Use `node --trace-warnings ...` to show where the warning was created)
(node:87224) Warning: Accessing non-existent property 'column' of module exports inside circular dependency
(node:87224) Warning: Accessing non-existent property 'filename' of module exports inside circular dependency
(node:87224) Warning: Accessing non-existent property 'lineno' of module exports inside circular dependency
(node:87224) Warning: Accessing non-existent property 'column' of module exports inside circular dependency
(node:87224) Warning: Accessing non-existent property 'filename' of module exports inside circular dependency
Copy
```

说实话我对 node.js 没啥了解，但是单词还是认识几个，看起来像是循环依赖的问题。（习惯性想起了一道面试题：Spring 是如何解决循环依赖的？）

这些 warnings 其实对 Hexo 程序运行没啥影响，只是看起来不舒服罢了。

但出于好奇和洁癖，就去 google 了一下。这里来总结一下原因及解决方案。



原因其实就是 [#29935](https://github.com/nodejs/node/pull/29935) 这个 pr 被合到 node.js 14.0.0 里边了，所以从 node.js 14 开始，这个问题就在网上不断被讨论了。

大家的解决办法也是五花八门，其中一个比较有代表性的是把 node 降级，降到 12 就不会报这个 warning 了

```
brew uninstall node
brew install node@12
brew link --overwrite --force node@12
Copy
```

但这样解决问题显然不是我的风格，继续翻 Github 上的 issues，发现**具体到 Hexo 这里的 warning**是由于 [stylus](https://github.com/stylus/stylus) 导致的，幸运的是 3 天前 stylus 在 0.54.8 版本修复了这个问题（见 pr [#2538](https://github.com/stylus/stylus/pull/2538) ）。

所以对于 Hexo 用户来说，重新装一下 `hexo-renderer-stylus` 就可以愉快的 `hexo s` 了

```
yarn remove hexo-renderer-stylus
yarn add hexo-renderer-stylus
Copy
```

至于其他的 package 导致的 warnings，可以使用如下方式来看看具体是哪个 package 导致的

```
npx cross-env NODE_OPTIONS="--trace-warnings" hexo s
Copy
```

UPDATE，接昨天说的：

> 刚写完准备睡觉，发现 `hexo s` 不报 warning 了，但是启动后又报了 😶

使用上边刚说的那个命令，发现这其实是 [nib@1.1.2](https://www.npmjs.com/package/nib) 这个包里的 stylus 引起的问题，nib 里的 dependencies 如下：

```
{
  "stylus": "0.54.5"
}
Copy
```

已经有人给 nib 提 issue 了，但看它最后一次更新已经是 4 years ago 了，估计是指望不上它更新了，那我们自己来解决吧！

在 package.json 里增加 `resolutions` 来覆盖版本定义

```
"resolutions": {
  "stylus": "^0.54.8"
}
Copy
```

然后重新 `yarn install` 一下就好了。

至此 hexo 就可以和 node.js 14 开始愉快的旅程了~

参考：

- https://www.haoyizebo.com/posts/710984d0/
- [module: warn on using unfinished circular dependency](https://github.com/nodejs/node/pull/29935)
- [Fix for Node v14 ‘Accessing non-existent property’ errors #2538](https://github.com/stylus/stylus/pull/2538)
- [NodeJS 14 warnings #2534](https://github.com/stylus/stylus/issues/2534)
- [Warning: Accessing non-existent property ‘lineno’ of module exports inside circular dependency #4257](https://github.com/hexojs/hexo/issues/4257)
- [选择性依赖项解决](https://classic.yarnpkg.com/zh-Hans/docs/selective-version-resolutions/)