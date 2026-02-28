---
title: Go routine调度
p: computer/golang/routine
date: 2019-09-07 11:34:46
tags:
  - Go
  - 调度
---
goroutine（有人也称之为协程）本质上go的用户级线程的实现，这种用户级线程是运行在内核级线程之上。当我们在go程序中创建goroutine的时候，我们的这些routine将会被分配到不同的内核级线程中运行。一个内核级线程可能会负责多个routine的运行。而保证这些routine在内内核级线程安全、公平、高效运行的工作，就由调度器来实现。
<!--more-->


&#8195;&#8195;go routine的调度原理和操作系统的线层调度是比较相似的。这里我们将介绍go routine的相关知识。

## goroutine
&#8195;&#8195;goroutine（有人也称之为协程）本质上go的用户级线程的实现，这种用户级线程是运行在内核级线程之上。当我们在go程序中创建goroutine的时候，我们的这些routine将会被分配到不同的内核级线程中运行。一个内核级线程可能会负责多个routine的运行。而保证这些routine在内内核级线程安全、公平、高效运行的工作，就由调度器来实现。

## Go的调度
&#8195;&#8195;Go的调度主要有四个结构组成，分别是：
- G：goroutine的核心结构，包括routine的栈、程序计数器pc、以及一些状态信息等；
- M：内核级线程。goroutine在M上运行。M中信息包括：正在运行的goroutine、等待运行的routine列表等。当然也包括操作系统线程相关信息，这些此处不讨论。
- P：processor，处理器，只要用于执行goroutine，维护了一个goroutine列表。其实P是可以从属于M的。当P从属于（分配给）M的时候，表示P中的某个goroutine得以运行。当P不从属于M的时候，表示P中的所有goroutine都需要等待被安排到内核级线程运行。
- Sched：调度器，存储、维护M，以及一个全局的goroutine等待队列，以及其他状态信息。



<p align="center">
<img src="img/go_architecture.png" alt="调度架构" style="width:600px"/>
</p>

### Go程序的启动过程：
- 初始化Sched：一个存储P的列表pidle。P的数量可以通过GOMAXPROCS设置，默认和计算机核数相同；
- 创建第一个goroutine。这个goroutine会创建一个M，这个内核级线程（sysmon）的工作是对goroutine进行监控。之后，这个goroutine开始我们在main函数里面的代码，此时，该goroutine就是我们说的主routine。


### 创建goroutine：
- goroutine创建时指定了代码段
- 然后，goroutine被加入到P中去等待运行。
- 这个新建的goroutine的信息包含：栈地址、程序计数器

### 创建内核级线程M
&#8195;&#8195;内核级线程由go的运行时根据实际情况创建，我们无法再go中创建内核级线程。那什么时候回创建内核级线程呢？当前程序等待运行的goroutine数量达到一定数量及存在空闲（为被分配给M）的P的时候，Go运行时就会创建一些M，然后将空闲的P分配给新建的内核级线程M，接着才是获取、运行goroutine。创建M的接口函数如下：
```
// 创建M的接口函数
void newm(void (*fn)(void), P *p)

// 分配P给M
if(m != &runtime·m0) {
	acquirep(m->nextp);
	m->nextp = nil;
}
// 获取goroutine并开始运行
schedule();
```


M的运行
```
static void schedule(void)
{
	G *gp;

	gp = runqget(m->p);
	if(gp == nil)
		gp = findrunnable();

  // 如果P的列表不止一个goroutine，且调度器中有空闲的的P，就唤醒其他内核级线程M
	if (m->p->runqhead != m->p->runqtail &&
		runtime·atomicload(&runtime·sched.nmspinning) == 0 &&
		runtime·atomicload(&runtime·sched.npidle) > 0)  // TODO: fast atomic
		wakep();
  // 执行goroutine
	execute(gp);
}
```

- runqget: 从P中获取goroutine即gp。gp可能为nil（如M刚创建时P为空；或者P的goroutine已经运行完了）。
- findrunnable：寻找空闲的goroutine（从全局的goroutine等待队列获取goroutine；如果所有goroutine都已经被分配了，那么从其他M的P的goroutine的goroutine列表获取一半）。如果获取到goroutine，就将他放入P中，并执行它；否则没能获取到任何的goroutine，该内核级线程进行系统调用sleep了。
- wakep：当当前内核级线程M的P中不止一个goroutine且调度器中有空闲的的P，就唤醒其他内核级线程M。（为了找些空闲的M帮自己分担）。


### 调度
&#8195;&#8195;前面说的是G，M是怎样创建的以及什么时候创建、运行。那么goroutine在M是是怎样进行调度的呢？这个才是goroutine的调度核心问题，即上面代码中的schedule。在说调度之前，我们必须知道goroutine的状态有什么，以及各个状态之间的关系。

![goroutine状态迁移图](https://pic2.zhimg.com/80/v2-36ea8b5fb3b947eee2434092a4c1d0c9_hd.jpg)

- Gidle：创建中的goroutine，实际上这个状态没有什么用；
- Grunnable：新创建完成的goroutine在完成了资源的分配及初始化后，会进入这个状态。这个新创建的goroutine会被分配到创建它的M的P中；
- Grunning：当Grunnable中的goroutine等到了空闲的cpu或者到了自己的时间片（Go1.2之后实现了一定程度的抢占式调度）的时候，就会进入Grunning状态。这个装下的goroutine可以被前文提到的findrunnable函数获取；
- Gwaiting：当正在运行的goroutine进行一些阻塞调用的时候，就会从Grunning状态进入Gwaiting状态。常见的调用有：写入一个满的channel、读取空的channel、IO操作、定时器Ticker等。当阻塞调用完成后，goroutine的状态就会从Gwaiting转变为Grunnable；
- Gsyscall：当正在运行的goroutine进行系统调用的时候，其状态就会转变为Gsyscall。当系统调用完成后goroutine的状态就会变为Grunnable。（前文提到的sysmon进程会监控所有的P，如果发现有的P的系统调用是阻塞式的或者执行的时间过长，就会将P从原来的M分离出来，并新建一个M，将P分配给这个新建的M）。

综上，触发调度的事件有：
- channel的阻塞读写；
- go routine的调用，如go func(){xxx}()；
- 阻塞式的系统调用，如耗时IO等；
- GC。（GC时间的时候goroutine会暂停运行）


&#8195;&#8195;本文并不属于原创，只是汇总、修改了个人看了的几篇blog而已。

## Ref
- https://zhuanlan.zhihu.com/p/29698463
- http://skoo.me/go/2013/11/29/golang-schedule?utm_campaign=studygolang.com&utm_medium=studygolang.com&utm_source=studygolang.com
- https://povilasv.me/go-scheduler/
- https://medium.com/@riteeksrivastava/a-complete-journey-with-goroutines-8472630c7f5c
- https://rakyll.org/scheduler/



# 当futex_wake的时候，唤醒的是M还是G？
你这个问题问到了**Go 运行时和 Linux 内核交互的核心边界**——结论先给你：
`futex_wake` 是**Linux 内核系统调用**，它唤醒的是**操作系统线程（M）**；但 Go 运行时会在被唤醒的 M 上，调度对应的 goroutine（G）执行。

我用「内核层 + Go 运行时层」的双层视角，把这个过程拆得明明白白，你会瞬间打通 Go 协程和系统线程的关联。

---

## 一、先明确核心概念（避免混淆）
| 层级       | 实体       | 含义                                                                 |
|------------|------------|----------------------------------------------------------------------|
| Linux 内核 | 线程（M）  | 操作系统级别的执行单元，对应内核的 `task_struct`，是 CPU 调度的最小单位 |
| Go 运行时  | 协程（G）  | Go 语言的轻量级执行单元，由 Go 运行时调度，不直接对接内核             |
| Go 运行时  | 线程（M）  | Go 对操作系统线程的封装，是 G 和内核之间的“桥梁”                      |
| 系统调用   | futex      | 只认内核线程（M），完全感知不到 G 的存在                             |

核心结论：
**内核不懂什么是 G，它只管理 M；Go 运行时把 G 绑定到 M 上执行，futex 操作的是 M，G 的唤醒由 Go 运行时接管。**

---

## 二、futex_wake 唤醒的完整流程（Go 锁场景）
以 `sync.Mutex` 阻塞/唤醒为例，拆解「G 阻塞 → M 挂起 → futex_wake 唤醒 M → G 恢复执行」的全过程：

### 步骤 1：G 抢锁失败，触发 M 阻塞
1. Goroutine G1 抢锁失败，自旋也没抢到，调用 `semacquire`（Go 运行时函数）；
2. `semacquire` 会把 G1 标记为“等待态”，并将对应的 M1（绑定 G1 的操作系统线程）调用 `futex_wait` 系统调用；
3. 内核收到 `futex_wait` 后，将 M1 从“运行态”转为“睡眠态”，从 CPU 上剥离，释放内核调度资源；
   - 此时 G1 跟着 M1 一起“休眠”，但内核完全不知道 G1 的存在，只看到 M1 被挂起。

### 步骤 2：其他 G 解锁，调用 futex_wake
1. Goroutine G2 解锁 `sync.Mutex`，调用 `semrelease`（Go 运行时函数）；
2. `semrelease` 调用 `futex_wake` 系统调用，告诉内核：“唤醒绑定在这个 futex 变量上的 M”；
3. 内核执行 `futex_wake`：
   - 找到该 futex 变量对应的等待队列（里面有挂起的 M1）；
   - 将 M1 从“睡眠态”转为“就绪态”，放入内核的就绪队列；
   - 内核调度器后续会把 M1 重新分配到 CPU 上运行。

### 步骤 3：M 被唤醒后，Go 运行时唤醒 G
1. M1 被内核调度到 CPU 上，恢复执行（从 `futex_wait` 系统调用返回）；
2. M1 回到 Go 运行时的 `semacquire` 函数中，Go 运行时感知到 M1 已唤醒；
3. Go 运行时将 G1 从“等待态”转为“就绪态”，放入 Go 的调度队列；
4. Go 调度器（P）将 G1 重新绑定到 M1（或其他空闲 M）上，G1 继续尝试抢锁、执行业务逻辑。

---

## 三、关键细节（为什么内核只认 M）
1. **futex 是内核接口，只对接内核线程**：
   - futex 的等待队列中存储的是内核线程的 `task_struct`（对应 Go 的 M），不是 Go 的 G；
   - 内核没有任何关于 G 的数据结构，也无法直接操作 G，只能通过 M 间接影响 G。

2. **G 的阻塞/唤醒是 Go 运行时的逻辑**：
   - G 的“等待态”“就绪态”是 Go 运行时的标记，和内核线程状态无关；
   - 即使 M 被内核唤醒，G 也不一定能立刻执行（还需 Go 调度器分配 P、抢锁等）。

3. **M 和 G 的对应关系**：
   - 正常情况下：1 个 M 绑定 1 个 G 执行；
   - 阻塞时：M 被内核挂起，G 被 Go 标记为等待；
   - 唤醒后：M 恢复运行，Go 调度 G 继续执行。

---

## 四、一句话总结（最核心）
- **futex_wake（内核层）**：唤醒的是**操作系统线程 M**；
- **Go 运行时（用户层）**：在被唤醒的 M 上，把对应的 G 从等待态转为就绪态，最终让 G 恢复执行。

内核只负责“唤醒线程”，而 Go 运行时负责“唤醒协程”——这是 Go 协程轻量化、低调度成本的关键：内核只需要管理少量 M（通常和 CPU 核心数一致），而 Go 可以管理上万 G。

---

## 补充：为什么 Go 不直接唤醒 G？
因为内核无法感知 G，Go 必须借助 M 作为“中介”：
1. G 是用户态的轻量级协程，无需内核参与调度，节省上下文切换成本；
2. 若让内核感知 G，会失去 Go 调度器的灵活性（比如抢占式调度、G 池管理）；
3. futex 只需要挂起/唤醒少量 M，就能支撑大量 G 的阻塞/唤醒，这是 Go 高并发的核心设计。


# 有没有可能所有M都被挂起？
你问的这个问题直击 Go 运行时调度的核心边界——结论先明确：
**理论上存在所有 M 被挂起的可能性，但 Go 运行时通过「动态创建新 M」「抢占式调度」「监控线程兜底」三大机制，从设计上杜绝了这种情况发生，保证程序不会彻底卡死。**

我会从「为什么会出现 M 全挂起」「Go 如何防住这个问题」「极端场景的兜底逻辑」三个维度讲透，让你理解 Go 调度器的“自救”机制。

---

## 一、先搞懂：什么情况下会出现“所有 M 被挂起”的风险？
M 被挂起的核心场景是：**M 绑定的 G 执行了阻塞系统调用（如 futex_wait、sleep、网络/磁盘 IO），导致 M 被内核挂起**。

假设有如下场景（无防护时会触发 M 全挂起）：
1. 你的 Go 程序设置了 `GOMAXPROCS = 4`（对应 4 个 P，调度器最多同时运行 4 个 M）；
2. 这 4 个 M 上的 G 都执行了 `sync.Mutex.Lock()` 且抢不到锁，最终全部调用 `futex_wait` 被内核挂起；
3. 此时没有任何空闲 M 来执行“解锁”的 G（解锁 G 也需要 M 才能运行）；
4. 结果：所有 M 被挂起，程序陷入“死等”，无法继续执行。

### 关键前提：
M 被挂起的本质是「G 触发了阻塞系统调用」，且**Go 运行时能识别这种阻塞**（如 futex、网络 IO、文件 IO）——这是 Go 能“自救”的基础。

---

## 二、Go 运行时的核心防护机制（杜绝 M 全挂起）
Go 调度器从 1.14 引入抢占式调度后，通过三大机制彻底解决了 M 全挂起的问题：

### 机制 1：**阻塞系统调用时，动态创建新 M 绑定 P**
这是最核心的“兜底”逻辑，流程如下：
1. G1 在 M1 上执行阻塞系统调用（如 futex_wait），Go 运行时在系统调用前会检测到“该调用会阻塞 M1”；
2. 运行时立刻将 M1 绑定的 P 剥离，然后**创建一个新的 M2**，将 P 绑定到 M2 上；
3. M2 从就绪 G 队列中取一个 G（比如“解锁”的 G2）执行；
4. 原 M1 被内核挂起，但 P 不会闲置，新 M2 继续处理其他 G；
5. 当 M1 被唤醒后，若有空闲 P 则继续运行，若无则把 G1 放入就绪队列，M1 进入缓存池（或销毁）。

#### 关键代码逻辑（简化版）：
```go
// 系统调用前的预处理（runtime/syscall.go）
func entersyscall() {
    // 剥离当前 M 绑定的 P
    releasep()
    // 标记 G 进入系统调用
    getg().m.syscall = true
    // 运行时会检测到 P 空闲，触发新 M 创建
}
```

### 机制 2：**监控线程（sysmon）兜底，强制抢占+清理阻塞 M**
Go 运行时启动时会创建一个**独立的监控线程（sysmon）**，它不占用 P，由内核直接调度，核心职责之一就是“防止所有 M 被挂起”：
1. **定时扫描**：每 10ms 扫描一次所有 M，检测是否有 M 长时间阻塞（如超过 10ms）；
2. **强制抢占**：若发现某个 G 长时间占用 M（如死循环、未触发系统调用的阻塞），会发送信号（SIGURG）给 M，触发 G 的抢占式调度，将 G 从 M 上剥离，让 M 处理其他 G；
3. **清理僵尸 M**：回收长时间闲置的 M，避免资源浪费；
4. **唤醒阻塞 M**：检测到“所有 M 被挂起”的风险时，会主动唤醒缓存池中的 M，或创建新 M。

### 机制 3：**抢占式调度（1.14+），避免 G 霸占 M**
在 Go 1.14 之前，若 G 执行无限循环（无系统调用、无函数调用），会一直霸占 M，导致 P 无法调度其他 G；1.14 引入的抢占式调度解决了这个问题：
1. 当 G 运行超过 10ms 且未触发调度点（如函数调用、系统调用），sysmon 会向 M 发送 SIGURG 信号；
2. M 收到信号后，暂停当前 G 的执行，将其放回就绪队列；
3. M 重新绑定 P，调度其他 G 执行，避免 M 被单个 G 霸占而“假死”。

---

## 三、极端场景验证：真的不会出现所有 M 被挂起吗？
我们假设最极端的场景：
- `GOMAXPROCS=1`（只有 1 个 P）；
- 当前 M1 上的 G1 调用 `futex_wait` 被挂起；
- 此时是否有 M 来执行“解锁”的 G2？

### 执行流程：
1. G1 执行 `futex_wait` 前，Go 运行时剥离 M1 的 P；
2. 运行时检测到 P 空闲，立刻创建新 M2，将 P 绑定到 M2；
3. M2 从就绪队列中取出 G2（解锁逻辑）执行，G2 调用 `futex_wake` 唤醒 M1；
4. M1 被内核唤醒后，若 P 已被 M2 占用，则 G1 进入就绪队列，M1 进入缓存池；
5. 最终：M2 正常运行，M1 被唤醒后等待复用，不会出现“所有 M 挂起”。

### 唯一例外（人为作死）：
如果你的代码**禁用了 Go 运行时的系统调用检测**（如通过 cgo 调用未被封装的阻塞系统调用），且所有 M 都被这类调用挂起，此时 Go 运行时无法感知，可能出现 M 全挂起——但这属于非标准用法，正常 Go 代码不会触发。

---

## 四、核心结论（关键点）
1. **理论风险**：若没有防护，所有 M 可能因阻塞系统调用被内核挂起，导致程序卡死；
2. **核心防护**：Go 运行时在 M 触发阻塞系统调用时，会**动态创建新 M 接管 P**，保证总有 M 能执行就绪 G；
3. **兜底保障**：监控线程（sysmon）定时扫描，强制抢占霸占 M 的 G，避免 P 闲置；
4. **最终结果**：正常 Go 程序中，**不可能出现所有 M 被挂起的情况**，调度器会自动“自救”。

简单来说：Go 调度器的设计目标就是“让 P 永远不闲置”——只要 P 有任务，就一定会有可用的 M 来执行，哪怕需要动态创建新 M。这也是 Go 能支撑高并发的核心原因之一：不会因为部分 G 阻塞，就导致整个程序停摆。

---

补充：M 的创建上限
Go 对 M 的创建有默认上限（默认 10000），可通过 `runtime/debug.SetMaxThreads` 调整，但正常场景下远达不到这个上限——因为阻塞的 M 被唤醒后会被复用，不会无限制创建。