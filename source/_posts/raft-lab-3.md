---
title:  MIT6.824 Lab3 实现及解析
date: 2019-08-29 15:05:38
tags: distributed-system
---

<!-- ---
layout: post
title:
author: Ray Chan(ray1888)
date: '2019-08-20 12:05:38 +0800'
category: 
summary: MIT6.824 Lab3 Implemantation and detail explains
thumbnail: distributed-db.jpg
--- -->

# 目录
1. 实验目的
2. 实验实现

# <a id=""><span class="toptitle">实验目的</span></a>
lab3需要实现一个建议的带客户端的 分布式KV的数据库，需要支持对外的Get(), Put(), Append()三个操作

整体的实验架构是这样的

<!-- ![lab3](/assets/img/posts/Lab3-Process.png) -->
{% asset_img Lab3-Process.png lab3 %}
架构图如上：
一个值得注意的点是，各个KV数据库层之间不会直接相互通信，只是会通过Raft层来同步操作。  
并且Client如果找到的不是Leader的节点，会直接放弃操作。然后请求集群内的下一个节点。  
PS: 这里的KvDatabase 和Raft总体包起来才是一个KVServer的实例，而不是单独脱离的。  

流程大概如下:
1. Client发送请求到KvDatabase
2. KvDatabase收到请求后，会把收到的命令重新封装，通过Raft提供的Start API来在Raft集群中进行同步的操作
3. Raft同步到其他的非Leader节点中
4. Raft层同步成功后，会通过ApplyCh返回信号的KvDatabase
5. KvDatabase对Client端做出应答（如果成功返回结果为成功，如果下面同步层失败导致超时，返回给客户端的结果为超时）

# <a id=""><span class="toptitle">实验实现</span></a>

此部分实现分为两个部分：
1. 基础的键值对的实现
2. 日志压缩与快照的部分


## <a id=""><span class="secondtitle">基础键值对的实现</span></a>

因为上面提及到这里的KvDatabase 和Raft总体包起来才是一个KVServer的实例，而不是单独脱离的。  
所以此处展示一下KvRaft所包含的结构
```
type KVServer struct {
	mu      sync.Mutex
	me      int
    // 包含了Raft的实例
	rf      *raft.Raft
    // Applych 是 用与接受Raft成形成共识后的返回
	applyCh chan raft.ApplyMsg

	maxraftstate int // snapshot if log grows this big
	database map[string]string
	dup map[int64]int
	chanResult map[int]chan Op
}

// Type OP
type Op struct {
    // Key 是 Get()、Put()、Append()三个都会用到的值
	Key      string
    // Value是Put()、Append()用到的字段，Get此处默认为空
	Value    string
    // 存放的是操作的名称
	Name     string
    // 用于表示客户端的来源
	ClientId int64
    // 给予序号
	Seq      int
}
```

上面所描述到的Op结构体的操作为什么要把Get加入，并且需要区分ClientId和Seq的原因，本质上都是需要在全序关系广播中实现线性化所需要的。如果不知道什么是全序关系广播，可以查看之前我的文章[分布式系统一致性与共识](https://ray1888.github.io/distributed-system/2019/08/20/distributed-consensus/)

此处还需要注意，因为我们暴露给客户端的操作是一个同步的操作，但是我们这层与Raft层是一个异步的操作，因此，需要我们这边等待Raft层异步返回成功，并且我们此层把数据保留下来后，才能继续返回

所以需要需要在另外一个协程中去读取ApplyCH的数据，然后继续对比之后再去创建返回给客户端

```
func (kv *KVServer) ApplyOPRoutine() {
	//this gorouine is to asyncly get the result of raft applych reply to
	// and to produce signal to reply client Rpc Request
	DPrintf("Apply gorountine runing ")
	for {
		msg := <-kv.applyCh
		//DPrintf("get apply msg from raftServer")
		if msg.CommandValid {
			index := msg.CommandIndex
			if cmd, ok := msg.Command.(Op); ok {
				kv.mu.Lock()
                // 对比单个客户端的序号，来减少重复的旧操作的更新操作
				if kv.dupcheck(cmd.ClientId, cmd.Seq) {
					if cmd.Name == PUT {
						kv.database[cmd.Key] = cmd.Value
					} else if cmd.Name == APPEND {
						if _, ok := kv.database[cmd.Key]; ok {
							kv.database[cmd.Key] += cmd.Value
						} else {
							kv.database[cmd.Key] = cmd.Value
						}
					}
					kv.dup[cmd.ClientId] = cmd.Seq
				}
				res := Op{cmd.Key, kv.database[cmd.Key], cmd.Name,
					cmd.ClientId, cmd.Seq}
				ch, ok := kv.chanResult[index]
				if ok {
					select {
					case <-ch:
					default:
					}
					ch <- res
					//DPrintf("the cmd has been commited , push request return to chan")
				}
				if kv.maxraftstate != -1 && kv.rf.GetStateSize() >= kv.maxraftstate && index == kv.rf.GetCommitedIndex() {
					DPrintf("Do snapshot for over the maxraftstate")
					kv.DoSnapShot(index)
				}
				kv.mu.Unlock()
			}
		} else {
			kv.LoadSnapShot(msg.Snapshot)

		}
	}
}

//  把OP传入到Raft共识层的函数
func(rf *Raft) StartCommand(cmd Op) (Err, string){
    index, _, isLeader := kv.rf.Start(cmd)
	//DPrintf("start command %s , client id is %d, key is %s, value is %s",
	//	cmd.Name, cmd.ClientId, cmd.Key, cmd.Value)
	if !isLeader {
		kv.mu.Unlock()
		//DPrintf("not leader ")
		return ERRWrongLeader, ""
	}
	ch := make(chan Op, 1)
	kv.chanResult[index] = ch
	kv.mu.Unlock()

	defer func() {
		// After finish the task
		kv.mu.Lock()
		delete(kv.chanResult, index)
		kv.mu.Unlock()
	}()
	select {
	case c := <-ch:
		// this channel return is get data from ApplyRoutine
		if kv.CheckSame(c, cmd) {
			resvalue := ""
			if cmd.Name == GET {
				resvalue = c.Value
			}
			return OK, resvalue
		} else {
			DPrintf("Leader has change, index %d op %s error", index, cmd.Name)
			return ERRWrongLeader, ""
		}
	case <-time.After(time.Duration(200) * time.Millisecond):
		DPrintf("log get agree timeout, index is %d", index)
		return ERRTimeout, ""
	}
}
```

## <a id=""><span class="secondtitle">日志压缩与快照</span></a>

此处的快照与Raft本身的快照多了两个KVDatabase 特有的属性， database（保存的数据） 和 dup （客户端操作序号的记录）这两个属性
```
func (kv *KVServer) DoSnapShot(index int) {
	kv.rf.SaveSnapShot(index, kv.database, kv.dup)
}


// 读取快照的函数
func (kv *KVServer) LoadSnapShot(snapshot []byte) {
	if snapshot == nil || len(snapshot) < 1 {
		kv.mu.Lock()
		kv.database = make(map[string]string)
		kv.mu.Unlock()
		return
	}
	s := bytes.NewBuffer(snapshot)
	decoder := labgob.NewDecoder(s)
	var kvdb map[string]string
	var kvdup map[int64]int
	if decoder.Decode(&kvdb) != nil || decoder.Decode(&kvdup) != nil {
		DPrintf("server %d, Decode Snapshot error", kv.me)
	} else {
		kv.mu.Lock()
		defer kv.mu.Unlock()
		kv.database = kvdb
		kv.dup = kvdup
		DPrintf("msg snapshot db is %v, dup is %v", kvdb, kvdup)
		DPrintf("server %d , load Snapshot success", kv.me)
	}
}

```

在本实验中，会触发日志保留的情况只是因为保存的Log> maxraftstate。 



