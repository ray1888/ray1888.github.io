---
layout: post
title: Ray 的算法探索-动态规划(TOFinish )
author: Ray Chan(ray1888)
date: '2020-11-23 22:19:38 +0800'
category: Algorithm
summary: ray's algorithm advanture- dynamic programming
thumbnail: Prometheus.png
---


# 算法-深入动态规划


动态规划的核心问题：把一个大问题分解成小的子问题，并且复用其小问题的解来构成更高一级问题的解


动态规划也可以理解为子问题依赖的DAG的拓扑排序。（Mit 算法导论中提到）,子问题的解不能成环。    

做动态规划的核心：猜！（我全都试）



DP ~= Recursion+ Memorize + guess (找出状态转移的方程)
DP ~= careful brute force
dp ~= shorest path in some DAG



## 5步帮你去解DP问题
1. 定义子问题(define subproblems) --(需要关注的点： 计算子问题的数量）
2. 猜(Guess part of solution)   --(需要关注的点： number of choices for guess ）
3. 关联子问题(Related subproblem solutions) -- (需要关注的点：time per subproblems）
4. 解出子问题(recurse & memoize || building table bottle up) -- (需要关注的点：子问题递归是否为不成环）
5. 解出原问题   -- (需要关注的点：extra time ）



##　ＴＯＤＯ　补充题目是如何应用上面的解法