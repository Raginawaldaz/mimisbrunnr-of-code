# Mímisbrunnr of Code

> The Well of Mímir — a living source of knowledge and wisdom. This repository is a growing collection of algorithm implementations and data-structure walkthroughs across multiple programming languages.

![GitHub last commit](https://img.shields.io/badge/last_commit-2026-blue)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

## 📖 About

**Mímisbrunnr of Code** is an open collection of algorithm notes and implementations. It aims to gather clear, well-documented examples of common algorithms and data structures — written in different languages by different contributors, so each one can be appreciated from a fresh perspective.

This is a community-driven repository: contributions of any language, algorithm, or data structure are warmly welcome.

## 🗂 Repository Structure

```
mimisbrunnr-of-code/
├── c/                         # C language implementations & notes
│   ├── linked_list.md         # Singly linked list — basic operations in C
│   ├── static_linked_list.md  # Static linked list (array-simulated)
│   ├── dynamic_array.md      # Array & dynamic array (vector)
│   └── stack.md               # Stack — sequential stack with dynamic growth
├── python/                    # Python implementations & notes
│   └── dynamic_array.md       # Dynamic array from scratch with ctypes
├── os/                        # Operating-system notes
│   └── deadlock.md            # Deadlock & Banker's Algorithm (C)
├── README.md                  # This file (English)
└── README-CN.md               # Chinese version
```

New languages are added as top-level directories (e.g. `python/`, `go/`, `rust/`, `java/`). Topic notes that are not tied to a single language (for example operating systems) live in their own top-level folders.

## ✅ Current Content

### C

- **Singly Linked List** — `c/linked_list.md`
  - Basic operations: define node, create, traverse, insert at head, insert at tail, delete, free the list, plus a complete runnable example and common interview questions.
- **Static Linked List** — `c/static_linked_list.md`
  - Array-simulated singly linked list: `val` / `ne` parallel arrays, `head` and `idx` as a simple allocator, typical contest-style operations.
- **Array & Dynamic Array** — `c/dynamic_array.md`
  - Static vs dynamic arrays; `data` / `size` / `capacity`; growth, random access, and a complete C implementation.
- **Stack** — `c/stack.md`
  - LIFO sequential stack with dynamic growth: `data` / `top` / `capacity`, push, pop, and related operations.

### Python

- **Dynamic Array** — `python/dynamic_array.md`
  - A from-scratch dynamic array using `ctypes` contiguous buffers (not wrapping built-in `list`), covering size, capacity, and growth.

### Operating Systems

- **Deadlock** — `os/deadlock.md`
  - Deadlock vs starvation vs infinite loop, the four necessary conditions, and a complete C implementation of the Banker's Algorithm (deadlock avoidance).

More items are on the way.

## 🎯 Roadmap

- [x] C — Singly Linked List
- [x] C — Static Linked List
- [x] C — Dynamic Array
- [x] C — Stack
- [x] Python — Dynamic Array
- [x] OS — Deadlock & Banker's Algorithm
- [ ] More C data structures
- [ ] More Python / Go / Rust / Java implementations
- [ ] Common algorithm categories (sorting, searching, dynamic programming, etc.)
- [ ] Tests and build instructions per language

## 🧑‍🤝‍🧑 Contributing

Contributions are what make this repository a true *Well of Mímir* — welcome and appreciated! Whether it's a new algorithm, a cleaner implementation, a better explanation, or a fix to an existing note, please **open a Pull Request**.

### How to contribute

1. **Fork** this repository.
2. Create a new **branch** for your change (`git checkout -b feat/pystack`).
3. Make your changes, keeping the existing style (clear comments, a runnable example where possible, and a short explanation).
4. **Commit** with a descriptive message.
5. **Push** to your fork and open a **Pull Request**.

### Guidelines

- Group code by language in the matching top-level directory; if the language is new, create a new directory.
- Add a short explanation alongside the code (what it does, how to run it).
- Keep it beginner-friendly — fine to explain, but don't pad.
- Follow [conventional commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `docs:`) for your commit messages.

## 📄 License

This project is licensed under the **Apache License 2.0**. See the [LICENSE](LICENSE) file for details.

---

*Drink deeply from the Well, and share what you learn.* 🌊
