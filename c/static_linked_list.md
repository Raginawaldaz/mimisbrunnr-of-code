# 静态链表（数组模拟单链表）

在 C 语言中，除了使用指针和 `malloc()` 实现的动态链表外，还可以使用**数组下标模拟指针**来实现链表，这种结构被称为**静态链表（Static Linked List）**。

在静态链表中：

1. **数据数组（val）**：存储节点的数据值
2. **指针数组（ne）**：存储下一个节点的**数组下标**（相当于指针域 `next`）
3. **全局变量 head**：存储链表头节点的数组下标（初始化为 `-1` 表示空表）
4. **全局变量 idx**：充当简易“内存分配器”，记录当前可用存储单元的下标

---

## 1. 定义与全局变量

通常使用两个平行数组或一个静态节点结构体数组。在算法竞赛与底层高效场景中，平行数组写法最为简洁高效：

```c
#include <stdio.h>

#define MAX_SIZE 100010

// 链表基本存储结构
int head;            // 头节点所在的数组下标，-1 表示空表
int idx;             // 当前申请到的最新节点下标（内存分配指针）
int val[MAX_SIZE];   // val[i] 表示下标为 i 的节点存放的数据
int ne[MAX_SIZE];    // ne[i]  表示下标为 i 的节点的下一个节点下标（next）

```

这里：

* `idx` 永远单调递增，每次需要新节点时分配当前的 `idx++`。
* `ne[i] = -1` 相当于传统链表中的 `node->next = NULL`。

---

## 2. 初始化链表

初始化只需将头节点标记为 `-1`，并将可用空间起点置为 `0`：

```c
void initList() {
    head = -1;  // 初始为空链表
    idx = 0;   // 从下标 0 开始分配节点
}

```

---

## 3. 内存与结构映射示意

依次插入节点 `30 -> 20 -> 10` 后的内存映射：

```text
全局指针状态:
head = 2 (指向 val=10 的下标)
idx  = 3 (下次分配下标)

数组物理存储 (连续):
下标 i :    [0]      [1]      [2]
val[i]:     30       20       10
ne[i] :     -1        0        1

逻辑链表结构 (通过下标串联):
head(2) ──> val[2]=10, ne[2]=1 ──> val[1]=20, ne[1]=0 ──> val[0]=30, ne[0]=-1 (NULL)

```

---

## 4. 遍历链表

沿 `ne[p]` 下标指针依次访问，直到下标为 `-1`：

```c
void printList() {
    int p = head;

    while (p != -1) {
        printf("%d -> ", val[p]);
        p = ne[p]; // 相当于传统指针的 p = p->next
    }

    printf("NULL\n");
}

```

---

## 5. 头插法（Insert to Head）

在链表头部插入一个值为 `x` 的节点，时间复杂度为 $O(1)$：

```c
void insertHead(int x) {
    val[idx] = x;     // 在新节点存入数值
    ne[idx] = head;   // 新节点指向当前的头节点下标
    head = idx;       // 头节点指针更新为当前新节点
    idx++;            // 内存池下标后移，完成分配
}

```

使用：

```c
initList();
insertHead(30);
insertHead(20);
insertHead(10);

```

结果：

```text
10 -> 20 -> 30 -> NULL

```

---

## 6. 在指定节点后插入（Insert After k）

在下标为 `k` 的节点后面插入新节点，时间复杂度为 $O(1)$：

```c
void insertAfter(int k, int x) {
    val[idx] = x;
    ne[idx] = ne[k]; // 新节点指向 k 的后继
    ne[k] = idx;     // k 的后继指向新节点
    idx++;
}

```

---

## 7. 删除节点（Delete After k）

将下标为 `k` 的节点的下一个节点从链表中移除：

```c
void deleteAfter(int k) {
    // 将 k 的下一个节点直接指向下下个节点（跳过待删节点）
    if (ne[k] != -1) {
        ne[k] = ne[ne[k]];
    }
}

// 专门删除头节点
void deleteHead() {
    if (head != -1) {
        head = ne[head];
    }
}

```

> **注意**：静态链表中通常不回收被删除节点的下标（逻辑跳过即可）。若空间极度敏感，可引入备用链表（Free List）进行空闲下标复用。

---

## 8. 完整示例

```c
#include <stdio.h>

#define MAX_SIZE 100010

int head;
int idx;
int val[MAX_SIZE];
int ne[MAX_SIZE];

void initList() {
    head = -1;
    idx = 0;
}

void insertHead(int x) {
    val[idx] = x;
    ne[idx] = head;
    head = idx;
    idx++;
}

void insertAfter(int k, int x) {
    val[idx] = x;
    ne[idx] = ne[k];
    ne[k] = idx;
    idx++;
}

void deleteAfter(int k) {
    if (ne[k] != -1) {
        ne[k] = ne[ne[k]];
    }
}

void deleteHead() {
    if (head != -1) {
        head = ne[head];
    }
}

void printList() {
    int p = head;
    while (p != -1) {
        printf("%d -> ", val[p]);
        p = ne[p];
    }
    printf("NULL\n");
}

int main() {
    // 1. 初始化
    initList();

    // 2. 头插法构建链表
    insertHead(30);
    insertHead(20);
    insertHead(10);
    printList(); // 10 -> 20 -> 30 -> NULL

    // 3. 在下标为 1 的节点 (val=20) 之后插入 25
    insertAfter(1, 25);
    printList(); // 10 -> 20 -> 25 -> 30 -> NULL

    // 4. 删除头节点
    deleteHead();
    printList(); // 20 -> 25 -> 30 -> NULL

    // 5. 删除下标为 1 的节点之后的节点 (即删除 25)
    deleteAfter(1);
    printList(); // 20 -> 30 -> NULL

    return 0;
}

```

输出：

```text
10 -> 20 -> 30 -> NULL
10 -> 20 -> 25 -> 30 -> NULL
20 -> 25 -> 30 -> NULL
20 -> 30 -> NULL

```

---

### 面试常见问题与应用场景

#### 1. 为什么不用指针，而用静态链表？

* **性能极致**：避开系统级调用 `malloc()` 和 `free()` 的堆管理开销，分配耗时基本为 1 条指令（$O(1)$ 极小常数）。
* **防止内存碎片与泄漏**：统一在大数组预分配空间，生命周期随程序结束自动回收，无悬挂指针与内存泄漏隐患。
* **语言兼容性**：在不支持显式指针的高级脚本语言中，可用此机制模拟树、链表与图结构。

#### 2. 核心应用场景

* **图论：邻接表 / 链式前向星**
存储稀疏图的核心利器，每个顶点的出边表本质就是一个基于数组模拟的头插法单链表：
```c
void addEdge(int u, int v, int w) {
    to[idx] = v;
    weight[idx] = w;
    ne[idx] = head[u];
    head[u] = idx++;
}

```


* **LRU 缓存 / 散列表拉链法（Chaining）**：
在固定内存限制的嵌入式系统或刷题场景下，常开平铺数组作为 Hash Bucket 解决冲突。
