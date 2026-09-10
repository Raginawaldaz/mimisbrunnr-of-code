# Mímisbrunnr of Code

> 密米尔之泉（The Well of Mímir）—— 知识与智慧生生不息的源泉。本仓库是一个不断生长的集合，收录了多种编程语言下的算法实现与数据结构讲解。

![GitHub last commit](https://img.shields.io/badge/last_commit-2026-blue)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

## 📖 简介

**Mímisbrunnr of Code** 是一个开放的算法笔记与实现集合。我们希望汇聚常见算法与数据结构的清晰、详尽的示例——由不同贡献者用不同语言书写，从而带来不同的视角。

这是一个社区驱动的仓库：无论什么语言、算法还是数据结构，都欢迎投稿。

## 🗂 目录结构

```
mimisbrunnr-of-code/
├── c/                         # C 语言实现与笔记
│   ├── linked_list.md         # C 语言单向链表 —— 基础操作
│   ├── static_linked_list.md  # 静态链表（数组模拟单链表）
│   ├── dynamic_array.md      # 数组与动态数组
│   └── stack.md               # 栈 —— 可动态扩容的顺序栈
├── python/                    # Python 实现与笔记
│   └── dynamic_array.md       # 用 ctypes 从零实现动态数组
├── os/                        # 操作系统笔记
│   └── deadlock.md            # 死锁与银行家算法（C 实现）
├── linux/                     # Linux 操作与排查笔记
│   └── linux_compass.md      # 核心命令、负载分析与 TCP 调优
├── README.md                  # 本项目说明（英文版）
└── README-CN.md               # 中文版说明
```

新增语言会以顶层目录的形式添加（例如 `python/`、`go/`、`rust/`、`java/`）。不绑定单一语言的主题笔记（例如操作系统、Linux）放在各自的顶层目录中。

## ✅ 已有内容

### C

- **单向链表** —— `c/linked_list.md`
  - 基础操作：定义节点、创建、遍历、头插、尾插、删除、释放链表，附完整可运行示例与常见面试题。
- **静态链表** —— `c/static_linked_list.md`
  - 用数组下标模拟指针：`val` / `ne` 平行数组、`head` 与简易分配器 `idx`，以及竞赛风格的基本操作。
- **数组与动态数组** —— `c/dynamic_array.md`
  - 静态数组与动态数组的区别；`data` / `size` / `capacity`；扩容、随机访问，以及完整的 C 实现。
- **栈** —— `c/stack.md`
  - 后进先出（LIFO）的动态顺序栈：`data` / `top` / `capacity`，入栈、出栈及相关操作。

### Python

- **动态数组** —— `python/dynamic_array.md`
  - 不封装内置 `list`，用 `ctypes` 申请物理连续内存，从零复刻动态数组（元素个数、容量与扩容）。

### 操作系统

- **死锁** —— `os/deadlock.md`
  - 死锁、饥饿与死循环的辨析，四大必要条件，以及银行家算法（死锁避免）的完整 C 实现。

### Linux

- **Linux 核心操作与系统排查** —— `linux/linux_compass.md`
  - 文件系统与 Inode、权限、管道与 grep/sed/awk 三剑客、Load Average 与 CPU/I/O 瓶颈判定、TCP `TIME_WAIT` 调优，以及巡检脚本示例与面试要点。

更多内容持续更新中。

## 🎯 后续计划

- [x] C —— 单向链表
- [x] C —— 静态链表
- [x] C —— 动态数组
- [x] C —— 栈
- [x] Python —— 动态数组
- [x] OS —— 死锁与银行家算法
- [x] Linux —— 核心操作与系统排查
- [ ] 更多 C 数据结构
- [ ] 更多 Python / Go / Rust / Java 实现
- [ ] 常见算法分类（排序、查找、动态规划等）
- [ ] 各语言的测试与编译说明

## 🧑‍🤝‍🧑 投稿与参与

正是大家的贡献，让这个仓库真正成为一座「密米尔之泉」——无论新增算法、优化实现、完善讲解，还是修正已有笔记，都非常欢迎，请提 **Pull Request**。

### 参与步骤

1. **Fork** 本仓库。
2. 为你的改动新建一个**分支**（`git checkout -b feat/pystack`）。
3. 进行修改，保持既有风格（注释清晰、尽量附带可运行示例，并有简短讲解）。
4. 用描述性的信息完成 **Commit**。
5. **Push** 到你的 fork，并提交 **Pull Request**。

### 约定

- 按语言放入对应的顶层目录；若为新语言，请新建一个目录。
- 代码旁附简短说明（实现做什么、如何运行）。
- 对新手友好——可以多解释，但别注水。
- 提交信息遵循 [Conventional Commits](https://www.conventionalcommits.org/)（`feat:`、`fix:`、`docs:`）。

## 📄 许可证

本项目采用 **Apache License 2.0** 许可证。详见 [LICENSE](LICENSE) 文件。

---

*畅饮此泉，与人分享你所学的。* 🌊
