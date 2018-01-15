---
title: "Golang tools"
subtitle: "Golang tools"
date: 2018-01-15T18:54:06+08:00
tags: ["golang","tools"]
type: "post"
categories: ["golang","tools"]
description: "tools of golang usage"
---

- [glide](#orgfb9872c)


<a id="orgfb9872c"></a>

# glide

使用 glide 进行包管理时，在国内会遇到这样的问题：

```sh
[WARN] Unable to checkout golang.org/x/crypto
[ERROR] Update failed for golang.org/x/crypto: Cannot detect VCS
[WARN] Unable to checkout golang.org/x/net
[ERROR] Update failed for golang.org/x/net: Cannot detect VCS
[ERROR] Failed to install: Cannot detect VCS
Cannot detect VCS
```

可以通过 `glide mirror set` 命令设置镜像解决，如

```sh
glide mirror set https://golang.org/x/sys https://github.com/golang/sys --vcs git
glide mirror set https://golang.org/x/image https://github.com/golang/image --vcs git
glide mirror set https://golang.org/x/text https://github.com/golang/text --vcs git
glide mirror set https://golang.org/x/tools https://github.com/golang/tools --vcs git
glide mirror set https://golang.org/x/net https://github.com/golang/net --vcs git
glide mirror set https://golang.org/x/crypto https://github.com/golang/crypto --vcs git
glide mirror set https://google.golang.org/grpc https://github.com/grpc/grpc-go --vcs git
glide mirror set https://google.golang.org/genproto https://github.com/google/go-genproto --vcs git

```
