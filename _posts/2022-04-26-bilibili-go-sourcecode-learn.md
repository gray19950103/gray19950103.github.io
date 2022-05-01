---
layout: post
tag: "Golang"
date: 2022-04-26
title: "Bilibili GO 源码学习"
desp: "Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nulla massa lorem, tempor vitae libero vitae, fringilla interdum mauris. Integer volutpat aliquam orci, non eleifend diam tempor sed. Aenean eget nulla non diam maximus porttitor eu in eros. Etiam auctor odio tempus molestie hendrerit. Proin tristique tincidunt nibh, vitae viverra sem scelerisque eget. Nam porta orci ac quam blandit, nec pellentesque erat volutpat. Phasellus turpis erat, dictum non tristique vitae, vehicula eu nisl. Duis ut quam nulla. Proin in sapien ex. Nullam porta, orci in elementum cursus, quam velit viverra elit, eu tristique turpis ex et tellus. Duis molestie arcu venenatis, cursus nisi in, malesuada metus."
---

辗转找到bilibili的GO源码，最初泄漏地址 [github.com/openbilibili](https://github.com/openbilibili/go-common/) 已经因为违反DMCA被移除。

项目根目录下的app目录包含全部应用源码：

| 目录名       | 内容     |
|-----------|--------|
| admin     | 管理后台   |
| common    | ？？？    |
| infra     | 基础设施类  |
| interface | 接口     |
| job       | 消息队列消费 |
| service   | 功能模块   |
| tool      | 工具类    |

## service 功能模块
功能模块中包含多个分类：  
- bbq
- ep
- live
- main
- openplatform
- ops
- video

每个分类中又包含多个微服务模块。

### 微服务模块 main/account

| 目录名     | 内容             |
|---------|----------------|
| api     | gorpc proto    |
| cmd     | 启动入口类 toml配置文件 |
| conf    | 配置类            |
| dao     | 数据链接层          |
| model   | 数据模型           |
| rpc     | rpc            |
| server  | grpc http      |
| service | 业务实现层          |

#### cmd/main.go
解析命令行参数后并初始化配置文件后，
分别启动 `rpcSvr` `wardenSvr` `httpSvr`，
并进入无限循环获取 `os.Signal` 信道的信息，
获得指定信号后中止应用。

#### server/http/http.go
cmd/main.go中调用http.Init()后，
根据传入配置构建`blademaster`的engine、注册路由及方法，
最后启动engine。

## job 消息队列消费模块
与其他模块一致，同样包含多个分类。

### 微服务模块 main/account-recovery
目录结构与上述微服务模块大体一致。

#### service/service.go
http/http.go的初始化中调用service.New()后，
根据传入配置构建service，
同时以Goroutine的方式调用service中的全部Consumeproc方法，
方法中无限循环获取对应`Databus`中`Message`信道的信息，
并反序列化后调用相应方法。

{% highlight go %}
import (
	"context"
	"sync"

	v1 "go-common/app/service/main/account/api"
	"go-common/library/stat/prom"
	"go-common/library/sync/errgroup"
)

var _ _cache

// Info get data from cache if miss will call source method, then add to cache.
func (d *Dao) Info(c context.Context, id int64) (res *v1.Info, err error) {
	addCache := true
	res, err = d.CacheInfo(c, id)
	if err != nil {
		addCache = false
		err = nil
	}
	if res != nil {
		prom.CacheHit.Incr("Info") //缓存击中统计
		return
	}
	prom.CacheMiss.Incr("Info")
	res, err = d.RawInfo(c, id)
	if err != nil {
		return
	}
	miss := res
	if !addCache {
		return
	}
	d.cache.Do(c, func(ctx context.Context) {
		d.AddCacheInfo(ctx, id, miss)
	})
	return
}
{% endhighlight %}