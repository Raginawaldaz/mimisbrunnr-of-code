在 C 语言中，链表（Linked List）是一种**动态数据结构**，每个节点包含：

1. **数据域（data）**：存储数据
2. **指针域（next）**：指向下一个节点

---

## 1. 定义链表节点

最常见的是单向链表：

```c
#include <stdio.h>
#include <stdlib.h>

// 定义链表节点
typedef struct Node {
    int data;           // 数据
    struct Node *next;  // 指向下一个节点
} Node;
```

这里：

```c
struct Node *next;
```

表示当前节点保存着下一个节点的地址。

---

## 2. 创建节点

通常使用 `malloc()` 动态申请内存：

```c
Node* createNode(int value) {
    Node *newNode = (Node*)malloc(sizeof(Node));

    if (newNode == NULL) {
        printf("内存申请失败\n");
        return NULL;
    }

    newNode->data = value;
    newNode->next = NULL;

    return newNode;
}
```

---

## 3. 创建一个简单链表

例如创建：

```text
10 -> 20 -> 30 -> NULL
```

```c
int main() {
    Node *head = createNode(10);
    Node *second = createNode(20);
    Node *third = createNode(30);

    head->next = second;
    second->next = third;

    return 0;
}
```

内存结构：

```text
head
 ↓
+------+------+
| 10   |  *------+
+------+------+
              ↓
        +------+------+
        | 20   |  *------+
        +------+------+
                      ↓
                +------+------+
                | 30   | NULL |
                +------+------+
```

---

## 4. 遍历链表

打印所有节点：

```c
void printList(Node *head) {
    Node *p = head;

    while (p != NULL) {
        printf("%d -> ", p->data);
        p = p->next;
    }

    printf("NULL\n");
}
```

调用：

```c
printList(head);
```

输出：

```text
10 -> 20 -> 30 -> NULL
```

---

## 5. 头插法

在链表头部插入节点：

```c
void insertHead(Node **head, int value) {
    Node *newNode = createNode(value);

    newNode->next = *head;
    *head = newNode;
}
```

使用：

```c
Node *head = NULL;

insertHead(&head, 30);
insertHead(&head, 20);
insertHead(&head, 10);
```

结果：

```text
10 -> 20 -> 30 -> NULL
```

---

## 6. 尾插法

在链表末尾插入：

```c
void insertTail(Node **head, int value) {
    Node *newNode = createNode(value);

    if (*head == NULL) {
        *head = newNode;
        return;
    }

    Node *p = *head;

    while (p->next != NULL) {
        p = p->next;
    }

    p->next = newNode;
}
```

使用：

```c
Node *head = NULL;

insertTail(&head, 10);
insertTail(&head, 20);
insertTail(&head, 30);
```

结果：

```text
10 -> 20 -> 30 -> NULL
```

---

## 7. 删除节点

删除值为 `20` 的节点：

```c
void deleteNode(Node **head, int value) {
    Node *cur = *head;
    Node *pre = NULL;

    while (cur != NULL) {
        if (cur->data == value) {

            if (pre == NULL) {
                *head = cur->next;
            } else {
                pre->next = cur->next;
            }

            free(cur);
            return;
        }

        pre = cur;
        cur = cur->next;
    }
}
```

执行：

```c
deleteNode(&head, 20);
```

结果：

```text
10 -> 30 -> NULL
```

---

## 8. 释放链表

动态申请的内存必须释放：

```c
void freeList(Node *head) {
    Node *tmp;

    while (head != NULL) {
        tmp = head;
        head = head->next;
        free(tmp);
    }
}
```

调用：

```c
freeList(head);
```

---

## 完整示例

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node {
    int data;
    struct Node *next;
} Node;

Node* createNode(int value) {
    Node *node = (Node*)malloc(sizeof(Node));
    node->data = value;
    node->next = NULL;
    return node;
}

void insertTail(Node **head, int value) {
    Node *newNode = createNode(value);

    if (*head == NULL) {
        *head = newNode;
        return;
    }

    Node *p = *head;

    while (p->next) {
        p = p->next;
    }

    p->next = newNode;
}

void printList(Node *head) {
    while (head) {
        printf("%d -> ", head->data);
        head = head->next;
    }

    printf("NULL\n");
}

int main() {
    Node *head = NULL;

    insertTail(&head, 10);
    insertTail(&head, 20);
    insertTail(&head, 30);

    printList(head);

    return 0;
}
```

输出：

```text
10 -> 20 -> 30 -> NULL
```

### 面试常见问题

链表相关高频题：

* 链表反转（Reverse Linked List）
* 快慢指针找中间节点
* 判断链表是否有环
* 合并两个有序链表
* 删除倒数第 N 个节点
* K 个一组翻转链表

掌握了“定义节点 → 创建 → 插入 → 遍历 → 删除 → 释放”这几个基本操作后，这些题目就容易理解很多。
