---
title:  MIT6.824 Lab2 实现及解析
date: 2019-08-28 15:05:38
tags:  distributed-system
---

<!-- ---
layout: post
title:
author: Ray Chan(ray1888)
date: '2019-08-20 12:05:38 +0800'
category: distributed-system
summary: MIT6.824 Lab2 Implemantation and detail explains
thumbnail: distributed-db.jpg
--- -->

# 目录
1. [Raft论文阅读](#PaperReading)  
2. [实现细节](#Detail)    
        2.1 [Raft整体流程](#2.1)    
        2.2 [Raft状态机维护](#2.2)  
        2.3 [Raft代码实现](#2.3)  

# <a id="PaperReading"><span class="toptitle">论文阅读及问题</span></a>

论文各个章节主要解析
1. 总体介绍，解析创造Raft的原因，并且简单描述Raft与Paxos的不同点
2. 简单介绍一下副本状态机这个概念
3. 简单描述Paxos的问题（包括了不好理解以及工业化上面的问题）
4. 阐述Raft为什么比Paxos好理解
5. Raft的算法的详细描述以及边界条件
6. Raft集群出现成员变动的时候如何处理
7. Raft的日志压缩和快照
8. Client与Raft集群的交互方式

实现的时候，重点要看Figure2中所提及的条件。

# <a id="Detail"><span class="toptitle">代码实现</span></a>
## <a id="2.1"><span class="secondtitle"> Raft整体流程</span></a>

<!-- ![Raft整体流程](/assets/img/posts/RaftProcess.png) -->
{% asset_img RaftProcess.png Raft整体流程 %}

此处只是一个比较简单的忽略具体可能出现错误细节的描述。
本质上Raft就是一个通过一个维护一个协程里面的状态机，并且通过其他做网络请求的协程达成大部分节点在共识的算法。
具体可以看这里的![演示动画](https://raft.github.io/)

## <a id="2.2"><span class="secondtitle"> Raft状态机的转换</span></a>
<!-- ![Raft整体流程](/assets/img/posts/RaftStateMachine.png) -->
{% asset_img RaftStateMachine.png Raft状态机流程 %}

此处是一个简单状态机的描述，具体转换的原因也已经在图上面有标出来。



## <a id="2.3"><span class="secondtitle"> Raft代码的实现</span></a>
代码实现在这个Lab中实际上分为了3part
1. PartA  完成选举
2. PartB  完成日志
3. PartC  完成持久化的工作，并且处理共识的边界条件


### PartA 的实现 
跟随论文中提及的RequestVote RPC 以及 AppendEntries RPC 实现即可
此处可以只需要完成心跳发送后可以保证非主节点不超时，不会使得Term有变动即可。

一个注意的点，发送这两个RPC的时候，都是需要并行发送的，因此每个请求需要使用一个GoRoutine去进行实现。
下面代码就是发送AppendEntries的代码的实例
```
    logs := make([]Log, 0)
    tmpIndex := min(max(0, rf.nextIndex[number]-1), rf.logs[rf.getLen()].Index)
    if tmpIndex < rf.logs[rf.getLen()].Index {
        logs = rf.logs[tmpIndex+1-firstIndex:]
        //DPrintf("server %d is Leader, tmpIndex is %d, firstIndex is %d, len of log is %d, commited index is %d",
        //	rf.me, tmpIndex, firstIndex, len(rf.logs), rf.commitedIndex)
    }
    args := AppendEntriesArgs{Term: rf.currentTerm, LeaderId: rf.me,
        PrevLogTerm: rf.logs[tmpIndex-firstIndex].Term, PrevLogIndex: tmpIndex,
        Entries: logs, Leadercommited: rf.commitedIndex}
    //  使用Goroutine来发送请求
    go func(args AppendEntriesArgs, number int) {
        reply := AppendEntriesReply{}
        ok := rf.sendAppendEntires(number, &args, &reply)
        rf.mu.Lock()
        defer rf.mu.Unlock()
        if !ok {
            DPrintf("allAppendEntries server %d call remote %d rpc AppendEntries failed", rf.me, number)
        }
        if args.Term != rf.currentTerm {
            return
        }
        if reply.Term > rf.currentTerm {
            //rf.mu.Lock()
            rf.state = Follower
            rf.currentTerm = reply.Term
            // TODO why Term voliate needed to persist?
            rf.persist()
            //rf.mu.Unlock()
            return
        }
        if reply.Success == true && rf.state == Leader {
            if len(args.Entries) > 0 {
                rf.nextIndex[number] = args.Entries[len(args.Entries)-1].Index + 1
                rf.matchIndex[number] = rf.nextIndex[number] - 1
                if rf.matchIndex[number] > rf.commitedIndex {
                    rf.updateCommit()
                }
            }
        } else if rf.state == Leader {
            if rf.nextIndex[number] > reply.PrevIndex+1 {
                rf.nextIndex[number] = reply.PrevIndex + 1
            } else {
                rf.nextIndex[number] = max(rf.nextIndex[number]-1, 1)
            }
        }
    }(args, number)
```
并且注意实现的时候的锁的使用，保证每次使用锁都有加锁和解锁的操作成对出现，否则可能会出现一些比较奇怪的情况。


### PartB的实现
此处需要接受模拟从客户端的请求进来然后把Log先放到本地的Log上面，然后把Log同步到其他的节点上面去进行共识的达成。

首先的条件，需要确保Start函数接受请求的时候必须是Leader的角色，否则不会接收请求。（论文中有提及，此处Raft集群中非Leader节点不会把日志写入的请求转发到Leader节点上，这个是为了维护的方便来做的）
```
func (rf *Raft) Start(command interface{}) (int, int, bool) {
	index := -1
	term := -1
	isLeader := true
	// Wanring: do not double add lock for GetState function
	rf.mu.Lock()
	defer rf.mu.Unlock()
	term, isLeader = rf.GetState()
	if isLeader {
		index = rf.logs[rf.getLen()].Index + 1
		//oldLogLen := len(rf.logs)
		rf.logs = append(rf.logs, Log{command, term, index})
		//DPrintf("server %d is Leader. new log has been appended, old length is %d, now is %d.  index number is %d, commitedindex is %d",
		//	rf.me, oldLogLen, len(rf.logs), index, rf.commitedIndex)
		if len(rf.chanNewLog) == 0 {
			rf.chanNewLog <- 1
		}
		rf.persist()
	}
	return index, term, isLeader
}
```
重点处理的地方是nextIndex和MatchIndex 的计算以及维护的问题
#### Apply的实现
因为对于外部客户端的行为来说，Start函数是一个异步的函数, 所以实际上客户端是通过等待Make函数中输入的ApplyCh来确定能够apply成功，保证对外的原子性
```
func (rf *Raft) doApply() {
	for {
		rf.mu.Lock()
		st := rf.state
		rf.mu.Unlock()
		if st == Killed {
			return
		}
		select {
		case <-rf.chanCommit:
			for {
				rf.mu.Lock()
				if !(rf.lastApplied < rf.commitedIndex && rf.lastApplied < rf.logs[rf.getLen()].Index) {
					rf.mu.Unlock()
					break
				}
				FirstIndex := rf.logs[0].Index
				if rf.lastApplied+1 >= FirstIndex {
					index := min(rf.lastApplied+1-FirstIndex, rf.getLen())
					msg := ApplyMsg{CommandValid: true, Command: rf.logs[index].Command, CommandIndex: rf.lastApplied + 1}
					rf.lastApplied++
					rf.mu.Unlock()
					//can't lock when send in channel, dead lock
					// canApplychan is to block the new
					rf.chanCanApply <- 1
					rf.chanApplyMsg <- msg
					<-rf.chanCanApply
					rf.mu.Lock()
				}
				rf.mu.Unlock()
			}
		}
	}
}

```


### PartC的实现
此处首先需要添加状态机的持久化的方法，并且需要在启动的时候添加一个读取旧的持久化状态机的方法。
```
func (rf *Raft) InstallSnapshot(args *InstallSnapshotArgs, reply *InstallSnapshotReply) {
	rf.chanCanApply <- 1
	rf.mu.Lock()
	reply.Success = false
	persistFlag := 0
	if args.Term < rf.currentTerm {
		reply.Term = rf.currentTerm
		rf.mu.Unlock()
		<-rf.chanCanApply
		return
	}
	if rf.currentTerm < args.Term {
		rf.currentTerm = args.Term
		rf.ClearChan()
		rf.voteFor = -1
		rf.state = Follower
		persistFlag = 1
	}
	reply.Term = rf.currentTerm
	// similar to appendEntries receive call
	rf.chanAppendEntries <- 1
	firstIndex := rf.logs[0].Index
	nowIndex := args.LastIncludeIndex - firstIndex
	if nowIndex < 0 {
		if persistFlag == 1 {
			rf.persist()
		}
		reply.PrevIndex = firstIndex
		rf.mu.Unlock()
		<-rf.chanCanApply
		return
	}
	rf.logs = args.Logs
	rf.lastApplied = args.LastIncludeIndex
	rf.commitedIndex = args.LeaderCommitIndex
	rf.persister.SaveStateAndSnapshot(rf.getPersistByte(), args.Snapshot)
	msg := ApplyMsg{CommandValid: false, Snapshot: args.Snapshot}
	rf.mu.Unlock()
	rf.chanApplyMsg <- msg
	<-rf.chanCanApply
	reply.Success = true

}

```

```
// 启动的时候添加这个方法把Snapshotload出来
rf.readPersist(persister.ReadRaftState())
```

并且需要注意在投票部分需要根据Safty那里提及的一个重要的地方，如果RequestVote RPC里面的日志Term与目前的Term不是一致的情况下，不能通过数票数的方法来进行计算。这个是为了整个集群的正确来考虑的。
```
func (rf *Raft) updateCommit() {
	N := rf.commitedIndex
	FirstIndex := rf.logs[0].Index
	for i := max(rf.commitedIndex+1, FirstIndex+1); i <= rf.logs[rf.getLen()].Index; i++ {
		num := 1
		for j := range rf.peers {
			if j != rf.me {
				if rf.matchIndex[j] >= i {
					/*
						this part is paper 5.4.2 limit, only can count replicate when log's term
						is equals to currentTerm
					*/
					if rf.logs[i-FirstIndex].Term == rf.currentTerm {
						num++
					}
				}
			}
		}
		if num > len(rf.peers)/2 {
			N = i
		}
	}
	if N > rf.commitedIndex && rf.state == Leader {
		rf.commitedIndex = min(N, rf.logs[rf.getLen()].Index)
		rf.chanCommit <- 1
	}
}
```