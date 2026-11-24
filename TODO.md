# TODO.md — 当前攻略

> 只写**当前小目标**。目标完成 → 整份清空重写。历史不在这里，在 `LOG.md`。
> 概念细节随提问补充；被纠正的误解同步在 `NOTE.md` 留一条浓缩结论。

---

## 当前单元

**U00 跑通编译环境**（0.5–1 天）

一句话：证明「写代码 → 编译 → 出一张图」这条链路是通的，且 Release/Debug 都能跑。
在链路通之前，不要碰任何数学。

---

## 小目标

### G1 · 从零构建成功

产出：`build/textenstein3d.exe`，运行后不崩（哪怕什么也不画）。

参考骨架（这是构建配置，不是算法；逐行看懂再敲）：

```cmake
cmake_minimum_required(VERSION 3.20)
project(textenstein3d LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(textenstein3d src/main.cpp)
```

要知道的三件事：

1. **target 是 CMake 的最小单位**。`add_executable` 造一个 target，名字就是产物名；后面所有配置（include 路径、源文件、优化选项）都挂在 target 上，不是全局。
2. **`-B build` 是 out-of-source 构建**。源码树保持干净，删 `build/` 就等于「干净构建」。验收必须从删掉 `build/` 开始。
3. **`CMAKE_BUILD_TYPE` 只在单配置生成器（Ninja、Makefile）生效**。用 Visual Studio 生成器时它是空的，要用 `--config Release`。先选定一种，别无脑照抄命令。

### G2 · 输出一张 800×600 的 PPM

产出：一个能被看图软件正常打开的纯色（或渐变）图。

1. 帧缓冲用 `std::vector<float>` 存 RGB，**不要用 uint8**。理由见下面「为什么先 float」。
2. 填一个线性空间颜色，比如 `(0.2, 0.4, 0.6)`。
3. 输出前做 gamma 编码，再转 `uint8`。
4. 写 P6 文件。
5. 用字节数校验（见完成条件）。

**P6 格式**（就这些，别想复杂）：

```
P6\n
<width> <height>\n
255\n
然后跟 width*height*3 个字节，RGB 顺序，逐行写，第一行是图像最上面一行
```

头部 `"P6\n800 600\n255\n"` 正好 **15 字节**（2 + 1 + 7 + 1 + 3 + 1）。

**为什么先 float**

`PROJECT.md:37` 定的是「线性 float RGB + float depth」。现在改 uint8，U13 上光照时你还得把整条链路倒回来。而且 float 留了 HDR 余量：光照把像素算到 3.0 也存得下，最后 tonemap 时才压回 [0,1]。这是 D01/M12 的第一次接触，别跳过。

**gamma 的落点**

光照和混合都在线性空间算，**只在最后写文件/写窗口那一步编码一次**。编码公式二选一：

- 快速近似：`encode = pow(linear, 1/2.2)`
- sRGB 精确曲线：`linear <= 0.0031308 ? 12.92*linear : 1.055*pow(linear, 1/2.4) - 0.055`

先随便用一个，但要知道两者差多少（线性 0.5 的输出值见完成条件）。

---

## 坑（按踩到的概率排序）

1. **Windows 下必须二进制模式写文件**。文本模式会把 `0x0A` 翻成 `0x0D 0x0A`，头部从 15 字节变 18，RGB 数据整体错位 3 字节 → 图偏色或斜纹。这是本项目第一个真 bug 的种子。
2. **头部里不要写 `\r\n`**。同上，看图软件会解析失败或偏移。
3. **width / height 写反**。症状是图被拉成竖条。
4. **像素行序搞反**。PPM 第一行是**图像顶部**；而很多图形 API 的缓冲区第一行是底部。现在先记死：PPM 自顶向下。
5. **没开 C++17**。`std::optional`、`if constexpr`、结构化绑定都会报错，报错信息还可能指向标准库内部。
6. **输出路径**。程序的工作目录是 `build/`，写 `out.ppm` 会落在 `build/out.ppm`。想固定位置就自己拼绝对路径或走命令行参数。

---

## 完成条件（可测，缺一不算）

- [ ] `rm -rf build && cmake -B build -S .` → `cmake --build build` 成功，**零 warning**（warning 留着会变成未来的 bug）
- [ ] `build/textenstein3d` 能运行并退出
- [ ] 输出文件字节数 **= 15 + 800×600×3 = 1,440,015**
- [ ] 看图软件打开：均匀色块，无斜纹、无错位、无颜色渐变异常
- [ ] 线性 0.5 的灰度编码后落在 **186–188**（纯 2.2 ≈ 186，sRGB 精确 ≈ 188）；差得多说明 gamma 重复做或漏做
- [ ] Release 和 Debug 都能出图

---

## 下一个目标（预告，先别展开）

U01：`Vec2/3/4` + `Mat4`，跑通 `test_math`。到那时才把 CTest 接进 CMake。
