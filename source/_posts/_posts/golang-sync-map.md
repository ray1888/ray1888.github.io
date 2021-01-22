---
title: sync.Map 源码阅读
date: 2020-12-10 16:07:38
tags: Go
---


<!-- ---
layout: post
title: sync.Map源码圆度
author: Ray Chan(ray1888)
date: '2020-12-10 16:07:38 +0800'
category: Golang
summary: sync.Map implate 
thumbnail: go.png
--- -->


# 前言
Sync.Map  是Golang 官方提供的线程安全的 map类库，因为Golang 本身自带的map并不是线程安全的，因为会有sync.Map这个类库的存在

# 实现
## 基础元素
```
type Map struct {
    mu Mutex

    // read contains the portion of the map's contents that are safe for
    // concurrent access (with or without mu held).
    //
    // The read field itself is always safe to load, but must only be stored with
    // mu held.
    //
    // Entries stored in read may be updated concurrently without mu, but updating
    // a previously-expunged entry requires that the entry be copied to the dirty
    // map and unexpunged with mu held.
    read atomic.Value // readOnly

    // dirty contains the portion of the map's contents that require mu to be
    // held. To ensure that the dirty map can be promoted to the read map quickly,
    // it also includes all of the non-expunged entries in the read map.
    //
    // Expunged entries are not stored in the dirty map. An expunged entry in the
    // clean map must be unexpunged and added to the dirty map before a new value
    // can be stored to it.
    //
    // If the dirty map is nil, the next write to the map will initialize it by
    // making a shallow copy of the clean map, omitting stale entries.
    dirty map[interface{}]*entry

    // misses counts the number of loads since the read map was last updated that
    // needed to lock mu to determine whether the key was present.
    //
    // Once enough misses have occurred to cover the cost of copying the dirty
    // map, the dirty map will be promoted to the read map (in the unamended
    // state) and the next store to the map will make a new dirty copy.
    misses int
}

// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
    m       map[interface{}]*entry
    amended bool // true if the dirty map contains some key not in m.
}
type entry struct {
    // p points to the interface{} value stored for the entry.
    //
    // If p == nil, the entry has been deleted and m.dirty == nil.
    //
    // If p == expunged, the entry has been deleted, m.dirty != nil, and the entry
    // is missing from m.dirty.
    //
    // Otherwise, the entry is valid and recorded in m.read.m[key] and, if m.dirty
    // != nil, in m.dirty[key].
    //
    // An entry can be deleted by atomic replacement with nil: when m.dirty is
    // next created, it will atomically replace nil with expunged and leave
    // m.dirty[key] unset.
    //
    // An entry's associated value can be updated by atomic replacement, provided
    // p != expunged. If p == expunged, an entry's associated value can be updated
    // only after first setting m.dirty[key] = e so that lookups using the dirty
    // map find the entry.
    p unsafe.Pointer // *interface{}
}
```

sync.Map 存储数据的分为两个部分：
1.read元素，使用的是atomic.Value 把map 当成interface进行存储；
2. dirty 一个golang中的一个Map，用于临时存储部分新的数据（具体场景会在后面进行描述）

## 写流程
```
1.  判断Key是否再Read部分，如果在read部分，则直接采用cas来更换，更换成功则直接返回
2.  否则就要加锁然后修改到dirty中，
      有三种情况：
      1. 在Read中，但是已经被修改，cas无法修改成功（会先把这个key对应的值与mark无效的指针做对比，如果是，则需要把这个更换了的key对应的放到dirty里面）
      2. 在dirty 中，直接进行修改
      3. 是一个新Key的情况下，如果dirty没有key不在read中，直接修改dirty,并且把需要读修正的Flag（read.amend = true置入）
3. 解锁
```
因此 如果当key在Read中，是可以保证一个比较快的实现，因为用的是cas的比较方法，而不是直接加锁去防止竞争锁带来的性能损失。
## 读流程
1.  读取Read中是否存在此Key，若存在则直接返回
2.  当发现key不存在于read，并且没有读修正的时候，直接返回
3.  发现需要读修正的情况下，会顺便把dirty中的数据置入到read中，并且返回值

## LoadORStore 读写一体流程


# 用于阅读sync.Map的触发代码
```
package main

import (
    "fmt"
    "sync"
)

func main() {
    // for testing sync.map work

    var ma sync.Map
    k, _ := ma.Load("a")
    fmt.Println(k)

    ma.Store("a", 1)
    _, _ = ma.Load("a")
    ma.Store("b", 2)
    ma.Store("a", 2)
    ma.Store("a", 3)
    p, _ := ma.Load("a")
    fmt.Println(p)
}

```

竞锁的情况
```

```



