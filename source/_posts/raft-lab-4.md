---
title: MIT6.824 Lab4 实现及解析
date: 2019-08-29 20:05:38
tags: distributed-system
---

<!-- ---
layout: post
title: MIT6.824 Lab4 实现及解析
author: Ray Chan(ray1888)
date: '2019-08-20 12:05:38 +0800'
category: distributed-system
summary: MIT6.824 Lab4 Implemantation and detail explains
thumbnail: distributed-db.jpg
--- -->

# 目录
1. [实验目的](#Purpose)  
2. [实验实现](#Implementation)  
   2.1 [lab4-1](#2.1)  
   2.2 [lab4-2](#2.2)  


# <a id="Purpose"><span class="toptitle">实验目的</span></a>

lab4需要实现两个模块:
1. lab4-1 完成一个基于KvRaft的ShardMaster（可以理解为一个分片调度的存放的机器），但是写入的是配置而不是简单的K-v的值（但是与Lab3的实现相当相似）
2. lab4-2 完成一个Sharding的kvRaft数据库（实现了Multi-raft)

整体的实验架构是这样的

<!-- ![lab4](/assets/img/posts/lab4.png) -->
{% asset_img lab4.png lab4 %}

架构图如上：
一个值得注意的点是，各个KV数据库层之间不会直接相互通信，只是会通过Raft层来同步操作。    
并且Client如果找到的不是Leader的节点，会直接放弃操作。然后请求集群内的下一个节点。    
PS: 这里的KvDatabase 和Raft总体包起来才是一个KVServer的实例，而不是单独脱离的。    

流程大概如下:
1. Client发送请求到ShardMaster，查询Key的具体对应的分片在哪个组里面
2. ShardMaster收到请求后，返回对应数据Key所在的组信息
3. Client发送请求到对应分片的Raft Leader中
4. Raft层同步成功后，会通过ApplyCh返回信号的KvDatabase
5. 分片的KvDatabase对Client端做出应答（如果成功返回结果为成功，如果下面同步层失败导致超时，返回给客户端的结果为超时）

## 副本与分片的问题
在上面的架构图中，Multi-Raft已经实现了分片的副本的实现。

分片： 只要是把数据进行划分（从大的数据变为只负责一部分的数据量）即是分片（在定义上面数据库的垂直划分和水平划分和此处的Key按键去Mod划分都是属于分片的操作，但是数据库表的划分和此处的Key的Mod划分不是在同一个层级上面的，理解不一样）  
副本： 是指数据的重复的数量。但是一般只有一份的数据我们不会称之为单副本，而是称为0副本。副本一般是>=2才叫。副本的目的是为了冗余的问题。防止因为单点故障而导致数据全部的丢失。  

# <a id="Implementation"><span class="toptitle">实验实现</span></a>

## <a id="2.1"><span class="secondtitle">lab4-1</span></a>

对比起与Lab3 的实现，它在这里需要支持的是Leave(), Move(), Join()， Query()的四种方法，因为分别对应节点的加入集群、退出集群、移动集群和集群配置查询的四种操作。
所以基本实现的思路与Lab3是类似的。  

```
type ShardMaster struct {
	mu      sync.Mutex
	me      int
	rf      *raft.Raft
	applyCh chan raft.ApplyMsg
	// Your data here.
    // Lab3 此处是为database 是一个map[string]string 的结构
	configs    []Config // indexed by config num
	dup        map[int64]Result
	chanResult map[int]chan Result
}

```

但是有一个比较重要的需求是，它需要完成一个shard 与 集群的绑定关系的变动，因为本来就是需要支持的目的就是数据随着Group的添加和变动来做到数据的均衡。  
并且一个比较不同的是，对于重复的Get操作操作，Lab3采用的是直接返回Kv的值，但是在此处，因为Config的存放机制是一个数组（里面的顺序就是配置有效的顺序），因此需要把重复的读配置，返回一个复制好的配置。  

## <a id="2.2"><span class="secondtitle">lab4-2</span></a>

这里的实现是需要首先保证Kv的功能可以使用，然后保证在配置变动并且数据搬移完成之后，才能继续对外提供Kv的服务。并且需要保证节点挂掉之后可以读取会最新的状态下来

总体流程：
在提供KV服务的同时，需要把配置定时进行更新，并且实际应用新配置之前，必须保证数据迁移成功。因此实际上用到了3个单独的协程去分别做这几个工作
1. 读取配置协程（定时向ShardMaster请求配置）
2. 数据迁移的协程
3. 应用数据同步的协程

因为这个实验中的目的有三种，因此我们的消息类型也定义了三种
1. 数据操作类型，与Lab3原来类似的OP类型（可以包含Get、Pull、Put、Append的操作）
2. 配置更新类型，把Lab4-1的Config类型封装一层进行使用
3. 真实的数据迁移类型，原因是：因为数据迁移的时候是两个RaftGroup的Leader相互通信，并且需要把原来数据KV格式同步进去到新的组的所有副本上面。因此单独分配一个数据类型来记录此类数据

向ShardMaster读取配置的协程的实现
```
func (kv *ShardKV) ConfigUpdateRoutine() {
	for {
		kv.mu.Lock()
		curConfigNum := kv.myconfig[0].Num
		kv.mu.Unlock()
        // 此处Query带上当前版本加1的原因，防止查询到的配置不正确，导致分片的节点之前一直无法达成配置上共识
		config := kv.mck.Query(curConfigNum + 1)
		//DPrintf("get newConfig from SM group is %v, shard is %v", config.Groups, config.Shards)
		kv.mu.Lock()
		if config.Num == kv.myconfig[0].Num+1 {
			// update with static NewConfig
			newConfig := kv.makeEmptyConfig()
			kv.CopyConfig(&config, &newConfig)
			if _, isLeader := kv.rf.GetState(); isLeader {
				cfg := Cfg{newConfig, int64(kv.gid), kv.myconfig[0].Num}
				kv.rf.Start(cfg)
				DPrintf("Config: group %d-%d is start config %d into consueum",
					kv.gid, kv.me, cfg.NewConfig.Num)
				//index, _, isleader := kv.rf.Start(cfg)
				//DPrintf("))))) server %d gid %d Start cfg， index is %d, isleader is %t, kv is %v",
				//	kv.me, kv.gid, index, isleader, kv.database)
			}
		} 
		kv.mu.Unlock()
		time.Sleep(100 * time.Millisecond)
	}
}

```

修改自己需要发送和接受Shard的配置部分的代码
```
case Cfg:
    if kv.CfgDupCheck(cmd.ClientId, cmd.Seq) {
        kv.SwitchConfig(cmd)
        if kv.CheckMigrateDone() {
            // if migrate done, use new config, if not, do nothing to avoid replying the old group replied
            kv.myconfig[0] = kv.myconfig[1]
            //DPrintf("group %d-%d is applied new config , shard is %v, kv is %v", kv.gid, kv.me, kv.myshards, kv.database)
        }
        kv.cfgdup[cmd.ClientId] = cmd.Seq
        if kv.maxraftstate != -1 {
            kv.SaveSnapshot(index)
        }
    }

func (kv *ShardKV) SwitchConfig(newcfg Cfg) {
	if newcfg.NewConfig.Num == kv.myconfig[0].Num+1 {
		if kv.myconfig[0].Num != 0 {
			kv.GenShardChangeList(newcfg)
		} else if kv.myconfig[0].Num == 0 {
			for i := 0; i < shardmaster.NShards; i++ {
				if newcfg.NewConfig.Shards[i] == kv.gid {
					kv.myshards[i] = 1
				}
			}
		}
		newc := kv.makeEmptyConfig()
		kv.CopyConfig(&newcfg.NewConfig, &newc)
		kv.myconfig[1] = newc
	}
}

// 此函数是用于生成需要发送和修改那些部分的参数
func (kv *ShardKV) GenShardChangeList(newcfg Cfg) {
	for i := 0; i < shardmaster.NShards; i++ {
		if kv.myconfig[0].Shards[i] == kv.gid && newcfg.NewConfig.Shards[i] != kv.gid {
			//need to send
			kv.needsend[i] = newcfg.NewConfig.Shards[i]
		}
		if kv.myconfig[0].Shards[i] != kv.gid && newcfg.NewConfig.Shards[i] == kv.gid {
			//need to recv
			_, ok := kv.needrecv[kv.myconfig[0].Shards[i]]
			if !ok {
				kv.needrecv[kv.myconfig[0].Shards[i]] = make([]int, 0)
			}
			kv.needrecv[kv.myconfig[0].Shards[i]] = append(kv.needrecv[kv.myconfig[0].Shards[i]], i)
		}
	}
	DPrintf("!!! group %d-%d, new config need to send is %v, need to receive is %v", kv.gid, kv.me, kv.needsend, kv.needrecv)
}
```


获取迁移的数据部分  
数据迁移的副本是只要检测到相关属性的变化之后（感知到数据的变化）后，新的数据所归属的Leader就会与旧Leader继续RPC的Pull调用， 去获取它的Database和DUP的部分  
当拉取到配置了之后，就会把数据变成日志应用到状态中，就可以实现分片数据的副本的性质。
```
func (kv *ShardKV) MigrationRoutine() {
	for {
		if _, isLeader := kv.rf.GetState(); isLeader {
			kv.mu.Lock()
			for k, v := range kv.needrecv {
				//DPrintf("group %d-%d needrecv ")
				needshard := make([]int, 0)
				for i := 0; i < len(v); i++ {
					needshard = append(needshard, v[i])
				}

				args := PullArgs{Shard: needshard, ClientId: int64(kv.gid), Seq: kv.myconfig[0].Num}
				go func(mgid int, arg *PullArgs) {
					servers := kv.myconfig[0].Groups[mgid]
					DPrintf("Migrate: group %d-%d get Gid %d Servers is %v",
						kv.gid, kv.me, mgid, servers)
					for {
						for _, si := range servers {
							reply := PullReply{}
							srv := kv.make_end(si)
							DPrintf("group %d-%d start call to gid %d", kv.gid, kv.me, mgid)
                            // 由GroupA
							ok := srv.Call("ShardKV.Pull", arg, &reply)
							//DPrintf("Migrate: group %d-%d  calling for server %v rpc pull, result is %t",
							//	    kv.gid, kv.me, si, ok)
							if !ok {
								DPrintf("Migrate Failed: group %d-%d  calling for server %v rpc pull, result is %t",
									kv.gid, kv.me, si, ok)
							}
							if ok && reply.WrongLeader == false {
								if reply.Err == ErrNeedWait {
									DPrintf("Migrate: waiting server %v to pull new config from SM", si)
									return
								}
								if _, isleader := kv.rf.GetState(); isleader {
									newmapkv := make(map[string]string)
									for k, v := range reply.MapKV {
										newmapkv[k] = v
									}
									var newdup [shardmaster.NShards]map[int64]int
									for i := 0; i < shardmaster.NShards; i++ {
										newdup[i] = make(map[int64]int)
										for k, v := range reply.ShardDup[i] {
											newdup[i][k] = v
										}
									}
									mig := Migrate{newmapkv, newdup, arg.Seq, mgid}
									kv.mu.Lock()
									// this is how partition data can be repliacated
									kv.rf.Start(mig)
									DPrintf("Migrate: group %d-%d start migrate the data pulled from %d", kv.gid, kv.me, mgid)
									kv.mu.Unlock()
									return
								}
							} else {
								DPrintf("Migrate Failed: group %d-%d call %d-%v meet wrong leader",
									kv.gid, kv.me, mgid, si)
								DPrintf("!!!server is %v", servers)
							}
							time.Sleep(20 * time.Millisecond)
						}
					}
				}(k, &args)
			}
			kv.mu.Unlock()
		}
		time.Sleep(100 * time.Millisecond)
	}
}

```

非Leader节点同步KV数据的方法
```
case Migrate:
    if kv.MigrateDupCheck(cmd.Gid, cmd.Num) {
        //DPrintf("group %d-%d apply the migrate data from %d and config num %d", kv.gid, kv.me, cmd.Gid, cmd.Num)
        //DPrintf("group %d-%d database before migrate is %v", kv.gid, kv.me, kv.database)
        for k, v := range cmd.MapKV {
            kv.database[k] = v
        }
        //DPrintf("group %d-%d database after migrate is %v", kv.gid, kv.me, kv.database)
        for i := 0; i < shardmaster.NShards; i++ {
            for k, v := range cmd.ShardDup[i] {
                kv.dup[i][k] = v
            }
        }
        for i := 0; i < len(kv.needrecv[cmd.Gid]); i++ {
            kv.myshards[kv.needrecv[cmd.Gid][i]] = 1
        }
        delete(kv.needrecv, cmd.Gid)
        if kv.CheckMigrateDone() {
            kv.myconfig[0] = kv.myconfig[1]
            DPrintf("Migrate: group %d-%d successful switch to config %d", kv.gid, kv.me, kv.myconfig[0].Num)
            //DPrintf("group %d-%d is applied new config , shard is %v", kv.gid, kv.me, kv.myshards)
        }
        kv.migratedup[cmd.Gid] = cmd.Num
        if kv.maxraftstate != -1 {
            kv.SaveSnapshot(index)
        }
    }
```