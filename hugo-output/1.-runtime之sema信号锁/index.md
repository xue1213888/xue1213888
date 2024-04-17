# 1. runtime之sema信号锁


假如我们有一个宰🐷厂`sema`，里面有很多的危险工具，比如刀。

我们有好几个仓库（不同的`addr`），里面放着各种各样的刀，每个仓库它们都记录着有多少把刀`addr存储的uint32数`。

它们可能好几个仓库里的比较近，会有一个大门统一排队管理（它们会选择到同一个`seamRoot`）。

这个大门上面记录了当前有多少人在排队`nwait`，记录了排队情况`treap`

大门外面要让去不同的仓库的人排着不同的队伍，这些队伍的头部通过`parent`、`next`、`prev`组成了一个`平衡二叉树`

队伍里面所有人就是一个`链表`，通过`waitlink`进行连接，每个人的`waittail`记录着它们队伍的最后一个人

大门的`treap`上面只记录了每个队伍的第一个人

## 获取门锁时传递的uint32地址有什么用

我们在获取和归还刀的时候，都需要从一个uint32的地址上进行寻找锁，这个uint32数值代表着当前仓库里面的刀的数量，uint32的地址就是仓库地址。

## 我们通过仓库地址找大门

```go
// 我们规划了251个大门，从semtable0～250下标
const semTabSize = 251 // semtable长度

// 每个semtable，里面就是一个大门
var semtable [semTabSize]struct {
    // semaRoot就是大门
	root semaRoot
	pad  [cpu.CacheLinePadSize - unsafe.Sizeof(semaRoot{})]byte
}

// 下面就是根据我们传进来的地址找大门的一个方法
func semroot(addr *uint32) *semaRoot {
   return &semtable[(uintptr(unsafe.Pointer(addr))>>3)%semTabSize].root
}
```

## 仓库大门长啥样

```go
// semaRoot 就是仓库大门，里面记录着门口等了多少人，排队的队伍，以及队伍里面的人是要去哪个小仓库
type semaRoot struct {
	lock  mutex
	treap *sudog // 大门进去会有很多小仓库，不同仓库排队队伍的第一个人
	nwait uint32 // 等待的人的数量
}
```

## 简单的拿一个刀

```go
// 我们获取刀的时候，如果仓库里面还有，那就直接拿走，并返回true，不然就返回false
func cansemacquire(addr *uint32) bool {
	for {
		v := atomic.Load(addr)
		if v == 0 {
			return false
		}
		if atomic.Cas(addr, v, v-1) {
			return true
		}
	}
}
```

## 要去拿一个刀

```go
// 这个是获取刀的真实的逻辑
func semacquire1(addr *uint32, lifo bool, profile semaProfileFlags, skipframes int) {
    // 是谁来获取，是gp
	gp := getg()
	if gp != gp.m.curg {
		throw("semacquire not on the G stack")
	}

	// 简单的case，就是仓库里面还有刀
	if cansemacquire(addr) {
		return
	}

    // 下面就是这4步
    // 1. 增加大门口等待排队的数量
    // 2. 尝试再去看一下我们的仓库里面还有没有刀，如果有，可能是有人归还了，我们就直接拿着走
    // 3. 去排队了
    // 4. 休眠
  	// 新建一个sudog
	s := acquireSudog()
    // 获取大门信息
	root := semroot(addr)
	t0 := int64(0)
	s.releasetime = 0
	s.acquiretime = 0
	
    // s的ticket是0
    s.ticket = 0
	
	for {
        // 大门加锁
		lockWithRank(&root.lock, lockRankRoot)
		
        // 给大门等待人数量上加1
		atomic.Xadd(&root.nwait, 1)
		
        // 判断他要去的仓库里面还有没有刀
		if cansemacquire(addr) {
			atomic.Xadd(&root.nwait, -1)
			unlock(&root.lock)
			break
		}
        
		// 暂时里面没有东西，我们等着吧，就要去排队
		root.queue(addr, s, lifo)
        // gopark阻塞这个g
		goparkunlock(&root.lock, waitReasonSemacquire, traceEvGoBlockSync, 4+skipframes)
        // 这边是唤醒了这个g
        // 我们的票据不是0的话，或者我们仓库里面有刀的话，就可以得到执行
		if s.ticket != 0 || cansemacquire(addr) {
			break
		}
	}
	
    // 释放sudog
	releaseSudog(s)
}
```

## 刀没直接拿到，要去排队等

```go
// 这个就是给semaRoot.treap上面进行排队
func (root *semaRoot) queue(addr *uint32, s *sudog, lifo bool) {
    // 拿到我们要排队的人
	s.g = getg()
    // 存一下它要去的仓库在哪
	s.elem = unsafe.Pointer(addr)
  	// 等我们下面再给他说，它该拍到哪里，前一个队伍是谁，后一个队伍是谁
	s.next = nil
	s.prev = nil
	
	var last *sudog
    // pt一开始就是第一个队伍第一个人
	pt := &root.treap
    // for循环，在每个队列里面找我们的小仓库是那个队伍
	for t := *pt; t != nil; t = *pt {
		if t.elem == unsafe.Pointer(addr) {
            // t就是我们小仓库的排队信息
			if lifo {
				// lifo是说，后入队的，先执行，排到最前面
                // prev和next都是队伍第一个人要知道的
                // 同一个队伍的不同人是通过waitlink连接的
                // 队伍的每个人都能通过waittail知道最后一个人的信息
				*pt = s
				s.ticket = t.ticket
				s.acquiretime = t.acquiretime
				s.parent = t.parent
				s.prev = t.prev
				s.next = t.next
				if s.prev != nil {
					s.prev.parent = s
				}
				if s.next != nil {
					s.next.parent = s
				}
				// Add t first in s's wait list.
				s.waitlink = t
				s.waittail = t.waittail
				if s.waittail == nil {
					s.waittail = t
				}
				t.parent = nil
				t.prev = nil
				t.next = nil
				t.waittail = nil
			} else {
				// 把人排队队伍的最后面
				if t.waittail == nil {
					t.waitlink = s
				} else {
					t.waittail.waitlink = s
				}
				t.waittail = s
				s.waitlink = nil
			}
			return
		}
        // 倒是第二个找到的队伍
		last = t
        // 因为我们存储的时候是这样存的，所以我们这边也是这样找的
		if uintptr(unsafe.Pointer(addr)) < uintptr(t.elem) {
			pt = &t.prev
		} else {
			pt = &t.next
		}
	}
    
    // ticket很经常和0进行比对，所以我们让最后1位为1的话，肯定就不是0
	s.ticket = fastrand() | 1
	s.parent = last
	*pt = s

	// 修复这个平衡二叉树，让这个树继续满足条件
	for s.parent != nil && s.parent.ticket > s.ticket {
		if s.parent.prev == s {
			root.rotateRight(s.parent)
		} else {
			if s.parent.next != s {
				panic("semaRoot queue")
			}
			root.rotateLeft(s.parent)
		}
	}
}
```

## 归还一个刀，让其他的人也能去用了

```go
// 归还一个刀
func semrelease1(addr *uint32, handoff bool, skipframes int) {
    // 找到大门
	root := semroot(addr)
    // 给仓库里面还一把刀
	atomic.Xadd(addr, 1)

	// 如果没有人在等，就直接结束就好了
	if atomic.Load(&root.nwait) == 0 {
		return
	}

	// 有人在等呢
    // 加锁
	lockWithRank(&root.lock, lockRankRoot)
    // 在判断一下，可能有其他人过来了
	if atomic.Load(&root.nwait) == 0 {
		unlock(&root.lock)
		return
	}
    // 我们找一个sudog让他出来拿刀工作
	s, t0 := root.dequeue(addr)
    // 如果找到了，就让大门的等待数量-1
	if s != nil {
		atomic.Xadd(&root.nwait, -1)
	}
    // 解锁大门
	unlock(&root.lock)
	if s != nil {
        // s.ticket != 0 证明这个ticket不太合法
        // 也就是说sudog要执行，必须要把票清空
		if s.ticket != 0 {
			throw("corrupted semaphore ticket")
		}
        // 我们再去判断一下能不能拿到刀，能拿到的话，ticket=1
		if handoff && cansemacquire(addr) {
			s.ticket = 1
		}
        // 恢复我们的g，将我们的g放在p的runnext
		readyWithTime(s, 5+skipframes)
        // 如果ticket是1，我们的g的m上没上锁的话
		if s.ticket == 1 && getg().m.locks == 0 {
			// 如果是饥饿模式，我们直接切换到g执行，不要等了
			goyield()
		}
	}
}
```

## 其他人还刀，通知我们来拿刀

```go
func (root *semaRoot) dequeue(addr *uint32) (found *sudog, now int64) {
    // ps就是门锁的第一个队伍的第一个人
	ps := &root.treap
	s := *ps
	for ; s != nil; s = *ps {
		if s.elem == unsafe.Pointer(addr) {
            // 找到我们的队伍了
			goto Found
		}
		if uintptr(unsafe.Pointer(addr)) < uintptr(s.elem) {
			ps = &s.prev
		} else {
			ps = &s.next
		}
	}
    // 没找到我们的队伍
	return nil, 0

Found:
    // 删除我们的sudog
	if t := s.waitlink; t != nil {
		*ps = t
		t.ticket = s.ticket
		t.parent = s.parent
		t.prev = s.prev
		if t.prev != nil {
			t.prev.parent = t
		}
		t.next = s.next
		if t.next != nil {
			t.next.parent = t
		}
		if t.waitlink != nil {
			t.waittail = s.waittail
		} else {
			t.waittail = nil
		}
		t.acquiretime = now
		s.waitlink = nil
		s.waittail = nil
	} else {
        // 删除这个队伍，因为没有人排队了，还要修复一下平衡二叉树
		for s.next != nil || s.prev != nil {
			if s.next == nil || s.prev != nil && s.prev.ticket < s.next.ticket {
				root.rotateRight(s)
			} else {
				root.rotateLeft(s)
			}
		}
		// Remove s, now a leaf.
		if s.parent != nil {
			if s.parent.prev == s {
				s.parent.prev = nil
			} else {
				s.parent.next = nil
			}
		} else {
			root.treap = nil
		}
	}
	s.parent = nil
	s.elem = nil
	s.next = nil
	s.prev = nil
	s.ticket = 0
	return s, now
}
```




