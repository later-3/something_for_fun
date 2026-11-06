# 22 RTOS 实时操作系统

> **模块一：嵌入式系统和网络（40 分）** 的一部分。你没做过 FreeRTOS，这份是从零建立的——但笔试考的是**概念**，不是让你调板子。
> 检验标准：能讲出优先级反转的三任务场景和两种解法，能说清信号量 vs 互斥锁的本质区别，能说出上下文切换保存什么。

---

# 第一部分：考点（24 条）

## 一、RTOS 基础

### 1.1 什么是 RTOS ★★
**RTOS = Real-Time Operating System**：任务的响应时间**有确定上界**（deadline 可保证）。

| 类型 | 定义 | 例子 |
|---|---|---|
| **硬实时** | 错过 deadline = 系统失败 | 安全气囊点火、电机伺服、ABS、飞行控制 |
| **软实时** | 偶尔超时可接受 | 视频播放丢帧、UI 响应、数据采集 |

### 1.2 RTOS vs Linux ★★
| | Linux | RTOS |
|---|---|---|
| 目标 | 吞吐量、公平 | **确定性**（延迟有上界） |
| 调度 | 分时（CFS） | 优先级抢占 |
| 调度延迟 | 几十 ms 且**不确定** | 微秒级且**确定** |
| 内存 | 有 MMU、虚拟内存 | 通常无 MMU，直接物理地址 |
| 体积 | 大（MB 级） | 小（KB 级） |
| 应用 | 通用计算 | 专用控制 |

**为什么 Linux 不实时**：CFS 追求公平、内核有不可抢占区段、缺页中断不可预测、cache/TLB 影响。实时补丁（PREEMPT_RT）可大幅改善但仍非硬实时。

### 1.3 RTOS vs 裸机（前后台）★
| | 裸机（前后台） | RTOS |
|---|---|---|
| 任务管理 | 单任务：主循环 + 中断 | 多任务并发、优先级 |
| 实时响应 | 依赖主循环结构，一个慢函数拖累全部 | 高优先级任务可抢占 |
| 复杂度 | 低，流程直观 | 高，需理解调度/栈/同步 |
| 适合 | 简单系统 | 多任务、复杂时序 |

### 1.4 常见 RTOS
| RTOS | 特点 |
|---|---|
| **FreeRTOS** | 最流行、免费（MIT）、文档完善、可裁剪到很小 |
| **Zephyr** | Linux 基金会，功能全（含网络/文件系统），配置复杂 |
| **RT-Thread** | 国产开源，组件丰富（文件系统/GUI/网络），生态好 |
| **µC/OS** | 商业（需授权），有安全认证（DO-178C/ IEC 61508），航天医疗 |
| **VxWorks** | 商业闭源，高可靠，国防航天 |
| **ThreadX（Azure RTOS）** | 微软，认证齐全 |

---

## 二、任务与状态

### 2.1 任务的四种状态 ★★★
| 状态 | 含义 |
|---|---|
| **Running（运行）** | 正在占用 CPU（单核同一时刻只有一个） |
| **Ready（就绪）** | 万事俱备，只差 CPU |
| **Blocked（阻塞）** | 等事件（信号量/队列/延时），即使 CPU 空闲也不运行 |
| **Suspended（挂起）** | 被 `vTaskSuspend()` 显式挂起，调度器完全不管（不响应任何事件） |

### 2.2 状态转换图 ★★★（必须能画）
```
                 调度器选中
        ┌─────────┐ ────────→ ┌─────────┐
        │  Ready   │ ←──────── │ Running │
        │  就绪    │  被抢占/   │  运行    │
        └─────────┘  时间片到  └─────────┘
             ↑                     │
             │ 事件到达/           │ 等事件/延时
             │ 延时到期            ↓
        ┌─────────┐           ┌─────────┐
        │ Blocked │           │ (阻塞)   │
        │ 阻塞    │ ←──────── │          │
        └─────────┘           └─────────┘

      任意状态 ⇄ Suspended（vTaskSuspend / vTaskResume）
```

**考点问答**：
- Running → Ready：被更高优先级任务抢占，或时间片用完
- Running → Blocked：主动等待事件（信号量、队列、延时）
- Blocked → Ready：事件到达（信号量被 give、队列有数据、延时到期）

### 2.3 TCB（任务控制块）★
每个任务一个 TCB，包含：**栈顶指针**（关键）、任务名、优先级、状态、事件列表项、栈起始与大小、运行统计。
**关键**：上下文切换时只需保存/恢复栈指针——因为所有寄存器都在栈里。

### 2.4 任务栈 ★
每个任务有**独立栈**，用于局部变量、函数调用、寄存器保存。
**大小评估**：局部变量总和 + 函数调用深度 × 每层开销 + 中断嵌套需求 + 余量（建议 20%）。
**FreeRTOS 栈大小单位是"字"（word，4 字节）**：`xTaskCreate(..., 128, ...)` = 512 字节。

---

## 三、调度

### 3.1 FreeRTOS 调度策略 ★★
- **抢占式（preemptive，默认）**：高优先级任务一旦就绪，**立即抢占**当前任务
- **同优先级时间片轮转**（需 `configUSE_TIME_SLICING = 1`）
- 可选协作式（cooperative）：任务主动让出（`taskYIELD()`）才切换

**核心原则**：**永远运行优先级最高的就绪任务**。

### 3.2 调度时机 ★
1. 任务主动阻塞（等信号量/延时）
2. 更高优先级任务就绪（被中断唤醒）
3. 时间片用完（同优先级轮转）
4. 任务主动 yield
5. 任务被删除/挂起

### 3.3 优先级的设计原则 ★
- **Rate Monotonic（速率单调）**：周期越短（频率越高）的任务优先级越高
- 关键任务（安全相关）优先级最高
- 耗时任务优先级较低（避免长期霸占 CPU）
- 优先级数量：`configMAX_PRIORITIES`（通常 5~32），越多越耗 RAM

### 3.4 上下文切换（Context Switch）★★★
**保存什么**（Cortex-M）：
| 寄存器 | 作用 |
|---|---|
| **R0-R12** | 通用寄存器 |
| **SP**（R13） | 栈指针 |
| **LR**（R14） | 链接寄存器（返回地址） |
| **PC**（R15） | 程序计数器 |
| **xPSR** | 程序状态寄存器（含标志位） |

**怎么保存**：压入**当前任务的栈**，TCB 记录新的栈顶指针；切换时从下一个任务的栈弹出。
**Cortex-M 的特殊设计**：硬件自动保存 R0-R3/R12/LR/PC/xPSR，软件只需保存 R4-R11 → 切换更快。

**在哪个异常里切换**：**PendSV**（可挂起系统服务异常）
- PendSV 被设为**最低优先级**，保证不会打断其他中断处理（在最安全的时刻切换）
- 触发方式：任务调用 `taskYIELD()` 或内核置 PendSV 挂起位

**切换开销**：约几微秒（保存/恢复寄存器 + 可能 cache 影响）。

### 3.5 空闲任务（Idle Task）★
- 调用 `vTaskStartScheduler()` 时**自动创建**，优先级 **0（最低）**
- 作用：① 保证 CPU 永远有任务可跑 ② 回收被删除任务的资源 ③ 提供低功耗入口（Idle Hook）
- **必须存在**，不能删除

### 3.6 系统节拍（Tick）★
- SysTick 定时器周期性中断（`configTICK_RATE_HZ`，通常 1000 = 1ms）
- 作用：任务延时计时、同优先级时间片轮转、软件定时器
- **tick 越大**：时间精度低但 CPU 开销小；**tick 越小**：时间精度高但中断频繁

---

## 四、任务间通信与同步 ★★★（最高频）

### 4.1 二值信号量 vs 互斥锁（Mutex）★★★
| | **互斥锁 Mutex** | **二值信号量 Binary Semaphore** |
|---|---|---|
| **所有权** | **有**（谁加锁谁解锁，其他任务不能解锁） | **无**（任何任务/ISR 都能 give/take） |
| **优先级继承** | **有**（FreeRTOS 的 mutex 自带） | 无 |
| **用途** | **保护临界资源**（互斥） | **事件通知 / 同步**（ISR → 任务） |
| **递归加锁** | 支持（递归互斥量） | 不适用 |
| **创建 API** | `xSemaphoreCreateMutex()` | `xSemaphoreCreateBinary()` |

**判断法则**：**保护共享资源用 mutex，通知事件用 semaphore。**

### 4.2 计数信号量
- 计数 = 可用资源数量
- **用途**：① 管理多个同类资源（如 3 个缓冲区）② 生产者-消费者计数
- API：`xSemaphoreCreateCounting(max, initial)`

### 4.3 队列（Queue）★★
- **任务间传数据**，FIFO
- **拷贝传递**（数据复制进队列，不是传指针）
- 长度与元素大小在创建时指定
- **阻塞语义**：队满时发送者阻塞（或超时），队空时接收者阻塞（或超时）
- API：`xQueueCreate` / `xQueueSend` / `xQueueReceive` / `xQueueSendFromISR`

**队列 vs 信号量**：
| | 队列 | 信号量 |
|---|---|---|
| 传数据 | 有（可带任意大小的数据） | 无（只有计数） |
| 用途 | 任务间通信 | 同步/互斥 |
| 拷贝 | 深拷贝到队列 | 无拷贝 |

### 4.4 任务通知（Task Notification）★
FreeRTOS 特有，**最快最省内存**——直接写到目标任务 TCB 里的一个 32 位值，**无需创建内核对象**。
- 比信号量快约 45%，省 RAM
- **限制**：只能**一对一**（发送方指定接收任务）；不能多任务等同一个通知
- API：`xTaskNotifyGive` / `ulTaskNotifyTake` / `xTaskNotify` / `xTaskNotifyWait`

### 4.5 事件组（Event Group）★
- 一个任务可以等**多个事件的组合**（AND / OR），用位图表示（通常 24 位可用）
- 用途：多个条件都满足才继续（如"传感器就绪 AND 通信就绪"）
- API：`xEventGroupSetBits` / `xEventGroupWaitBits`

### 4.6 选择表（考试常问"该用哪个"）★★
| 需求 | 用什么 |
|---|---|
| 保护共享变量/临界区 | **互斥锁** |
| ISR 通知任务"数据来了" | **二值信号量** / 任务通知 |
| 任务间传数据 | **队列** |
| 限制同时访问数（最多 3 个） | **计数信号量** |
| 等多个条件同时满足 | **事件组** |
| 简单单任务通知（一对一，追求效率） | **任务通知** |

---

## 五、优先级反转 ★★★（必考）

### 5.1 场景（必须能完整讲述）★
三个任务：**H**（高优先级）、**M**（中优先级）、**L**（低优先级）。

1. **L** 获得锁，进入临界区
2. **H** 就绪，抢占 L → 但 H 也需要同一把锁 → **H 被阻塞，等 L 释放**
3. **M** 就绪 → M 优先级高于 L → **M 抢占 L 运行**
4. → **结果：H（最高优先级）被 M（中优先级）间接阻塞** ——这就是优先级反转

**危害**：高优先级任务的响应时间变得不可预测（取决于中优先级任务跑多久），破坏实时性。

### 5.2 两种解决方法 ★★★
| 方案 | 做法 | 特点 |
|---|---|---|
| **优先级继承（Priority Inheritance）** | 当 H 等 L 持有的锁时，**L 的优先级临时提升到 H 的优先级**，让它尽快跑完放锁；放锁后恢复原优先级 | FreeRTOS 的 **mutex 自带**（`xSemaphoreCreateMutex`）；只提升到等待者中的最高优先级 |
| **优先级天花板（Priority Ceiling）** | 每把锁预设一个"天花板优先级"（= 所有可能使用它的任务中最高者），任务一拿到锁就提升到该优先级 | 更严格，实时性分析更容易，但需要预先知道所有使用者 |

**经典案例（说出来加分）**：1997 年 **火星探路者号**（Mars Pathfinder）探测器反复重启——优先级反转导致低优先级任务长期持锁，高优先级任务（负责喂看门狗）拿不到锁，看门狗超时复位。NASA 远程打补丁把 mutex 改成支持优先级继承后解决。

### 5.3 与死锁的关系
优先级反转 ≠ 死锁（反转是"延迟"，死锁是"永久卡住"）。但 RTOS 同样存在死锁（四条件相同）。
**RTOS 上的死锁预防**：统一加锁顺序、带超时获取（`xSemaphoreTake(sem, timeout)`）、避免嵌套锁。

---

## 六、中断与 RTOS ★★

### 6.1 中断处理原则 ★★
- **短**：只做最紧急的（清标志、取数据、发信号）——耗时处理交给任务
- **不能做的事**：`printf`（慢、不可重入）、延时/`vTaskDelay`（会阻塞）、`malloc`（不可重入、耗时）、调用非 ISR 版本的 RTOS API、浮点运算（若不保存 FPU 上下文）

### 6.2 FromISR 版本 API ★★★
中断里**必须**用带 `FromISR` 后缀的 API：
```c
void USART1_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    // 从中断中释放信号量
    xSemaphoreGiveFromISR(xSem, &xHigherPriorityTaskWoken);

    // 若有更高优先级任务被唤醒，立即切换（不等到下一个 tick）
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```
**`xHigherPriorityTaskWoken` 的作用**：告诉移植层"这次操作是否唤醒了更高优先级的任务"。如果是 `pdTRUE`，`portYIELD_FROM_ISR` 会触发 PendSV，**中断返回后立即切到高优先级任务**，而不是等下一个调度点——保证实时性。

### 6.3 中断优先级与 RTOS 的配合 ★
- Cortex-M 的**硬件中断优先级数值越小越高**
- FreeRTOS 有 `configMAX_SYSCALL_INTERRUPT_PRIORITY`（通常设为 5）
- **优先级数值高于该阈值的中断**（即数值 <5，更紧急）**不能调用任何 RTOS API**——因为它们不被 RTOS 的临界区屏蔽，可能破坏内核数据结构
- **优先级数值 ≤ 阈值的**（即数值 ≥5，较不紧急）可以调用 `FromISR` 版本
- FreeRTOS 的 SysTick 和 PendSV 都设为**最低优先级**（15）

### 6.4 中断上下文中为什么不能睡眠 ★
中断上下文没有任务栈、没有 TCB，无法被调度器保存/恢复；睡眠会让 CPU 无法处理其他中断，且无法被唤醒。

---

## 七、内存管理 ★

### 7.1 FreeRTOS 五种堆方案 ★★
| 方案 | 特点 | 适用 |
|---|---|---|
| **heap_1** | 只能分配，不能释放（简单、无碎片） | 系统启动时一次性创建所有对象 |
| **heap_2** | 可释放，但**不合并**相邻空闲块（会碎片） | 已不推荐 |
| **heap_4** | 可释放 + **合并相邻空闲块** | **最常用，推荐** |
| **heap_5** | 同 heap_4，支持**多个不连续** RAM 区域 | RAM 分散的 MCU |
| **heap_3** | 包一层标准库 malloc/free（加锁） | 需要线程安全的标准库 |

### 7.2 嵌入式内存原则 ★★
1. **优先静态分配**：编译期确定，无碎片，最坏情况可分析
2. **用内存池**：预分配固定大小块，O(1) 分配，无碎片
3. **避免频繁 malloc/free**：长时间运行会碎片化
4. `xTaskCreateStatic()` / `xQueueCreateStatic()` 静态创建

### 7.3 栈溢出检测 ★★
**FreeRTOS 两种方法**（`configCHECK_FOR_STACK_OVERFLOW`）：
- **方法 1**（值 1）：任务切换时检查栈指针是否越界（简单，但可能漏掉"刚好写出去又回来"的情况）
- **方法 2**（值 2）：栈底填充魔数（`0xA5A5A5A5`，通常 16 字节），切换时检查魔数是否被破坏（更可靠）
- 触发时调用 `vApplicationStackOverflowHook()`（用户实现）

**调试手段**：
- `uxTaskGetStackHighWaterMark()`：查历史最小剩余栈（值越小越危险）
- 看 map 文件核算最坏栈用量
- MPU 保护栈区

---

## 八、其他

### 8.1 软件定时器 ★
- 由**定时器服务任务（Timer Service Task/Daemon Task）** 统一管理回调（回调在任务上下文执行，不是中断）
- 单次（one-shot）/ 周期（periodic）
- **回调不能阻塞**（会阻塞整个定时器任务）
- API：`xTimerCreate` / `xTimerStart` / `xTimerChangePeriod`

### 8.2 Tickless 低功耗 ★
- 空闲时不产生周期 tick 中断，让 MCU 进入深度睡眠
- 睡眠前算出"下一个任务最早到期时间"，用 RTC/低功耗定时器设定唤醒
- 唤醒后补偿 tick 计数
- `configUSE_TICKLESS_IDLE = 1`

### 8.3 Hook 函数 ★
| Hook | 何时调用 |
|---|---|
| `vApplicationIdleHook()` | 空闲任务中（低功耗、清理） |
| `vApplicationTickHook()` | 每个 tick 中断（必须极短） |
| `vApplicationMallocFailedHook()` | 内存分配失败 |
| `vApplicationStackOverflowHook()` | 检测到栈溢出 |
| `vApplicationDaemonTaskStartupHook()` | 调度器启动时 |

### 8.4 临界区保护 ★
```c
taskENTER_CRITICAL();      // 屏蔽中断 + 挂起调度（Cortex-M3 上是设置 BASEPRI）
// 临界区代码
taskEXIT_CRITICAL();
// 中断中使用：
UBaseType_t saved = taskENTER_CRITICAL_FROM_ISR();
// ...
taskEXIT_CRITICAL_FROM_ISR(saved);
```
**代价**：关中断会影响实时响应，临界区必须**极短**。

### 8.5 裁剪与配置（了解）
关键配置项：
| 配置 | 含义 |
|---|---|
| `configUSE_PREEMPTION` | 1 = 抢占式 |
| `configTICK_RATE_HZ` | 节拍频率（1000 = 1ms） |
| `configMAX_PRIORITIES` | 优先级数量 |
| `configMINIMAL_STACK_SIZE` | 最小任务栈（字） |
| `configTOTAL_HEAP_SIZE` | heap_4 的总堆大小 |
| `configMAX_SYSCALL_INTERRUPT_PRIORITY` | 可调用 RTOS API 的中断优先级阈值 |
| `configCHECK_FOR_STACK_OVERFLOW` | 栈溢出检测（0/1/2） |

---

# 第二部分：题目（30 题，答案已给）

### Q1 ★★ 什么是 RTOS？和 Linux 的核心区别？
**答案**：RTOS 保证任务响应时间有确定上界（deadline 可保证）。
区别：Linux 追求吞吐与公平，调度延迟几十 ms 且不确定；RTOS 追求确定性，调度延迟微秒级有上界；Linux 有 MMU/虚拟内存、体积大；RTOS 通常无 MMU、KB 级。

### Q2 ★ 硬实时和软实时的区别？各举一例。
**答案**：硬实时=错过 deadline 即系统失败（安全气囊、电机伺服、ABS）；软实时=偶尔超时可接受（视频播放丢帧、UI 响应）。

### Q3 ★ 裸机（前后台）和 RTOS 的区别？
**答案**：裸机单任务（主循环+中断），实时响应依赖主循环结构，一个慢函数拖累全部；RTOS 多任务并发、优先级抢占，高优先级任务可立即响应，但复杂度更高（调度/栈/同步）。

### Q4 ★★★ FreeRTOS 任务有哪几种状态？画出状态转换图。
**答案**：见 2.1/2.2。Running / Ready / Blocked / Suspended。
转换：Running→Ready（被抢占/时间片到）、Running→Blocked（等事件/延时）、Blocked→Ready（事件到达/延时到期）、任意状态⇄Suspended。

### Q5 ★ 什么情况下任务从 Running 变成 Ready？
**答案**：① 被更高优先级任务抢占 ② 同优先级时间片用完 ③ 主动调用 `taskYIELD()`。

### Q6 ★ Blocked 和 Suspended 有什么区别？
**答案**：Blocked 是**等事件**（信号量/队列/延时），事件到达会自动转 Ready；Suspended 是**被显式挂起**（`vTaskSuspend`），调度器完全不理会它，即使它等的事件发生了也不会就绪，必须 `vTaskResume` 才恢复。

### Q7 ★★ FreeRTOS 的调度原则是什么？
**答案**：抢占式调度（高优先级任务就绪立即抢占）+ 同优先级时间片轮转（需开启）。核心：永远运行优先级最高的就绪任务。

### Q8 ★★ 任务优先级怎么设计？
**答案**：Rate Monotonic 原则——周期越短优先级越高；安全/关键任务优先级最高；耗时任务优先级低；优先级数量受 `configMAX_PRIORITIES` 限制（多则耗 RAM）。

### Q9 ★★★ 上下文切换保存哪些寄存器？在哪个异常中执行？
**答案**：保存 R0-R12、SP、LR、PC、xPSR（Cortex-M 上硬件自动保存 R0-R3/R12/LR/PC/xPSR，软件保存 R4-R11）。
在 **PendSV** 异常中执行——它被设为最低优先级，保证不打断其他中断，在最安全的时刻切换。

### Q10 ★ 为什么上下文切换用 PendSV 而不用 SysTick？
**答案**：SysTick 是系统节拍中断，优先级通常高于普通中断，在其中切换会打断正在处理的中断逻辑。PendSV 设为**最低优先级**，只在所有中断处理完后才执行，能安全地保存/恢复全部上下文。而且可以通过"挂起 PendSV"来延后切换（不要每 tick 都切，只在需要时切）。

### Q11 ★ 空闲任务的作用？能删除吗？
**答案**：`vTaskStartScheduler` 自动创建，优先级 0。作用：保证 CPU 永远有任务可跑、回收被删除任务的资源、提供低功耗入口（Idle Hook）。**不能删除**。

### Q12 ★★ 队列和信号量的区别？
**答案**：队列**传数据**（深拷贝到队列，可带任意大小数据）；信号量**只有计数**，用于同步/互斥，不传数据。队列有深度和元素大小配置；信号量只有计数。

### Q13 ★★★ 二值信号量和互斥锁的本质区别？
**答案**：
- 互斥锁**有所有权**（谁加锁谁解锁，其他任务不能解）、**有优先级继承**，用于**保护临界资源**
- 二值信号量**无所有权**（任何任务/ISR 都能 give/take）、**无优先级继承**，用于**事件通知/同步**
判据：保护共享资源用 mutex，ISR 通知任务用 semaphore。

### Q14 ★ 什么时候用计数信号量？
**答案**：管理多个同类资源时（如 3 个缓冲区/3 台打印机），计数 = 剩余可用数量。也用于生产者-消费者计数。

### Q15 ★ 任务通知（Task Notification）相比信号量有什么优势？限制是什么？
**答案**：优势：直接写目标任务的 TCB（32 位值），无需创建内核对象，速度更快（约快 45%）、省 RAM。
限制：**只能一对一**（发送方指定接收任务），不能多个任务等同一个通知；通知值只有 32 位。

### Q16 ★ 事件组（Event Group）解决什么问题？
**答案**：需要一个任务同时等**多个事件**的组合时（如"传感器就绪 AND 通信就绪"才继续）。位图表示，支持 AND/OR 等待。

### Q17 ★★ 一个 ISR 收到数据后要通知任务处理，应该用哪个 API？写出代码。
**答案**：用二值信号量（或任务通知）的 FromISR 版本：
```c
void UART_IRQHandler(void) {
    BaseType_t woken = pdFALSE;
    xSemaphoreGiveFromISR(xSem, &woken);
    portYIELD_FROM_ISR(woken);
}
```
**必须用 FromISR 版本**，普通版本可能阻塞，不能在中断上下文使用。

### Q18 ★★★ `xHigherPriorityTaskWoken` 有什么作用？
**答案**：告诉移植层"这次操作是否唤醒了更高优先级的任务"。初值 `pdFALSE`；若 FromISR API 唤醒了优先级高于当前被中断任务的任务，则被置为 `pdTRUE` 并记录当前最高的就绪任务。
`portYIELD_FROM_ISR(woken)` 在 `pdTRUE` 时触发 PendSV，使**中断返回后立即切换到高优先级任务**，而不是等下一个 tick——保证实时性。

### Q19 ★★ 中断里为什么不能调用普通（非 FromISR）的 RTOS API？
**答案**：普通版本可能在内部阻塞（等待资源）、可能操作调度器数据结构但没有针对中断上下文做保护（因为中断上下文不能被视为一个任务，没有自己的栈和 TCB），可能破坏内核状态或死锁。FromISR 版本不阻塞且安全。

### Q20 ★★ 中断里能做什么、不能做什么？
**答案**：
能做：清中断标志、读写硬件寄存器、把数据放入缓冲区/队列、发信号量（FromISR）、置标志位。
不能做：printf（慢、不可重入）、延时/sleep、malloc（不可重入、耗时不确定）、调用非 ISR 的 RTOS API、浮点运算（若中断不保存 FPU 上下文）、耗时计算。

### Q21 ★★ 中断优先级和 RTOS 如何配合？
**答案**：Cortex-M 中**数值越小优先级越高**。FreeRTOS 设 `configMAX_SYSCALL_INTERRUPT_PRIORITY`（如 5）。
- 优先级数值 **< 5**（更紧急）的中断**不能调用任何 RTOS API**
- 数值 **≥ 5** 的中断可以调用 `FromISR` 版本
- SysTick 和 PendSV 设为最低优先级（15）
原因：低于阈值的中断不被 RTOS 临界区屏蔽，若在其中调用 API 可能破坏内核数据结构。

### Q22 ★★★ 什么是优先级反转？描述完整的场景。
**答案**：见 5.1。三任务 H(高)、M(中)、L(低)：
1. L 持锁
2. H 就绪并抢占 L，但需要同一把锁 → H 阻塞等 L
3. M 就绪，优先级高于 L → M 抢占 L
4. 结果 H 被 M 间接阻塞 → 高优先级的响应时间取决于 M
这就是优先级反转，破坏实时性。

### Q23 ★★★ 如何解决优先级反转？FreeRTOS 用哪种？
**答案**：两种方法——
- **优先级继承**：H 等锁时，持锁的 L 优先级临时提升到 H，尽快执行完放锁，之后恢复。FreeRTOS 的 **mutex 自带**此机制。
- **优先级天花板**：锁预设最高优先级（所有可能使用者中最高），拿到锁就提升到该级。
案例：1997 火星探路者因优先级反转导致看门狗复位，靠优先级继承修复。

### Q24 ★★ RTOS 中如何避免死锁？
**答案**：① 统一加锁顺序（所有任务按相同顺序获取多把锁）② 带超时获取（`xSemaphoreTake(sem, timeout)`，失败则回退释放已持有的锁）③ 避免嵌套锁 ④ 尽量缩小临界区、少用锁。

### Q25 ★⚠ 临界区保护有哪些方式？各自的代价？
**答案**：
- 关中断（`taskENTER_CRITICAL`，Cortex-M 设 BASEPRI）：最快，但会延迟中断响应，必须极短
- 互斥锁：可睡眠等待，但有获取开销
- 信号量/自旋锁
代价：关中断影响实时性；加锁有开销且可能引入优先级反转/死锁。

### Q26 ★★ FreeRTOS 有哪几种堆管理方案？推荐哪个？
**答案**：heap_1（只分配不释放）、heap_2（可释放不合并，已不推荐）、heap_4（可释放 + 合并相邻空闲块，**推荐**）、heap_5（支持多个不连续 RAM 区域）、heap_3（包装标准库 malloc/free）。
最常用 heap_4。

### Q27 ★ 嵌入式为什么优先静态分配？怎么静态创建任务？
**答案**：静态分配无碎片、执行时间确定、最坏情况可分析——符合实时性要求。
```c
StaticTask_t tcb;
StackType_t stack[128];
xTaskCreateStatic(task_fn, "t1", 128, NULL, 2, stack, &tcb);
```

### Q28 ★★ 如何检测和调试栈溢出？
**答案**：
检测：`configCHECK_FOR_STACK_OVERFLOW = 2`（栈底魔数 0xA5A5A5A5 检测），触发 `vApplicationStackOverflowHook`。
调试：`uxTaskGetStackHighWaterMark()` 看历史最小剩余；看 map 文件核算最坏用量；MPU 保护栈区；在栈底填特征值后观察。

### Q29 ★ 软件定时器的回调在哪里执行？能阻塞吗？
**答案**：在**定时器服务任务（Timer Service/Daemon 任务）**的上下文中执行，不是中断。
**不能阻塞**——会阻塞整个定时器任务，导致其他定时器全部延迟。回调应极短，耗时处理应投递到队列交给普通任务。

### Q30 ★ Tickless 低功耗模式是什么？
**答案**：空闲时停止周期性 tick 中断，让 MCU 进入深度睡眠；睡眠前计算下一个任务的最早到期时间，用 RTC/低功耗定时器设定唤醒时刻；唤醒后补偿 tick 计数。可大幅降低空闲功耗（`configUSE_TICKLESS_IDLE`）。

---

# 第三部分：入门级补充（从未做过 RTOS 要知道的）

## 最小可用代码示例
```c
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"

QueueHandle_t xQueue;
TaskHandle_t xTaskA;

void vProducer(void *pv) {
    int val = 0;
    for (;;) {
        val++;
        xQueueSend(xQueue, &val, portMAX_DELAY);   // 队满则等
        vTaskDelay(pdMS_TO_TICKS(100));            // 让出 CPU 100ms
    }
}

void vConsumer(void *pv) {
    int val;
    for (;;) {
        if (xQueueReceive(xQueue, &val, portMAX_DELAY) == pdTRUE) {
            // 处理数据
        }
    }
}

int main(void) {
    xQueue = xQueueCreate(10, sizeof(int));        // 深度 10
    xTaskCreate(vProducer, "prod", 256, NULL, 2, NULL);
    xTaskCreate(vConsumer, "cons", 256, NULL, 3, NULL);  // 优先级更高
    vTaskStartScheduler();                         // 启动，不会返回
    for (;;);                                      // 只有内存不足才会走到这
}
```

## API 一览（记住常用即可）
| 类别 | 常用 API |
|---|---|
| 任务 | `xTaskCreate` / `vTaskDelete` / `vTaskDelay` / `vTaskDelayUntil` / `vTaskSuspend` / `vTaskResume` |
| 队列 | `xQueueCreate` / `xQueueSend` / `xQueueReceive` / `xQueueSendFromISR` |
| 信号量 | `xSemaphoreCreateBinary` / `xSemaphoreCreateMutex` / `xSemaphoreCreateCounting` / `xSemaphoreTake` / `xSemaphoreGive` |
| 任务通知 | `xTaskNotifyGive` / `ulTaskNotifyTake` |
| 事件组 | `xEventGroupCreate` / `xEventGroupSetBits` / `xEventGroupWaitBits` |
| 定时器 | `xTimerCreate` / `xTimerStart` / `xTimerStop` |

## `vTaskDelay` vs `vTaskDelayUntil` ★
- `vTaskDelay(n)`：**相对延时**，从调用时刻起阻塞 n 个 tick
- `vTaskDelayUntil(&last, period)`：**绝对延时**（周期）——每次都等到"上次唤醒 + period"这个绝对时刻
- **周期任务必须用 `vTaskDelayUntil`**，否则任务执行时间会累积到周期里，导致周期漂移

---

# 第四部分：易错速查

| 陷阱 | 记住 |
|---|---|
| 二值信号量 vs 互斥锁 | 所有权 + 优先级继承：只有 mutex 有 |
| 优先级反转 | 三任务：低持锁、高等锁、中抢占低 |
| 上下文切换 | Cortex-M 里在 PendSV 做，最低优先级 |
| FromISR | 中断里必须用，配 `xHigherPriorityTaskWoken` |
| 中断优先级 | 数值越小越高；低于阈值的不能调 API |
| 周期任务 | 用 `vTaskDelayUntil`，不用 `vTaskDelay` |
| 定时器回调 | 在 Daemon 任务里，不能阻塞 |
| 空闲任务 | 优先级 0，不能删除 |
| heap_4 | 会合并空闲块，推荐 |
| 栈大小单位 | 字（4 字节），128 = 512 字节 |
| 任务通知 | 最快但只能一对一 |
| SysTick vs PendSV | SysTick 做计时，PendSV 做切换 |
