---
title: "使用Macos中遇到的一些问题"
subtitle: "Macos issue"
date: 2017-12-21T17:29:07+08:00
tags: ["macos"]
type: "post"
categories: ["macos"]
description: "mac 上遇到的一些问题以及解决方法"

---
- [mac 上遇到的一些问题](#org27640a5)
  - [crontab 问题](#orgc8b5629)


<a id="org27640a5"></a>

# mac 上遇到的一些问题

1.  alfred 更新 alfred 后，提示不合法，可通过命令 `xattr -rc /Applications/Alfred\ 3.app`

解决


<a id="orgc8b5629"></a>

## crontab 问题

想在 mac 上添加一条 crontab 定时任务，用户自动更新 gfwlist, mac 默认没有/etc/crontab 文件，需要自己创建后，将以下命令粘贴到文件中，

```sh
30 10 * * * /Users/song/.ShadowsocksX/update_gfwlist.sh
```

这样 crontab 定时任务及完成，可以参考[Clarencep 的博客](https://www.clarencep.com/2015/12/16/how-to-enable-crontab-on-osx/)
