---
title:  Prometheus服务发现源码阅读
date: 2019-10-05 20:51:38
tags: Prometheus
---


<!-- ---
layout: post
title: Prometheus服务发现源码阅读
author: Ray Chan(ray1888)
date: '2019-10-05 20:51:38 +0800'
category: Prometheus
summary: Prometheus Service Discovery Module
thumbnail: Prometheus.png
--- -->

# 目录
1. 使用的目的  
2. 代码实现  
   2.1 主协程逻辑  
   2.2 子协程逻辑

# 代码版本

基于 prometheus项目的master branch的5a554df0855bf707a8c333c0dd830067d03422cf commit

# 使用目的
服务发现是Prometheus中最重要的功能之一，因为它是支撑Prometheus可以在容器的环境下的最重要的功能。因为应用的容器部署的弹性和有效的时长远与传统的基于服务器（无论是实体机还是虚拟机[OpenStack]这一类的机器）部署， 都会变动的更加快。因此为了适应这种弹性大、变化快的环境，它需要基于不同平台来支持服务发现这个功能。

Prometheus的服务发现模块是在prometheus/discovery的目录下面，它在Prometheus中的体系支撑了采集器的发现和AlertManager的发现。
可以看下面的代码。

```
//  此代码在cmd/main.go中的 350-352行
	ctxScrape, cancelScrape = context.WithCancel(context.Background())
	discoveryManagerScrape  = discovery.NewManager(ctxScrape, log.With(logger, "component", "discovery manager scrape"), discovery.Name("scrape"))

	ctxNotify, cancelNotify = context.WithCancel(context.Background())
	discoveryManagerNotify  = discovery.NewManager(ctxNotify, log.With(logger, "component", "discovery manager notify"), discovery.Name("notify"))

//  具体注入到两个功能的使用
/* 
	cmd/main.go line 555, 把Manager的输出的信道传递给scrapeManager，然后后面就会去restore scrape的方式。
	后面会有单独文章写Scrape（抓取数据）的部分
	*/
err := scrapeManager.Run(discoveryManagerScrape.SyncCh())

/* 
	cmd/main.go line 726, 把Manager的输出的信道传递给notifierManager，然后后面就会去获取更新alertManager组件的方式。
	后面会有单独文章写Notifer（抓取数据）的部分
	*/
notifierManager.Run(discoveryManagerNotify.SyncCh())

// 在main.go line 735, run Group 会执行上面注册在g 里面的这两个discoveryManager的Run方法。
```

所以整体的代码执行的入口流程
1. 在main.go实例化两个discoveryManager来进行服务发现的操作
2. 把注入的discovery config进行解析，并且生成对应的Provider协程
3. 主协程执行上面两个discoveryManager的Run方法

# 代码实现

## 模块的运行逻辑图

<!-- ![ServiceDiscoveryModule](/assets/img/posts/prometheus/DiscoverManager.png) -->
{% asset_img DiscoverManager.png ServiceDiscoveryModule %}

## 主协程的逻辑
1. 先去读取服务发现相关的配置，调用ApplyConfig()读取存在的service discovery的方式，并且加载到NewManager的结构体中。可以同同时支持多个service Discovery的方式。使用add函数把配置变为Providers结构体。
```
add := func(cfg interface{}, newDiscoverer func() (Discoverer, error)) {
		t := reflect.TypeOf(cfg).String()
		for _, p := range m.providers {
			if reflect.DeepEqual(cfg, p.config) {
				p.subs = append(p.subs, setName)
				added = true
				return
			}
		}

		d, err := newDiscoverer()
		if err != nil {
			level.Error(m.logger).Log("msg", "Cannot create service discovery", "err", err, "type", t)
			failedCount++
			return
		}

		provider := provider{
			name:   fmt.Sprintf("%s/%d", t, len(m.providers)),
			d:      d,
			config: cfg,
			subs:   []string{setName},
		}
		m.providers = append(m.providers, &provider)
		added = true
	}
// 支持多种方式的配置 具体可以看代码的discovery/manager.go 356-417行
```

2. 然后为每个Provider生成两个协程去进行服务发现的功能。 
```
	// discovery/manager.go line 223-226
	func (m *Manager) startProvider(ctx context.Context, p *provider) {
		level.Debug(m.logger).Log("msg", "Starting provider", "provider", p.name, "subs", fmt.Sprintf("%v", p.subs))
		ctx, cancel := context.WithCancel(ctx)
		// 此处Update是执行协程与主协程的交流通道，传输Watch的变动
		updates := make(chan []*targetgroup.Group)
		m.discoverCancel = append(m.discoverCancel, cancel)

		//  执行服务发现的Watch
		go p.d.Run(ctx, updates)
		// 把每个协程生成的updates放入到Manager中，使得子协程中发现到服务的变动的时候可以通知到主协程
		go m.updater(ctx, p, updates)
	}

	// discovery/manager.go line 205-207
	for _, prov := range m.providers {
			m.startProvider(m.ctx, prov)
	}
```

ApplyConfig完成后，Run方法如下
```
	func (m *Manager) Run() error {
		go m.sender()
		for range m.ctx.Done() {
			m.cancelDiscoverers()
			return m.ctx.Err()
		}
		return nil
	}
```

Manager会派生多一个协程去定时检查配置的变动，本质上是检查上面派生的updater协程时候有传入信号到triggerchan中,主协程就是为了防止泄露一直去遍历context的是否done，保持主协程阻塞，维持这个模块的运行。我们主要看一下Manager.Updater()和Manager.Sender()这两个函数。

此处 Manager.trigglechan is 长度为1的buffered channel。updates := make(chan []*targetgroup.Group), 是unbuffered channel 
```
	func (m *Manager) updater(ctx context.Context, p *provider, updates chan []*targetgroup.Group) {
		for {
			select {
			case <-ctx.Done():
				return
			case tgs, ok := <-updates:
				receivedUpdates.WithLabelValues(m.name).Inc()
				if !ok {
					level.Debug(m.logger).Log("msg", "discoverer channel closed", "provider", p.name)
					return
				}

				for _, s := range p.subs {
					m.updateGroup(poolKey{setName: s, provider: p.name}, tgs)
				}

				select {
				case m.triggerSend <- struct{}{}:
				default:
				}
			}
		}
	}

	func (m *Manager) sender() {
		ticker := time.NewTicker(m.updatert)
		defer ticker.Stop()

		for {
			select {
			case <-m.ctx.Done():
				return
			case <-ticker.C: // Some discoverers send updates too often so we throttle these with the ticker.
				select {
				case <-m.triggerSend:
					sentUpdates.WithLabelValues(m.name).Inc()
					select {
					case m.syncCh <- m.allGroups():
					default:
						delayedUpdates.WithLabelValues(m.name).Inc()
						level.Debug(m.logger).Log("msg", "discovery receiver's channel was full so will retry the next cycle")
						select {
						case m.triggerSend <- struct{}{}:
						default:
						}
					}
				default:
				}
			}
		}
	}
```

步骤
1. Provider 传入变化了的update到update channel中
2. Manager.Updater协程收集到了变化的内容后，修改Manager里面Group的内容。塞入信号到triggerSend channel中
3. Manager.Sender协程定时去进行获取通知，检查到triggerSend Channel中可以获取之后，就会尝试塞到syncCh中，使得订阅者可以收到这个消息。(即Main里面的两个放入syncCh方法的ScrapeManager和NotiferManager可以收到)

然后整个的流程就是这样

## 派生协程的逻辑  
### Provider协程的逻辑  
```
type Discoverer interface {
	// Run hands a channel to the discovery provider (Consul, DNS etc) through which it can send
	// updated target groups.
	// Must returns if the context gets canceled. It should not close the update
	// channel on returning.
	Run(ctx context.Context, up chan<- []*targetgroup.Group)
}
```
Provider协程主要时根据具体注入的配置来生成出与对应的系统进行服务发现的能力。每个Provider中的Discoverer都实现了自己对应的Run方法即上面p.d.Run()的方法。（如何生成上面已经描述，此处不再复述）

下面我们简单的看一下其中两个Provider，基于配置文件的Provider和基于zookeeper的Provider来看看具体的流程是怎样处理的。
对于Provider，我们可以理解为一个watch&notify的模型，但是是基于不同平台给予的Api继续watch&notify的操作。

#### FileProvider执行

FileProvider支持解析json和yaml的格式内容  
```
// discovery/file/file.go line 39-46
var (
	patFileSDName = regexp.MustCompile(`^[^*]*(\*[^/]*)?\.(json|yml|yaml|JSON|YML|YAML)$`)

	// DefaultSDConfig is the default file SD configuration.
	DefaultSDConfig = SDConfig{
		RefreshInterval: model.Duration(5 * time.Minute),
	}
)
```

下面我们看看主要实现Discoverer的Run方法的逻辑
```
func (d *Discovery) Run(ctx context.Context, ch chan<- []*targetgroup.Group) {
	watcher, err := fsnotify.NewWatcher()
	if err != nil {
		level.Error(d.logger).Log("msg", "Error adding file watcher", "err", err)
		return
	}
	d.watcher = watcher
	defer d.stop()

    //  协程内第一次执行把conf添加到discoverer中
	d.refresh(ctx, ch)

	ticker := time.NewTicker(d.interval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return

		case event := <-d.watcher.Events:
			// fsnotify sometimes sends a bunch of events without name or operation.
			// It's unclear what they are and why they are sent - filter them out.
			if len(event.Name) == 0 {
				break
			}
			// Everything but a chmod requires rereading.
			if event.Op^fsnotify.Chmod == 0 {
				break
			}
			// Changes to a file can spawn various sequences of events with
			// different combinations of operations. For all practical purposes
			// this is inaccurate.
			// The most reliable solution is to reload everything if anything happens.
			d.refresh(ctx, ch)

		case <-ticker.C:
			// Setting a new watch after an update might fail. Make sure we don't lose
			// those files forever.
			d.refresh(ctx, ch)

		case err := <-d.watcher.Errors:
			if err != nil {
				level.Error(d.logger).Log("msg", "Error watching file", "err", err)
			}
		}
	}
}
```

上面的死循环可以看出，对于File Discoverer， 它支持两种的Watch的方式，一个是定时监控(ticker.C)，一个是事件触发监控（event := <-d.watcher.Events)。

所以重点是在于Refresh函数，本质上就是一个重新解析并且检查文件内容是否有变动的函数，解析完文件之后就可以传出配置到对应的update channel中。
```
func (d *Discovery) refresh(ctx context.Context, ch chan<- []*targetgroup.Group) {
	t0 := time.Now()
	defer func() {
		fileSDScanDuration.Observe(time.Since(t0).Seconds())
	}()
	ref := map[string]int{}
	// 把初始化传入的配置的路径进行检查
	for _, p := range d.listFiles() {
		//  把单个文件进行解析，获取出里面的targets
		tgroups, err := d.readFile(p)
		if err != nil {
			fileSDReadErrorsCount.Inc()

			level.Error(d.logger).Log("msg", "Error reading file", "path", p, "err", err)
			// Prevent deletion down below.
			ref[p] = d.lastRefresh[p]
			continue
		}
		select {
			// 把新传入的传入到Update中
		case ch <- tgroups:
		case <-ctx.Done():
			return
		}

		ref[p] = len(tgroups)
	}
	// Send empty updates for sources that disappeared.
	for f, n := range d.lastRefresh {
		m, ok := ref[f]
		if !ok || n > m {
			level.Debug(d.logger).Log("msg", "file_sd refresh found file that should be removed", "file", f)
			d.deleteTimestamp(f)
			for i := m; i < n; i++ {
				select {
				case ch <- []*targetgroup.Group{{Source: fileSource(f, i)}}:
				case <-ctx.Done():
					return
				}
			}
		}
	}
	d.lastRefresh = ref

	d.watchFiles()
}
```

#### zookeeperProvider执行

Zookeeper的Provider与上面的模式类似，差别在于：
1. 初始化的时候需要创建zookeeper的链接（配置中需要传入配置好zookeeper相关的配置）
2. 分为了ServerSetPoint和NerveEndPoint两个类型的Discovery，因此抽象了一个Discovery的结构体
3. 没有了定时检查（因为不存在文件系统问题的）这种检查的方式

```
	type Discovery struct {
		conn *zk.Conn

		sources map[string]*targetgroup.Group

		updates     chan treecache.ZookeeperTreeCacheEvent
		pathUpdates []chan treecache.ZookeeperTreeCacheEvent
		treeCaches  []*treecache.ZookeeperTreeCache

		parse  func(data []byte, path string) (model.LabelSet, error)
		logger log.Logger
	}

	func (d *Discovery) Run(ctx context.Context, ch chan<- []*targetgroup.Group) {
		defer func() {
			for _, tc := range d.treeCaches {
				tc.Stop()
			}
			for _, pathUpdate := range d.pathUpdates {
				// Drain event channel in case the treecache leaks goroutines otherwise.
				for range pathUpdate {
				}
			}
			d.conn.Close()
		}()

		for _, pathUpdate := range d.pathUpdates {
			go func(update chan treecache.ZookeeperTreeCacheEvent) {
				for event := range update {
					select {
					case d.updates <- event:
					case <-ctx.Done():
						return
					}
				}
			}(pathUpdate)
		}

		for {
			select {
			case <-ctx.Done():
				return
			case event := <-d.updates:
				tg := &targetgroup.Group{
					Source: event.Path,
				}
				if event.Data != nil {
					labelSet, err := d.parse(*event.Data, event.Path)
					if err == nil {
						tg.Targets = []model.LabelSet{labelSet}
						d.sources[event.Path] = tg
					} else {
						delete(d.sources, event.Path)
					}
				} else {
					delete(d.sources, event.Path)
				}
				select {
				case <-ctx.Done():
					return
				case ch <- []*targetgroup.Group{tg}:
				}
			}
		}
	}
```

Run的流程
1. 给每一个需要检查的路径派生一个协程
2. 死循环获取是否有更新
3. 如果有，则使用传入的parse函数去进行解析，然后把结果发送到update的channel中，即ch变量中