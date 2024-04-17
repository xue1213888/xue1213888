# 3. runtime之lockRank锁排名机制


在`runtime.mutex`中内嵌了一个叫`lockRankStruct`的结构体，看名字我们就知道是做锁排名的。`mutex`我们在`runtime系列`的`第四篇`文章讲到。

它其实很简单，一开始我们的锁，只能有一个人持有，等他释放了，其他人才能来获取锁。

现在有了这个锁排名机制，当我们要来获取锁的时候，我们首先看这个锁有没有被持有，最后来的一个持有这个锁的人的等级是不是比我们低，如果比我们低的话，我们要去查一下我们的表，看看当他持有锁的时候，我们能不能继续获取锁。如果能，我们就能直接获取锁。

这个静态排名检查功能，我们要在`GOEXPERIMENT`开启`staticklockranking`才有效

## 排名结构

```go
// 锁的排名本质就是一个数字
type lockRank int

// 包装了一个排名结构，里面的pad是用来填充，让lockRankStruct是8字节的
type lockRankStruct struct {
	rank lockRank
	pad int
}

// 这就是我们查询是否能同时持有锁的一个表
// 一维数组表示不同的锁，里面存着它们的等级，刚好它们的下标也是它们的等级
// 二维数组就是当我们要持有锁的时候查询表来看看前一个是不是在自己的合法锁里面，如果不在，我们也不能同时持有这两个锁
var lockPartialOrder [][]lockRank = [][]lockRank{
	lockRankDummy:         {},
	lockRankSysmon:        {},
	lockRankScavenge:      {lockRankSysmon},
	lockRankForcegc:       {lockRankSysmon},
	lockRankSweepWaiters:  {},
	lockRankAssistQueue:   {},
	lockRankCpuprof:       {},
	lockRankSweep:         {},
	lockRankPollDesc:      {},
	lockRankSched:         {lockRankSysmon, lockRankScavenge, lockRankForcegc, lockRankSweepWaiters, lockRankAssistQueue, lockRankCpuprof, lockRankSweep, lockRankPollDesc},
	lockRankDeadlock:      {lockRankDeadlock},
	lockRankAllg:          {lockRankSysmon, lockRankSched},
	lockRankAllp:          {lockRankSysmon, lockRankSched},
	lockRankTimers:        {lockRankSysmon, lockRankScavenge, lockRankPollDesc, lockRankSched, lockRankAllp, lockRankTimers},
	lockRankItab:          {},
	lockRankReflectOffs:   {lockRankItab},
	lockRankHchan:         {lockRankScavenge, lockRankSweep, lockRankHchan},
	lockRankTraceBuf:      {lockRankSysmon, lockRankScavenge},
	lockRankFin:           {lockRankSysmon, lockRankScavenge, lockRankSched, lockRankAllg, lockRankTimers, lockRankReflectOffs, lockRankHchan, lockRankTraceBuf},
	lockRankNotifyList:    {},
	lockRankTraceStrings:  {lockRankTraceBuf},
	lockRankMspanSpecial:  {lockRankSysmon, lockRankScavenge, lockRankAssistQueue, lockRankCpuprof, lockRankSweep, lockRankSched, lockRankAllg, lockRankAllp, lockRankTimers, lockRankItab, lockRankReflectOffs, lockRankHchan, lockRankTraceBuf, lockRankNotifyList, lockRankTraceStrings},
	lockRankProf:          {lockRankSysmon, lockRankScavenge, lockRankAssistQueue, lockRankCpuprof, lockRankSweep, lockRankSched, lockRankAllg, lockRankAllp, lockRankTimers, lockRankItab, lockRankReflectOffs, lockRankHchan, lockRankTraceBuf, lockRankNotifyList, lockRankTraceStrings},
	lockRankGcBitsArenas:  {lockRankSysmon, lockRankScavenge, lockRankAssistQueue, lockRankCpuprof, lockRankSched, lockRankAllg, lockRankTimers, lockRankItab, lockRankReflectOffs, lockRankHchan, lockRankTraceBuf, lockRankNotifyList, lockRankTraceStrings},
	lockRankRoot:          {},
	lockRankTrace:         {lockRankSysmon, lockRankScavenge, lockRankForcegc, lockRankAssistQueue, lockRankSweep, lockRankSched, lockRankHchan, lockRankTraceBuf, lockRankTraceStrings, lockRankRoot},
	lockRankTraceStackTab: {lockRankScavenge, lockRankForcegc, lockRankSweepWaiters, lockRankAssistQueue, lockRankSweep, lockRankSched, lockRankAllg, lockRankTimers, lockRankHchan, lockRankTraceBuf, lockRankFin, lockRankNotifyList, lockRankTraceStrings, lockRankRoot, lockRankTrace},
	lockRankNetpollInit:   {lockRankTimers},

	lockRankRwmutexW: {},
	lockRankRwmutexR: {lockRankSysmon, lockRankRwmutexW},

	lockRankSpanSetSpine:  {lockRankSysmon, lockRankScavenge, lockRankForcegc, lockRankAssistQueue, lockRankCpuprof, lockRankSweep, lockRankPollDesc, lockRankSched, lockRankAllg, lockRankAllp, lockRankTimers, lockRankItab, lockRankReflectOffs, lockRankHchan, lockRankTraceBuf, lockRankNotifyList, lockRankTraceStrings},
	lockRankGscan:         {lockRankSysmon, lockRankScavenge, lockRankForcegc, lockRankSweepWaiters, lockRankAssistQueue, lockRankCpuprof, lockRankSweep, lockRankPollDesc, lockRankSched, lockRankTimers, lockRankItab, lockRankReflectOffs, lockRankHchan, lockRankTraceBuf, lockRankFin, lockRankNotifyList, lockRankTraceStrings, lockRankProf, lockRankGcBitsArenas, lockRankRoot, lockRankTrace, lockRankTraceStackTab, lockRankNetpollInit, lockRankSpanSetSpine},
	lockRankStackpool:     {lockRankSysmon, lockRankScavenge, lockRankSweepWaiters, lockRankAssistQueue, lockRankCpuprof, lockRankSweep, lockRankPollDesc, lockRankSched, lockRankTimers, lockRankItab, lockRankReflectOffs, lockRankHchan, lockRankTraceBuf, lockRankFin, lockRankNotifyList, lockRankTraceStrings, lockRankProf, lockRankGcBitsArenas, lockRankRoot, lockRankTrace, lockRankTraceStackTab, lockRankNetpollInit, lockRankRwmutexR, lockRankSpanSetSpine, lockRankGscan},
	lockRankStackLarge:    {lockRankSysmon, lockRankAssistQueue, lockRankSched, lockRankItab, lockRankHchan, lockRankProf, lockRankGcBitsArenas, lockRankRoot, lockRankSpanSetSpine, lockRankGscan},
	lockRankDefer:         {},
	lockRankSudog:         {lockRankHchan, lockRankNotifyList},
	lockRankWbufSpans:     {lockRankSysmon, lockRankScavenge, lockRankSweepWaiters, lockRankAssistQueue, lockRankSweep, lockRankPollDesc, lockRankSched, lockRankAllg, lockRankTimers, lockRankItab, lockRankReflectOffs, lockRankHchan, lockRankFin, lockRankNotifyList, lockRankTraceStrings, lockRankMspanSpecial, lockRankProf, lockRankRoot, lockRankGscan, lockRankDefer, lockRankSudog},
	lockRankMheap:         {lockRankSysmon, lockRankScavenge, lockRankSweepWaiters, lockRankAssistQueue, lockRankCpuprof, lockRankSweep, lockRankPollDesc, lockRankSched, lockRankAllg, lockRankAllp, lockRankTimers, lockRankItab, lockRankReflectOffs, lockRankHchan, lockRankTraceBuf, lockRankFin, lockRankNotifyList, lockRankTraceStrings, lockRankMspanSpecial, lockRankProf, lockRankGcBitsArenas, lockRankRoot, lockRankSpanSetSpine, lockRankGscan, lockRankStackpool, lockRankStackLarge, lockRankDefer, lockRankSudog, lockRankWbufSpans},
	lockRankMheapSpecial:  {lockRankSysmon, lockRankScavenge, lockRankAssistQueue, lockRankCpuprof, lockRankSweep, lockRankPollDesc, lockRankSched, lockRankAllg, lockRankAllp, lockRankTimers, lockRankItab, lockRankReflectOffs, lockRankHchan, lockRankTraceBuf, lockRankNotifyList, lockRankTraceStrings},
	lockRankGlobalAlloc:   {lockRankProf, lockRankSpanSetSpine, lockRankMheap, lockRankMheapSpecial},
	lockRankPageAllocScav: {lockRankMheap},

	lockRankGFree:     {lockRankSched},
	lockRankHchanLeaf: {lockRankGscan, lockRankHchanLeaf},
	lockRankPanic:     {lockRankDeadlock}, // plus any other lock held on throw.

	lockRankNewmHandoff:   {},
	lockRankDebugPtrmask:  {},
	lockRankFaketimeState: {},
	lockRankTicks:         {},
	lockRankRaceFini:      {},
	lockRankPollCache:     {},
	lockRankDebug:         {},
}

```

## 锁排名的初始化

```go
// 其实就是给mutex上加上了这个排名数字
func lockInit(l *mutex, rank lockRank) {
	l.rank = rank
}
```

## 直接获取锁上面的排名

```go
func getLockRank(l *mutex) lockRank {
	return l.rank
}
```

## 带排名加锁

就是检查这个排名的锁能不能合法的lock，如果可以就lock

```go
// 第二个参数就是我们锁排名
func lockWithRank(l *mutex, rank lockRank) {
   if l == &debuglock || l == &paniclk {
      // 这俩东西不要做排名
      // debuglock是调试输出我们当前m上面持有锁和排名信息的
      // paniclk同理也差不多，是异常错误后执行的
      // lock2和unlock2都是和mutex有关的，第四篇文章讲到
      lock2(l)
      return
   }
   // 如果rank == 0，我们的rank就是一个数值最大的锁，它只会判断前一个锁是不是lockRankLeafRank来决定能不能获取锁
   if rank == 0 {
      rank = lockRankLeafRank
   }
   // 获取我们执行的g
   gp := getg()
   // 切换系统栈，也就是g0的栈中执行下面
   systemstack(func() {
      // i是我们gp上面持有的锁信息长度
      i := gp.m.locksHeldLen
      // 如果我们locksHeldLen显示的长度>=10，那么就报错
      // m 持有最多 10 个锁，由锁排名代码维护。
      if i >= len(gp.m.locksHeld) {
         throw("too many locks held concurrently for rank checking")
      }
      // 把这个锁信息（排名和锁地址）记录到m上
      gp.m.locksHeld[i].rank = rank
      gp.m.locksHeld[i].lockAddr = uintptr(unsafe.Pointer(l))
      gp.m.locksHeldLen++
		
      // i==0的话，证明是第一个锁，直接放上去就好了，不用检查的
      // i > 0的话，证明不是第一个锁，我们就要判断我们新锁能不能加入进来
      if i > 0 {
         checkRanks(gp, gp.m.locksHeld[i-1].rank, rank)
      }
      // 如果能加入进来，我们就上锁了
      lock2(l)
   })
}
```

相同的还有一个函数`func acquireLockRank(rank lockRank)`他只是判断排名是否合法，然后把排名信息挂载到m上去，不进行锁操作，因为它也没给传一个锁，内部逻辑其实是一样的

## 真正的锁排名检查机制

```go
func checkRanks(gp *g, prevRank, rank lockRank) {
   // 一开始设置我们的rankOK为false
   rankOK := false
   // prevRank是前一个锁的等级，rank是我们现在要来的锁等级
   if rank < prevRank {
      // 我们新来的锁比前一个锁的等级低，我们不能上锁
      rankOK = false
   } else if rank == lockRankLeafRank {
      // 我们新来的锁，是一个最大锁，我们就判断前一个不是最大锁就行了
      rankOK = prevRank < lockRankLeafRank
   } else {
      // 来到这块证明我们的新来的锁比前一个锁等级要高，我们得查表看看他们能不能一起使用
      // lockPartialOrder是一个二维数组，我们在上面看到过
      list := lockPartialOrder[rank]
      for _, entry := range list {
         // 判断我们的前一个锁，有没有出现在自己的合法列表里面，如果出现了那么就是true
         if entry == prevRank {
            rankOK = true
            break
         }
      }
   }
   // 如果没出现，就打印调试信息，报错就好了
   if !rankOK {
      printlock()
      println(gp.m.procid, " ======")
      printHeldLocks(gp)
      throw("lock ordering problem")
   }
}
```

## 释放锁以及排名信息

如果我们开启`staticklockranking`功能后，我们的解锁需要把m上面挂在的锁信息也给删一下

```go
func unlockWithRank(l *mutex) {
   if l == &debuglock || l == &paniclk {
      // 与lockWithRank同理
      unlock2(l)
      return
   }
   // 下面就是简单的摘掉一个锁的过程
   gp := getg()
   systemstack(func() {
      found := false
      for i := gp.m.locksHeldLen - 1; i >= 0; i-- {
         if gp.m.locksHeld[i].lockAddr == uintptr(unsafe.Pointer(l)) {
            // 我们要的锁找到了
            found = true
            copy(gp.m.locksHeld[i:gp.m.locksHeldLen-1], gp.m.locksHeld[i+1:gp.m.locksHeldLen])
            gp.m.locksHeldLen--
            break
         }
      }
      // 没找到的话，就调试，还有报错
      if !found {
         println(gp.m.procid, ":", l.rank.String(), l.rank, l)
         throw("unlock without matching lock acquire")
      }
      // 解锁
      unlock2(l)
   })
}
```

相同的还有一个函数`func releaseLockRank(rank lockRank)`他只是释放m上面绑定的一个排名信息，锁地址是0的一个玩意。与`acquireLockRank`对照

## 判断排名合不合法

不做任何操作也不往m上挂载锁信息

```go
// lockWithRankMayAcquire 判断锁排名是否合法，不合法就报错
func lockWithRankMayAcquire(l *mutex, rank lockRank) {
   // 拿到g
   gp := getg()
   // 判断有没有锁
   if gp.m.locksHeldLen == 0 {
      // 没有锁，不可能出现锁排序出错问题
      return
   }
   // 切换系统栈
   systemstack(func() {
      i := gp.m.locksHeldLen
      if i >= len(gp.m.locksHeld) {
         throw("too many locks held concurrently for rank checking")
      }
      // 这边就是临时挂载上去，然后做了一个判断
      gp.m.locksHeld[i].rank = rank
      gp.m.locksHeld[i].lockAddr = uintptr(unsafe.Pointer(l))
      gp.m.locksHeldLen++
      checkRanks(gp, gp.m.locksHeld[i-1].rank, rank)
      gp.m.locksHeldLen--
   })
}
```

## 判断g是否持有锁l

```go
// 判断m有没有持有锁l
func checkLockHeld(gp *g, l *mutex) bool {
	for i := gp.m.locksHeldLen - 1; i >= 0; i-- {
		if gp.m.locksHeld[i].lockAddr == uintptr(unsafe.Pointer(l)) {
			return true
		}
	}
	return false
}
```

`func assertLockHeld(l *mutex)`省略了一个g参数，这个里面是获取当前执行的g的m

`func assertRankHeld(r lockRank)`这个和上面的逻辑差不多，你们下去自己看，就是判断m是否持有这个排名

