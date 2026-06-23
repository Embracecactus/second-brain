---
tags:
  - embedded-linux
  - bsp
  - lockdep
  - locking
  - kernel-debug
  - deadlock
category: embedded-linux
created: 2026-06-23
updated: 2026-06-23
status: active
kernel: Linux 6.1.141
parent: bsp-interrupt-concurrency
---

# Lockdep 原理与源码深挖 — 内核锁依赖验证器

> **前置笔记**：[[bsp-interrupt-concurrency]] — 阶段三：并发
>
> **核心文件**：`kernel/locking/lockdep.c` (6614 行)
> `include/linux/lockdep.h`, `include/linux/lockdep_types.h`
> `include/linux/spinlock_api_smp.h`

---

## 一、核心思想

Lockdep 不是运行时锁调试器，而是**依赖图验证器**：

- 把所有锁看成**节点**，锁的获取顺序看成**有向边**
- 构建全局有向图：`A → B` 表示"在持有 A 时获取 B"
- 每次发现新边，用 BFS 检查是否会形成**环路**
- 环路意味着潜在死锁 — 即使当前没有死锁，也**报告警告**

三个维度：

| 检查项 | 含义 | 检测方法 |
|--------|------|----------|
| 循环依赖 | A→B→C→A | BFS 从 next 能否回到 prev |
| IRQ 上下文 | irq-safe 锁在 irq-unsafe 锁之后取 | usage_mask 位运算 |
| 锁类型兼容性 | spinlock 内取 mutex (会睡眠) | wait_type 检查 |

```
                  ┌─────────────────────────┐
                  │    依赖图 (全局)          │
                  │                         │
                  │  lock_class_A ──→ lock_class_B  │
                  │       ↑                    │
                  │       │                    ↓
                  │  lock_class_D ←── lock_class_C  │
                  │                         │
                  │  ★ 环路 = 潜在死锁       │
                  └─────────────────────────┘
```

---

## 二、数据结构

### 2.1 `lock_class` — 锁类

```c
// include/linux/lockdep_types.h:91
struct lock_class {
    // 哈希链 (class hash table)
    struct hlist_node hash_entry;

    // 全局链表: all_lock_classes / free_lock_classes
    struct list_head lock_entry;

    // ★★★ 核心: 依赖图边
    struct list_head locks_after;   // 持有本锁之后获取的锁集合
    struct list_head locks_before;  // 获取本锁之前持有的锁集合

    const struct lockdep_subclass_key *key;  // 静态地址 = 类标识
    unsigned int subclass;

    // IRQ 上下文使用位图
    unsigned long usage_mask;
    const struct lock_trace *usage_traces[LOCK_TRACE_STATES];
    //   掩码位: LOCKF_USED_IN_HARDIRQ_READ 等

    const char *name;
    u8 wait_type_inner;
    u8 wait_type_outer;
    u8 lock_type;
};
```

**关键点**：`key` 是每个锁的静态地址（`&__key` in `raw_spin_lock_init`），不同实例但同类型的锁共享同一个类。这就是为什么同一驱动中所有 spinlock 被当做一个类处理。

### 2.2 `lockdep_map` — 嵌入锁对象

```c
// 嵌入在 spinlock_t / mutex / rwsem 中:
struct lockdep_map {
    struct lock_class_key *key;                             // 指向 lock_class_key
    struct lock_class *class_cache[NR_LOCKDEP_CACHING_CLASSES]; // 缓存 class
    const char *name;
    u8 wait_type_outer;   // 本锁可以在什么上下文中获取
    u8 wait_type_inner;   // 持有本锁时呈现什么上下文
    u8 lock_type;
};
```

### 2.3 `held_lock` — 当前持有锁栈

```c
// 每个 task_struct 中:
//   struct held_lock held_locks[MAX_LOCK_DEPTH];
//   unsigned int lockdep_depth;

struct held_lock {
    u64 prev_chain_key;      // 链式哈希
    struct lockdep_map *instance;
    unsigned int class_idx;  // class - lock_classes 索引
    unsigned long acquire_ip;
    u64 waittime_stamp;
    u64 holdtime_stamp;
    int pin_count;
    u8 irq_context;          // 0=正常, 1=softirq, 2=hardirq
    u8 trylock;
    u8 read;
    u8 check;
    u8 hardirqs_off;
    u8 references;
};
```

### 2.4 `lock_list` — 图的边

```c
// lock_class->locks_after / locks_before 链表元素:
struct lock_list {
    struct list_head entry;
    struct lock_class *class;     // 目标类
    struct lock_trace *trace;     // 谁创建的这条边
    u16 distance;                 // 距离
    u8 dep;                       // 依赖类型掩码
    u8 only_xr;                   // 是否仅读+递归
};
```

### 2.5 `circular_queue` — BFS 队列

```c
// kernel/locking/lockdep.c:1451
#define MAX_CIRCULAR_QUEUE_SIZE (4096UL)

struct circular_queue {
    struct lock_list *element[MAX_CIRCULAR_QUEUE_SIZE];
    unsigned int front, rear;
};

static struct circular_queue lock_cq;  // 全局单例
```

---

## 三、锁获取完整路径

### 3.1 `spin_lock()` → `raw_spin_lock()` → `__raw_spin_lock()`

```c
// include/linux/spinlock.h
#define raw_spin_lock(lock)  _raw_spin_lock(lock)

// kernel/locking/spinlock.c (若 CONFIG_UNINLINE_SPIN_UNLOCK):
noinline void __lockfunc _raw_spin_lock(raw_spinlock_t *lock)
{
    __raw_spin_lock(lock);
}

// include/linux/spinlock_api_smp.h:130 — ★ 关键 inline 函数:
static inline void __raw_spin_lock(raw_spinlock_t *lock)
{
    preempt_disable();                            // 1. 关抢占
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_); // 2. ★ lockdep
    LOCK_CONTENDED(lock, do_raw_spin_trylock,      // 3. 真正锁硬件
                   do_raw_spin_lock);
    //   → do_raw_spin_trylock → arch_spin_trylock
    //   → 如果失败: lock_contended → do_raw_spin_lock → arch_spin_lock
    //   → 然后: lock_acquired (lockstat)
}
```

### 3.2 `spin_acquire` → `lock_acquire` → `__lock_acquire`

```c
// include/linux/lockdep.h:526
#define spin_acquire(l, s, t, i)  lock_acquire_exclusive(l, s, t, NULL, i)
#define lock_acquire_exclusive(l, s, t, n, i)  lock_acquire(l, s, t, 0, 1, n, i)

// kernel/locking/lockdep.c:5627
void lock_acquire(struct lockdep_map *lock, unsigned int subclass,
                  int trylock, int read, int check,
                  struct lockdep_map *nest_lock, unsigned long ip)
{
    trace_lock_acquire(lock, subclass, trylock, read, check, nest_lock, ip);

    raw_local_irq_save(flags);           // 锁住 IRQ (防止递归)
    check_flags(flags);
    lockdep_recursion_inc();
    __lock_acquire(lock, subclass, trylock, read, check,
                   irqs_disabled_flags(flags), nest_lock, ip, 0, 0);
    lockdep_recursion_finish();
    raw_local_irq_restore(flags);
}
```

### 3.3 `__lock_acquire()` — 核心获取

```c
// kernel/locking/lockdep.c:4903
static int __lock_acquire(struct lockdep_map *lock, unsigned int subclass,
                          int trylock, int read, int check, int hardirqs_off,
                          struct lockdep_map *nest_lock, unsigned long ip,
                          int references, int pin_count)
{
    struct task_struct *curr = current;
    struct lock_class *class = NULL;
    struct held_lock *hlock;
    unsigned int depth;

    // 1. ★★ 查找/注册锁类 (静态 key → class)
    class = lock->class_cache[subclass];
    if (unlikely(!class)) {
        class = register_lock_class(lock, subclass, 0);
        //   → hash table 查找 lock->key
        //   → 未找到: 从 free_lock_classes 分配
        //   → 每个 lock_class_key 全局唯一
    }

    // 2. ★ 如果是嵌套重复取锁 (references 计数)
    //    (同一 task 在同一个锁类上再次 acquire)
    if (depth) {
        hlock = curr->held_locks + depth - 1;
        if (hlock->class_idx == class_idx && nest_lock) {
            hlock->references += references;
            return 2;  // 简单计数, 跳过验证
        }
    }

    // 3. 填充 held_lock 条目
    hlock = curr->held_locks + depth;
    hlock->class_idx = class_idx;
    hlock->acquire_ip = ip;
    hlock->trylock = trylock;
    hlock->read = read;
    hlock->check = check;
    hlock->hardirqs_off = !!hardirqs_off;
    // ...

    // 4. ★ 检查 wait context (mutex in spinlock?)
    check_wait_context(curr, hlock);
    //   → 比较 hlock->wait_type_outer ← lock->wait_type_outer
    //   → 如果持有 spinlock (wait_type = 0) 却获取 mutex (wait_type = 2)
    //     则 WARN: "possible circular locking dependency detected"

    // 5. 标记使用位 (usage_mask)
    mark_usage(curr, hlock, check);
    //   → 根据当前 irq_context 设置对应的 usage_mask 位
    //     LOCKF_USED_IN_HARDIRQ / LOCKF_ENABLED_HARDIRQ ...

    // 6. ★★★ 依赖验证
    validate_chain(curr, hlock, chain_head, chain_key);
    //   → 见第四章

    // 7. 更新 per-task 状态
    curr->curr_chain_key = chain_key;
    curr->lockdep_depth++;
}
```

### 3.4 `register_lock_class()` — 类注册

```c
// kernel/locking/lockdep.c
static struct lock_class *register_lock_class(struct lockdep_map *lock,
                                               unsigned int subclass, int force)
{
    struct lock_class_key *key = lock->key;
    struct lock_class *class;

    // 1. 哈希查找: key→class
    hash = class_hash(key, subclass);
    hlist_for_each_entry(class, &classhash_table[hash], hash_entry) {
        if (class->key == key && class->subclass == subclass)
            return class;  // 已注册
    }

    // 2. 分配新 class (从 free_lock_classes)
    class = list_first_entry_or_null(&free_lock_classes,
                                      struct lock_class, lock_entry);
    class->key = key;
    class->name = lock->name;
    // 初始化 locks_after, locks_before 链表
    INIT_LIST_HEAD(&class->locks_after);
    INIT_LIST_HEAD(&class->locks_before);

    // 3. 缓存到 lockdep_map 中
    lock->class_cache[subclass] = class;

    return class;
}
```

**关键**：同一个 `.c` 文件中 `DEFINE_SPINLOCK(foo)` 的所有实例共享同一个 `lock_class_key` → 同一类。

---

## 四、依赖图验证 — `validate_chain()`

### 4.1 入口

```c
// kernel/locking/lockdep.c:3778
static int validate_chain(struct task_struct *curr,
                          struct held_lock *hlock,
                          int chain_head, u64 chain_key)
{
    // 跳过 trylock (trylock 不产生依赖)
    if (!hlock->trylock && hlock->check &&
        lookup_chain_cache_add(curr, hlock, chain_key)) {
        // ★ 新链 → 执行 O(N²) 验证

        // 1. 同一锁类重复获取?
        ret = check_deadlock(curr, hlock);
        //   → 遍历 held_locks[0..depth-1], 找 class_idx 相同
        //   → 找到且非嵌套 → 报告 AA deadlock

        // 2. ★★ 检查与已有依赖的兼容性
        if (!chain_head && ret != 2) {
            check_prevs_add(curr, hlock);
            //   → 遍历 held_locks[0..depth-1], 对每个 prev
            //     调用 check_prev_add(curr, prev, hlock, ...)
        }
    }
}
```

### 4.2 `check_prev_add()` — 新边验证与添加

```c
// kernel/locking/lockdep.c:3055
static int check_prev_add(struct task_struct *curr,
                          struct held_lock *prev, struct held_lock *next,
                          u16 distance, struct lock_trace **const trace)
{
    // ★ Step 1: 环路检测 — BFS 从 next 能否回到 prev
    ret = check_noncircular(next, prev, trace);
    if (ret == BFS_RMATCH) {
        print_circular_bug(...);
        return 0;  // → 死锁报告!
    }

    // ★ Step 2: IRQ 上下文兼容性
    if (!check_irq_usage(curr, prev, next))
        return 0;  // → irq-safe vs irq-unsafe 死锁报告

    // ★ Step 3: 边已存在? → 更新 dep 位并返回
    list_for_each_entry(entry, &hlock_class(prev)->locks_after, entry) {
        if (entry->class == hlock_class(next)) {
            entry->dep |= calc_dep(prev, next);
            // 同步更新 next->locks_before 的反向边
            return 1;
        }
    }

    // ★ Step 4: 冗余检查 — 已有替代路径?
    ret = check_redundant(prev, next);
    if (ret == BFS_RMATCH)
        return 2;

    // ★ Step 5: 真正添加边
    add_lock_to_list(hlock_class(next), hlock_class(prev),
                     &hlock_class(prev)->locks_after, ...);
    add_lock_to_list(hlock_class(prev), hlock_class(next),
                     &hlock_class(next)->locks_before, ...);
    return 2;
}
```

### 4.3 `check_noncircular()` — BFS 环路检测

```
                    BFS from next
                    ↓
    lock_class_A → lock_class_B → lock_class_C
                    ↑                        ↑
                    └─────── 能回到 prev? ────┘

    搜索过程:
    init:  queue = [next]         (next = lock_class_B)
    iter1: pop B, expand locks_after → C
           queue = [C]
    iter2: pop C, expand locks_after → D, E
           queue = [D, E]
    iter3: pop D, expand locks_after → ...
           ...
           if any reached {prev} → BFS_RMATCH → 死锁!
```

```c
// kernel/locking/lockdep.c:2147
static int check_noncircular(struct held_lock *src, struct held_lock *target,
                              struct lock_trace **const trace)
{
    struct lock_list src_entry;
    enum bfs_result ret;

    bfs_init_root(&src_entry, src);           // 根节点: src (next 锁)
    ret = check_path(target, &src_entry,      // BFS: 能否到达 target (prev 锁)
                     hlock_conflict, NULL, &target_entry);
    //   → __bfs_forwards (沿 locks_after 搜索)

    if (ret == BFS_RMATCH) {
        print_circular_bug(&src_entry, target_entry, src, target);
        return BFS_RMATCH;
    }
    return ret;
}
```

### 4.4 `__bfs()` — 通用 BFS

```c
// kernel/locking/lockdep.c:1715
static enum bfs_result __bfs(struct lock_list *source_entry,
                             void *data,
                             bool (*match)(...),
                             bool (*skip)(...),
                             struct lock_list **target_entry,
                             int offset)  // locks_after 或 locks_before
{
    struct circular_queue *cq = &lock_cq;

    __cq_init(cq);
    __cq_enqueue(cq, source_entry);          // 入队根

    while ((lock = __bfs_next(lock, offset)) ||  // 遍历兄弟
           (lock = __cq_dequeue(cq))) {           // 出队下一层
        if (lock_accessed(lock))
            continue;
        mark_lock_accessed(lock);

        // Step 2: 强依赖路径检查
        //   -(*R)-> + -(S*)-> 不同组成强依赖
        if (lock->parent) {
            u8 dep = lock->dep;
            if (lock->parent->only_xr)
                dep &= ~(DEP_SR_MASK | DEP_SN_MASK);
            if (!dep)
                continue;
            lock->only_xr = !(dep & (DEP_SN_MASK | DEP_EN_MASK));
        }

        // Step 3: match 检查
        if (match(lock, data)) {
            *target_entry = lock;
            return BFS_RMATCH;     // 找到目标!
        }

        // Step 4: 扩展邻接 (入队第一个子节点)
        head = get_dep_list(lock, offset);  // locks_after / locks_before
        list_for_each_entry_rcu(entry, head, entry) {
            visit_lock_entry(entry, lock);
            if (!first)
                continue;   // 只入队第一个,
            __cq_enqueue(cq, entry);  // 其余通过 __bfs_next 遍历
            first = false;
        }
    }
    return BFS_RNOMATCH;  // 未找到 → 安全
}
```

---

## 五、IRQ 上下文检测

Lockdep 最强大的功能之一是检测 IRQ 相关的死锁:

```
CPU0:  spin_lock(&A)
       → <interrupt>  // 中断发生
       → spin_lock(&B)  // 在中断上下文中获取 B

CPU1:  spin_lock(&B)
       → spin_lock(&A)  // 进程中获取 A

★ 死锁: CPU0 持有 A, 等 B; CPU1 持有 B, 等 A
        但 CPU0 在中断中, CPU1 无法释放 B 给 CPU0
```

### 5.1 `usage_mask` 位

```c
// 每个 lock_class 的 usage_mask 记录锁被使用过的上下文:
#define LOCKF_USED_IN_HARDIRQ       (1 << 0)  // 曾在 hardirq context 中获取
#define LOCKF_USED_IN_HARDIRQ_READ  (1 << 1)
#define LOCKF_USED_IN_SOFTIRQ       (1 << 2)
#define LOCKF_USED_IN_SOFTIRQ_READ  (1 << 3)
#define LOCKF_ENABLED_HARDIRQ       (1 << 6)  // 在允许 hardirq 时获取
#define LOCKF_ENABLED_SOFTIRQ       (1 << 7)
// ...
```

### 5.2 `mark_usage()`

```c
// __lock_acquire → mark_usage:
static int mark_usage(struct task_struct *curr, struct held_lock *hlock, int check)
{
    // 根据当前 irq_context 设置:
    //   hardirq 上下文         → USED_IN_HARDIRQ
    //   softirq 上下文         → USED_IN_SOFTIRQ
    //   irq 使能 (非中断)      → ENABLED_HARDIRQ
    if (!hlock->trylock) {
        if (hlock->read) {
            if (curr->hardirq_context)
                mark_lock(curr, hlock, LOCK_USED_IN_HARDIRQ_READ);
            if (curr->softirq_context)
                mark_lock(curr, hlock, LOCK_USED_IN_SOFTIRQ_READ);
        } else {
            if (curr->hardirq_context)
                mark_lock(curr, hlock, LOCK_USED_IN_HARDIRQ);
            if (curr->softirq_context)
                mark_lock(curr, hlock, LOCK_USED_IN_SOFTIRQ);
        }
    }
    // ...ENABLED 位: 在 irq 使能时获取
    if (curr->hardirqs_enabled)
        mark_lock(curr, hlock, LOCK_ENABLED_HARDIRQ);
    // ...
}
```

### 5.3 `check_irq_usage()` — IRQ 死锁检测

```c
// check_prev_add → check_irq_usage:
//   prev->usage_mask & USED_IN_HARDIRQ && next->usage_mask & ENABLED_HARDIRQ?
//   → 如果 A 在 hardirq 中被获取, 但 B 的获取不关 hardirq
//     那么 hardirq 中可以 A → B 但进程路径 B → A → 死锁!

static int check_irq_usage(struct task_struct *curr,
                            struct held_lock *prev, struct held_lock *next)
{
    // BFS 向后搜索: 从 prev 出发沿 locks_before 找 usage 冲突
    // BFS 向前搜索: 从 next 出发沿 locks_after 找 usage 冲突
    //   → mark_lock_irq() 负责 usage bit 的传播
    //   → 如果发现 safe→unsafe 则报告死锁
}
```

---

## 六、链式哈希缓存

### 6.1 为什么需要 chain key

```
CPU0: A → B → C     (chain: A→B→C)
CPU1: A → C          (chain: A→C)

★ 这两条链不同 → 需要不同的验证
```

每次 `__lock_acquire` 计算链式哈希：

```c
// 起点
chain_key = INITIAL_CHAIN_KEY;   // (0ULL)

// 每步迭代
// kernel/locking/lockdep.c
#define iterate_chain_key(key1, key2) \
    (((key1) << MAX_LOCKDEP_KEYS_BITS) ^ \
     (key1) >> (64 - MAX_LOCKDEP_KEYS_BITS) ^ \
     (key2))
```

### 6.2 `lookup_chain_cache_add()`

```c
// 验证前, 先查 chain hash:
//   如果 chain_key 已存在 → 之前已验证过, 跳过
//   如果是新的 → 需要做完整 validate_chain
//     → 加 graph_lock
//     → check_prev_add → BFS 验证
//     → 解锁 graph_lock
```

---

## 七、Lockdep Splat 解读

典型报错:

```
======================================================
WARNING: possible circular locking dependency detected
6.1.141 #1 Tainted: G           O
------------------------------------------------------
my_driver/1234 is trying to acquire lock:
 (&dev->lock){+.+.}-{3:3}, at: my_func+0x14/0x100

but task is already holding lock:
 (&other_lock){+.+.}-{3:3}, at: my_other_func+0x20/0x200

which lock already depends on the new lock.

-> existing dependency chain (in reverse order):
               → &dev->lock (S1)
               → &other_lock (S2)           // 之前: other_lock → dev_lock

-> other info that might help us debug this:
 Possible unsafe locking scenario:

       CPU0                    CPU1
       ----                    ----
  lock(&other_lock);
                               lock(&dev->lock);
                               lock(&other_lock);  // ← 循环!
  lock(&dev->lock);

 *** DEADLOCK ***

stack backtrace:
 [<...>] check_prev_add+0x.../0x...
 [<...>] __lock_acquire+0x.../0x...
 [<...>] lock_acquire+0x.../0x...
 [<...>] _raw_spin_lock+0x.../0x...
```

每个字段含义:

| 字段 | 含义 |
|------|------|
| `{+.+.}` | lockdep 依赖类型位: `+` = 读/写, `.` = 递归位, `+` = 硬/软 IRQ |
| `-{3:3}` | wait_type: 第一个数字是 inner, 第二个是 outer, 越大表示越"重" |
| `Tainted: G O` | G=GPL, O=外部模块 |

---

## 八、实践指南

### 8.1 内核配置

```c
CONFIG_LOCKDEP=y             // 基本 lockdep
CONFIG_PROVE_LOCKING=y       // 完整证明 + BFS
CONFIG_LOCK_STAT=y           // 锁统计
CONFIG_DEBUG_LOCKDEP=y       // 内部一致性检查 (慢)
CONFIG_LOCK_TORTURE_TEST=m   // 锁压力测试
```

### 8.2 常见错误

| 错误 | 含义 |
|------|------|
| `possible circular locking dependency detected` | 环路, 有死锁可能 |
| `possible irq lock inversion dependency detected` | IRQ 上下文导致的 A-B-B-A |
| `INFO: inconsistent lock state` | 锁在 irq 开启/关闭时行为不一致 |
| `INFO: possible recursive locking detected` | 同一线程试图再次获取同一锁 (非嵌套) |
| `BUG: MAX_LOCK_DEPTH too low!` | 锁嵌套太深, 需增大 MAX_LOCK_DEPTH |

### 8.3 lockdep 注释

```c
// raw_spin_lock_init: 从 __key 的地址确定 class
#define raw_spin_lock_init(lock)
    do {
        static struct lock_class_key __key;
        __raw_spin_lock_init((lock), #lock, &__key, LD_WAIT_SPIN);
    } while (0)

// 锁类划分:
//   每个 DEFINE_SPINLOCK(foo) 在 .data 段,
//   因此不同文件中的 foo 是不同 class
//   ★ 但同一文件内所有 DEFINE_SPINLOCK(foo) 的 foo 都相同 key

// 避免 lockdep 警告:
//   1. 加锁顺序全局一致
//   2. 中断中只用 spin_lock_irqsave (展示关闭 IRQ)
//   3. 嵌套锁用 spin_lock_nest_lock
//   4. 释放顺序与获取顺序相反
```

### 8.4 运行时调试

```bash
# 查看 lockdep 统计
cat /proc/lockdep                # 所有锁类
cat /proc/lockdep_stats          # 统计信息
cat /proc/lock_stat              # 锁等待/持有时间 (需要 CONFIG_LOCK_STAT)

# 内核日志:
dmesg | grep "LOCKDEP\|lockdep\|possible deadlock"

# lockdep 没有 splat → 不一定没有死锁 (可能场景未触发)
# 但 lockdep 没有 splat + 没有死锁 → 大概率没有死锁
```

### 8.5 lockdep 与线程 IRQ

```c
// Lockdep + threaded IRQ 注意事项:

// request_threaded_irq 注册的 handler 在 hardirq 中运行
// → lockdep 认为 handler 中取锁是在 hardirq context
// → 如果某个锁在 hardirq 中获取, 且在进程上下文中也获取
//   但进程上下文中不禁用 IRQ → lockdep 报告 warning

// 解决方法: 使用 spin_lock_irqsave 而非 spin_lock

// IRQF_ONESHOT: threaded handler 运行时中断线仍被 mask
// → lockdep 不视 thread_fn 为 hardirq context
// → thread_fn 中可以用 mutex (但通常不建议)
```

---

## 九、思考题

1. `check_prev_add` 中为什么先 `check_noncircular` 再 `check_redundant`？验证顺序有讲究吗？

2. 为什么 trylock 不参与依赖图？试想一个场景：trylock 导致的 A-B-A 模式如何不会被 lockdep 检测到？

3. Lockdep 的 BFS 使用 `MAX_CIRCULAR_QUEUE_SIZE = 4096`。如果依赖图非常大（深度嵌套模块），搜索可能被截断？结果会怎样？

4. 两个不同的 `DEFINE_SPINLOCK(my_lock)` 实例（在同一个 .c 文件中），lockdep 如何区分它们？如果不能区分，会有什么后果？

5. `lockdep_map` 中的 `wait_type_outer` 和 `wait_type_inner` 如何防止 spinlock 中取 mutex？

---

## 相关笔记

- [[bsp-interrupt-concurrency]] — 阶段三：中断处理 + 并发
- [[bsp-interrupt-fullpath-deep]] — 中断处理完整路径
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
