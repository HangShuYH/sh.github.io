# 浅析RT-mutex
**Version: Linux 6.6.0**
## why RT-mutex
PREEMPT_RT是将Linux转换为RTOS的一个史诗级的补丁，从Linux 6.12开始，所有发行版都将包含实时Linux的代码。PREEMPT_RT 的关键在于让Linux内核中不可抢占的代码尽可能减少，其中一个重要改进方向就是让spinlock变得可抢占，因此PREEMPT_RT引入了RT-mutex来代替spinlock。

在CONFIG_PREEMPT_RT未开启时，spinlock的底层实现实际是raw_spinlock。在获取raw_spinlock时，内核会关抢占。低优先级的线程只要在spinlock保护的临界区内，高优先级的线程都无法抢占CPU，实时性就无法得到保证。

在CONFIG_PREEMPT_RT开启时，spinlock的底层实现实际是RT-mutex。RT-mutex是一种互斥锁，因此即便线程获取了锁处于临界区内，也可以被高优先级的线程抢占。Linux中确实需要关闭抢占的代码(有一百多处)需要使用raw_spinlock而不是spinlock。

与一般的互斥锁不同的点在于RT-mutex解决了unbouned priority inversion problem,即无限制优先级反转问题。

## Priority Inversion
在实时操作系统中，低优先级的线程运行时，高优先级的线程无法抢占CPU执行，就产生了优先级反转问题。优先级反转的因素有很多，大多数时候都无法避免。例如，当低优先级线程获取了共享资源，高优先级线程想要获取该资源时就不得不等待低优先级线程主动释放。

RT-mutex解决的是无限制优先级反转问题。无限制优先级反转问题的一个典型例子就是有三个线程A,B,C，假设P(A)>P(B)>P(C)。
**P(A)代表线程A的优先级**
当A试图获取一个C已经持有的共享资源X时，A必须等待C主动释放X。但是在A等待的过程中，B不需要获取该资源且由于P(B)>P(C)，B就会抢占C。注意，此时优先级低于A的B事实上在阻塞A的执行，因为C被抢占之后无法释放X。对于A来说，C释放X的时间是无法预料的，因此A的实时性就无法得到保证。更糟糕的情况是，如果有很多优先级处于A和C之间的线程，这些线程会不停抢占C，高优先级的A甚至可能被饿死。这就是无限制优先级反转问题。

[优先级反转那点事儿](https://zhuanlan.zhihu.com/p/146132061)一文对优先级反转的问题有很详细的解释。

## RT-mutex Design
### Priority Inheritance(PI)
RT-mutex解决优先级反转问题的方法是Priority Inheritance(PI)，即优先级继承。PI就是当高优先级线程A阻塞在锁上时，这个锁的持有线程B(低优先级)会临时继承A的优先级。线程B释放锁的时候，再将自己的优先级调整回去。

还是上面的那个例子，当线程C持有线程A需要的锁时，线程A去申请获取这把锁，会导致线程C的优先级临时提升为线程A的优先级。此时线程B由于优先级低于线程C，就无法进行抢占。当线程C释放锁的时候，它的优先级被调整回原来的低优先级，线程A就可以顺利抢占线程C的执行。这样无限制优先级反转问题就被解决了。
### PI chain
在真实场景下，当然不会只有3个线程和一把锁。当有一系列线程和锁参与进来时，优先级继承就需要顺着PI chain传播下去。
例如：
```
    Process:  A, B, C, D, E
    Mutexes:  L1, L2, L3, L4
```

![chain1](https://github.com/user-attachments/assets/b496e2d8-ab45-4f6e-a7cf-fd4c563b1e1e)

PI chain的含义是：线程E等待锁L4，锁L4被线程D持有，线程D等待锁L3，L3被线程C持有，线程C等待锁L2，锁L2被线程B持有，线程B等待锁L1，锁L1被线程A持有。
当这个PI chain存在时，五个线程的优先级应该满足处于链右边的线程的优先级应当不低于处于链左边的优先级。只有该条件满足时，无限制优先级反转问题才不会出现。

当然，PI chain并不一定是单链，因为一把锁可能会有多个线程在等待。假设此时一个线程F申请锁L2，且F也有自己持有的锁L5，L5有一个等待者G。那么以A为链尾的PI chain就会和以F为链尾的PI chain合并为一个PI chain：

![chain2](https://github.com/user-attachments/assets/a1bcc712-5f98-438d-9d37-8e992271e693)

思考一下，当F去尝试获取锁L2时，链上的优先级应该发生怎样的变化？如果F的优先级比同样在等待锁L2的C高，那么F的优先级应该顺着链往右传播，使得线程B和线程A的优先级至少不低于F。

如果是线程G去获取L2呢？这种情况不可能发生，因为一个线程不会在多个锁上等待。但是一个锁可以有多个等待者。

锁的释放顺序应该是怎样的呢？当然是逆着链的顺序来释放，因为当某个线程还在等待一把锁的时候，当然不会去尝试释放它自己拥有的锁。

## RT-mutex Implementation
### Data Structures
#### RT-Mutex
RT-Mutex的定义如下：
```c
struct rt_mutex_base {
	raw_spinlock_t		wait_lock;
	struct rb_root_cached   waiters;
	struct task_struct	*owner;
};
```
rt_mutex_base中的waiters就是下文的Mutex Waiter Tree。wait_lock用以保护waiters。owner存储锁的持有者。

#### Mutex Waiter Tree
RT-mutex需要知道当前等待在锁上的线程有哪些。等待者使用以线程优先级为key的红黑树来存储，这样一把锁能够很快找到当前等待自己的最高优先级的线程，高优先级的线程能够优先竞争到该锁(相同优先级的线程按照FIFO的策略加入到Tree中)。

#### Mutex owner
RT-mutex会保存它的持有者的指针。如果mutex当前没有被持有，则owner指针为NULL。task_struct是两字节对齐的，因此RT-mutex还使用了owner的最低位来保存当前RT-mutex是否有waiters(Has Waiters flag)。
owner的状态图如下：
| owner | bit0 | Notes |
|--------|--------|--------|
| NULL | 0 | lock is free (fast acquire possible) |
| NULL | 1 | lock is free and has waiters and the top waiter is going to take the lock |
| taskpointer | 0 | lock is held (fast release possible) |
| taskpointer | 1 | lock is held and has waiters |

当owner为NULL，bit0为0时，RT-mutex的获取通过快速路径获取锁。
```c
static __always_inline bool rt_mutex_cmpxchg_acquire(struct rt_mutex_base *lock,
						     struct task_struct *old,
						     struct task_struct *new)
{
	return try_cmpxchg_acquire(&lock->owner, &old, new);
}
```
owner为NULL，bit0为1是一个过渡状态。当线程要通过慢速路径去获取RT-mutex的时候，需要将bit0设置为1，以避免快速路径的获取和释放。

当owner为taskpointer，bit0为0时，RT-mutex通过快速路径释放锁。
```c
static __always_inline bool rt_mutex_cmpxchg_release(struct rt_mutex_base *lock,
						     struct task_struct *old,
						     struct task_struct *new)
{
	return try_cmpxchg_release(&lock->owner, &old, new);
}
```

#### Task PI Tree
我们先看一下task_struct中关于RT-mutex的字段：
```c
struct task_struct {
    ...
	/* Protection of the PI data structures: */
	raw_spinlock_t			pi_lock;

	struct wake_q_node		wake_q;

#ifdef CONFIG_RT_MUTEXES
	/* PI waiters blocked on a rt_mutex held by this task: */
	struct rb_root_cached		pi_waiters;
	/* Updated under owner's pi_lock and rq lock */
	struct task_struct		*pi_top_task;
	/* Deadlock detection and priority inheritance handling: */
	struct rt_mutex_waiter		*pi_blocked_on;
#endif
    ...
}
```
为了构建PI chain，每一个线程都有用红黑树组织的pi_waiters，用以保存当前线程持有的RT-mutex的top waiters。top_waiters是指锁的最高优先级的等待线程。假设线程A持有锁L1和L2，那么A的task_struct的pi_waiters就会保存L1等待者中的最高优先级线程和L2等待者中的最高优先级线程。
为了实现优先级继承，线程并不需要知道它持有锁的所有等待者，只需要知道其中优先级最高那一个等待即可。例如，当线程B尝试去获取L1时，线程B只需要知道等待L1的最高优先级线程，如果线程B的优先级更高，则需要更新线程A的pi_waiters，如果线程B的优先级是pi_waiters中的最高的，则还需要临时调整线程A的优先级。

### Implementation Details
#### Taking of a mutex
```c
static __always_inline void rtlock_lock(struct rt_mutex_base *rtm)
{
	if (unlikely(!rt_mutex_cmpxchg_acquire(rtm, NULL, current)))
		rtlock_slowlock(rtm);
}
```
RT-mutex先尝试通过原子指令的快速路径获取锁，如果获取失败则通过慢速路径获取。慢速路径的获取和释放都需要获取wait_lock并且关中断。
```c
static __always_inline void __sched rtlock_slowlock(struct rt_mutex_base *lock)
{
	unsigned long flags;

	raw_spin_lock_irqsave(&lock->wait_lock, flags);
	rtlock_slowlock_locked(lock);
	raw_spin_unlock_irqrestore(&lock->wait_lock, flags);
}
```
```c
static void __sched rtlock_slowlock_locked(struct rt_mutex_base *lock)
{
	struct rt_mutex_waiter waiter;
	struct task_struct *owner;

	lockdep_assert_held(&lock->wait_lock);

	if (try_to_take_rt_mutex(lock, current, NULL))
		return;

	rt_mutex_init_rtlock_waiter(&waiter);

	/* Save current state and set state to TASK_RTLOCK_WAIT */
	current_save_and_set_rtlock_wait_state();

	trace_contention_begin(lock, LCB_F_RT);

	task_blocks_on_rt_mutex(lock, &waiter, current, NULL, RT_MUTEX_MIN_CHAINWALK);

	for (;;) {
		/* Try to acquire the lock again */
		if (try_to_take_rt_mutex(lock, current, &waiter))
			break;

		if (&waiter == rt_mutex_top_waiter(lock))
			owner = rt_mutex_owner(lock);
		else
			owner = NULL;
		raw_spin_unlock_irq(&lock->wait_lock);

		if (!owner || !rtmutex_spin_on_owner(lock, &waiter, owner))
			schedule_rtlock();

		raw_spin_lock_irq(&lock->wait_lock);
		set_current_state(TASK_RTLOCK_WAIT);
	}

	/* Restore the task state */
	current_restore_rtlock_saved_state();

	/*
	 * try_to_take_rt_mutex() sets the waiter bit unconditionally.
	 * We might have to fix that up:
	 */
	fixup_rt_mutex_waiters(lock, true);
	debug_rt_mutex_free_waiter(&waiter);

	trace_contention_end(lock, 0);
}
```
由于有些体系结构不支持cmpxchg的快速路径，所以慢速路径首先会调用try_to_take_rt_mutex函数去获取锁。在慢速路径中，每当线程尝试获取锁时，都会调用try_to_take_rt_mutex。
try_to_take_mutex首先会将owner的bit0设置为1以避免快速路径。在两种情况下try_to_take_mutex会成功获取到锁：(1)锁没有owner，即锁没有被持有。(2)当前线程是锁的waiters中优先级最高的。
当前线程通过try_to_take_mutex获取锁时，需要将锁的top waiter加入到当前线程的pi_waiters中，然后将锁的owner修改为自己。
```c
static int __sched
try_to_take_rt_mutex(struct rt_mutex_base *lock, struct task_struct *task,
		     struct rt_mutex_waiter *waiter)
{
	lockdep_assert_held(&lock->wait_lock);

	/*
	 * Before testing whether we can acquire @lock, we set the
	 * RT_MUTEX_HAS_WAITERS bit in @lock->owner. This forces all
	 * other tasks which try to modify @lock into the slow path
	 * and they serialize on @lock->wait_lock.
	 *
	 * The RT_MUTEX_HAS_WAITERS bit can have a transitional state
	 * as explained at the top of this file if and only if:
	 *
	 * - There is a lock owner. The caller must fixup the
	 *   transient state if it does a trylock or leaves the lock
	 *   function due to a signal or timeout.
	 *
	 * - @task acquires the lock and there are no other
	 *   waiters. This is undone in rt_mutex_set_owner(@task) at
	 *   the end of this function.
	 */
	mark_rt_mutex_waiters(lock);

	/*
	 * If @lock has an owner, give up.
	 */
	if (rt_mutex_owner(lock))
		return 0;

	/*
	 * If @waiter != NULL, @task has already enqueued the waiter
	 * into @lock waiter tree. If @waiter == NULL then this is a
	 * trylock attempt.
	 */
	if (waiter) {
		struct rt_mutex_waiter *top_waiter = rt_mutex_top_waiter(lock);

		/*
		 * If waiter is the highest priority waiter of @lock,
		 * or allowed to steal it, take it over.
		 */
		if (waiter == top_waiter || rt_mutex_steal(waiter, top_waiter)) {
			/*
			 * We can acquire the lock. Remove the waiter from the
			 * lock waiters tree.
			 */
			rt_mutex_dequeue(lock, waiter);
		} else {
			return 0;
		}
	} else {
		/*
		 * If the lock has waiters already we check whether @task is
		 * eligible to take over the lock.
		 *
		 * If there are no other waiters, @task can acquire
		 * the lock.  @task->pi_blocked_on is NULL, so it does
		 * not need to be dequeued.
		 */
		if (rt_mutex_has_waiters(lock)) {
			/* Check whether the trylock can steal it. */
			if (!rt_mutex_steal(task_to_waiter(task),
					    rt_mutex_top_waiter(lock)))
				return 0;

			/*
			 * The current top waiter stays enqueued. We
			 * don't have to change anything in the lock
			 * waiters order.
			 */
		} else {
			/*
			 * No waiters. Take the lock without the
			 * pi_lock dance.@task->pi_blocked_on is NULL
			 * and we have no waiters to enqueue in @task
			 * pi waiters tree.
			 */
			goto takeit;
		}
	}

	/*
	 * Clear @task->pi_blocked_on. Requires protection by
	 * @task->pi_lock. Redundant operation for the @waiter == NULL
	 * case, but conditionals are more expensive than a redundant
	 * store.
	 */
	raw_spin_lock(&task->pi_lock);
	task->pi_blocked_on = NULL;
	/*
	 * Finish the lock acquisition. @task is the new owner. If
	 * other waiters exist we have to insert the highest priority
	 * waiter into @task->pi_waiters tree.
	 */
	if (rt_mutex_has_waiters(lock))
		rt_mutex_enqueue_pi(task, rt_mutex_top_waiter(lock));
	raw_spin_unlock(&task->pi_lock);

takeit:
	/*
	 * This either preserves the RT_MUTEX_HAS_WAITERS bit if there
	 * are still waiters or clears it.
	 */
	rt_mutex_set_owner(lock, task);

	return 1;
}
```

try_to_take_rt_mutex失败后，当前线程调用rt_mutex_init_rtlock_waiter设置waiter的task为当前线程，lock为需要获取的锁，waiter的rbtree node设置为当前线程的优先级，然后调用task_blocks_on_rt_mutex将自己阻塞在锁上。(task_blocks_on_rt_mutex函数代码较长，就不贴代码了)
如果当前线程是当前正在等待该锁的最高优先级线程，删除来自所有者的 pi_waiters 的前一个顶级等待线程（如果存在），并将当前线程添加到owner的pi_waiters中。由于owner的 pi_waiter发生变化，调用rt_mutex_adjust_prio(后续会详细说明该函数)来确定是否需要调整owner的优先级。如果owner也阻塞在锁上，且owner的pi_waiters发生了变化，则需要调用rt_mutex_adjust_prio_chain沿着PI chain更新链上线程的优先级。

随后当前线程进入无限的for循环中，每次在循环开始时，lock都会尝试去获取锁。如果当前线程是锁的top_waiter，则会自旋等待，否则睡眠。

#### Unlocking the Mutex
RT-mutex的unlock和lock一样，先尝试通过cmpxchg的快速路径(快速路径要求owner为NULL，bit0为0)，如果快速路径失败，则进入慢速路径。
```c
static __always_inline void __rt_mutex_unlock(struct rt_mutex_base *lock)
{
	if (likely(rt_mutex_cmpxchg_release(lock, current, NULL)))
		return;

	rt_mutex_slowunlock(lock);
}
```

如果锁没有waiters，则只需要将锁的owner设置为NULL(这部分代码比较微妙，建议仔细阅读一下注释)。
如果锁有waiters，则需要将锁的top_waiter移除出锁的waiters Tree和owner的pi_waiters Tree。然后唤醒该线程即可。

```c
static void __sched rt_mutex_slowunlock(struct rt_mutex_base *lock)
{
	DEFINE_RT_WAKE_Q(wqh);
	unsigned long flags;

	/* irqsave required to support early boot calls */
	raw_spin_lock_irqsave(&lock->wait_lock, flags);

	debug_rt_mutex_unlock(lock);

	while (!rt_mutex_has_waiters(lock)) {
		/* Drops lock->wait_lock ! */
		if (unlock_rt_mutex_safe(lock, flags) == true)
			return;
		/* Relock the rtmutex and try again */
		raw_spin_lock_irqsave(&lock->wait_lock, flags);
	}

	/*
	 * The wakeup next waiter path does not suffer from the above
	 * race. See the comments there.
	 *
	 * Queue the next waiter for wakeup once we release the wait_lock.
	 */
	mark_wakeup_next_waiter(&wqh, lock);
	raw_spin_unlock_irqrestore(&lock->wait_lock, flags);

	rt_mutex_wake_up_q(&wqh);
}
```

#### Priority adjustments
rt_mutex_adjust_prio函数用来临时调整一个线程的优先级，前面我们提到，如果当前线程是正在等待锁的最高优先级线程，它需要将自己加入到锁的owner的pi_waiters中。由于owner的pi_waiters发生改变，需要调用rt_mutex_adjust_prio更新owner的优先级。rt_mutex_setprio是最终真正更新线程优先级的函数。

```c
static __always_inline void rt_mutex_adjust_prio(struct rt_mutex_base *lock,
						 struct task_struct *p)
{
	struct task_struct *pi_task = NULL;

	lockdep_assert_held(&lock->wait_lock);
	lockdep_assert(rt_mutex_owner(lock) == p);
	lockdep_assert_held(&p->pi_lock);

	if (task_has_pi_waiters(p))
		pi_task = task_top_pi_waiter(p)->task;

	rt_mutex_setprio(p, pi_task);
}
```
如果owner的优先级被更新且也被阻塞在RT-mutex上(即owner不是链上的最后一个线程，还需要随着链更新其它线程的优先级)，或者需要检测死锁，则还需要调用rt_mutex_adjust_prio_chain。

### Other Issues
#### Per CPU Variables
现在的spinlock不会禁止内核抢占，所以对per-cpu变量的使用需要额外小心，线程被抢占后可能被调度到其它CPU上执行，这时per cpu变量就出错了。因此需要使用per-cpu lock来保护per-cpu变量。

#### Task State
由于现在spinlock会发生睡眠，在sched函数中，会将task的state修改为TASK_RUNNING。这时候可能会发生错误，考虑下面这段代码：
```c
spin_lock(&mylock1);
current->state = TASK_UNINTERRUPTIBLE;
spin_lock(&mylock2);
blah();
spin_unlock(&mylock2);
spin_unlock(&mylock1);
```
blah函数可能预期current->state是TASK_UNINTERRUPTIBLE，然后由于spinlock的睡眠，current->state被sched函数修改为了TASK_RUNNING，就有可能会出现意料不到的错误。

解决办法是在慢速路径中调用current_save_and_set_rtlock_wait_state函数将当前线程的state保存到saved_state中。在线程被唤醒后，将根据情况判断是将线程的state恢复为saved_state或者设置为TASK_RUNNING。

## References
[优先级反转那点事儿](https://zhuanlan.zhihu.com/p/146132061)
[A realtime preemption overview](https://lwn.net/Articles/146861/)
[rt-mutex-design](https://docs.kernel.org/locking/rt-mutex-design.html)
[rt-mutex](https://docs.kernel.org/locking/rt-mutex.html)