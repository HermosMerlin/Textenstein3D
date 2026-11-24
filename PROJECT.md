# PROJECT.md — Textenstein 3D

一句话：用 C++17 从零写一个 CPU 软光栅器，把 Textenstein 的 2D 地图渲染成可自由漫游的 3D 场景，并用它验证变换、光栅化、深度、透视校正、纹理和 Blinn-Phong 六个知识块。

---

## 1. 项目定位

### 是什么

- **学习项目**：核心算法由你手写，AI 只解释、review、出题。
- **CPU 软光栅器**：不调用 OpenGL / D3D / Vulkan。
- **Textenstein 的 3D 续作**：复用它的地图和墙面纹理，把 2.5D raycaster 升级成真 3D。
- **作品集项目**：有截图、性能数字、3 个 bug、90 秒视频、20 分钟讲解。

### 不是什么

- 不是 ray tracer。
- 不是 GPU 渲染器；OpenGL 移植是下个月或 stretch。
- 不是完整游戏：不做敌人、射击、声音、UI。
- 不是 PBR 引擎：v0.1 用 Blinn-Phong。
- 不是模型加载器：v0.1 只渲染地图生成的网格。
- 不是多线程渲染器：单线程正确并测量后，才考虑多线程。

---

## 2. 技术约束

| 项 | 约束 |
|---|---|
| 语言 | C++17 |
| 构建 | CMake 3.20+，Ninja 或 MSVC |
| 平台 | Windows 10/11 x64 |
| 依赖 | C++ 标准库 + Windows SDK；无第三方库 |
| 窗口 | Win32 + DIB section，800×600，32-bit top-down |
| 无窗口模式 | PPM (P6) 输出，用于测试和截图 |
| 渲染目标 | 线性 float RGB + float depth |
| 颜色 | 光照在线性空间；输出前 tonemap + gamma |
| 输入 | WASD 移动，方向键 yaw/pitch，Esc 退出，F1 性能面板，F2 线框，F3 深度视图，F12 存 PPM |
| 鼠标 | 可选；先键盘，鼠标 look 是 stretch |
| 构建类型 | Debug（`-O0 -g`）找 bug；Release（`-O2` / `/O2`）测性能 |
| 目标 | 800×600 / ≥5k 三角形 / ≥30fps（Release） |

### 坐标系契约（必须写进 README）

这一条是第一个月最容易翻车的地方。先定死，后面不要改：

- **世界**：右手系，+X 右，+Y 上，+Z 朝屏幕外；相机看向 -Z。
- **地图**：`map.x` → world X，`map.y` → world Z；墙高 1.0；相机 eye 在 y=0.5。
- **矩阵**：column-major，`v_clip = P * V * M * v`；`Mat4` 存 16 个 float。
- **NDC**：x,y ∈ [-1,1]，z ∈ [-1,1]（OpenGL 风格）；近平面 z=-1，远平面 z=+1。
- **深度**：存 NDC z；less 比较；clear 1.0。
- **视口**：屏幕 y 向下；`screen_x = (ndc_x*0.5+0.5)*W`，`screen_y = (1-(ndc_y*0.5+0.5))*H`。
- **绕序**：NDC 里逆时针为正面；经过 y 翻转后，屏幕空间变成顺时针。用单三角形测试确认，不要凭感觉。

如果你后来改成 D3D 风格（左手、NDC z [0,1]），可以，但必须全项目统一，并在 README 写清楚。

### 分辨率与像素中心

- 内部渲染分辨率 800×600；窗口固定 800×600（不做缩放）。
- 像素中心在 `(x+0.5, y+0.5)`。
- 三角形覆盖测试用 top-left rule，避免相邻三角形裂缝和重复着色。

---

## 3. 资产

从 Textenstein 复制，不修改原仓库：

```bash
cd F:/code/Textenstein3D
mkdir -p assets
cp ../Textenstein/assets/default_map.txt assets/
cp ../Textenstein/assets/complex_map.txt assets/
cp ../Textenstein/assets/wall_brick_16.txt assets/
```

- **地图格式**：`#` 墙，`.` 空地，`P` 玩家起点，`E` 敌人（v0.1 忽略）。
- **纹理格式**：32×32 的 ASCII 数字，0–9；按 `digit/9` 转灰度，或自己定一套 palette。
- **颜色空间**：把 ASCII 灰度当作 sRGB 编码值；加载时解码到线性，输出时再编码。
- v0.1 不需要 OBJ / glTF / PNG。想加 PPM (P6) 作为第二种纹理格式可以，但先让 ASCII 纹理跑通。

### 地图到 3D 的映射

- 每个 `#` 格子是一个 1×1×1 的墙块。
- 只生成与空地相邻的墙面；两个墙块之间的面不生成。
- 每面墙是一个四边形，拆成两个三角形；UV 按 0–1。
- 地板和天花板各是一个大四边形，纹理 repeat；如果地图很大，可以按格子拆分，但先做一个大四边形。
- 墙高 1.0；相机 eye 高度 0.5；玩家起点 `P` 的格子中心。

---

## 4. 功能范围

### v0.1 必须

1. 加载 `.map`，生成墙面四边形；只生成与空地相邻的面。
2. 地板和天花板：两个大四边形，纹理 repeat。
3. 自由相机：WASD + 方向键；透视投影。
4. 三角形光栅化：edge function、重心坐标、top-left rule、包围盒裁剪。
5. 背面剔除 + 近平面裁剪。
6. z-buffer：逐像素深度测试；深度视图。
7. 透视校正插值：UV、法线、世界坐标。
8. 纹理：nearest + bilinear；wrap repeat/clamp；mipmap；sRGB 解码。
9. Blinn-Phong：ambient + diffuse + specular；一个方向光。
10. 性能面板：帧时间、FPS、三角形数、着色像素数；CSV 输出。
11. PPM 截图；F12 保存。
12. 测试：math、raster、texture、scene。
13. README、90 秒视频、20 分钟提纲、3 个 bug 记录。

### 可选 / stretch（最多选一个）

- 鼠标 look。
- 阴影贴图（S06）。
- OpenGL 最小对照（A03）。
- 多线程（C06 / P05）。
- 法线贴图（S04）。
- MSAA（R10）。
- 视锥剔除（R11）。

### 不做

- PBR / IBL / GI。
- 后处理链（bloom / DOF / motion blur）。
- 动画 / 骨骼 / 粒子。
- 模型加载。
- 编辑器 / UI。
- 网络 / 存档。
- 敌人 / 射击 / 声音。

---

## 5. 阶段与验收

时间不锁死；按阶段推进。每个阶段结束必须满足“验收”，否则不进下一阶段。
建议每阶段 1 周，但进度落后时按第 11 节砍功能，不砍验收标准。

### Stage 1 · 变换与相机

**交付**

- `Vec2` / `Vec3` / `Vec4`、`Mat4`、`Image` / `Framebuffer`。
- `.map` 加载 + 墙面 mesh 生成。
- 自由相机 + view / projection。
- 线框 PPM：从 3D 看地图。

**验收**

- 白板推 MVP；能说清 model / world / view / clip / NDC / screen 每一步。
- 单位立方体 8 个角点投影到屏幕，与手算误差 < 1 px。
- `Mat4` 单测：identity、multiply、lookAt、perspective。
- 线框图能看出地图结构；相机移动方向正确。
- 证据：`docs/evidence/MVP-whiteboard.*`、`M01-M05-tests.txt`、`Stage1-wireframe.ppm`。

**知识**：M01–M05、R07、R08；M06、C01–C03。

---

### Stage 2 · 光栅化与深度

**交付**

- 三角形填充；top-left rule；相邻三角形无裂缝。
- 背面剔除。
- 近平面裁剪（Sutherland–Hodgman 或等价）。
- z-buffer + 深度视图。
- 第一版帧时间测量。

**验收**

- 单三角形覆盖像素与手算一致；两个相邻三角形无缝隙、无重复。
- 两个相交三角形遮挡正确；深度视图能看出前后关系。
- 走进墙面时，跨近平面的三角形不爆炸、不整块消失。
- near=0.1 vs near=1.0 的深度精度对比；能解释 z-fighting。
- 800×600 下记录第一组帧时间。
- 证据：`R02-near-clip.png`、`R03-triangle.png`、`R05-depth.csv`、`R09-backface.png`。

**知识**：R02、R03、R05、R09；R01、M13、C05、C09、P01。

---

### Stage 3 · 纹理与透视

**交付**

- 透视校正 UV / 法线 / 世界坐标。
- 纹理类：ASCII 加载、nearest、bilinear、wrap、mipmap、sRGB 解码。
- 地板棋盘格；墙面纹理。
- 过滤对比图。

**验收**

- 45° 地板棋盘格直线不弯；与 affine 插值对比能看出差别。
- 远处棋盘格不闪；mipmap 开/关对比明显。
- nearest vs bilinear 对比；wrap repeat 在地板上正确。
- sRGB vs linear 对比；光照颜色正确。
- 选一个像素，用射线–平面求交算 UV，与插值结果误差 < 1e-3。
- 证据：`R04-perspective.png`、`S05-filtering.png`、`M11-mipmap.png`、`M12-linear-vs-srgb.png`。

**知识**：R04、S05；M11、M12、D01。

---

### Stage 4 · 光照与交付

**交付**

- Blinn-Phong：ambient / diffuse / specular；方向光。
- 性能面板 + CSV；overdraw 统计。
- README、90 秒视频、20 分钟提纲、3 个 bug。
- 最终截图 / GIF。

**验收**

- 光照随法线和光线方向正确变化；specular 高光可见。
- 800×600 / ≥5k 三角形 ≥30fps；记录 min / avg / p95。
- 能说出瓶颈：三角形数、像素数、overdraw、cache、分支。
- 90 秒视频能讲清“是什么、最难的决定、一个 bug”。
- 20 分钟提纲覆盖：约束 → 架构 → 三个难点 → 三个 bug → 性能 → 如果重做。
- 证据：`S01-blinn-phong.png`、`P01-perf.csv`、`P03-overdraw.png`、`W04-90s.mp4`。

**知识**：S01；P01、P03、C10；W04（讲解，交付要求，不计入 28 个技术节点）。

---

## 6. 验收标准（可测）

| 项 | 标准 |
|---|---|
| 投影 | 8 个立方体角点屏幕坐标误差 < 1 px |
| 光栅 | 相邻三角形无裂缝；top-left rule 不重复着色 |
| 深度 | 3 组相交三角形排序正确；深度视图可读 |
| 近裁剪 | 相机贴墙时无爆炸 / 整块消失 |
| 透视校正 | 45° 棋盘格直线不弯；UV 与解析求交误差 < 1e-3 |
| 纹理 | nearest / bilinear / wrap / mipmap 四项都有对比图 |
| 颜色 | 光照在线性空间；输出 gamma 正确 |
| 光照 | Blinn-Phong 三项可分别开关；高光随视角变化 |
| 性能 | 800×600 / ≥5k 三角形 ≥30fps（Release）；有 CSV |
| 代码 | 无裸 `new` / `delete`；核心类有 RAII；无第三方依赖 |
| 测试 | `ctest` 全绿；至少 4 个测试文件 |
| 文档 | README 有构建 / 控制 / 架构 / 性能 / 3 bug / 局限 |

---

## 7. 性能与测量

- **计时**：`QueryPerformanceCounter`；每帧记录 CPU 时间。
- **统计**：300 帧（60 帧 warmup）的 min / avg / p95 / max。
- **指标**：帧时间、FPS、三角形数、着色像素数、overdraw 比。
- **输出**：`docs/evidence/perf-<stage>.csv`；F1 面板实时显示。
- **优化顺序**：包围盒 → 背面剔除 → early depth（先测深度再着色）→ 减少 per-pixel 分配 → 多线程（stretch）。
- **规则**：不许先优化后测量；每次优化记录 before / after。

### 目标数字

- 800×600，场景 ≥5k 三角形，Release，≥30fps。
- 如果地图三角形不足，细分地板补到 5k，作为压力测试。
- 如果达不到 30fps，先记录瓶颈和尝试过的优化，不要藏数字。

---

## 8. 测试

- **math**：Vec / Mat 基本运算、lookAt、perspective、已知点投影。
- **raster**：单三角形覆盖、top-left rule、z-buffer、背面。
- **texture**：bilinear 已知值、wrap、mipmap LOD。
- **scene**：1×1 地图单面墙，断言中心像素颜色；地板 UV 解析对比。
- **golden image**：固定相机渲染 PPM；逐像素 diff，允许 ≤2/255 的误差和 1% 离群点。
- **测试运行器**：简单 `CHECK` 宏 + CTest；不引入 GTest。
- **测试不依赖窗口**：用内存 framebuffer 或 PPM。

### 至少覆盖的场景

1. 单三角形：覆盖像素与手算一致。
2. 两个相邻三角形：无裂缝、无重复。
3. 两个相交三角形：深度排序正确。
4. 近平面裁剪：三角形跨近平面时被裁，不是被丢。
5. 纹理：bilinear 在 0.5 偏移处的已知值。
6. 地板：UV 与解析射线–平面求交一致。

---

## 9. 代码结构建议（可改）

```
include/
  math/vec.h, mat.h
  render/framebuffer.h, rasterizer.h, texture.h, camera.h, mesh.h, scene.h
  platform/window.h
src/
  main.cpp
  math/mat.cpp
  render/rasterizer.cpp
  render/texture.cpp
  platform/win32_window.cpp
test/
  test_math.cpp
  test_raster.cpp
  test_texture.cpp
  test_scene.cpp
assets/
docs/evidence/
```

这是建议，不是硬性目录。但“地图 → 网格 → 变换 → 光栅 → 着色 → 输出”这条数据流必须能画出来：

```
map.txt → MapGrid → MeshBuilder → Mesh
input → Camera → view / projection
Mesh + Camera + Light + Texture → Rasterizer → Framebuffer(linear) + Depth
Framebuffer → tonemap / gamma → BGRA8 → Win32 DIB / PPM
```

### 数据结构建议

- `Vertex { Vec3 position; Vec3 normal; Vec2 uv; }`
- `Mesh { std::vector<Vertex> vertices; std::vector<uint32_t> indices; }`
- `Texture { int width, height; std::vector<Vec3> texels; }`（线性值）
- `Framebuffer { int width, height; std::vector<Vec3> color; std::vector<float> depth; }`
- `Camera { Vec3 position; float yaw, pitch; float fov, near, far; Mat4 view, proj; }`

这些只是建议；你可以用不同的名字或布局，但要在 README 的架构一节写清楚。

---

## 10. 交付物

- **`README.md`**：一句话、截图/GIF、构建、控制、架构图、性能表、3 个 bug、局限、如果重做。
- **`LEARNING_LOG.md`**：每周一页；你写了什么、卡在哪、怎么出来的、下周验证什么。
- **`docs/evidence/`**：每个知识节点的证据。
- **`docs/20min-outline.md`**：20 分钟讲解提纲。
- **90 秒视频**：是什么、最难的决定、一个 bug。
- **3 个 bug**：现象、定位、根因、修复、如何防。

### README 必须回答

1. 这是什么？约束是什么？
2. 怎么构建、怎么操作？
3. 一帧的数据流是什么？
4. 性能数字是多少？瓶颈在哪？
5. 最难的一个决定是什么？为什么？
6. 三个 bug 是什么？怎么定位的？
7. 如果重做，先改哪三件事？

---

## 11. 砍功能顺序

如果进度落后，按这个顺序砍，不要砍验收标准：

1. 鼠标 look → 只用键盘。
2. 天花板 → 只做地板。
3. bilinear → nearest。
4. mipmap → 只做 nearest + LOD bias。
5. specular → 只做 diffuse。
6. 窗口 → 只输出 PPM。
7. 纹理地板 → 纯色地板。

**不能砍**：MVP、三角形填充、z-buffer、近平面裁剪、透视校正、单测、帧时间测量。

---

## 12. Stretch（最多一个）

- **S06 阴影贴图**：需要深度纹理 + 灯光矩阵；约一周。
- **A03 OpenGL 最小对照**：同一场景、同一 MVP，渲染一张对比图；2–3 天。
- **C06 / P05 多线程**：按屏幕分块；先测量再并行；2–3 天。
- **S04 法线贴图**：墙面增加切线空间；1–2 天。
- **R10 MSAA**：2× / 4×；1 天。
- **R11 视锥剔除**：AABB + 视锥平面；1 天。

不要同时做两个 stretch。先把 v0.1 讲清楚。

---

## 13. 常见坑（先别找答案）

这些是你大概率会遇到的，遇到时先自己调 30 分钟，再问 AI：

1. 矩阵 row/column-major 混了，画面全黑或镜像。
2. `P*V*M` 顺序反了。
3. NDC z 范围和深度比较方向不一致。
4. 视口 y 翻转后，背面剔除绕序反了。
5. 透视插值忘了除以 `w`，纹理扭曲。
6. 深度用 view z 存、却用 NDC z 插值。
7. 纹理 V 轴翻转，图上下颠倒。
8. 近平面裁剪没做，贴墙时三角形爆炸。
9. DIB 的 stride 不等于 width*4，图像斜。
10. 整数除法 / 隐式截断，重心坐标全错。
11. 像素中心没用 +0.5，三角形边缘裂缝。
12. 线性 / sRGB 重复或漏做 gamma，画面灰或过亮。

---

## 14. 20 分钟讲解提纲

1. **30s**：是什么 + 约束（C++17、零依赖、800×600、CPU）。
2. **2min**：demo + 数字。
3. **3min**：一帧的数据流（map → mesh → MVP → raster → depth → texture → lighting → present）。
4. **3min**：难点 1 — MVP 与坐标约定。
5. **3min**：难点 2 — 透视校正；错误 / 正确对比。
6. **2min**：难点 3 — 深度精度与 z-fighting。
7. **2min**：性能 — 帧预算、overdraw、瓶颈。
8. **2min**：3 个 bug 与定位过程。
9. **2min**：如果重做 — OpenGL 移植、阴影、PBR、多线程。
10. **30s**：Q&A。

---

## 15. 完成定义

- 干净构建能跑；`ctest` 全绿。
- Stage 1–4 验收全部有证据。
- `KNOWLEDGE.md` 的 28 个节点都有证据链接。
- README / 视频 / 提纲 / 3 bug 齐全。
- 你能不看代码回答：
  - MVP 每一步在哪个空间？除以了几次 w？
  - 为什么纹理要透视校正？不校正会怎样？
  - 为什么 z-buffer 要逐像素？列级深度为什么不够？
  - near 变小，深度精度怎么变？z-fighting 怎么解决？
  - 800×600 一帧多少 ms？瓶颈在哪？
  - 如果重做，你会先改哪三件事？

---

## 16. 版本控制建议

建议 `.gitignore`：

```gitignore
build/
*.exe
*.pdb
*.ilk
*.ppm
docs/evidence/*.mp4
```

- `AGENTS.md`、`PROJECT.md`、`KNOWLEDGE.md` 建议入库；它们记录了学习规则和范围。
- 截图（PNG）可以入库，视频不要入库，放链接。
- 每个 commit 的源码必须是你写的；AI 最多帮你改 commit message 的措辞。
