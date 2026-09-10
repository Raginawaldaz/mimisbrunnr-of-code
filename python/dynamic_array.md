# 动态数组（Dynamic Array - Python 实现）

在 Python 中，内置的列表（`list`）底层正是由 C 语言实现的动态数组（`PyListObject`）。为了深入理解其运作机制，我们不直接封装现成的 `list`，而是借助标准库 `ctypes` 分配底层的**物理连续内存块**，从零复刻一个自适应长度的动态数组。

动态数组是一种**基于连续内存的数据结构**，每个动态数组包含三个核心部分：

1. **底层缓冲区（`_data`）**：通过底层连续空间存放对象引用的内存块
2. **元素数量（`_size`）**：当前数组中已经存放的有效元素个数
3. **数组容量（`_capacity`）**：当前连续内存块最多能容纳的元素总数

---

## 1. 定义动态数组类与内存分配

利用 `ctypes.py_object` 创建定长的物理连续引用数组，模拟底层的裸数组：

```python
import ctypes

class DynamicArray:
    """基于 ctypes 连续内存模拟实现的动态数组"""

    def __init__(self, initial_capacity: int = 4):
        if initial_capacity <= 0:
            initial_capacity = 4
        self._size = 0                          # 当前有效元素个数
        self._capacity = initial_capacity      # 当前最大容量
        self._data = self._make_array(self._capacity)  # 物理连续缓冲区

    def _make_array(self, capacity: int):
        """在底层申请一个大小为 capacity 的连续引用数组"""
        return (capacity * ctypes.py_object)()

```

这里：

* `(capacity * ctypes.py_object)()` 会在内存中分配一块严格连续的内存空间，用于存储 Python 对象的引用（指针）。
* 当 `_size == _capacity` 时，说明当前连续空间已被占满，需要触发**扩容机制**。

---

## 2. 内存结构示意

例如容量为 4，已存入 `10, 20, 30` 时的内存结构：

```text
DynamicArray 对象 (Python 对象区)
+----------------+----------+---------------+
| _data (指针)   | _size: 3 | _capacity: 4  |
+-------+--------+----------+---------------+
        |
        ↓ 物理连续引用缓冲区 (ctypes.py_object)
+-------+-------+-------+-------+
|  [0]  |  [1]  |  [2]  |  [3]  |
+---+---+---+---+---+---+-------+
    |       |       |       未分配
    ↓       ↓       ↓
+------+ +------+ +------+
|  10  | |  20  | |  30  |  (Python 堆区真实对象)
+------+ +------+ +------+
|<-- 有效元素 (_size: 3) -->|<- 剩余容量 ->|

```

由于底层引用数组的物理内存严格连续，支持 **$O(1)$ 随机访问**，第 $i$ 个引用的物理寻址公式为：

$$\text{Address}(i) = \text{BaseAddress} + i \times \text{sizeof}(\text{PyObject*})$$

---

## 3. 基础属性与随机访问

实现 `__len__`、`__getitem__` 以及格式化打印：

```python
    def __len__(self) -> int:
        """返回数组中有效元素的数量"""
        return self._size

    @property
    def capacity(self) -> int:
        """返回当前容量"""
        return self._capacity

    def is_empty(self) -> bool:
        """检查数组是否为空"""
        return self._size == 0

    def __getitem__(self, index: int):
        """按索引安全获取元素，支持负数索引"""
        if index < 0:
            index += self._size
        if not 0 <= index < self._size:
            raise IndexError(f"Index out of range: {index}")
        return self._data[index]

    def __repr__(self) -> str:
        """字符串表示"""
        if self._size == 0:
            return "[] (empty)"
        elements = [str(self._data[i]) for i in range(self._size)]
        return f"[{', '.join(elements)}] (size: {self._size}, capacity: {self._capacity})"

```

---

## 4. 扩容机制与尾部追加（append）

在末尾追加元素是最常见的操作。当容量已满时，通常采用 **翻倍扩容策略（2x）**，使得单次追加的均摊时间复杂度为 $O(1)$：

```python
    def _resize(self, new_capacity: int):
        """调整底层连续数组容量"""
        new_data = self._make_array(new_capacity)
        # 将原数组中的所有对象引用搬移到新数组
        for i in range(self._size):
            new_data[i] = self._data[i]
        self._data = new_data
        self._capacity = new_capacity

    def append(self, value):
        """在数组末尾追加元素"""
        # 容量已满，触发自动翻倍扩容
        if self._size == self._capacity:
            new_capacity = 4 if self._capacity == 0 else self._capacity * 2
            self._resize(new_capacity)

        self._data[self._size] = value
        self._size += 1

```

---

## 5. 指定位置插入（insert）

在索引 `index` 处插入元素时，该位置及之后的所有元素都需要**依次向后平移一位**，最坏与平均时间复杂度为 $O(n)$：

```python
    def insert(self, index: int, value):
        """在指定索引处插入元素"""
        if index < 0:
            index += self._size
        if not 0 <= index <= self._size:
            raise IndexError(f"Index out of range: {index}")

        # 检查是否需要扩容
        if self._size == self._capacity:
            new_capacity = 4 if self._capacity == 0 else self._capacity * 2
            self._resize(new_capacity)

        # 从后往前依次右移一位腾出空间
        for i in range(self._size, index, -1):
            self._data[i] = self._data[i - 1]

        self._data[index] = value
        self._size += 1

```

---

## 6. 删除元素与惰性缩容（pop / delete）

删除指定位置的元素，其后所有元素依次向前平移覆盖。为防止内存闲置浪费，当元素数量降至容量的 $\frac{1}{4}$ 时触发缩容（缩至 $\frac{1}{2}$），以避免反复在临界点操作引起的“复杂度震荡”：

```python
    def pop(self, index: int = -1):
        """移除并返回指定索引处的元素（默认移除末尾）"""
        if self._size == 0:
            raise IndexError("pop from empty array")

        if index < 0:
            index += self._size
        if not 0 <= index < self._size:
            raise IndexError(f"Index out of range: {index}")

        value = self._data[index]

        # 从前往后依次左移覆盖
        for i in range(index, self._size - 1):
            self._data[i] = self._data[i + 1]

        # 清空原末尾引用，以便 Python 垃圾回收器及时回收对象
        self._data[self._size - 1] = None
        self._size -= 1

        # 惰性缩容策略：当有效元素 <= 容量的 1/4 且当前容量较大时，减半容量
        if 0 < self._size <= self._capacity // 4 and self._capacity // 2 >= 4:
            self._resize(self._capacity // 2)

        return value

```

---

## 完整示例

```python
import ctypes

class DynamicArray:
    """基于 ctypes 连续内存底层复刻的动态数组"""

    def __init__(self, initial_capacity: int = 4):
        if initial_capacity <= 0:
            initial_capacity = 4
        self._size = 0
        self._capacity = initial_capacity
        self._data = self._make_array(self._capacity)

    def _make_array(self, capacity: int):
        return (capacity * ctypes.py_object)()

    def __len__(self) -> int:
        return self._size

    @property
    def capacity(self) -> int:
        return self._capacity

    def is_empty(self) -> bool:
        return self._size == 0

    def __getitem__(self, index: int):
        if index < 0:
            index += self._size
        if not 0 <= index < self._size:
            raise IndexError(f"Index out of range: {index}")
        return self._data[index]

    def _resize(self, new_capacity: int):
        new_data = self._make_array(new_capacity)
        for i in range(self._size):
            new_data[i] = self._data[i]
        self._data = new_data
        self._capacity = new_capacity

    def append(self, value):
        if self._size == self._capacity:
            new_cap = 4 if self._capacity == 0 else self._capacity * 2
            self._resize(new_cap)
        self._data[self._size] = value
        self._size += 1

    def insert(self, index: int, value):
        if index < 0:
            index += self._size
        if not 0 <= index <= self._size:
            raise IndexError(f"Index out of range: {index}")

        if self._size == self._capacity:
            new_cap = 4 if self._capacity == 0 else self._capacity * 2
            self._resize(new_cap)

        for i in range(self._size, index, -1):
            self._data[i] = self._data[i - 1]
        self._data[index] = value
        self._size += 1

    def pop(self, index: int = -1):
        if self._size == 0:
            raise IndexError("pop from empty array")
        if index < 0:
            index += self._size
        if not 0 <= index < self._size:
            raise IndexError(f"Index out of range: {index}")

        val = self._data[index]
        for i in range(index, self._size - 1):
            self._data[i] = self._data[i + 1]

        self._data[self._size - 1] = None  # 辅助垃圾回收
        self._size -= 1

        if 0 < self._size <= self._capacity // 4 and self._capacity // 2 >= 4:
            self._resize(self._capacity // 2)

        return val

    def __repr__(self) -> str:
        if self._size == 0:
            return "[] (empty)"
        elements = [str(self._data[i]) for i in range(self._size)]
        return f"[{', '.join(elements)}] (size: {self._size}, capacity: {self._capacity})"


if __name__ == "__main__":
    # 1. 初始化容量为 2 的动态数组
    arr = DynamicArray(initial_capacity=2)

    # 2. 尾部追加并触发自动扩容
    arr.append(10)
    arr.append(20)
    arr.append(30)  # 触发扩容：capacity 从 2 扩容至 4
    print(arr)      # [10, 20, 30] (size: 3, capacity: 4)

    # 3. 指定位置插入
    arr.insert(1, 99)
    print(arr)      # [10, 99, 20, 30] (size: 4, capacity: 4)

    # 4. 删除指定位置元素
    removed = arr.pop(1)  # 删除 99
    print(f"弹出的元素: {removed}")
    print(arr)      # [10, 20, 30] (size: 3, capacity: 4)

    # 5. 索引访问与长度
    print(f"首元素: {arr[0]}, 末元素: {arr[-1]}, 当前长度: {len(arr)}")

```

输出：

```text
[10, 20, 30] (size: 3, capacity: 4)
[10, 99, 20, 30] (size: 4, capacity: 4)
弹出的元素: 99
[10, 20, 30] (size: 3, capacity: 4)
首元素: 10, 末元素: 30, 当前长度: 3

```

---

### 面试常见问题

#### 1. 经典算法题型

* **双指针法**：对撞指针（盛最多水的容器、三数之和）、快慢指针（移除元素、移动零）
* **滑动窗口**：长度最小的子数组、无重复字符的最长子串
* **前缀和与差分**：区域和检索、航班预订统计
* **合并与区间**：合并两个有序数组、合并区间

#### 2. 底层机制高频考点

* **CPython 中 `list` 的扩容公式是怎样的？**
* CPython 并没有采用纯粹的 2 倍扩容，其源码（`listobject.c`）中的过量分配公式大致为：

$$\text{new\_allocated} = \text{newsize} + (\text{newsize} \gg 3) + (\text{newsize} < 9 \mathrel{?} 3 : 6)$$


* 扩容倍数约在 **1.125 倍（$\approx \frac{9}{8}$）** 左右。这种较平缓的增长策略可以在内存占用与重新分配频率之间取得更好平衡，避免大列表扩容时产生大量未利用的内存碎片。


* **为什么 Python 动态数组存储的是“对象引用”而非“真实值”？**
* Python 的变量名本质是指向对象的指针，`list` 存储的是 `PyObject*`。这带来了极致的灵活性（同一个列表中可以存放不同类型的对象，如 `[1, "hello", [3]]`）；
* 代价是**缓存命中率不如纯数值数组**：遍历列表时，虽然指针数组本身的访问是连续的，但在解引用读取实际对象数值时会发生离散寻址，导致 CPU Cache Miss。这也是科学计算领域普遍采用原生连续存储的 **NumPy `ndarray**` 的主要原因。


* **`list.append()` 与 `list.insert(0, ...)` 的性能差异？**
* `append()` 操作在末尾，均摊时间复杂度为 $O(1)$；
* `insert(0, ...)` 在头部插入，必须将现有所有元素的引用依次右移，时间复杂度为严格的 $O(n)$。若频繁需要在序列两端进行高效入队与出队操作，推荐使用标准库双端队列 `collections.deque`（底层为基于双向链表的分块缓冲区）。