# 死锁（Deadlock）

在多道程序并发执行的环境中，多个进程可能因竞争有限的系统资源而相互等待。若无外力作用，这些进程都将永远无法向前推进，这种僵局状态称为**死锁（Deadlock）**。

在考研 408 计算机操作系统中，死锁是进程管理的核心高频考点。本文梳理死锁的基本概念、四大必要条件，并以 **C 语言完整实现经典死锁避免算法——银行家算法（Banker's Algorithm）**。

---

## 1. 核心概念辨析

在操作系统中，需严格区分以下三种易混淆的进程受阻状态：

| 状态 | 定义 | 进程位置 / CPU 状态 | 是否占有资源 |
| --- | --- | --- | --- |
| **死锁（Deadlock）** | 两个或多个进程相互等待对方释放资源，均无法推进 | 阻塞态（等待特定事件发生） | 占有部分资源并请求新资源 |
| **饥饿（Starvation）** | 调度策略长期偏向高优先级进程，导致低优先级进程长期无法获取资源推进 | 就绪态 / 阻塞态 | 可能不占有任何资源 |
| **死循环（Infinite Loop）** | 程序逻辑错误导致的死循环 | 运行态（持续霸占 CPU） | 通常不涉及跨进程资源互锁 |

---

## 2. 死锁产生的四大必要条件

死锁发生必须**同时满足**以下四个条件，破坏其中任意一个即可预防死锁：

```text
       ┌─────────────┐
       │   互斥条件   │  (资源排他使用，独占访问)
       └──────┬──────┘
              ▼
       ┌─────────────┐
       │ 不剥夺条件  │  (进程未使用完前不能被强行抢占)
       └──────┬──────┘
              ▼
       ┌─────────────┐
       │ 请求和保持  │  (占有已有资源的同时继续请求新资源)
       └──────┬──────┘
              ▼
       ┌─────────────┐
       │ 循环等待条件│  (存在进程-资源的闭环等待链：P0->P1->...->Pn->P0)
       └─────────────┘

```

1. **互斥条件（Mutual Exclusion）**：资源在一段时间内只能被一个进程占用。
2. **不剥夺条件（Non-preemption）**：进程获得的资源在未使用完之前，不能被外部强行剥夺，只能由该进程自愿释放。
3. **请求和保持条件（Hold and Wait）**：进程已经保持了至少一个资源，但又提出了新的资源请求，而该资源已被其他进程占有，此时请求进程被阻塞，但对自己已获得的资源保持不放。
4. **循环等待条件（Circular Wait）**：存在一种进程资源的循环等待链，链中每一个进程已获得的资源同时被链中下一个进程所请求。
> **注意**：循环等待是死锁的必要条件，但**不是充分条件**（当同类资源数大于 1 时，有循环等待环路不一定会发生死锁）。



---

## 3. 死锁处理策略

操作系统处理死锁主要有四种策略：

```text
死锁处理策略
├── 1. 鸵鸟策略（Ignore）：视而不见，适用于死锁极少发生且修复成本高（如 Linux/Windows 默认策略）
├── 2. 死锁预防（Prevention）：严格破坏四大必要条件之一（保守，资源利用率低）
├── 3. 死锁避免（Avoidance）：动态评估系统安全性，代表为“银行家算法”（折中方案）
└── 4. 死锁检测与解除（Detection & Recovery）：允许死锁发生，定期检测并通过剥夺资源/撤销进程解除

```

### 死锁预防的手段与代价

* **破坏互斥**：引入 SPOOLing 技术（将独占设备虚拟化为共享设备），但并不是所有物理资源都能虚拟化。
* **破坏不剥夺**：当进程请求新资源得不到满足时必须主动释放所有已有资源；常用于易于保存恢复的 CPU 和内存，用于打印机等设备代价极大。
* **破坏请求和保持**：采用**静态分配策略**（进程运行前一次性申请全部所需资源），严重降低资源利用率。
* **破坏循环等待**：采用**顺序资源分配法**（对所有资源编号，进程必须严格按序号递增申请），限制了编程灵活性。

---

## 4. 安全状态与死锁避免

死锁避免并不要求破坏四大必要条件，而是在资源动态分配过程中，预先评估分配后的系统状态：

* **安全状态（Safe State）**：系统能找到至少一个**安全序列（Safe Sequence）** $\langle P_1, P_2, \dots, P_n \rangle$，使得每个进程 $P_i$ 后续所需的资源量都不超过“系统当前空闲资源 + 排在它前面的所有进程释放的资源总和”。
* **核心结论**：

$$\text{安全状态} \subset \text{无死锁状态} \quad \Longleftrightarrow \quad \text{非安全状态} \not\Rightarrow \text{死锁发生（但有死锁风险）}$$



```text
+---------------------------------------------+
|               所有系统分配状态               |
|  +---------------------------------------+  |
|  |             无死锁状态                |  |
|  |  +---------------------------------+  |  |
|  |  |           安全状态              |  |  |
|  |  |    (必能找到安全序列，绝对安全)   |  |  |
|  |  +---------------------------------+  |  |
|  |       不安全状态 (可能演化为死锁)       |  |
|  +---------------------------------------+  |
|       死锁状态 (不可逆僵局)                  |
+---------------------------------------------+

```

---

## 5. 银行家算法（Banker's Algorithm）C 语言实现

银行家算法包含两大部分：

1. **安全性检查子算法（Safety Check）**：验证当前剩余资源是否能够构成安全序列。
2. **资源请求处理流程（Request Processing）**：预分配资源 $\to$ 试探性安全性检查 $\to$ 安全则正式分配，不安全则回滚状态并拒绝请求。

### 核心数据结构数学对应关系

设进程总数为 $n$，资源类别总数为 $m$：

* `Available[m]`：可利用资源向量
* `Max[n][m]`：各进程最大需求矩阵
* `Allocation[n][m]`：各进程已分配资源矩阵
* `Need[n][m]`：各进程尚需资源矩阵，满足：

$$\text{Need}[i][j] = \text{Max}[i][j] - \text{Allocation}[i][j]$$

### 完整可运行代码

```c
#include <stdio.h>
#include <stdbool.h>

#define PROCESS_NUM 5   // 进程数量 (P0 ~ P4)
#define RESOURCE_NUM 3  // 资源种类 (A, B, C)

// 系统全局资源状态
int Available[RESOURCE_NUM] = {3, 3, 2}; // 初始空闲资源向量

int Max[PROCESS_NUM][RESOURCE_NUM] = {
    {7, 5, 3}, // P0
    {3, 2, 2}, // P1
    {9, 0, 2}, // P2
    {2, 2, 2}, // P3
    {4, 3, 3}  // P4
};

int Allocation[PROCESS_NUM][RESOURCE_NUM] = {
    {0, 1, 0}, // P0
    {2, 0, 0}, // P1
    {3, 0, 2}, // P2
    {2, 1, 1}, // P3
    {0, 0, 2}  // P4
};

int Need[PROCESS_NUM][RESOURCE_NUM];

// 初始化计算 Need 矩阵
void initNeed() {
    for (int i = 0; i < PROCESS_NUM; i++) {
        for (int j = 0; j < RESOURCE_NUM; j++) {
            Need[i][j] = Max[i][j] - Allocation[i][j];
        }
    }
}

// 打印当前系统矩阵信息
void printSystemState() {
    printf("\n============================ 当前系统资源分配表 ============================\n");
    printf("进程\tMax\t\tAllocation\tNeed\t\tAvailable\n");
    for (int i = 0; i < PROCESS_NUM; i++) {
        printf("P%d\t", i);
        for (int j = 0; j < RESOURCE_NUM; j++) printf("%d ", Max[i][j]);
        printf("\t\t");
        for (int j = 0; j < RESOURCE_NUM; j++) printf("%d ", Allocation[i][j]);
        printf("\t\t");
        for (int j = 0; j < RESOURCE_NUM; j++) printf("%d ", Need[i][j]);
        if (i == 0) {
            printf("\t\t");
            for (int j = 0; j < RESOURCE_NUM; j++) printf("%d ", Available[j]);
        }
        printf("\n");
    }
    printf("===========================================================================\n\n");
}

// 安全性检查算法
bool isSafe(int safeSequence[PROCESS_NUM]) {
    int work[RESOURCE_NUM];
    bool finish[PROCESS_NUM] = {false};

    // 1. 初始化 Work 向量为当前 Available
    for (int i = 0; i < RESOURCE_NUM; i++) {
        work[i] = Available[i];
    }

    int count = 0;

    // 2. 寻找满足 Finish[i] == false 且 Need[i] <= Work 的进程
    while (count < PROCESS_NUM) {
        bool found = false;

        for (int i = 0; i < PROCESS_NUM; i++) {
            if (!finish[i]) {
                bool canAllocate = true;
                for (int j = 0; j < RESOURCE_NUM; j++) {
                    if (Need[i][j] > work[j]) {
                        canAllocate = false;
                        break;
                    }
                }

                // 找到满足条件的进程，释放其持有的全部资源
                if (canAllocate) {
                    for (int j = 0; j < RESOURCE_NUM; j++) {
                        work[j] += Allocation[i][j];
                    }
                    finish[i] = true;
                    safeSequence[count++] = i;
                    found = true;
                }
            }
        }

        // 若一轮循环未找到任何可满足进程，说明无法进入安全状态
        if (!found) {
            return false;
        }
    }

    return true;
}

// 银行家算法：请求分配资源
bool requestResources(int processId, int request[RESOURCE_NUM]) {
    printf(">>> 进程 P%d 申请资源: [%d, %d, %d]\n", processId, request[0], request[1], request[2]);

    // 步骤 1: 检查请求量是否超出它声明的最大尚需量
    for (int j = 0; j < RESOURCE_NUM; j++) {
        if (request[j] > Need[processId][j]) {
            printf("[拒绝]: 申请资源超出进程声明的最大需求量 Need\n");
            return false;
        }
    }

    // 步骤 2: 检查系统当前可用资源是否足够
    for (int j = 0; j < RESOURCE_NUM; j++) {
        if (request[j] > Available[j]) {
            printf("[等待]: 当前可用资源不足，进程需等待\n");
            return false;
        }
    }

    // 步骤 3: 试探性分配（预分配）
    for (int j = 0; j < RESOURCE_NUM; j++) {
        Available[j] -= request[j];
        Allocation[processId][j] += request[j];
        Need[processId][j] -= request[j];
    }

    // 步骤 4: 执行安全性检查
    int safeSequence[PROCESS_NUM];
    if (isSafe(safeSequence)) {
        printf("[批准]: 系统处于安全状态，资源已成功分配！\n");
        printf("安全序列为: ");
        for (int i = 0; i < PROCESS_NUM; i++) {
            printf("P%d%s", safeSequence[i], (i == PROCESS_NUM - 1) ? "\n" : " -> ");
        }
        return true;
    } else {
        // 预分配导致系统不安全，状态回滚（Rollback）
        printf("[拒绝]: 分配后系统将进入不安全状态，正在回滚...\n");
        for (int j = 0; j < RESOURCE_NUM; j++) {
            Available[j] += request[j];
            Allocation[processId][j] -= request[j];
            Need[processId][j] += request[j];
        }
        return false;
    }
}

int main() {
    initNeed();
    printSystemState();

    // 初始状态安全性检查
    int safeSequence[PROCESS_NUM];
    if (isSafe(safeSequence)) {
        printf("初始状态安全！安全序列为: ");
        for (int i = 0; i < PROCESS_NUM; i++) {
            printf("P%d%s", safeSequence[i], (i == PROCESS_NUM - 1) ? "\n" : " -> ");
        }
    } else {
        printf("初始状态即处于不安全状态！\n");
    }

    // 测试场景 1: P1 申请资源 [1, 0, 2]（应被批准）
    printf("\n----------------------- 测试 1 -----------------------\n");
    int req1[RESOURCE_NUM] = {1, 0, 2};
    requestResources(1, req1);
    printSystemState();

    // 测试场景 2: P4 申请资源 [3, 3, 0]（可用资源不足，应被拒绝）
    printf("\n----------------------- 测试 2 -----------------------\n");
    int req2[RESOURCE_NUM] = {3, 3, 0};
    requestResources(4, req2);

    // 测试场景 3: P0 申请资源 [0, 2, 0]（导致不安全，回滚）
    printf("\n----------------------- 测试 3 -----------------------\n");
    int req3[RESOURCE_NUM] = {0, 2, 0};
    requestResources(0, req3);
    printSystemState();

    return 0;
}

```

---

## 6. 死锁定理与资源分配图化简

在死锁检测策略中，系统使用资源分配图（Resource Allocation Graph, RAG）表示：

* **圆圈节点（P）**：进程
* **矩形节点（R）**：资源类（框内的点代表资源实例个数）
* **分配边（$R_j \to P_i$）**：资源已被分配给该进程
* **请求边（$P_i \to R_j$）**：进程正在申请该资源

### 死锁定理（化简法判断死锁）

1. 在资源分配图中，找到一个**既不阻塞又非孤立**的进程节点 $P_i$（即其所有请求边均能被系统中现有的空闲资源满足）。
2. 消除该进程的所有请求边和分配边（释放该进程占有的全部资源），使它成为孤立点。
3. 重复以上过程，直到不能继续化简。
4. **结论**：若图**不能完全化简**（仍有边残留），则此时系统必然发生了**死锁**。

---

### 408 考研高频考点总结

1. **同类资源最小免死锁数量计算公式**：
* 设有 $n$ 个进程，每个进程最大需要 $m$ 个同类资源。
* 系统发生死锁的最恶劣情况：所有进程都占有了 $m - 1$ 个资源，且都在等待最后一个资源。
* **系统绝不发生死锁的最小资源数条件**：

$$\text{TotalResources} \ge n \times (m - 1) + 1$$




2. **银行家算法复杂度**：
* 假设有 $n$ 个进程、$m$ 类资源，安全性检查算法的时间复杂度为 $O(m \times n^2)$。


3. **死锁解除的代价顺序**：
* **资源剥夺法**：挂起某些死锁进程并强行夺取其资源（需注意防止被剥夺进程长期饥饿）。
* **撤销进程法**：按代价（优先级、占用资源多少、运行剩余时间）逐一强制终止死锁进程，直至死锁解除。
* **进程回退法**：基于检查点（Checkpoint）回滚到死锁发生前的状态。