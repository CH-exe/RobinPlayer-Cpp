# RobinPlayer（C++ 版）

[![build](https://github.com/CH-exe/RobinPlayer-Cpp/actions/workflows/build.yml/badge.svg)](https://github.com/CH-exe/RobinPlayer-Cpp/actions/workflows/build.yml)

在 Windows 控制台里播放《使一颗心免于哀伤》：用 `#` 逐像素画出知更鸟，
滚动词曲，同时用 MCI 播放 MIDI 音乐；歌曲中段还会有一个"逐点点亮全屏"的小彩蛋。

这是 [SyrieYume/RobinPlayer](https://github.com/SyrieYume/RobinPlayer)（C 语言，GPL-3.0）的
**C++ 重写版**。原版 `main.c` 把音乐、歌词、两张位图用 Base64 拆成碎字符串藏在源码里，
本仓库沿用这个思路，把改写后的逻辑和这四份资源同样放进**一个 `.cpp`**，
所以编译时不需要任何外部资源文件，程序首次运行会自动把资源解到当前目录。

```
RobinPlayer-Cpp/
├── main.cpp          # C++17 版：string_view / chrono / mt19937，代码更现代
├── main_cpp98.cpp    # C++98 兼容版：任何编译器默认设置都能直接编过（推荐）
├── LICENSE           # GPL-3.0
├── README.md
└── .github/workflows/build.yml
```

## 为什么有两个版本？

`main.cpp` 用了 `std::string_view`、`constexpr`、`std::chrono`、`std::mt19937`，**需要 C++17**。
如果 IDE 默认按 C++98/14 编译（Visual Studio 新建项目默认就是 C++14，老版 Dev-C++ 默认 gnu++98），
会在这些行报错：

```
constexpr std::string_view kMusicPath = "music.mid";   //error: 'string_view' is not a member of 'std'
```

`main_cpp98.cpp` 把所有 C++11/17 特性都换成了 C++98 就有的写法，
**打开 IDE → 编译 → 运行**即可，不用改任何编译选项：

| C++17 版 | C++98 兼容版 |
| --- | --- |
| `std::string_view` | `const char*` + 长度 |
| `constexpr` | `enum` 常量 / `const char[]` |
| `std::chrono` + `sleep_until` | `GetTickCount()` + 精确 `sleepUntil()` |
| `std::mt19937` + `std::shuffle` | `srand/rand` + 手写 Fisher-Yates |
| `std::to_string` | `sprintf` |
| `std::size` / `std::array` / `nullptr` | `sizeof` / C 数组 / `NULL` |

两份实现的功能、画面和输出完全一致（见文末"验证"）。

## 编译

只依赖 C++ 标准库和 `windows.h`，`winmm.dll` 是运行时用 `LoadLibrary` 动态加载的，
**不需要** `-lwinmm`。源文件请以 UTF-8 保存（仓库里的文件带 BOM，MSVC 也能正确识别）。

MinGW-w64 / Dev-C++：

```bat
g++ -O2 -o RobinPlayer.exe main_cpp98.cpp
:: 想用现代版：
g++ -std=c++17 -O2 -o RobinPlayer.exe main.cpp
```

MSVC：

```bat
cl /nologo /EHsc /O2 main_cpp98.cpp
:: 想用现代版：
cl /nologo /EHsc /O2 /std:c++17 /utf-8 main.cpp
```

## 运行

双击 `RobinPlayer.exe`，等窗口出现后把它调整到合适大小，然后欣赏。
首次运行会在当前目录生成 `music.mid`、`lyric.lrc`、`pic1.bmp`、`pic2.bmp`
（已存在则跳过，不会覆盖）。

```bat
RobinPlayer.exe             :: 正常播放
RobinPlayer.exe --extract   :: 只解出这 4 个资源文件后退出，方便调试
```

要求 **Windows 10 及以上**（需要控制台的虚拟终端序列支持，用来显示 ANSI 真彩色）。

## 实现说明

保持原版的功能和视觉效果不变，只把实现方式换成 C++：

| 原 C 版本 | 本仓库 |
| --- | --- |
| `malloc/free`、`Pixel*` | `Bitmap` 类 + `std::vector<Pixel>`，自动释放 |
| 手写 `decode_table` 查表的 Base64 解码 | 标准 Base64 流式解码 |
| 固定 90 KB `strBuffer` + `sprintf_s` | `std::string` 追加 + 局部 `sprintf` |
| `clock()` + `while(...) Sleep(100)` 轮询 | `GetTickCount`（现代版用 `steady_clock`）+ 精确等待 |
| `srand(time(NULL)) + rand()` 手写洗牌 | 同样的随机源（现代版用 `mt19937 + shuffle`） |
| 出错 `system("pause"); exit(-1)` | 抛 `std::runtime_error`，`main` 统一捕获并恢复光标 |
| `_access` / `fopen_s` | `GetFileAttributesA` / `std::ofstream` |
| 动态取 `mciSendStringA` | 封装成 RAII 的 `MciPlayer`，析构时自动 `stop` + `close` |

顺手修掉的两处隐患：

- BMP 按 4 字节行对齐读取并校验文件头（原版直接按 `width * height * 3` 读，遇到带行填充的 BMP 会错位）；
- 程序结束时恢复光标（`ESC[?25h`）并重置颜色。

## 验证

- `main_cpp98.cpp` 在 `-std=c++98 / c++11 / c++14 / c++17 / c++20` 下均编译通过，
  `-Wall -Wextra -pedantic` 零警告；
- 两份实现用同一段驱动代码渲染同一批画面（图片 + 26 帧歌词），输出**逐字节相同**；
- 解出的 4 个资源与原版 C 代码 `base64_decode` 的结果**逐字节相同**
  （10512 / 2432 / 4112 / 10856 字节）；
- `pic1` / `pic2` 的像素数据与原 C 算法（读 `w*h*3` 再上下翻转）**差异 0 字节**。

## 版权与致谢

- 代码：基于 [SyrieYume/RobinPlayer](https://github.com/SyrieYume/RobinPlayer)，遵循 **GPL-3.0**，
  见 [LICENSE](LICENSE)。
- 音乐、歌词、图片等素材来自游戏《崩坏：星穹铁道》，版权归 **miHoYo / HoYoverse** 所有。
  本仓库只是沿用上游项目的做法，把这些素材作为演示数据内嵌在源码里；
  如有侵权请联系删除。
