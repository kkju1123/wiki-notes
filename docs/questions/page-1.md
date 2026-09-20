---
title: 面试题 · 第 1 页
url: wikibar://questions/page-1
source_type: questions
folder: questions
count: 2
fetched_at: '2026-09-20T01:15:43.206884+00:00'
---

# 面试题 · 第 1 页

## Q1: 什么是 TCP 三次握手？

三次握手是 TCP 建立可靠连接的过程：
1. 客户端发送 SYN=1, seq=x，进入 SYN_SENT；
2. 服务端回复 SYN=1, ACK=1, seq=y, ack=x+1，进入 SYN_RCVD；
3. 客户端发送 ACK=1, seq=x+1, ack=y+1，双方进入 ESTABLISHED。
核心作用：同步双方初始序列号、确认双方收发能力、防止旧重复连接被错误建立。
常见追问：为什么不是两次？两次无法确认客户端接收能力，且旧 SYN 可能让服务端单方面建立连接。为什么不是四次？四次多余，三次已经完成双向确认。


## Q2: 讲讲 Python 的 GIL 是什么，有什么影响？

GIL 是 CPython 解释器的全局解释器锁，保证同一时刻只有一个线程执行 Python 字节码。
影响：CPU 密集型多线程无法利用多核，甚至因线程切换更慢；IO 密集型多线程仍有效，因为等待 IO 时会释放 GIL。
存在原因：CPython 内存管理和对象引用计数需要线程安全，GIL 实现简单且兼容大量 C 扩展。
应对方案：CPU 密集型用 multiprocessing 多进程；IO 密集型用线程或 asyncio；也可用 C 扩展在耗时计算中主动释放 GIL，或考虑其他解释器/自由线程版本。
