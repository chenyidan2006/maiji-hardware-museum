# 麦极 · 硬件数字展厅

三层堆叠硬件的交互式数字展厅。模型来自真实的 STEP 工程文件，可以逐层、逐区域点开，
看每一块板、每一个功能区域在整条信号链里做什么。

## 打开方式

### 在线链接（已部署）

**https://chenyidan2006.github.io/maiji-hardware-museum/**

仓库：https://github.com/chenyidan2006/maiji-hardware-museum
（GitHub Pages 免费版要求仓库公开，所以这个仓库是 public 的。）

改完东西要更新线上：

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
tools/                     STEP → glTF 的转换脚本（需要 FreeCAD）
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
