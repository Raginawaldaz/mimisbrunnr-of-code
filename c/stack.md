# 栈（Stack）

在 C 语言中，栈（Stack）是一种**操作受限的线性数据结构**。它严格遵循 **LIFO（Last In, First Out，后进先出）** 原则，所有元素的插入（入栈）和删除（出栈）操作均只能在**栈顶（Top）**一端进行，另一端称为**栈底（Bottom）**。

常见的栈实现方式有基于数组的**顺序栈（含动态扩容）**和基于链表的**链式栈**。由于顺序栈具有出色的 CPU 缓存局部性且实现精简，工业实践中最常用的是支持自动扩容的**动态顺序栈**。

一个动态顺序栈包含三个核心属性：

1. **数据指针（data）**：指向堆区存放元素的连续内存空间
2. **栈顶索引（top）**：指向当前栈顶元素的下标（空栈时约定为 `-1`）
3. **栈容量（capacity）**：当前连续空间最多可容纳的元素总数

---

## 1. 定义栈结构体

使用结构体封装这三个属性：

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

// 定义动态顺序栈结构体
typedef struct Stack {
    int *data;      // 指向连续内存块的首地址
    int top;        // 栈顶元素下标（-1 表示空栈）
    int capacity;   // 栈的最大容量
} Stack;

```

这里：

* 元素个数始终为 `top + 1`。
* 当 `top == capacity - 1` 时，说明当前空间已满，需要触发扩容。

---

## 2. 创建与初始化

在堆上动态分配控制结构体与底层存储缓冲区：

```c
Stack* createStack(int initialCapacity) {
    if (initialCapacity <= 0) {
        initialCapacity = 4; // 默认初始容量
    }

    Stack *stack = (Stack*)malloc(sizeof(Stack));
    if (stack == NULL) {
        printf("栈控制块内存申请失败\n");
        return NULL;
    }

    stack->data = (int*)malloc(sizeof(int) * initialCapacity);
    if (stack->data == NULL) {
        printf("栈数据区内存申请失败\n");
        free(stack);
        return NULL;
    }

    stack->top = -1; // 初始为空栈
    stack->capacity = initialCapacity;

    return stack;
}

```

---

## 3. 内存结构示意

例如容量为 4，依次入栈 `10, 20, 30` 后的内存状态：

```text
Stack 结构体 (栈/堆)
+----------------+---------+-------------+
| data (指针)    | top: 2  | capacity: 4 |
+-------+--------+---------+-------------+
        |
        ↓ 堆区物理连续内存块
+-------+-------+-------+-------+
|  10   |  20   |  30   |  未   |
+-------+-------+-------+-------+
  [0]     [1]     [2]     [3]
   ↑               ↑
 栈底             栈顶 (top)
|<--- 有效元素 (3) --->|<- 剩余容量 ->|

```

---

## 4. 状态判断与查看栈顶（Peek）

基础状态检查与栈顶查看的时间复杂度均为 $O(1)$：

```c
// 判空
bool isEmpty(Stack *stack) {
    return stack == NULL || stack->top == -1;
}

// 获取当前元素个数
int getSize(Stack *stack) {
    return (stack == NULL) ? 0 : stack->top + 1;
}

// 查看栈顶元素（不弹出）
bool peek(Stack *stack, int *outValue) {
    if (isEmpty(stack)) {
        printf("栈为空，无法查看栈顶\n");
        return false;
    }
    *outValue = stack->data[stack->top];
    return true;
}

```

---

## 5. 入栈操作（Push）与动态扩容

入栈将新元素放置在 `top + 1` 位置。若容量满，则以 **2 倍** 扩容，确保均摊时间复杂度为 $O(1)$：

```c
// 内部扩容函数
static bool resize(Stack *stack, int newCapacity) {
    int *newData = (int*)realloc(stack->data, sizeof(int) * newCapacity);
    if (newData == NULL) {
        printf("扩容失败: 内存不足\n");
        return false;
    }
    stack->data = newData;
    stack->capacity = newCapacity;
    return true;
}

// 入栈
void push(Stack *stack, int value) {
    if (stack == NULL) return;

    // 容量已满，触发翻倍扩容
    if (stack->top == stack->capacity - 1) {
        int newCap = (stack->capacity == 0) ? 4 : stack->capacity * 2;
        if (!resize(stack, newCap)) {
            return;
        }
    }

    stack->top++;
    stack->data[stack->top] = value;
}

```

---

## 6. 出栈操作（Pop）

出栈返回并移除栈顶元素，只需将 `top` 递减即可，时间复杂度为 $O(1)$：

```c
bool pop(Stack *stack, int *outValue) {
    if (isEmpty(stack)) {
        printf("栈下溢（Stack Underflow）：栈已空\n");
        return false;
    }

    *outValue = stack->data[stack->top];
    stack->top--;
    return true;
}

```

---

## 7. 释放栈内存

遵循“**先释放底层数据缓冲区，再释放控制结构体**”的原则，防止内存泄漏：

```c
void freeStack(Stack *stack) {
    if (stack != NULL) {
        if (stack->data != NULL) {
            free(stack->data);
            stack->data = NULL;
        }
        free(stack);
    }
}

```

---

## 8. 完整示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

typedef struct Stack {
    int *data;
    int top;
    int capacity;
} Stack;

Stack* createStack(int initialCapacity) {
    if (initialCapacity <= 0) initialCapacity = 4;
    Stack *stack = (Stack*)malloc(sizeof(Stack));
    if (!stack) return NULL;

    stack->data = (int*)malloc(sizeof(int) * initialCapacity);
    if (!stack->data) {
        free(stack);
        return NULL;
    }

    stack->top = -1;
    stack->capacity = initialCapacity;
    return stack;
}

static bool resize(Stack *stack, int newCapacity) {
    int *newData = (int*)realloc(stack->data, sizeof(int) * newCapacity);
    if (!newData) return false;
    stack->data = newData;
    stack->capacity = newCapacity;
    return true;
}

bool isEmpty(Stack *stack) {
    return stack == NULL || stack->top == -1;
}

int getSize(Stack *stack) {
    return (stack == NULL) ? 0 : stack->top + 1;
}

void push(Stack *stack, int value) {
    if (!stack) return;
    if (stack->top == stack->capacity - 1) {
        int newCap = (stack->capacity == 0) ? 4 : stack->capacity * 2;
        if (!resize(stack, newCap)) return;
    }
    stack->data[++stack->top] = value;
}

bool pop(Stack *stack, int *outValue) {
    if (isEmpty(stack)) return false;
    *outValue = stack->data[stack->top--];
    return true;
}

bool peek(Stack *stack, int *outValue) {
    if (isEmpty(stack)) return false;
    *outValue = stack->data[stack->top];
    return true;
}

void freeStack(Stack *stack) {
    if (stack) {
        if (stack->data) free(stack->data);
        free(stack);
    }
}

int main() {
    // 1. 初始化容量为 2 的栈
    Stack *stack = createStack(2);

    // 2. 入栈并触发自动扩容
    push(stack, 10);
    push(stack, 20);
    push(stack, 30); // 触发扩容：capacity 从 2 变为 4

    printf("当前栈大小: %d, 容量: %d\n", getSize(stack), stack->capacity);

    // 3. 查看栈顶元素
    int topVal;
    if (peek(stack, &topVal)) {
        printf("栈顶元素: %d\n", topVal);
    }

    // 4. 依次出栈
    printf("出栈顺序: ");
    int val;
    while (pop(stack, &val)) {
        printf("%d ", val);
    }
    printf("\n");

    // 5. 判空
    printf("栈是否为空: %s\n", isEmpty(stack) ? "是" : "否");

    // 6. 释放内存
    freeStack(stack);
    return 0;
}

```

输出：

```text
当前栈大小: 3, 容量: 4
栈顶元素: 30
出栈顺序: 30 20 10 
栈是否为空: 是

```

---

### 面试常见问题

#### 1. 经典算法题型

* **有效括号匹配（Valid Parentheses）**：遇到左括号入栈，遇到右括号比对栈顶并出栈。
* **单调栈（Monotonic Stack）**：
* 下一个更大元素（Next Greater Element）
* 接雨水（Trapping Rain Water）
* 柱状图中最大的矩形（Largest Rectangle in Histogram）


* **最小栈（Min Stack）**：维护一个辅助栈或差值变量，支持 $O(1)$ 获取当前栈内最小元素。
* **逆波兰表达式求值（Evaluate Reverse Polish Notation）**：后缀表达式的运算解析。
* **双栈模拟队列（Implement Queue using Stacks）**：一个输入栈负责入队，一个输出栈负责出队。

#### 2. 底层机制高频考点

* **顺序栈（基于数组） vs 链式栈（基于单链表）如何权衡？**
* **顺序栈**：内存物理连续，具备优秀的 CPU 缓存命中率；但扩容时偶尔会有数据拷贝的性能抖动（尽管均摊为 $O(1)$）。
* **链式栈**：没有固定容量限制，每次入栈仅动态分配一个节点，避免了扩容拷贝；但每个节点多了一个指针域（64 位系统下多耗费 8 字节），且频繁 `malloc/free` 会带来系统调用开销与内存碎片。


* **数据结构中的“栈”与系统运行时的“函数调用栈（Call Stack）”有什么区别？**
* **数据结构中的栈**：是一种逻辑抽象模型，用于程序中数据的存取组织。
* **调用栈**：是操作系统和 CPU 硬件共同支持的一块固定内存区域（通常只有几 MB），用于记录函数调用的活动记录（返回地址、形参、局部变量等）。递归过深导致的 `Stack Overflow`（栈溢出）正是该物理内存用尽所致。