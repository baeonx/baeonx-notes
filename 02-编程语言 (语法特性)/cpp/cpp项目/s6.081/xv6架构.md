# 丢失唤醒（Lost Wakeup）：真正的多核竞态

## 真正的场景：两颗 CPU 并行

xv6 是 `-smp 3`（多核）。A 在 CPU0，B 在 CPU1，同时执行：

```text
时间 →    CPU0 (进程 A)                    CPU1 (进程 B)
          release(lk)
          [还没执行到 SLEEPING]  ←─同时─→  满足条件，调 wakeup(chan)
                                          扫到 A: p->state == RUNNING
                                          → 不唤醒 A          ✗ 信号丢了!
          p->state = SLEEPING
          sched() 睡下
          → 从此再没人叫醒 A（因为 B 那次 wakeup 已经错过了）
```

A 不需要被切走——B 在另一颗核上并行地跑。A 在"放了 lk、还没设 SLEEPING"的这个极短窗口里，CPU1 上的 B 已经扫过 A 了，看到它是 RUNNING（不是 SLEEPING），就跳过了。

## 为什么不是"抢占"

- **单颗 CPU 上**，持有自旋锁期间中断是关的（`push_off`），不会有抢占让你在中间被切走。
- 所以这个 bug **只在多核（SMP）下发生**：另一颗 CPU 的线程真·并发地跑。

## 修复后：拿 `p->lock` 挡住另一颗 CPU

```c
acquire(&p->lock);   // A 拿住自己的锁
release(lk);
p->state = SLEEPING;
```

现在 A 从 `acquire(&p->lock)` 到睡着被切走，一直持有 `p->lock`。CPU1 上的 B 要操作 A，B 必须：

```c
acquire(&A->lock);   // ← B 卡在这，因为 A 拿着！
```

所以 B 在 A 睡着之前进不来，只能自旋等。等 B 拿到 `A->lock` 时（调度器已释放），A 必然是 SLEEPING → B 唤醒 A。**竞态被消除了。**

## 双 CPU 图（修复后）

```text
时间 →    CPU0 (A)                          CPU1 (B)
          acquire(&A->lock)  ← 拿住
          release(lk)
          p->state = SLEEPING
          sched() 切走
            └ 调度器 release(&A->lock) ──┐
                                         │
                                          ▼  B 这时才 acquire(&A->lock) ✓
                                             A->state == SLEEPING ✓
                                             A->state = RUNNABLE → 唤醒
```

## 一句话

你说得对：A 是**先设 SLEEPING 再自己 sched 切走**的，不是被动切走。我那个例子的意思是——**"B 在另一颗 CPU 上并发执行"——丢失唤醒是多核竞态**，不是抢占。而修复的 `acquire(&p->lock)` 正是为了挡住"另一颗 CPU 上的 wakeup"在这个窗口里乱看 A 的状态。