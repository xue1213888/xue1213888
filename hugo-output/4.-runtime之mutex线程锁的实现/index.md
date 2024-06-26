# 4. runtime之mutex线程锁的实现


`sync.Mutex` 和 `sync.Cond` 都是对于协程 `g` 来说的，而 `m.mOS.mutex` 和 `m.mOS.cond` 是对线程 `m` 来说的，底层用法大概相同。

`runtime.mutex` 在 `sema` 的实现方案下（不同系统决定着实现方案的不同），底层依赖 `m.mOS.mutex` 和 `m.mOS.cond` 来加解锁操控 `m` ，这时候 `mutex` 中的 `key` 是 `waitm` ，一个等待的 `m` 的地址。

`runtime.mutex` 在 `futex` 的实现方案下，`key` 存储着锁的状态

## mutex基于信号sema的实现

### 信号的实现

- `func semacreate(mp *m)` 为 `m` 创建信号量，如果它还没有的话。
- `func semasleep(ns int64) int32` 最长阻塞ns纳秒来得到一个信号量，成功返回0，失败返回-1
- `func semawakeup(mp *m)` 激活一个信号量

我们使用 `semacreate` 创建信号量之后，我们的信号量是空的，我们使用 `semawakeup` 信号量才会得到一个信号，我们使用 `semasleep` 是为了得到一个信号量，这个信号量是 `semawakeup` 发送过来的，如果在我们规定的时间拿不到信号量，或者程序中断，都会返回-1。

#### 持有结构

```go
// initialized 是否被初始化
// mutex 互斥锁
// cond 条件通知
// count 当前可直接获取的信号量数量
type mOS struct {
	initialized bool
	mutex       pthreadmutex
	cond        pthreadcond
	count       int
}
```

`semasleep` 和 `semawakeup` 本质都是操作 count` `的

`semasleep` 是 `count - 1`，如果 `count == 0`，就要等一个 `semawakeup` 给它加1

`semawakeup` 是 `count + 1`，如果 +1 后的 `count` 大于 0， 我们就要唤醒一个正在`sleep`

#### semacreate

```go
// 给m.mOS的mutex和cond做初始化，如果已经被初始化了就算了
// 这里的mutex就是一个互斥体，防止多个线程并发访问修改数据导致出错的
func semacreate(mp *m) {
    // 如果已经被初始化，就不要再初始化了
	if mp.initialized {
		return
	}
	mp.initialized = true
   	// 初始化锁，互斥体
	if err := pthread_mutex_init(&mp.mutex, nil); err != 0 {
		throw("pthread_mutex_init")
	}
    // 初始化条件通知
	if err := pthread_cond_init(&mp.cond, nil); err != 0 {
		throw("pthread_cond_init")
	}
}
```

#### semasleep

```go
// 阻塞来获取信号
// ns如果小于0，它会一直阻塞等到能够获取信号为止
// ns如果大于等于0，它会最大阻塞ns这么长的时间来获取信号
// 成功返回0，失败返回其他
func semasleep(ns int64) int32 {
	var start int64
    // 如果我们有最大耗时，我们就要先记录一下我们调用时候的时间，方便后面算花费了多长时间
	if ns >= 0 {
		start = nanotime()
	}
    // 得到m
	mp := getg().m
    // 上锁
	pthread_mutex_lock(&mp.mutex)
	for {
        // mp.count只有在唤醒一个信号的时候才会增加
		if mp.count > 0 {
            // 如果里面得到一个信号，我们就让它--
			mp.count--
            // 解锁就好了
			pthread_mutex_unlock(&mp.mutex)
			return 0
		}
        // ns >= 0 说明我们有一个耗时检测
		if ns >= 0 {
            // 判断一下我们当前花费的时间是不是超过我们的耗时了
			spent := nanotime() - start
			if spent >= ns {
                // 超过我们就解锁并返回-1
				pthread_mutex_unlock(&mp.mutex)
				return -1
			}
            // t是一个时间量
			var t timespec
            // 计算剩余t的值
			t.setNsec(ns - spent)
            // 我们开始等待一个条件通知，让我们能继续执行，最长阻塞时间为t，这里面传入mutex也是为了上锁，里面好修改状态
			err := pthread_cond_timedwait_relative_np(&mp.cond, &mp.mutex, &t)
            // 如果超时，我们就返回-1
			if err == _ETIMEDOUT {
				pthread_mutex_unlock(&mp.mutex)
				return -1
			}
            // 没有超时，我们就继续循环了
            // 如果没有超时，证明cond得到了一个通知，我们的m.count是会++的，在上面会正常结束返回0
		} else {
            // 如果我们让无限等待，且当前没有得到一个信号，我们就直接等通知就好了
			pthread_cond_wait(&mp.cond, &mp.mutex)
            // 拿到通知后，再循环到上面count肯定也会大于0，就返回0就好了
		}
	}
}
```

#### semawakeup

```go
// 唤醒一个信号，为我们的count++，并使用cond_signal进行通知
func semawakeup(mp *m) {
   // 加锁，因为要操作m了
   pthread_mutex_lock(&mp.mutex)
   mp.count++
   if mp.count > 0 {
      // 如果count>0，代表持有信号量，我们就通知cond
      pthread_cond_signal(&mp.cond)
   }
   pthread_mutex_unlock(&mp.mutex)
}
```

### mutex 的加解锁操作

```go
// locked 用最低位是不是1来判断是否处于锁定状态
// active_spin 积极自旋的次数
// passive_spin 消极自旋的次数
// active_spin_cnt 用来自旋等待的
const (
	locked uintptr = 1

	active_spin     = 4
	active_spin_cnt = 30
	passive_spin    = 1
)
```

#### 获取锁 lock

```go
func lock(l *mutex) {
    // lockWithRank 在第三篇文章讲到了，就不说了
    // 判断锁排名是否合法
	lockWithRank(l, getLockRank(l))
}

// 加锁操作
func lock2(l *mutex) {
    // 得到g
	gp := getg()
    // 判断m上锁的数量是不是异常的
	if gp.m.locks < 0 {
		throw("runtime·lock: lock count")
	}
    // 给lock++
	gp.m.locks++

	// cas操作，判断l.key是不是0，如果是0，我们设置成locked，就结束了
	if atomic.Casuintptr(&l.key, 0, locked) {
		return
	}
    // 如果不是0，我们先初始化一下信号量
	semacreate(gp.m)

	// 单核处理器不自旋，多核处理器自旋
	spin := 0
	if ncpu > 1 {
        // 自旋4次
		spin = active_spin
	}
Loop:
    // 进入一个循环
	for i := 0; ; i++ {
        // 获取key的值
		v := atomic.Loaduintptr(&l.key)
        // 判断key的最低位是不是0
		if v&locked == 0 {
			// 是0的话，证明可以直接加锁
            // 我们尝试cas加锁，如果失败了，我们就让重新开始自旋
			if atomic.Casuintptr(&l.key, v, v|locked) {
				return
			}
            // 如果还是被锁定的，我们就重新自旋
			i = 0
		}
        
		if i < spin {
            // i 如果小于自旋次数，调用procyield，资源消耗小
            // 阻塞一下
			procyield(active_spin_cnt)
		} else if i < spin+passive_spin {
            // 如果i大于等于自旋次数，我们判断是不是还小于加上消极的自旋次数，就执行osyield，资源消耗大
            // 阻塞一下
			osyield()
		} else {
            // 不用自旋了，我们要给这个mutex进行入队等待了
			// l.key 挂载的是等待这个锁的 m 的链表
			// 通过 nextwaitm 来排队
			for {
                // gp.m.nextwaitm 设置成的 v中的m，也就是原来的m
                // l.key设置成现在的m
                // 那么关系就是l.key是现在的m
                // l.key的m的nextwaitm是上一个m了
                // 相当于入队到头部了
				gp.m.nextwaitm = muintptr(v &^ locked)
                // 如果v在中途没有被解锁，我们就把v设置成当前m的地址，最后一位设置成锁定中
				if atomic.Casuintptr(&l.key, v, uintptr(unsafe.Pointer(gp.m))|locked) {
					break
				}
                // 如果v被其他线程改变了，重新去一下v的新值（其他线程可能解锁了这个v）
				v = atomic.Loaduintptr(&l.key)
                // 如果v当前处于解锁状态了
				if v&locked == 0 {
                    // 继续外部循环了
					continue Loop
				}
			}
            // 如果v锁定中，我们就挂起等待
			if v&locked != 0 {
				// 等待
				semasleep(-1)
				i = 0
			}
		}
	}
}
```

##### 当一个线程需要获取一个锁的时候，他执行流程如下

1. 判断key的值等于0（也就是一个新锁），那么就直接设置为1锁定状态，并返回
2. 如果key不是0，证明里面要么是1，要么是存了一个waitm，我们就要初始化信号锁（更底层，仅会初始化一次）
3. 获取是否应该自旋和设置自旋次数
4. 进入自旋也就是循环，此时key可能是1或者是一个waitm，waitm的最低位标志着当前锁的状态，我们判断最低位等于0，我们直接设置最低位为1，返回即可
5. 如果设置最低位为1失败（其他线程抢先一步修改了），我们需要重新自旋
6. 在上一步验证结束后，如果自旋次数还未循环结束，我们让处理器执行等待，后继续循环获取锁状态
7. 如果自旋次数结束，无法获取到锁，我们就让当前m的nextwaitm指向key，也就是前面正在阻塞的m的地址，让我们的key存储当前m的地址
8. 如果存储我们的m失败，证明当前锁已经被其他线程释放了，我们就继续判断，重新进入自旋状态
9. 如果存储成功，我们就等待sema信号的通知即可

##### 为什么m要通过那种方式设置，我们举个例子，他这个算法很巧妙。

|       | 地址与计算             | 备注                           |
| :---: | ---------------------- | ------------------------------ |
|   m   | 0x1000                 | m就是我们的线程地址            |
| waitm | 0x1001 (0x1000 &^ 0x1) | waitm就是我们的nextwaitm       |
|  key  | 0x1001 (0x1000 \| 0x1) | key就是mutex里面存储的         |
| getm  | 0x1000 (0x1001 &^ 0x1) | getm就是释放锁时从key里取到的m |

#### 释放锁

```go
func unlock(l *mutex) {
	unlockWithRank(l)
}

func unlock2(l *mutex) {
    // 获取g
	gp := getg()
	var mp *m
	for {
        // 获取锁上存的key，也就是waitm以及锁定状态
		v := atomic.Loaduintptr(&l.key)
        // 如果锁定中，直接解锁
		if v == locked {
            // cas操作解锁
			if atomic.Casuintptr(&l.key, locked, 0) {
				break
			}
		} else {
            // 当前v不是单纯的锁定状态，里面存储了m信息
            // 取到m的指针
			mp = muintptr(v &^ locked).ptr()
            // 让我们的key指向下一个等待的m身上
			if atomic.Casuintptr(&l.key, v, uintptr(mp.nextwaitm)) {
                // 唤醒mp
				semawakeup(mp)
				break
			}
		}
	}
    // 解锁成功
	gp.m.locks--
	if gp.m.locks < 0 {
		throw("runtime·unlock: lock count")
	}
    // 如果m上没有锁了，就恢复抢占请求
	if gp.m.locks == 0 && gp.preempt { // restore the preemption request in case we've cleared it in newstack
		gp.stackguard0 = stackPreempt
	}
}

```

## mutex基于futex的实现

### 获取锁

```go
func lock(l *mutex) {
	lockWithRank(l, getLockRank(l))
}

func lock2(l *mutex) {
    // 拿到g
	gp := getg()
	if gp.m.locks < 0 {
		throw("runtime·lock: lock count")
	}
	gp.m.locks++

	// 把l.key设置成mutex_locked
	v := atomic.Xchg(key32(&l.key), mutex_locked)
    // 判断原来l.key是不是解锁状态，拿到锁了
	if v == mutex_unlocked {
		return
	}

    // wait 现在是 MUTEX_LOCKED 或者 MUTEX_SLEEPING状态
	wait := v

	spin := 0
	if ncpu > 1 {
		spin = active_spin
	}
	for {
		// 积极自旋，再次尝试解锁
		for i := 0; i < spin; i++ {
            // 如果其他线程把锁释放了，我们这边就能拿到了，并且把锁状态，设置成我们上面本来的状态
			for l.key == mutex_unlocked {
				if atomic.Cas(key32(&l.key), mutex_unlocked, wait) {
					return
				}
			}
			procyield(active_spin_cnt)
		}

		// 消极自旋，与上面相同，就是换成了osyield
		for i := 0; i < passive_spin; i++ {
			for l.key == mutex_unlocked {
				if atomic.Cas(key32(&l.key), mutex_unlocked, wait) {
					return
				}
			}
			osyield()
		}

		// 我们再获取一次，获取到了就ok，再挂载到futex的时候一定是sleeping状态
		v = atomic.Xchg(key32(&l.key), mutex_sleeping)
		if v == mutex_unlocked {
			return
		}
        // 来到这块，证明我们的锁获取不到了，我们要去睡眠了
		wait = mutex_sleeping
		futexsleep(key32(&l.key), mutex_sleeping, -1)
	}
}
```

### 释放锁

```go
func unlock(l *mutex) {
	unlockWithRank(l)
}

func unlock2(l *mutex) {
    // 把锁设置成mutex_unlocked
	v := atomic.Xchg(key32(&l.key), mutex_unlocked)
    // 判断原来是不是就是解锁状态
	if v == mutex_unlocked {
		throw("unlock of unlocked lock")
	}
    // 如果锁在睡眠中，我们就唤醒一个
	if v == mutex_sleeping {
		futexwakeup(key32(&l.key), 1)
	}

	gp := getg()
	gp.m.locks--
	if gp.m.locks < 0 {
		throw("runtime·unlock: lock count")
	}
	if gp.m.locks == 0 && gp.preempt { // restore the preemption request in case we've cleared it in newstack
		gp.stackguard0 = stackPreempt
	}
}
```


