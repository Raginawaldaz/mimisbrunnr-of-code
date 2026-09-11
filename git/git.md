在软件开发与团队协作中，**Git** 是一种**分布式版本控制系统**。它将项目的历史演进视为一系列微型文件系统的“快照”，其核心建立在两个机制之上：

1. **三层工作区域**：管理代码从修改、暂存到固化存档的流转状态
2. **提交指针链表（Commit History）**：每个提交记录一次快照，并通过指针指向父提交

---

## 1. Git 的核心心智模型：三大工作区

与直接保存文件的传统软件不同，Git 显式地将文件的修改划分为三个区域：

```text
+---------------------+        git add        +---------------------+       git commit      +---------------------+
|      工作区          | ───────────────────> |      暂存区          | ───────────────────> |      版本库          |
| (Working Directory) |                       |  (Staging / Index)  |                      | (Repository / HEAD) |
+---------------------+ <─────────────────── +---------------------+ <─────────────────── +---------------------+
      日常改写代码的地方                         挑选准备存档的修改                          永久保存的快照历史

```

---

## 2. 基础身份配置

首次使用 Git 时，必须配置提交者的身份信息（用于记录代码由谁提交）：

```bash
# 配置全局用户名
git config --global user.name "YourName"

# 配置全局邮箱
git config --global user.email "your_email@example.com"

# 检查当前配置
git config --list

```

---

## 3. 初始化仓库

在项目根目录下创建一个隐藏的 `.git` 文件夹，将其转换为 Git 仓库：

```bash
# 进入你的项目目录
cd my_project

# 初始化仓库
git init

```

执行后提示：

```text
Initialized empty Git repository in /path/to/my_project/.git/

```

---

## 4. 追踪并暂存文件（git add）

新创建的文件处于“未追踪（Untracked）”状态，需要将其移入暂存区：

```bash
# 查看当前文件状态
git status

# 暂存单个文件
git add main.py

# 暂存当前目录下的所有修改
git add .

```

状态检查：

```bash
git status

```

输出：

```text
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   main.py

```

---

## 5. 提交版本快照（git commit）

将暂存区的内容固化为一个不可逆的独立版本（节点）：

```bash
# 提交并附带简要说明
git commit -m "feat: 初始化项目并添加主入口文件"

```

每个提交在底层都会生成一个唯一的哈希值（如 `a1b2c3d`），用于标识该历史节点。

---

## 6. Commit Message 规范化（约定式提交）

随意书写提交信息（如 `git commit -m "fix"` 或 `git commit -m "update"`) 会导致项目后期难以追溯历史。业界通用的是 **Conventional Commits（约定式提交）** 规范。

基础结构格式：

```text
<type>(<scope>): <subject>

[可选的 body: 详细描述改动原因]
[可选的 footer: 关联的 issue 等]

```

常用 `type` 类别：

* **`feat`**：引入新特性/新功能（Feature）
* **`fix`**：修复 Bug
* **`docs`**：仅修改文档（Documentation）
* **`style`**：不影响代码逻辑的变动（如空格、格式化、缺少分号等）
* **`refactor`**：代码重构（既不是新增功能，也不是修复 Bug）
* **`perf`**：提升性能的代码变动（Performance）
* **`test`**：增加或修正测试用例
* **`chore`**：构建流程、依赖包管理或辅助工具变动

标准示例：

```bash
# 单行标准提交
git commit -m "feat(auth): 新增手机短信验证码登录"
git commit -m "fix(parser): 修复空字符串解析报错的问题"

# 多行详细提交（不加 -m 参数直接输入 git commit，会弹出文本编辑器）
git commit

```

多行提交信息内容范例：

```text
fix(cart): 修复结算金额浮点数精度溢出问题

原因：直接相加导致 JavaScript 浮点数存在误差。
解决：引入 Decimal 精度运算库对商品单价进行统一换算。

Closes #104

```

---

## 7. 查看版本历史（git log）

查看整个项目的提交演进链条：

```bash
# 单行紧凑查看提交记录
git log --oneline --graph

```

输出：

```text
* 3c8e12a (HEAD -> main) docs: 补充使用文档
* 7f91a0b feat: 新增数据处理函数
* e104d5c feat: 初始化项目并添加主入口文件

```

---

## 8. 分支机制：创建与切换（git branch / switch）

分支本质上只是一个**指向特定提交节点的轻量级可移动指针**：

```text
                    (feature)
                        ↓
                 +------------+
                 | Commit C   |
                 +------------+
                       ↑
+------------+   +------------+
| Commit A   | ← | Commit B   |
+------------+   +------------+
                       ↑
                 (main / HEAD)

```

常用命令：

```bash
# 查看所有分支
git branch

# 创建并切换到新功能分支
git switch -c feature-login
# （注：旧版 Git 使用 git checkout -b feature-login）

# 切换回主分支
git switch main

```

---

## 9. 分支合并（git merge）

将特性分支的代码汇入当前分支：

```bash
# 1. 确保先切回接收修改的目标分支（如 main）
git switch main

# 2. 合并指定分支
git merge feature-login

```

如果两个分支没有修改同一文件的同一位置，Git 会自动完成“快进合并（Fast-forward）”或生成一次“合并提交（Merge Commit）”。

---

## 完整操作示例

从空白文件夹开始，走完“新建 $\to$ 修改 $\to$ 暂存 $\to$ 规范化提交 $\to$ 分支合并”的全流程命令：

```bash
# 1. 创建空目录并进入
mkdir demo_git && cd demo_git

# 2. 初始化 Git 仓库
git init

# 3. 创建文件并写入内容
echo "print('Hello Git!')" > app.py

# 4. 查看状态（此时为 Untracked）
git status -s

# 5. 添加至暂存区并进行规范化提交
git add app.py
git commit -m "feat: 初次提交，创建基础打印程序"

# 6. 创建并切换至新分支开发功能
git switch -c dev
echo "def add(a, b): return a + b" >> app.py
git add app.py
git commit -m "feat(math): 增加两数求和工具函数"

# 7. 切回主分支并完成合并
git switch main
git merge dev

# 8. 查看最终提交图
git log --oneline --graph

```

输出：

```text
* 89a2bc1 (HEAD -> main, dev) feat(math): 增加两数求和工具函数
* 4f17c80 feat: 初次提交，创建基础打印程序

```

---

### 新手常见避坑指南

* **`git add .` 加错了不想提交的文件怎么办？**
* 使用 `git restore --staged <文件名>` 将文件从暂存区撤回工作区，文件内容不会丢失。


* **刚提交完发现提交信息写错了怎么办？**
* 执行 `git commit --amend -m "fix: 正确的规范化描述"`，可直接覆盖最近一次提交的信息。


* **合并时出现冲突（Merge Conflict）怎么处理？**
* 打开冲突文件，搜索 `<<<<<<<`、`=======`、`>>>>>>>` 冲突标记。
* 人工决定保留哪一部分代码，删除标记符号后重新执行 `git add <文件名>` 和 `git commit` 即可闭环。


* **`.gitignore` 文件是干什么的？**
* 存放不需要被版本控制的文件规则（如编译产物 `.pyc`、系统隐藏文件 `.DS_Store`、依赖包目录 `node_modules/`、大型数据集）。写入该文件的路径会被 Git 自动忽略。