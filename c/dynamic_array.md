# 数组与动态数组（Array & Dynamic Array）

在 C 语言中，原生数组（静态数组）的大小在编译或声明时固定。为了支持自适应长度，通常使用**动态数组（Dynamic Array / Vector）**。

动态数组是一种**基于连续内存的数据结构**，每个动态数组包含三个核心部分：

1. **数据指针（data）**：指向堆区申请的连续内存空间
2. **元素数量（size）**：当前数组中已经存放的有效元素个数
3. **数组容量（capacity）**：当前连续内存块最多能容纳的元素总数

---

## 1. 定义动态数组结构体

使用结构体封装这三个属性：

```c
#include <stdio.h>
#include <stdlib.h>

// 定义动态数组结构体
typedef struct DynamicArray {
    int *data;      // 指向连续内存块的首地址
    int size;       // 当前有效元素个数
    int capacity;   // 当前已分配的最大容量
} DynamicArray;

```

这里：

* `int *data` 保存堆内存首地址，支持像原生数组一样通过下标 `data[i]` 随机访问。
* 当 `size == capacity` 时，说明底层空间已用满，需要触发**扩容机制**。

---

## 2. 创建与初始化

动态数组的控制结构体和底层数据缓冲均在堆上通过 `malloc()` 动态分配：

```c
DynamicArray* createArray(int initialCapacity) {
    if (initialCapacity <= 0) {
        initialCapacity = 4; // 默认初始容量
    }

    DynamicArray *arr = (DynamicArray*)malloc(sizeof(DynamicArray));
    if (arr == NULL) {
        printf("控制块内存申请失败\n");
        return NULL;
    }

    arr->data = (int*)malloc(sizeof(int) * initialCapacity);
    if (arr->data == NULL) {
        printf("数据区内存申请失败\n");
        free(arr);
        return NULL;
    }

    arr->size = 0;
    arr->capacity = initialCapacity;

    return arr;
}

```

---

## 3. 内存结构示意

例如容量为 6，已存入 `10, 20, 30`：

```text
DynamicArray 结构体 (栈/堆)
+----------------+---------+-------------+
| data (指针)    | size: 3 | capacity: 6 |
+-------+--------+---------+-------------+
        |
        ↓ 堆区物理连续内存块
+-------+-------+-------+-------+-------+-------+
|  10   |  20   |  30   |  未   |  未   |  未   |
+-------+-------+-------+-------+-------+-------+
  [0]     [1]     [2]     [3]     [4]     [5]
|<--- 有效元素 (size) --->|<-- 空闲容量 (capacity - size) -->|

```

由于物理内存连续，数组具备 **$O(1)$ 随机访问** 特性，寻址公式为：


$$\text{Address}(i) = \text{BaseAddress} + i \times \text{sizeof}(int)$$

---

## 4. 遍历与随机访问

遍历打印当前存储的所有有效元素：

```c
void printArray(DynamicArray *arr) {
    if (arr == NULL || arr->size == 0) {
        printf("[] (empty)\n");
        return;
    }

    printf("[");
    for (int i = 0; i < arr->size; i++) {
        printf("%d", arr->data[i]);
        if (i < arr->size - 1) {
            printf(", ");
        }
    }
    printf("] (size: %d, capacity: %d)\n", arr->size, arr->capacity);
}

```

按索引安全获取元素：

```c
int getAt(DynamicArray *arr, int index, int *outValue) {
    if (index < 0 || index >= arr->size) {
        printf("索引越界: %d\n", index);
        return 0; // 访问失败
    }
    *outValue = arr->data[index];
    return 1; // 访问成功
}

```

---

## 5. 扩容机制与尾部追加（Push Back）

在末尾追加元素是最常见操作。如果容量已满，一般采取 **翻倍扩容策略（2x）**，确保均摊时间复杂度为 $O(1)$：

```c
// 扩容/调整容量内部函数
static int resize(DynamicArray *arr, int newCapacity) {
    int *newData = (int*)realloc(arr->data, sizeof(int) * newCapacity);
    if (newData == NULL) {
        printf("扩容失败: 内存不足\n");
        return 0;
    }

    arr->data = newData;
    arr->capacity = newCapacity;
    return 1;
}

// 尾部追加
void pushBack(DynamicArray *arr, int value) {
    if (arr == NULL) return;

    // 容量已满，触发扩容
    if (arr->size == arr->capacity) {
        int newCap = (arr->capacity == 0) ? 4 : arr->capacity * 2;
        if (!resize(arr, newCap)) {
            return;
        }
    }

    arr->data[arr->size] = value;
    arr->size++;
}

```

使用：

```c
DynamicArray *arr = createArray(2);
pushBack(arr, 10);
pushBack(arr, 20);
pushBack(arr, 30); // 触发自动翻倍扩容

```

输出：

```text
[10, 20, 30] (size: 3, capacity: 4)

```

---

## 6. 指定位置插入

在指定索引 `index` 处插入元素，其后的所有元素都需要**向后平移一位**，时间复杂度为 $O(n)$：

```c
void insertAt(DynamicArray *arr, int index, int value) {
    if (arr == NULL || index < 0 || index > arr->size) {
        printf("插入位置非法: %d\n", index);
        return;
    }

    // 检查扩容
    if (arr->size == arr->capacity) {
        int newCap = (arr->capacity == 0) ? 4 : arr->capacity * 2;
        if (!resize(arr, newCap)) {
            return;
        }
    }

    // 从后往前依次右移
    for (int i = arr->size; i > index; i--) {
        arr->data[i] = arr->data[i - 1];
    }

    arr->data[index] = value;
    arr->size++;
}

```

使用：

```c
// 在索引 1 处插入 99
insertAt(arr, 1, 99);

```

结果：

```text
[10, 99, 20, 30] (size: 4, capacity: 4)

```

---

## 7. 删除元素与缩容

删除指定索引处的元素，后续元素**向前平移一位**覆盖。为了避免空间闲置浪费，当实际元素数量远低于容量时（如 $size \le \frac{capacity}{4}$），可选择缩容：

```c
void deleteAt(DynamicArray *arr, int index) {
    if (arr == NULL || index < 0 || index >= arr->size) {
        printf("删除索引非法: %d\n", index);
        return;
    }

    // 从前往后依次左移覆盖
    for (int i = index; i < arr->size - 1; i++) {
        arr->data[i] = arr->data[i + 1];
    }
    arr->size--;

    // 惰性缩容策略：当元素占用小于等于容量的 1/4 且容量较大时，缩容至 1/2
    if (arr->size > 0 && arr->size <= arr->capacity / 4 && arr->capacity / 2 >= 4) {
        resize(arr, arr->capacity / 2);
    }
}

```

使用：

```c
deleteAt(arr, 1); // 删除索引 1 处的 99

```

结果：

```text
[10, 20, 30] (size: 3, capacity: 4)

```

---

## 8. 释放动态数组

必须遵循“**先释放数据缓冲，再释放控制结构体**”的顺序，避免悬垂指针与内存泄漏：

```c
void freeArray(DynamicArray *arr) {
    if (arr != NULL) {
        if (arr->data != NULL) {
            free(arr->data);
            arr->data = NULL;
        }
        free(arr);
    }
}

```

调用：

```c
freeArray(arr);
arr = NULL;

```

---

## 完整示例

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct DynamicArray {
    int *data;
    int size;
    int capacity;
} DynamicArray;

DynamicArray* createArray(int initialCapacity) {
    if (initialCapacity <= 0) initialCapacity = 4;
    DynamicArray *arr = (DynamicArray*)malloc(sizeof(DynamicArray));
    if (!arr) return NULL;

    arr->data = (int*)malloc(sizeof(int) * initialCapacity);
    if (!arr->data) {
        free(arr);
        return NULL;
    }
    arr->size = 0;
    arr->capacity = initialCapacity;
    return arr;
}

static int resize(DynamicArray *arr, int newCapacity) {
    int *newData = (int*)realloc(arr->data, sizeof(int) * newCapacity);
    if (!newData) return 0;
    arr->data = newData;
    arr->capacity = newCapacity;
    return 1;
}

void pushBack(DynamicArray *arr, int value) {
    if (!arr) return;
    if (arr->size == arr->capacity) {
        int newCap = (arr->capacity == 0) ? 4 : arr->capacity * 2;
        if (!resize(arr, newCap)) return;
    }
    arr->data[arr->size++] = value;
}

void insertAt(DynamicArray *arr, int index, int value) {
    if (!arr || index < 0 || index > arr->size) return;
    if (arr->size == arr->capacity) {
        int newCap = (arr->capacity == 0) ? 4 : arr->capacity * 2;
        if (!resize(arr, newCap)) return;
    }
    for (int i = arr->size; i > index; i--) {
        arr->data[i] = arr->data[i - 1];
    }
    arr->data[index] = value;
    arr->size++;
}

void deleteAt(DynamicArray *arr, int index) {
    if (!arr || index < 0 || index >= arr->size) return;
    for (int i = index; i < arr->size - 1; i++) {
        arr->data[i] = arr->data[i + 1];
    }
    arr->size--;
}

void printArray(DynamicArray *arr) {
    if (!arr || arr->size == 0) {
        printf("[] (empty)\n");
        return;
    }
    printf("[");
    for (int i = 0; i < arr->size; i++) {
        printf("%d", arr->data[i]);
        if (i < arr->size - 1) printf(", ");
    }
    printf("] (size: %d, capacity: %d)\n", arr->size, arr->capacity);
}

void freeArray(DynamicArray *arr) {
    if (arr) {
        if (arr->data) free(arr->data);
        free(arr);
    }
}

int main() {
    // 1. 初始化容量为 2 的动态数组
    DynamicArray *arr = createArray(2);

    // 2. 尾插触发自动扩容
    pushBack(arr, 10);
    pushBack(arr, 20);
    pushBack(arr, 30);
    printArray(arr); // [10, 20, 30] (size: 3, capacity: 4)

    // 3. 在中间位置插入
    insertAt(arr, 1, 99);
    printArray(arr); // [10, 99, 20, 30] (size: 4, capacity: 4)

    // 4. 删除元素
    deleteAt(arr, 1);
    printArray(arr); // [10, 20, 30] (size: 3, capacity: 4)

    // 5. 释放内存
    freeArray(arr);
    return 0;
}

```

输出：

```text
[10, 20, 30] (size: 3, capacity: 4)
[10, 99, 20, 30] (size: 4, capacity: 4)
[10, 20, 30] (size: 3, capacity: 4)

```

---

### 面试常见问题

#### 1. 经典算法题型

* **双指针法**：对撞指针（两数之和、盛最多水的容器）、快慢指针（原地移除元素、移动零）
* **滑动窗口**：长度最小的子数组、无重复字符的最长子串
* **二分查找**：有序数组中查找目标值及其左右边界
* **前缀和与差分**：区间和检索、批量区间增减操作
* **合并区间 / 合并两个有序数组**

#### 2. 底层机制高频考点

* **扩容倍数为什么通常选 1.5 倍或 2 倍？**
* 选 2 倍逻辑清晰，平摊复杂度为均摊 $O(1)$；
* 部分工业级实现（如 MSVC STL、Facebook 的 `folly::fbvector`）选约 1.5 倍，能保证之后申请的内存块有机会**复用之前已经释放的内存**，大幅减少内存碎片。


* **为什么缩容阈值常设为 $\frac{1}{4}$ 而不是 $\frac{1}{2}$？**
* 防止**复杂度震荡**（如果 $\frac{1}{2}$ 就缩容，一旦在容量边界反复交替 `push` 与 `pop`，会导致每次操作都要重新分配内存并复制全部数据，均摊复杂度退化为 $O(n)$）。


* **动态数组 vs 链表的核心区别？**
* 动态数组物理地址连续，具备 $O(1)$ 随机索引与出色的 **CPU 缓存局部性（Cache Locality）**；
* 链表支持非连续存储与高效的头尾插入，但牺牲了随机访问，且每个节点额外的指针域会有额外内存开销。
