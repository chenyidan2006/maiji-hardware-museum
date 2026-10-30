# 麦极 · 硬件数字展厅

三层堆叠硬件的交互式数字展厅。模型来自真实的 STEP 工程文件，可以逐层、逐区域点开，
看每一块板、每一个功能区域在整条信号链里做什么。

## 打开方式

### 在线链接（已部署）

**https://chenyidan2006.github.io/maiji-hardware-museum/**

仓库：https://github.com/chenyidan2006/maiji-hardware-museum
（GitHub Pages 免费版要求仓库公开，所以这个仓库是 public 的。）

改完东西要更新线上：

> 如果 
pm run deploy 报 Failed to connect to github.com port 443，说明这台机器的 git 解析到了被墙的 IP，pi.github.com 却是通的。用 
pm run deploy:api 代替——它走 GitHub REST API 推送，不经过 git push。

```bash
npm run deploy      # 构建 + 推送到 main，GitHub 约 30 秒后自动重新发布
```

想换自定义域名：在仓库 Settings → Pages → Custom domain 里填，然后在 DNS 加一条
CNAME 指到 `chenyidan2006.github.io`。

### 发一个文件给别人

**`麦极硬件数字展厅.html`**（4.7 MB，单个文件）**双击就能看** —— 不需要装软件、
不需要服务器、不需要联网，模型全部内嵌在里面。微信 / 邮件 / 网盘都能传。

- 重新生成：`npm run pack`
- 对方要用**真正的浏览器**打开（Chrome / Edge / Safari）。手机微信里点 `.html`
  常常只显示源码或提示打不开，需要选「用其他应用打开」→ 浏览器。
- 3D 用 WebGL，近五年的设备都能跑；极老的机器或远程桌面里可能黑屏。

### 本地开发

```bash
npm install
npm run dev        # http://127.0.0.1:5173
npm run build      # 生成静态站点到 dist/
npm run preview    # 预览 dist/，http://127.0.0.1:4173
```

## 怎么逛

| 操作 | 结果 |
| --- | --- |
| 拖动 | 水平 360°、垂直 360° 自由旋转（可以绕到模型下方） |
| 滚轮 | 缩放 |
| 点模型上的板 / 左侧板层 | 进入单板聚焦，其余两层淡化退开 |
| 点板面上的发光标记 | 打开该区域；**如果该区元件在板子背面，板会自动翻过来** |
| 「翻到背面 / 翻到正面」 | 自己把当前这块板翻个面 |
| 「展开」 | 三层拉开，露出中间的激光头与同轴排线 |
| 点空白 / Esc | 逐级退出：区域 → 单板 → 三层全景 |

## 关于双面板

三块板都是**双面板**，元件分布在正反两面。所以：

- 每个区域的元件都记录了 `正面 / 背面 / 两面` 的分布（右侧面板「元件位置」一行）；
- 单板列表里每个区域带 `正面为主 / 背面为主 / 双面分布` 标签；
- 板面永远不透明 —— 透光会让前后两层糊在一起。要看另一面，是把板子转过来，不是把板子变透明。

## 换新的 STEP 文件

一句话：**STEP 先经 FreeCAD 导成中间 glTF，再由管线压成站点资产**。分两步，各一条命令。

### 0. 一次性准备

- 装 **FreeCAD 1.0+**。关键：必须能用 **GUI 版 `freecad.exe`**，因为 console 版
  `freecadcmd` 加载不了 `ImportGui`，而 STEP 导出依赖它。
  本机已验证路径：`C:\Users\peony\AppData\Local\Programs\FreeCAD 1.1\bin\freecad.exe`
  （`npm run step:export` 会自动找到它；换机器可设 `FREECAD_CMD` 指过去）
- `npm install`

### 1. STEP → glTF

```bash
npm run step:export -- --board top --step "D:\project\PPG\硬件\第二版\3D_PCB6_顶板.step"
npm run step:export -- --board bottom --step "...\3D_PCB3_底板.step"
npm run step:export -- --board laser  --step "...\3D_PCB4_激光头.step"
```

产物落在 `step-export/`（已 gitignore）：

| `--board` | 输出文件 |
| --- | --- |
| `top` | `step-export/top_board.glb` |
| `bottom` | `step-export/bottom_board.glb` |
| `laser` | `step-export/laser_head.glb` |

只换其中一块，就只跑那一块，其余两块的中间文件保留即可。

### 2. glTF → 站点资产

```bash
npm run assets
```

这一步做的是：丢掉重复装配、剔除原点上的封装库模板、把元件按最近的连接器/芯片
聚成功能区、按「分区 + 材质」合并几何、算每区正反面分布，最后 `gltfpack` 量化和
meshopt 压缩。输出 `public/assets/`（3 个 glb + manifest.json）。

跑完看控制台：每块板会打印板尺寸、元件数、每个分区的三角面/元件数/正反面。
`npm run dev` 立刻就能看效果。

### 如果新板子长得不一样

管线不是全自动的，有两处是按当前这批 STEP 调的，在 `tools/pipeline/config.mjs`：

```js
cull: [-0.009, -0.006, 0.010, 0.007],   // 原点附近要丢弃的封装库模板范围（米）
anchors: [/^TYPE-C/, /^CONN-SMD/, /^QFN-56/, /^RF-/],  // 每块板的功能区锚点（封装名前缀）
zones: 4,                                // 分几个区
```

- **板子尺寸/原点变了** → 调 `cull`。它圈的是 STEP 里那堆停在原点的封装库实体；
  圈小了会看到一堆重复元件飘在板外，圈大了会误删真元件。
- **换了主芯片/连接器** → 调 `anchors`，用封装名前缀（`QFN-56`、`TYPE-C`、`HDR` 这些）。
- 管线会自己校验：认不出板子实体、元件数不够分区，都会停下来告诉你该改哪里，
  不会闷头输出一个空站点。

### 分区的文字还得手动对

`src/content.js` 里的 `ZONES` 是按分区 id（`top-z0`…）写的文案。分区数量和顺序
如果变了，`npm run assets` 会打印新的分区列表（含每区的元件构成），照着改文案即可。
id 没变的话文案不用动。

### 微调某个零件的位置（不改 STEP）

有时只是想让某个零件挪几毫米（比如排针在板上的位置）：不用回 EDA 改、也不用重新导出，
改一个数字就行。改 `tools/pipeline/config.mjs` 里对应板子的 `nudges`：

```js
nudges: [
  { name: '左侧排针', dz: 7, box: { x: [-17.6, -14.4], z: [-16.4, 2.4], y: [-2, 13] } },
  { name: '右侧排针', dz: 7, box: { x: [16.6, 19.4], z: [-16.0, 2.0], y: [-2, 13] } },
],
```

- **`dz`** 要挪多少毫米：正数往 **+z**，负数往 **−z**。
  在**俯视图**里 **+z 是屏幕下方、−z 是上方**（想「往下挪」就填正数）。
- **`box`** 只有落在里面的顶点会动。`x / z / y` 是毫米、以板中心为原点、Y 向上，
  和 `manifest.json` 里是同一套坐标。只动元件节点，**PCB 本体永远不会被移动**。

先看会命中什么，再决定：

```bash
npm run nudge:check -- --board bottom      # 只报告，不改文件
npm run nudge -- --board bottom --dz -3    # 临时试一个值，不动 config
npm run nudge -- --board bottom            # 按 config 应用
npm run build && npm run deploy
```

它是**幂等**的：每次从 `step-export/bottom.base.raw.glb` 这份基准副本重算，
所以反复改 `dz` 不会越挪越远。

不确定该圈多大范围时，用探针看那个区域里到底有什么：

```bash
node tools/pipeline/probe-box.mjs --board bottom --x -17.6 -14.4 --z -16.4 2.4
```

按节点分组打印命中顶点数、包围盒和每 1mm 的高度分布，能确认有没有误圈别的东西。

## 目录

```
index.html                 页面骨架
src/main.js                状态机、UI 渲染、事件
src/stage.js               three.js 场景：装配、分区、翻面、排线、拾取
src/cables.js              同轴线 / FFC 排线的扫掠几何（端点跟随板子移动）
src/content.js             全部中文文案（按区域 id 对应真实分区）
src/style.css              展厅视觉
public/assets/manifest.json 分区、元件统计、正反面分布、连接器锚点
public/assets/*.glb        每块板一个 GLB，按功能分区合并成少量 draw call
step-export/               FreeCAD 导出的中间 glTF（gitignore，换 STEP 时重新生成）
tools/pipeline/            资产管线：config.mjs（板子配置）、build.mjs、from-step.mjs、
                           nudge.mjs（微调零件位置）、probe-box.mjs、selftest.mjs
tools/export_step_gltf.py  FreeCAD 调用的 STEP → glTF 脚本
tools/deploy-pages.mjs     一键发布到 GitHub Pages
```

## 模型是怎么来的

1. `tools/export_step_gltf.py` —— 用 FreeCAD 把 `硬件/第二版/*.step` 导成 glTF。
   原始导出每颗元件有 3 份重复（裸实体 + 位号组内副本 + 整个 `EasyEDA PCB Model` 重复装配），
   还有一堆停在原点的封装库模板，单板 142 万三角面、上千个 draw call。
2. 资产管线（在本仓的 `_gen/` 工作区里）做了：丢弃重复装配、剔除库模板、
   按最近的连接器/芯片把元件聚成功能分区、按分区+材质合并几何，
   最后用 `gltfpack -tc -c` 做量化 + meshopt 压缩。
   结果：**每板 142 万 → 32~37 万三角面，文件 21 MB → 1.3 MB，draw call 上千 → 一百多**。
3. 排线（两根 IPEX 同轴线 + 两条 FFC 软排线）是运行时按连接器锚点生成的扫掠几何，
   板子移动或翻面时会实时跟随。

> 注意：`_gen/` 里的中间产物（原始 OpenCascade GLB）已经被清掉了。要重新生成，
> 先跑 `tools/export_step_gltf.py` 从 STEP 导出，再重跑资产管线。
