# 2. runtime之notifyList条件唤醒


假如我们去餐馆吃饭，下单后会给你一个编号，然后你等着叫号拿菜就好了。

- 我们去下单了，把我们就记录到系统里面了`wait+1`
- 叫号`Signal`，他会按照`notify`序号来叫号，如果叫到你了，你就可以来拿餐了，然后它还会把`notify`序号加一
- 全部都做好了`Broadcast`，把所有人都叫过来拿餐，把`notify`序号更新成最大的出单号
- `head`就是指向我们排队的第一个人`(sudog)`
- `tail`就是我们队伍的最后一个人`(sudog)`
- 排队是通过`sudog.next`进行连接的

## 结构

```go
type notifyList struct {
   wait uint32
   notify uint32
   lock mutex
   head *sudog
   tail *sudog
}
```

- `wait` 当前出单时的序号
- `notify` 当前正在出餐的序号，餐出好后，会按照这个序号叫号
- `lock` 锁
- `head` 当前排队等餐人的第一个
- `tail` 当前排队等餐人的最后一个

## 获取出餐号

```go
func notifyListAdd(l *notifyList) uint32 {
   // 给wait加上1，返回原来wait的序号，wait加完1之后就是下一次点单要出的号
   return atomic.Xadd(&l.wait, 1) - 1
}
```

## 人去排队

```go
func notifyListWait(l *notifyList, t uint32) {
   // 加锁
   lockWithRank(&l.lock, lockRankNotifyList)

   // 我们拿着单号去排队，先看一下我们的单号是不是叫号系统当前应该叫的号小，如果小的话，肯定叫不到我们，我们走就好了
   if less(t, l.notify) {
      unlock(&l.lock)
      return
   }

   // 新建一个sudog
   s := acquireSudog()
   // 获取当前要阻塞的g
   s.g = getg()
   // 把我们的t放到sudog
   s.ticket = t

   // 如果队伍没有尾巴，那我们就是一个
   if l.tail == nil {
      // 没有尾巴，也就是没人排队，我就来当头
      l.head = s
   } else {
      l.tail.next = s
   }
   // 每次排队肯定会设置尾巴的
   l.tail = s
   // 排完队，我们就等着吧，可以站在原地休息一会
   goparkunlock(&l.lock, waitReasonSyncCondWait, traceEvGoBlockCond, 3)
   
   // 执行到这里，就证明，叫到我们了
   if t0 != 0 {
      blockevent(s.releasetime-t0, 2)
   }
   // 释放sudog
   releaseSudog(s)
}
```

## 叫号，出了一个餐了

```go
func notifyListNotifyOne(l *notifyList) {
   // 判断我们当前要出单的号码，和我们轮到叫的号码是不是想等的，如果想等，那就没人排队，就没有单子在做
   if atomic.Load(&l.wait) == atomic.Load(&l.notify) {
      return
   }
   // 加锁
   lockWithRank(&l.lock, lockRankNotifyList)

   // t就有我们需要通知到的序号
   t := l.notify
   // 如果t是我们将要出单的序号的话，那么肯定就没有g在等待我们出餐
   if t == atomic.Load(&l.wait) {
      unlock(&l.lock)
      return
   }

   // 更新一下notify的序号
   atomic.Store(&l.notify, t+1)
   // 在队伍里面找到第一个人，然后给他通知说可以了
   // 并且，让队伍往前走一步
   for p, s := (*sudog)(nil), l.head; s != nil; p, s = s, s.next {
      if s.ticket == t {
         // 找到我们要通知的sudog了
         n := s.next
         if p != nil {
            p.next = n
         } else {
            l.head = n
         }
         if n == nil {
            l.tail = p
         }
         unlock(&l.lock)
         s.next = nil
         // 通知餐好了，你可以来拿餐了
         readyWithTime(s, 4)
         return
      }
   }
   unlock(&l.lock)
}
```

## 唤醒所有

```go
func notifyListNotifyAll(l *notifyList) {
   if atomic.Load(&l.wait) == atomic.Load(&l.notify) {
      return
   }

   lockWithRank(&l.lock, lockRankNotifyList)
   // 把队伍记载了临时的s里面
   s := l.head
   // 把队伍清空了
   l.head = nil
   l.tail = nil

   // 更新一下叫号
   atomic.Store(&l.notify, atomic.Load(&l.wait))
   unlock(&l.lock)

   // 把队伍里面所有人都通知一遍，菜好了
   for s != nil {
      next := s.next
      s.next = nil
      readyWithTime(s, 4)
      s = next
   }
}
```
