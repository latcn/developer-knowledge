
---
## 🧰 基础工具库

### 1. MyTinySTL

**GitHub 地址**：https://github.com/Alinshans/MyTinySTL

一个用 C++11 实现的小型 STL（标准模板库），包含容器库和算法库。代码结构清晰规范，配有中文文档与注释，并自带简单的测试框架，非常适合新手学习 STL 底层实现原理。项目从 2.x.x 版本开始进入长期维护阶段，主要修复 bug 不再增加新内容。支持 Linux、Windows、macOS 平台，需 g++ 5.4 以上或 clang++ 3.5 以上编译器。

### 2. fmtlib/fmt

**GitHub 地址**：https://github.com/fmtlib/fmt

一个现代 C++ 格式化库，提供快速且安全的格式化方案，可替代 C 的 `printf`/`sprintf` 和 C++ 的 iostreams。实现了 C++20 的 `std::format` 和 C++23 的 `std::print`，格式化语法类似 Python 的 `str.format`。该库**完全类型安全**，格式字符串错误可在编译期报告，自动内存管理防止缓冲区溢出。无外部依赖，采用 MIT 许可证。

### 3. spdlog

**GitHub 地址**：https://github.com/gabime/spdlog

一个**非常快速**的 C++ 日志库，支持 header-only 或编译模式。基于 fmtlib 实现，提供高性能的日志记录能力，支持多线程安全、异步日志、自定义格式等特性，广泛应用于各类 C++ 工程中。

### 4. cereal

**GitHub 地址**：https://github.com/USCiLab/cereal

一个 C++11 的**序列化库**，header-only，无外部依赖。可以将任意数据类型序列化为紧凑的二进制编码、XML 或 JSON 等不同格式。采用 BSD 许可证。使用方式非常简洁——只需为自定义类型编写 `serialize` 函数即可完成序列化/反序列化。

### 5. nlohmann/json

**GitHub 地址**：https://github.com/nlohmann/json

被誉为“**JSON for Modern C++**”，整个库仅由**单个头文件** `json.hpp` 构成，无任何依赖。设计目标是提供**直觉的语法**，让 C++ 中的 JSON 操作像 Python 一样自然。支持 JSON 与 STL 容器的双向转换、JSON Pointer、JSON Patch、以及 BSON/CBOR/MessagePack 等二进制格式。采用 MIT 许可证。

### 6. ctpl

**GitHub 地址**：https://github.com/vit-vit/CTPL

一个**现代高效的 C++11 线程池库**。线程池是一种编程模式，用于并行执行任务，可以最小化线程创建和销毁的开销，同时限制并行任务的数量。代码量极小（约 500 行），适合学习 C++ 并发编程的核心模型。

### 7. cxxopts

**GitHub 地址**：https://github.com/jarro2783/cxxopts

一个**轻量级的 C++ 命令行选项解析库**，支持标准的 GNU 风格选项语法。header-only，采用 MIT 许可证。API 设计简洁，支持选项的链式配置和类型安全的参数解析。

### 8. tinyexpr

**GitHub 地址**：https://github.com/codeplea/tinyexpr

一个**非常小的递归下降表达式解析器和求值引擎**，用于在运行时动态计算数学表达式。如果你需要在项目中加入数学表达式求值功能，又不想引入大量依赖，tinyexpr 是理想选择。采用 MIT 许可证。代码精简，适合学习递归下降解析和抽象语法树（AST）的构建。

---

## 🌐 网络编程

### 9. Tinyhttpd

**GitHub 地址**：https://github.com/EZLippi/Tinyhttpd

一个**不到 500 行的超轻量级 HTTP 服务器**，由 J. David Blackstone 于 1999 年用 C 语言编写。主要用于**教育目的**，帮助开发者深入理解 HTTP 协议和网络编程的基本原理。支持多线程处理、静态请求和 CGI 协议。虽然不适合生产环境，但却是理解 Web 服务器本质的绝佳入门项目。

### 10. TinyWebServer

**GitHub 地址**：https://github.com/qinguoyi/TinyWebServer

一个**Linux 下 C++ 轻量级 Web 服务器**，助力初学者快速实践网络编程。采用 **线程池 + 非阻塞 socket + epoll（ET 和 LT 均实现）+ 事件处理（Reactor 和模拟 Proactor 均实现）** 的并发模型。经 Webbench 压力测试可实现上万的并发连接数据交换。代码量约 2000 行，是理解高并发网络编程的经典范例。

### 11. muduo

**GitHub 地址**：https://github.com/chenshuo/muduo

陈硕开发的**基于 Reactor 模式的多线程 C++ 网络库**。用于构建高性能的多线程服务器和应用程序。此外，社区有 **`muduo_cpp11_std`** 重构版（https://github.com/Kirin9900/muduo_cpp11_std），去除了所有 Boost 依赖，采用标准库模式和智能指针，充分利用 C++11/14/17 新特性，更便于阅读和使用。

---

## 📚 综合学习资源

### 12. CPlusPlusThings

**GitHub 地址**：https://github.com/Light-City/CPlusPlusThings

一个国人开源的**系统性 C++ 学习项目**，将 C++ 学习分为**基础进阶、实战系列、C++2.0 新特性、并发编程、C++ 惯用法**等多个模块。适合初学者从**入门到进阶**，解决了学习者想要深入 C++ 及如何入坑的问题。配有配套网站和微信公众号，是系统性学习 C++ 的优质资源。

### 13. C-Plus-Plus (TheAlgorithms)

**GitHub 地址**：https://github.com/TheAlgorithms/C-Plus-Plus

收录了用 C++ 实现的**各种算法的集合**，涵盖数学、机器学习、计算机科学、物理等领域。所有实现均**使用 STL 类，无外部库依赖**。代码严格遵循 **C++17 标准**，保证了代码在不同平台（Windows、MacOS、Linux）和嵌入式系统上的可移植性。每个程序都包含自检机制确保正确性。如果你想了解某个特定算法（如排序、搜索、图论）的多种 C++ 实现，这里是绝佳的资源库。

---

## 学习路径建议

| 阶段 | 推荐项目 | 学习目标 |
| :--- | :--- | :--- |
| **入门（第1-2周）** | MyTinySTL、cxxopts | 理解 STL 容器实现、模板编程、RAII |
| **进阶（第3-4周）** | nlohmann/json、spdlog、cereal | 学习现代 C++ API 设计、序列化、类型安全 |
| **并发（第5周）** | ctpl | 理解线程池、future/promise、并发模型 |
| **网络编程（第6-8周）** | Tinyhttpd → TinyWebServer → muduo | 从 HTTP 协议到高并发 Reactor 模型的完整链路 |
| **持续学习** | CPlusPlusThings、C-Plus-Plus | 系统性查漏补缺、算法实现巩固 |

祝学习顺利！