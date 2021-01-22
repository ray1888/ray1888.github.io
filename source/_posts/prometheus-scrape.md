---
title:  Prometheus(1)- 数据抓取源码阅读
date: 2019-10-06 15:31:38
tags: Prometheus
---

<!-- ---
layout: post
title: Prometheus数据抓取源码阅读
author: Ray Chan(ray1888)
date: '2019-10-06 15:31:38 +0800'
category: Prometheus
summary: Prometheus Scrape Module
thumbnail: Prometheus.png
--- -->


# 目录
1. 使用的目的
2. 代码实现

# 代码版本

基于 prometheus项目的master branch的5a554df0855bf707a8c333c0dd830067d03422cf commit

# 使用目的

Prometheus 是一个基于Pull模型所进行数据采集的系统，因此，需要在主体项目中有一个抓取数据的模块，而Scrape就是这样的模块。因此这个也是Prometheus的一个主要部分。

```
入口代码部分
main.go
// line 356
scrapeManager = scrape.NewManager(log.With(logger, "component", "scrape manager"), fanoutStorage)
// line 427
scrapeManager.ApplyConfig
// line 555
err := scrapeManager.Run(discoveryManagerScrape.SyncCh())
```

# 代码实现 

## 整体的流程图

<!-- ![ServiceDiscoveryModule](/assets/img/posts/prometheus/ScrapeModule.png) -->
{% asset_img ScrapeModule.png ServiceDiscoveryModule %}

## 目录结构
```
-rw-r--r-- 1 ray 197121  2239 9月  30 16:09 helpers_test.go
-rw-r--r-- 1 ray 197121  7543 9月  30 16:09 manager.go   //主要的控制的模块Manager
-rw-r--r-- 1 ray 197121 10727 9月  30 16:09 manager_test.go
-rw-r--r-- 1 ray 197121 35749 9月  30 16:09 scrape.go    // 主要进行采集的模块
-rw-r--r-- 1 ray 197121 40682 9月  30 16:09 scrape_test.go
-rw-r--r-- 1 ray 197121 11743 9月  30 16:09 target.go    //  抓取的公用部分的逻辑
-rw-r--r-- 1 ray 197121  9542 9月  30 16:09 target_test.go

```

## 重要数据结构描述
ScrapeManager 是管理所有抓取的一个抽象
```
type Manager struct {
	logger    log.Logger
	append    Appendable
	graceShut chan struct{}

	jitterSeed    uint64     // Global jitterSeed seed is used to spread scrape workload across HA setup.
	mtxScrape     sync.Mutex // Guards the fields below.
	scrapeConfigs map[string]*config.ScrapeConfig
	scrapePools   map[string]*scrapePool
	targetSets    map[string][]*targetgroup.Group

	triggerReload chan struct{}
}
```

ScrapePools 是单个的Job的抓取目标的工作单位
```
appendable Appendable
	logger     log.Logger

	mtx    sync.RWMutex
	config *config.ScrapeConfig
	client *http.Client
	// Targets and loops must always be synchronized to have the same
	// set of hashes.
	activeTargets  map[uint64]*Target
	droppedTargets []*Target
	loops          map[uint64]loop
	cancel         context.CancelFunc

	// Constructor for new scrape loops. This is settable for testing convenience.
	newLoop func(scrapeLoopOptions) loop
```

loop是单个Target的执行单位，是一个接口。在这里主要使用的是ScrapeLoop的实例
```
type loop interface {
	run(interval, timeout time.Duration, errc chan<- error)
	stop()
}

type scrapeLoop struct {
	scraper         scraper
	l               log.Logger
	cache           *scrapeCache
	lastScrapeSize  int
	buffers         *pool.Pool
	jitterSeed      uint64
	honorTimestamps bool

	appender            func() storage.Appender
	sampleMutator       labelsMutator
	reportSampleMutator labelsMutator

	parentCtx context.Context
	ctx       context.Context
	cancel    func()
	stopped   chan struct{}
}

```

scraper接口时具体的执行单位，scrapeLoop也是调用scraper的方法来进行数据的抓取, Prometheus默认使用targetScraper去抓取数据

```
type scraper interface {
	scrape(ctx context.Context, w io.Writer) (string, error) // 抓取数据的方法
	report(start time.Time, dur time.Duration, err error)    // 上报数据的方法
	offset(interval time.Duration, jitterSeed uint64) time.Duration // 记录数据偏移的方法
}

type targetScraper struct {
	*Target  //包含了report和offset方法

	client  *http.Client //因为Prometheus的exporter是以http接口进行数据的暴露的，所以会有httpclient的结构包含在里面
	req     *http.Request
	timeout time.Duration

	gzipr *gzip.Reader
	buf   *bufio.Reader
}


```

## 主协程逻辑

跟着调用的部分，我们先从初始化后的manager的ApplyConfig方法开始看起。
```
func (m *Manager) ApplyConfig(cfg *config.Config) error {
	m.mtxScrape.Lock()
	defer m.mtxScrape.Unlock()

	c := make(map[string]*config.ScrapeConfig)
    
	for _, scfg := range cfg.ScrapeConfigs {
		c[scfg.JobName] = scfg
	}
	m.scrapeConfigs = c
    // 使用全局配置来生成一个集群内不重复的seed
	if err := m.setJitterSeed(cfg.GlobalConfig.ExternalLabels); err != nil {
		return err
	}

	// Cleanup and reload pool if the configuration has changed.
	var failed bool
    // 根据解析出来的配置生成对应的ScrapePool， 如果有并且数据没有改变的话，那就不进行操作，否则
	for name, sp := range m.scrapePools {
		if cfg, ok := m.scrapeConfigs[name]; !ok {
			sp.stop()
			delete(m.scrapePools, name)
		} else if !reflect.DeepEqual(sp.config, cfg) {
			err := sp.reload(cfg)
			if err != nil {
				level.Error(m.logger).Log("msg", "error reloading scrape pool", "err", err, "scrape_pool", name)
				failed = true
			}
		}
	}

	if failed {
		return errors.New("failed to apply the new configuration")
	}
	return nil
}

func (sp *scrapePool) reload(cfg *config.ScrapeConfig) error {
	targetScrapePoolReloads.Inc()
	start := time.Now()

	sp.mtx.Lock()
	defer sp.mtx.Unlock()

	client, err := config_util.NewClientFromConfig(cfg.HTTPClientConfig, cfg.JobName, false)
	if err != nil {
		targetScrapePoolReloadsFailed.Inc()
		return errors.Wrap(err, "error creating HTTP client")
	}
	sp.config = cfg
	oldClient := sp.client
	sp.client = client

	var (
		wg              sync.WaitGroup
		interval        = time.Duration(sp.config.ScrapeInterval)
		timeout         = time.Duration(sp.config.ScrapeTimeout)
		limit           = int(sp.config.SampleLimit)
		honorLabels     = sp.config.HonorLabels
		honorTimestamps = sp.config.HonorTimestamps
		mrc             = sp.config.MetricRelabelConfigs
	)

	for fp, oldLoop := range sp.loops {
		var (
			t       = sp.activeTargets[fp]
			s       = &targetScraper{Target: t, client: sp.client, timeout: timeout}
			newLoop = sp.newLoop(scrapeLoopOptions{
				target:          t,
				scraper:         s,
				limit:           limit,
				honorLabels:     honorLabels,
				honorTimestamps: honorTimestamps,
				mrc:             mrc,
			})
		)
		wg.Add(1)

		go func(oldLoop, newLoop loop) {
			oldLoop.stop()
			wg.Done()

			go newLoop.run(interval, timeout, nil)
		}(oldLoop, newLoop)

		sp.loops[fp] = newLoop
	}

	wg.Wait()
	oldClient.CloseIdleConnections()
	targetReloadIntervalLength.WithLabelValues(interval.String()).Observe(
		time.Since(start).Seconds(),
	)
	return nil
}

```

1. 把解析好的配置，遍历，变为一个jobName为key，配置值为Value的map
2. 对比自己的配置，如果之前已经存在但是配置发生变动的，则去reload scraper pool的配置。
3. （Reload） 如果需要reload配置的情况下，会重新生成scrapePool后，派生多一个线程去执行scraperPool.Sync()，知道Manager的targetSet被遍历完为止。Sync方法的内容会后面进行详细讲解。

Run()方法
```
func (m *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) error {
	go m.reloader()
	for {
		select {
		case ts := <-tsets:
			m.updateTsets(ts)

			select {
			case m.triggerReload <- struct{}{}:
			default:
			}

		case <-m.graceShut:
			return nil
		}
	}
}


func (m *Manager) reloader() {
	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-m.graceShut:
			return
		case <-ticker.C:
			select {
			case <-m.triggerReload:
				m.reload()
			case <-m.graceShut:
				return
			}
		}
	}
}
```
功能：
1. 等待从Main.go中传入的discoveryManager的SyncCh是否有变动，如果有变动，更新Targetset。
2. 派生出了一个Reloader协程，Reloader协程会定时检查是否有关闭的信号或者Reload信号（triggerReload channel,就是外部给与主协程的刺激产生的二级信号），如果有，则执行reload操作。


## 子协程逻辑

### ScraperPool
```
// Sync converts target groups into actual scrape targets and synchronizes
// the currently running scraper with the resulting set and returns all scraped and dropped targets.
func (sp *scrapePool) Sync(tgs []*targetgroup.Group) {
	start := time.Now()

	var all []*Target
	sp.mtx.Lock()
	sp.droppedTargets = []*Target{}
	for _, tg := range tgs {
		targets, err := targetsFromGroup(tg, sp.config)
		if err != nil {
			level.Error(sp.logger).Log("msg", "creating targets failed", "err", err)
			continue
		}
		for _, t := range targets {
			if t.Labels().Len() > 0 {
				all = append(all, t)
			} else if t.DiscoveredLabels().Len() > 0 {
				sp.droppedTargets = append(sp.droppedTargets, t)
			}
		}
	}
	sp.mtx.Unlock()
	sp.sync(all)

	targetSyncIntervalLength.WithLabelValues(sp.config.JobName).Observe(
		time.Since(start).Seconds(),
	)
	targetScrapePoolSyncsCounter.WithLabelValues(sp.config.JobName).Inc()
}
```

Sync函数是一个对外暴露函数的接口：
1. 把配置解析出来的target结构化。
2. 调用内部方法sync()来进行数据抓取的执行
3. 一些计数器添加计数

值得注意的是Append方法，是一个封装了的方法，是同是进行对变量的修改，并且包含了采集到的数据持久化的操作。


```
// sync takes a list of potentially duplicated targets, deduplicates them, starts
// scrape loops for new targets, and stops scrape loops for disappeared targets.
// It returns after all stopped scrape loops terminated.
func (sp *scrapePool) sync(targets []*Target) {
	sp.mtx.Lock()
	defer sp.mtx.Unlock()

	var (
		uniqueTargets   = map[uint64]struct{}{}
		interval        = time.Duration(sp.config.ScrapeInterval)
		timeout         = time.Duration(sp.config.ScrapeTimeout)
		limit           = int(sp.config.SampleLimit)
		honorLabels     = sp.config.HonorLabels
		honorTimestamps = sp.config.HonorTimestamps
		mrc             = sp.config.MetricRelabelConfigs
	)

	for _, t := range targets {
		t := t
		hash := t.hash()
		uniqueTargets[hash] = struct{}{}

		if _, ok := sp.activeTargets[hash]; !ok {
			s := &targetScraper{Target: t, client: sp.client, timeout: timeout}
			l := sp.newLoop(scrapeLoopOptions{
				target:          t,
				scraper:         s,
				limit:           limit,
				honorLabels:     honorLabels,
				honorTimestamps: honorTimestamps,
				mrc:             mrc,
			})

			sp.activeTargets[hash] = t
			sp.loops[hash] = l

			go l.run(interval, timeout, nil)
		} else {
			// Need to keep the most updated labels information
			// for displaying it in the Service Discovery web page.
			sp.activeTargets[hash].SetDiscoveredLabels(t.DiscoveredLabels())
		}
	}

	var wg sync.WaitGroup

	// Stop and remove old targets and scraper loops.
	for hash := range sp.activeTargets {
		if _, ok := uniqueTargets[hash]; !ok {
			wg.Add(1)
			go func(l loop) {

				l.stop()

				wg.Done()
			}(sp.loops[hash])

			delete(sp.loops, hash)
			delete(sp.activeTargets, hash)
		}
	}

	// Wait for all potentially stopped scrapers to terminate.
	// This covers the case of flapping targets. If the server is under high load, a new scraper
	// may be active and tries to insert. The old scraper that didn't terminate yet could still
	// be inserting a previous sample set.
	wg.Wait()
}
```

主要逻辑：
1. 把传入的Target列表进行遍历
   1.1 如果target不在active的map中， 生成targetScraper，然后把targetScraper放入Loop里面，调用Loop.run()在协程中进行逻辑
   1.2 否则， 会先删除旧的协程，然后重新生成协程。

### ScraperLoop

```
func (sl *scrapeLoop) run(interval, timeout time.Duration, errc chan<- error) {
	select {
	case <-time.After(sl.scraper.offset(interval, sl.jitterSeed)):
		// Continue after a scraping offset.
	case <-sl.ctx.Done():
		close(sl.stopped)
		return
	}

	var last time.Time

	ticker := time.NewTicker(interval)
	defer ticker.Stop()

mainLoop:
	for {
		select {
		case <-sl.parentCtx.Done():
			close(sl.stopped)
			return
		case <-sl.ctx.Done():
			break mainLoop
		default:
		}

		var (
			start             = time.Now()
			scrapeCtx, cancel = context.WithTimeout(sl.ctx, timeout)
		)

		// Only record after the first scrape.
		if !last.IsZero() {
			targetIntervalLength.WithLabelValues(interval.String()).Observe(
				time.Since(last).Seconds(),
			)
		}

		b := sl.buffers.Get(sl.lastScrapeSize).([]byte)
		buf := bytes.NewBuffer(b)

		contentType, scrapeErr := sl.scraper.scrape(scrapeCtx, buf)
		cancel()

		if scrapeErr == nil {
			b = buf.Bytes()
			// NOTE: There were issues with misbehaving clients in the past
			// that occasionally returned empty results. We don't want those
			// to falsely reset our buffer size.
			if len(b) > 0 {
				sl.lastScrapeSize = len(b)
			}
		} else {
			level.Debug(sl.l).Log("msg", "Scrape failed", "err", scrapeErr.Error())
			if errc != nil {
				errc <- scrapeErr
			}
		}

		// A failed scrape is the same as an empty scrape,
		// we still call sl.append to trigger stale markers.
		total, added, seriesAdded, appErr := sl.append(b, contentType, start)
		if appErr != nil {
			level.Warn(sl.l).Log("msg", "append failed", "err", appErr)
			// The append failed, probably due to a parse error or sample limit.
			// Call sl.append again with an empty scrape to trigger stale markers.
			if _, _, _, err := sl.append([]byte{}, "", start); err != nil {
				level.Warn(sl.l).Log("msg", "append failed", "err", err)
			}
		}

		sl.buffers.Put(b)

		if scrapeErr == nil {
			scrapeErr = appErr
		}

		if err := sl.report(start, time.Since(start), total, added, seriesAdded, scrapeErr); err != nil {
			level.Warn(sl.l).Log("msg", "appending scrape report failed", "err", err)
		}
		last = start

		select {
		case <-sl.parentCtx.Done():
			close(sl.stopped)
			return
		case <-sl.ctx.Done():
			break mainLoop
		case <-ticker.C:
		}
	}

	close(sl.stopped)

	sl.endOfRunStaleness(last, ticker, interval)
}
```

ScraperLoop是单个Target进行获取的执行单位，协程使用死循环进行占用，然后调用scraper接口的Scrape方法去抓取数据，并且调用Stroage模块的Appender的接口金属数据的持久化，然后继续定时休眠的过程。我们需要更加具体的看一下实例Scraper的Scrape方法。

ScraperLoop把Scraper抽象出来的三个接口都进行了调用：
1. 开始部分的Select代码段中的Offset是用于控制第一次执行的时候等待的间隔
2. Scrape方法就是直接进行数据的抓取，下面有详细解析
3. report方法，修改Scraper中Target自己保存的状态。



### TargerScraper
```
func (s *targetScraper) scrape(ctx context.Context, w io.Writer) (string, error) {
	if s.req == nil {
		req, err := http.NewRequest("GET", s.URL().String(), nil)
		if err != nil {
			return "", err
		}
		req.Header.Add("Accept", acceptHeader)
		req.Header.Add("Accept-Encoding", "gzip")
		req.Header.Set("User-Agent", userAgentHeader)
		req.Header.Set("X-Prometheus-Scrape-Timeout-Seconds", fmt.Sprintf("%f", s.timeout.Seconds()))

		s.req = req
	}

	resp, err := s.client.Do(s.req.WithContext(ctx))
	if err != nil {
		return "", err
	}
	defer func() {
		io.Copy(ioutil.Discard, resp.Body)
		resp.Body.Close()
	}()

	if resp.StatusCode != http.StatusOK {
		return "", errors.Errorf("server returned HTTP status %s", resp.Status)
	}

	if resp.Header.Get("Content-Encoding") != "gzip" {
		_, err = io.Copy(w, resp.Body)
		if err != nil {
			return "", err
		}
		return resp.Header.Get("Content-Type"), nil
	}

	if s.gzipr == nil {
		s.buf = bufio.NewReader(resp.Body)
		s.gzipr, err = gzip.NewReader(s.buf)
		if err != nil {
			return "", err
		}
	} else {
		s.buf.Reset(resp.Body)
		if err = s.gzipr.Reset(s.buf); err != nil {
			return "", err
		}
	}

	_, err = io.Copy(w, s.gzipr)
	s.gzipr.Close()
	if err != nil {
		return "", err
	}
	return resp.Header.Get("Content-Type"), nil
}

```

Scrape方法是使用HttpClient进行对target url 的数据抓取，抓取的内容在context中进行传递，得到返回后，继续解析，返回给ScraperLoop的Run方法使用。