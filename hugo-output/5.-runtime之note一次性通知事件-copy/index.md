# 5. runtime之note一次性通知事件


对一次性事件进行睡眠和唤醒。在调用 `notesleep` 或者 `notewakeup` 之前，必须调用 `noteclear` 来初始化 `note` 。然后，一个线程可以调用 `notesleep` ，一个线程可以调用 `notewakeup` 一次，一点调用了 `notewakeup` ， `notesleep` 就会返回。后续调用 `notesleep` 将立即返回。之后的 `noteclear` 必须在之前的 `notesleep` 返回后调用。比如，不允许在 `notewakeup` 之后直接调用 `noteclear`。

`notetsleep` 和 `notesleep` 类似，但即使事件尚未发生，也会在给定的ns后唤醒。如果一个 `goroutine` 使用 `notetsleep` 提前唤醒，它必须等待调用 `noteclear` ，直到可以确定没有其他 `goroutine` 在调用 `notewakeup`。

`notesleep` 和 `notetsleep` 通常被 `g0` 调用。

`notetsleepg` 与 `notetsleep` 相似，但被用户 `g` 调用。

这是一家只能打包带走餐厅。

小明来买午饭，需要通过 `noteclear` 拿走一个叫号牌，通过 `notesleep` 告诉老板钱付过了，这个牌子的号码绑定的就是我的午餐。老板把饭做好了要叫号 `notewakeup` ，我们听到叫号，就去拿餐，阻塞就结束了。老板只会叫一次号，可能我们没听到叫号，但是我们过去柜台看的话，餐就在那里放着，上面也写着是多少号的餐。也就是 `notewakeup` 后，继续调用 `notesleep` 不会阻塞，立即返回。

今天比较着急，我只愿意等10分钟，超时小明上课就迟到了，奖学金就没有了，所以小明通过 `notetsleep` 告诉老板我很急，只能等10分钟，小明在10分钟拿到餐就是正常情况，但可能老板没做出来，小明就不等了，老板做出来之后，看了看号码牌，好像没人要这个餐了。小明课上完如果想起来自己还有个餐没拿，过来也是能直接拿到的。

## 结构

基于futex实现，key是一个uint32的值

基于sema实现，key是一个waitm

```go
type note struct {
	key uintptr
}
```

## 基于sema实现

### 初始化一次性通知或者清零一次性通知

```go
// 就是给key设置成了0
func noteclear(n *note) {
	if GOOS == "aix" {
		atomic.Storeuintptr(&n.key, 0)
	} else {
		n.key = 0
	}
}
```

### 阻塞等待通知

```go
func notesleep(n *note) {
    // 得到g
	gp := getg()
    // 如果 g 不是 g0，我们抛出错误
	if gp != gp.m.g0 {
		throw("notesleep not on g0")
	}
    // 初始化信号
	semacreate(gp.m)
    // 如果 key 是 0 ，证明这是一个新初始化的 note
    // 我们让他存储当前 m 的地址
	if !atomic.Casuintptr(&n.key, 0, uintptr(unsafe.Pointer(gp.m))) {
        // 如果存储失败，证明 n.key 不是 0。
		// 如果我们的已经调用了 notewakeup ， 导致 n.key 是 locked，那么我们就返回
        // 如果 n.key 不是 locked 也不是锁定状态，我们这边还要阻塞的话，就报错。
		if n.key != locked {
			throw("notesleep - waitm out of sync")
		}
		return
	}
	// n.key 存储了 m 的地址，成功了，我们要让他等待
	gp.m.blocked = true
	if *cgo_yield == nil {
        // 无限等待通知
		semasleep(-1)
	} else {
		// 休眠一个任意但适中的间隔来轮询 libc 拦截器。
		const ns = 10e6
        // 如果这个时候 n.key 是 0 的话，证明被 wakeup 了
		for atomic.Loaduintptr(&n.key) == 0 {
			semasleep(ns)
			asmcgocall(*cgo_yield, nil)
		}
	}
	gp.m.blocked = false
}

```

#### 流程如下

1. 现在 `g0` 要阻塞直到得到一个通知。
2. 检查 `note.key != 0` ，证明这个 `note` 已经被 `wakeup(==locked)` 或者 当前正在 `sleep(==waitm)` 中。
3. 继续判断如果 `note.key != locked` ，也就是已经 `sleep` ，我们就报错，因为已经这个 `note` 上面正在睡眠等待，如果 `note.key == locked` 证明已经被唤醒，我们就直接返回，因为我们只会唤醒一次，后续调用 `sleep` 将不会进行阻塞
4. 如果 `note.key == 0` ，我们就将 `m` 的地址存入 `note.key` 中。开启信号量等待就好了

### 唤醒通知

```go
func notewakeup(n *note) {
	var v uintptr
	for {
        // v 是我们 n.key 的值
		v = atomic.Loaduintptr(&n.key)
        // 如果 v 没有被其他线程修改，我们就让它的状态为locked
		if atomic.Casuintptr(&n.key, v, locked) {
			break
		}
	}

	// 这里的 v 还是之前 key 的值
	switch {
	case v == 0:
        // 如果之前 v == 0 证明之前就是一个解锁的状态，并且没有存储waitm，我们直接结束就好了，加锁已经完成
	case v == locked:
		// 如果之前就是锁定的，我们就报错
		throw("notewakeup - double wakeup")
	default:
		// v 里面存储的是一个 waitm
		semawakeup((*m)(unsafe.Pointer(v)))
	}
}
```

### 带最大唤醒等待时间的等待

```go
func notetsleep(n *note, ns int64) bool {
	gp := getg()
	if gp != gp.m.g0 {
		throw("notetsleep not on g0")
	}
	semacreate(gp.m)
	return notetsleep_internal(n, ns, nil, 0)
}

func notetsleep_internal(n *note, ns int64, gp *g, deadline int64) bool {
	// gp 和 deadline 按逻辑来说是局部变量，但是为了存在调用者的栈中，我们写成了参数
    // 这减少了 notetsleep_internal 的 nosplit 占用空间。
    // 拿到 g ， 这个 g 是 g0
	gp = getg()

	// 如果 n.key != 0
	if !atomic.Casuintptr(&n.key, 0, uintptr(unsafe.Pointer(gp.m))) {
		if n.key != locked {
            // 对已经阻塞的 note ，继续阻塞
			throw("notetsleep - waitm out of sync")
		}
        // 已经被唤醒的note
		return true
	}
    // 如果 n.key == 0，n.key = uintptr(unsafe.Pointer(gp.m))
    // 当前 key 是要阻塞 m 的地址
	if ns < 0 {
		// 小于0，就是无限等待，跟上面逻辑一样
		gp.m.blocked = true
		if *cgo_yield == nil {
			semasleep(-1)
		} else {
			// Sleep in arbitrary-but-moderate intervals to poll libc interceptors.
			const ns = 10e6
			for semasleep(ns) < 0 {
				asmcgocall(*cgo_yield, nil)
			}
		}
		gp.m.blocked = false
		return true
	}
	// 当 ns >= 0，证明要有一个超时唤醒时间
	deadline = nanotime() + ns
	for {
		gp.m.blocked = true
		if *cgo_yield != nil && ns > 10e6 {
			ns = 10e6
		}
        // semasleep(ns) >= 0 代表着在规定时间内正常被唤醒，我们返回true
		if semasleep(ns) >= 0 {
			gp.m.blocked = false
			// Acquired semaphore, semawakeup unregistered us.
			// Done.
			return true
		}
        // 小小休息一下，防止空耗cpu
		if *cgo_yield != nil {
			asmcgocall(*cgo_yield, nil)
		}
		gp.m.blocked = false
		// Interrupted or timed out. Still registered. Semaphore not acquired.
		ns = deadline - nanotime()
        // 等待超时了
		if ns <= 0 {
			break
		}
		// Deadline hasn't arrived. Keep sleeping.
	}

	// 我们超时拿不到信号量通知，要放弃了
	for {
        // v 就是 waitm
		v := atomic.Loaduintptr(&n.key)
		switch v {
		case uintptr(unsafe.Pointer(gp.m)):
			// 取消注册，返回，清空这个key，如果别人再来唤醒，也根本啥事不做了
			if atomic.Casuintptr(&n.key, v, 0) {
				return false
			}
		case locked:
			// 其他线程把note唤醒
			gp.m.blocked = true
			if semasleep(-1) < 0 {
                // 无限等待，如果中断，抛出异常
				throw("runtime: unable to acquire - semaphore out of sync")
			}
			gp.m.blocked = false
            // 唤醒了
			return true
		default:
            // 意外的情况
			throw("runtime: unexpected waitm - semaphore out of sync")
		}
	}
}
```

### 用户 g 调用的等待

```go
func notetsleepg(n *note, ns int64) bool {
	gp := getg()
	if gp == gp.m.g0 {
		throw("notetsleepg on g0")
	}
	semacreate(gp.m)
	entersyscallblock()
	ok := notetsleep_internal(n, ns, nil, 0)
	exitsyscall()
	return ok
}
```


