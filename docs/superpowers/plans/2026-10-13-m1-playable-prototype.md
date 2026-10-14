# M1 可玩原型 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 先验证 three.js 能在抖音小游戏里跑通"像素 3D"的三层合成；然后做出只有周一的可玩原型：家里看电脑、自己决定关机、关机等待，然后在一条斜 45° 像素 3D 直路上跑回学校（3 条道、跳、钻、冲刺和体力、3 种阻碍），最后在校门判定按时或迟到并显示结算。

**Architecture:** 抖音原生小游戏，纯 JS + CommonJS，结构参照 `../nine_to_nope`。引擎（`js/engine/`）只管逻辑，用 `on/emit` 发事件、`view()` 出快照。画面分三层：跑步段的 3D 世界由 `world3d.js` 用 three.js 搭建，渲染到 180×320 的渲染目标；家里、标题、结算等 2D 画面和 HUD 画在 180×320 的 Canvas 2D 像素画布上；中文文字画在屏幕分辨率的文字画布上。`presenter-three.js` 把三层按整数倍放大合成到屏幕，是唯一碰 WebGL 的模块。像素素材在 `js/art/` 里用"调色板 + 字符矩阵"定义，2D 画面和 3D 纹理共用。

**Tech Stack:** JavaScript（CommonJS，ES2017 以内的语法）、three.js r162（esbuild 裁剪打包为 `js/libs/three.js`）、Canvas 2D、抖音小游戏 `tt.*` API、`node --test`（测试文件用 `.cjs`）。

**Spec:** `docs/superpowers/specs/2026-10-13-retro-run-design.md`（第 2 版；本计划实现其中第 12 节的 M1）

## Global Constraints

- 平台：抖音小游戏原生工程，`project.config.json` 里 `douyinProjectType: "native"`，`game.json` 里 `deviceOrientation: "portrait"`。
- 所有 `tt.*` 调用只能出现在 `js/platform.js`。
- 测试文件只能是 `test/*.test.cjs`（开发者工具会编译所有 `.js`，测试里的 `node:*` 模块会导致编译失败）。测试命令：`npm test`（即 `node --test "test/**/*.test.cjs"`）。
- 引擎不碰平台和绘制；所有随机来自 `createRng(seed)`；引擎只用累计的 `dt`，单帧 `dt` 最多 `0.1` 秒，不调用 `Date.now()`。
- 渲染器只读快照，`hitTest(x, y)` 只把坐标翻译成 `{ action }`。
- three.js 固定 `three@0.162.0`（最后一个支持 WebGL 1 的版本），只通过 `js/libs/three.js` 引入；这个文件由 `npm run build:three` 生成并提交，不手改。
- 只有 `js/renderer/presenter-three.js` 创建 `WebGLRenderer`；`world3d.js` 只创建场景对象（Node 里不需要 WebGL 也能测）。
- 跑步段：正交摄像机斜 45° 俯视，学生朝屏幕左上方跑；轨道第 z 米 = three.js 里的 `-z`；路段都是直的。3D 纹理一律最近邻过滤、不生成 mipmap。
- 时钟：从 12:25 开始，13:00 打铃；1 游戏分钟 = 3 真实秒；家里段和跑步段连续走，出门转场 2 秒不走时钟。
- 过瘾值每游戏分钟 +10；关机等待 1～2 游戏分钟；20% 概率"程序没有响应"，点"结束任务"后额外等 0.5 游戏分钟。
- 跑步：3 条道；冲刺速度 ×1.5；体力 100，冲刺每秒 -25，不冲刺每秒 +8；体力到 0 喘气 1.5 秒、速度减半，之后体力回到 30。
- 撞到障碍不会死，只扣时间。M1 没有好人好事，所以迟到就是失败。
- 像素画面 180×320，按设备像素整数倍放大，放在安全区内居中。
- 版权：画面里不出现原作名字、Logo（M1 的屏幕画面只是一个"恐龙回合制"占位画）。
- **Task 3 是关卡**：技术验证在抖音上不通过，就停下来和 Rainman 讨论，不继续往下做。
- 不包含：音频、存档、路口、补给、好人好事、其他四天。

## Review Focus

- 一直不关机，挂到 13:00：应当立刻结束这一局，显示"13:00 了，你还坐在电脑前"的失败结算，而不是卡住或进入跑步段。→ Task 8 测试
- 按按钮按得慢（按住超过 180 毫秒才松开）：按钮也必须生效，不能因为被识别成"长按"就吞掉。→ Task 9 测试
- 切到后台（来电、下拉通知）：时钟停止、冲刺松开，回来后点一下屏幕继续；暂停期间的滑动不生效。→ Task 8、Task 14 测试
- 卡顿导致一帧很长（比如 5 秒）：时钟最多走 0.1 秒对应的量，跑步者不能借着大步长穿过障碍。→ Task 8 测试
- 不同屏幕比例（很长的全面屏、iPad、小屏安卓）：画面整数倍放大、居中、不压到刘海和底部横条，点击坐标换算正确。→ Task 11 测试

---

## 文件结构

```
game.js                     入口：require('./js/main')
game.json                   竖屏
project.config.json         原生工程；打包忽略 test/、docs/、tools/、node_modules/
package.json                test、build:three 脚本；开发依赖 three@0.162.0、esbuild
.gitignore                  加上 .claude/
tools/
  three-entry.mjs           列出用到的 three.js 类
  build-three.cjs           esbuild 裁剪打包 → js/libs/three.js
js/
  libs/three.js             生成的 three.js（提交）
  main.js                   组装模块、触摸 → 手势 → 引擎、主循环；导出 { engine, renderer } 供测试和预览
  config.js                 全局常量
  platform.js               tt.* 适配（WebGL 上屏画布、离屏画布、安全区、触摸、震动、生命周期）
  util.js                   clamp / pad2 / createRng / pickWeighted
  input.js                  GestureTracker：滑动、长按、点击
  data/
    days.js                 周一的配置
    obstacles.js            三种阻碍
  engine/
    clock.js                午休时钟
    home.js                 家里段：过瘾值、诱惑、关机流程
    runner.js               跑步者
    track.js                轨道：物件
    game-engine.js          单局总控
  art/
    palette.js              全局调色板
    sprites/student.js      学生
    sprites/props.js        自行车、积水、店铺、树、校门
    sprites/stone.js        电脑屏幕里的恐龙回合制占位画
    sprites.js              素材表 + 缓存 + 绘制
  renderer/
    presenter-three.js      三层合成上屏（唯一碰 WebGL 的模块）
    world3d.js              跑步段的 3D 场景
    pixel.js                像素绘图工具：矩形、进度条、像素数字、时钟牌
    renderer.js             场景路由、像素画布、文字画布、hitTest、抖动和浮字
    scene-title.js / scene-home.js / scene-leaving.js / scene-run.js / scene-result.js
test/
  project-layout  three-bundle  util  clock  home  runner  track  game-engine  input
  sprites  render-smoke  main-wiring（都是 .test.cjs）
.claude/launch.json         本地预览服务器（不提交）
.claude/preview/index.html  浏览器里模拟 tt 运行游戏（不提交）
```

---

### Task 1: 工程骨架和工具函数

**Files:**
- Create: `game.js`, `game.json`, `project.config.json`, `package.json`, `js/util.js`
- Modify: `.gitignore`
- Test: `test/project-layout.test.cjs`, `test/util.test.cjs`

**Interfaces:**
- Produces: `js/util.js` 导出 `{ clamp(v, a, b), pad2(n), createRng(seed) → () => [0,1), pickWeighted(rng, { key: weight }) → key }`

- [ ] **Step 1: 写失败的测试**

`test/project-layout.test.cjs`:

```js
const test = require('node:test');
const assert = require('node:assert/strict');
const fs = require('node:fs');
const path = require('node:path');

const root = path.join(__dirname, '..');

// 抖音开发者工具会编译工程里所有 .js，测试里的 node:* 模块会导致编译失败，所以测试文件只能用 .cjs
test('test/ 目录下没有 .js 文件', () => {
  const js = fs.readdirSync(__dirname).filter((f) => f.endsWith('.js'));
  assert.deepEqual(js, []);
});

test('project.config.json：原生工程，打包时忽略 test、docs、tools、node_modules', () => {
  const cfg = JSON.parse(fs.readFileSync(path.join(root, 'project.config.json'), 'utf8'));
  assert.equal(cfg.douyinProjectType, 'native');
  assert.equal(cfg.projectname, 'retro_run');
  const folders = cfg.packOptions.ignore.filter((r) => r.type === 'folder').map((r) => r.value);
  for (const f of ['test', 'docs', 'tools', 'node_modules']) assert.ok(folders.includes(f), f);
});

test('game.json：竖屏', () => {
  const cfg = JSON.parse(fs.readFileSync(path.join(root, 'game.json'), 'utf8'));
  assert.equal(cfg.deviceOrientation, 'portrait');
});
```

`test/util.test.cjs`:

```js
const test = require('node:test');
const assert = require('node:assert/strict');
const { clamp, pad2, createRng, pickWeighted } = require('../js/util');

test('clamp / pad2', () => {
  assert.equal(clamp(5, 0, 3), 3);
  assert.equal(clamp(-1, 0, 3), 0);
  assert.equal(pad2(7), '07');
  assert.equal(pad2(12), '12');
});

test('createRng：同一种子序列相同，取值在 [0,1)', () => {
  const a = createRng(42);
  const b = createRng(42);
  for (let i = 0; i < 100; i++) {
    const x = a();
    assert.equal(x, b());
    assert.ok(x >= 0 && x < 1);
  }
  assert.notEqual(createRng(1)(), createRng(2)());
});

test('pickWeighted：按权重落区间，忽略权重为 0 的项', () => {
  const w = { a: 1, b: 3, c: 0 };
  assert.equal(pickWeighted(() => 0, w), 'a');
  assert.equal(pickWeighted(() => 0.24, w), 'a');
  assert.equal(pickWeighted(() => 0.26, w), 'b');
  assert.equal(pickWeighted(() => 0.999, w), 'b');
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test "test/**/*.test.cjs"`
Expected: FAIL（找不到 `project.config.json`、`../js/util`）

- [ ] **Step 3: 写实现**

`game.js`:

```js
// 抖音小游戏入口
// 文档：https://developer.open-douyin.com/docs/resource/zh-CN/mini-game/guide/minigame/introduction
require('./js/main');
```

`game.json`:

```json
{
    "deviceOrientation": "portrait"
}
```

`project.config.json`:

```json
{
    "setting": {
        "urlCheck": true,
        "es6": true,
        "postcss": true,
        "minified": true,
        "newFeature": true,
        "autoCompile": true,
        "compileHotReLoad": true,
        "nativeCompile": true
    },
    "packOptions": {
        "ignore": [
            { "type": "folder", "value": "test" },
            { "type": "folder", "value": "docs" },
            { "type": "folder", "value": "tools" },
            { "type": "folder", "value": "node_modules" }
        ]
    },
    "appid": "testAppId",
    "projectname": "retro_run",
    "douyinProjectType": "native"
}
```

`package.json`:

```json
{
  "name": "retro_run",
  "private": true,
  "scripts": {
    "test": "node --test \"test/**/*.test.cjs\""
  }
}
```

`.gitignore`（在现有三行后追加一行）:

```
.DS_Store
.remember/
node_modules/
.claude/
```

`js/util.js`:

```js
const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
const pad2 = (n) => String(n).padStart(2, '0');

// 可复现的伪随机数（mulberry32）。引擎只用它，不用 Math.random，方便测试
function createRng(seed) {
  let a = seed >>> 0;
  return function rng() {
    a = (a + 0x6D2B79F5) >>> 0;
    let t = a;
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}

// weights: { key: 权重 }，按权重随机返回一个 key
function pickWeighted(rng, weights) {
  const keys = Object.keys(weights).filter((k) => weights[k] > 0);
  const total = keys.reduce((s, k) => s + weights[k], 0);
  let r = rng() * total;
  for (const k of keys) {
    r -= weights[k];
    if (r < 0) return k;
  }
  return keys[keys.length - 1];
}

module.exports = { clamp, pad2, createRng, pickWeighted };
```

注意：此时 `game.js` 引用的 `js/main.js` 还不存在（Task 3 先写临时的验证版，Task 14 换成正式版），这不影响测试。

- [ ] **Step 4: 运行，确认通过**

Run: `npm test`
Expected: PASS（6 个测试）

- [ ] **Step 5: 提交**

```bash
git add game.js game.json project.config.json package.json .gitignore js/util.js test/project-layout.test.cjs test/util.test.cjs
git commit -m "chore: scaffold douyin native project with util helpers"
```

---

### Task 2: three.js 裁剪打包

**Files:**
- Modify: `package.json`
- Create: `tools/three-entry.mjs`, `tools/build-three.cjs`, `js/libs/three.js`（生成）
- Test: `test/three-bundle.test.cjs`

**Interfaces:**
- Produces: `js/libs/three.js`，CommonJS，导出 `REVISION`（`'162'`）、`WebGLRenderer`、`WebGLRenderTarget`、`Scene`、`Group`、`Mesh`、`Sprite`、`OrthographicCamera`、`BoxGeometry`、`PlaneGeometry`、`CircleGeometry`、`MeshBasicMaterial`、`MeshLambertMaterial`、`SpriteMaterial`、`CanvasTexture`、`Color`、`Fog`、`HemisphereLight`、`DirectionalLight`、`NearestFilter`、`LinearFilter`、`BasicShadowMap`、`SRGBColorSpace`。后续任务只用这些名字；要用新的类就加到 `tools/three-entry.mjs` 里重新打包。

- [ ] **Step 1: 写失败的测试**

`test/three-bundle.test.cjs`:

```js
const test = require('node:test');
const assert = require('node:assert/strict');
const fs = require('node:fs');
const path = require('node:path');

const root = path.join(__dirname, '..');
const BUNDLE = path.join(root, 'js', 'libs', 'three.js');
const NAMES = ['WebGLRenderer', 'WebGLRenderTarget', 'Scene', 'Group', 'Mesh', 'Sprite', 'OrthographicCamera',
  'BoxGeometry', 'PlaneGeometry', 'CircleGeometry', 'MeshBasicMaterial', 'MeshLambertMaterial', 'SpriteMaterial',
  'CanvasTexture', 'Color', 'Fog', 'HemisphereLight', 'DirectionalLight',
  'NearestFilter', 'LinearFilter', 'BasicShadowMap', 'SRGBColorSpace'];

test('package.json 固定 three@0.162.0（最后支持 WebGL 1 的版本）', () => {
  const pkg = JSON.parse(fs.readFileSync(path.join(root, 'package.json'), 'utf8'));
  assert.equal(pkg.devDependencies.three, '0.162.0');
  assert.ok(pkg.devDependencies.esbuild);
  assert.equal(pkg.scripts['build:three'], 'node tools/build-three.cjs');
});

test('js/libs/three.js 是 r162，导出了用到的所有类', () => {
  const T = require(BUNDLE);
  assert.equal(T.REVISION, '162');
  for (const n of NAMES) assert.ok(T[n] !== undefined, n);
});

test('体积在 700KB 以内', () => {
  const size = fs.statSync(BUNDLE).size;
  assert.ok(size < 700 * 1024, `three.js 打包后 ${Math.round(size / 1024)}KB`);
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test test/three-bundle.test.cjs`
Expected: FAIL（`pkg.devDependencies` 是 undefined）

- [ ] **Step 3: 安装依赖并写打包脚本**

Run: `npm install --save-dev --save-exact three@0.162.0 esbuild`

然后在 `package.json` 的 `scripts` 里加一行，结果应当是（esbuild 的版本号以 npm 实际装上的为准）：

```json
{
  "name": "retro_run",
  "private": true,
  "scripts": {
    "test": "node --test \"test/**/*.test.cjs\"",
    "build:three": "node tools/build-three.cjs"
  },
  "devDependencies": {
    "esbuild": "x.y.z",
    "three": "0.162.0"
  }
}
```

`tools/three-entry.mjs`:

```js
// 游戏里用到的 three.js 类（设计文档 10.1）。要用新的类就加在这里，再 npm run build:three
export {
  REVISION,
  WebGLRenderer, WebGLRenderTarget,
  Scene, Group, Mesh, Sprite, OrthographicCamera,
  BoxGeometry, PlaneGeometry, CircleGeometry,
  MeshBasicMaterial, MeshLambertMaterial, SpriteMaterial,
  CanvasTexture, Color, Fog,
  HemisphereLight, DirectionalLight,
  NearestFilter, LinearFilter, BasicShadowMap, SRGBColorSpace,
} from 'three';
```

`tools/build-three.cjs`:

```js
/**
 * 把 three.js 里用到的类裁剪打包成一个压缩的 CommonJS 文件（设计文档 10.1、10.3）
 * 输出 js/libs/three.js 并提交，抖音开发者工具里不需要再构建
 * 用法：npm run build:three
 */
const path = require('node:path');
const fs = require('node:fs');
const esbuild = require('esbuild');

const out = path.join(__dirname, '..', 'js', 'libs', 'three.js');
esbuild.buildSync({
  entryPoints: [path.join(__dirname, 'three-entry.mjs')],
  outfile: out,
  bundle: true,
  format: 'cjs',
  platform: 'neutral',
  mainFields: ['module', 'main'],
  target: 'es2017',
  minify: true,
  legalComments: 'none',
  banner: { js: '/* three.js r162 (MIT License, https://github.com/mrdoob/three.js) — 由 tools/build-three.cjs 生成，不要手改 */' },
});
console.log(`js/libs/three.js ${Math.round(fs.statSync(out).size / 1024)}KB`);
```

Run: `npm run build:three`
Expected: 输出 `js/libs/three.js` 和体积（预计 400～600KB）。

- [ ] **Step 4: 运行，确认通过**

Run: `npm test`
Expected: PASS。如果体积超过 700KB，先检查 `three-entry.mjs` 是否误导出了整个库，不要直接调高预算。

- [ ] **Step 5: 提交**

```bash
git add package.json package-lock.json tools/three-entry.mjs tools/build-three.cjs js/libs/three.js test/three-bundle.test.cjs
git commit -m "build: bundle trimmed three.js r162 for douyin"
```

---

### Task 3: 技术验证——three.js 在抖音上跑通三层合成（关卡）

这一步写的 `platform.js` 和 `presenter-three.js` 是正式代码，后面直接用；`js/main.js` 是临时的验证程序，Task 14 会整个替换掉。WebGL 在 Node 里没法跑，所以这一步不写单元测试，靠浏览器预览和抖音开发者工具验证。

**Files:**
- Create: `js/platform.js`, `js/renderer/presenter-three.js`, `js/main.js`（临时）
- Create（不提交）: `.claude/launch.json`, `.claude/preview/index.html`

**Interfaces:**
- Consumes: `js/libs/three.js`（Task 2）
- Produces:
  - `js/platform.js` 导出 `{ width, height, dpr, safe: { top, bottom }, createScreenCanvas() → { canvas, gl }, createOffscreen(w, h) → canvas, onTouchStart(fn), onTouchMove(fn), onTouchEnd(fn), onTouchCancel(fn), onHide(fn), onShow(fn), vibrate(kind), getStorage(key), setStorage(key, value), raf(fn), now() }`；触摸回调签名 `fn(id, x, y)`；`createScreenCanvas` 必须最先调用
  - `js/renderer/presenter-three.js` 导出 `class ThreePresenter`：`new ThreePresenter({ canvas, gl, W, H, dpr })`；`present({ world, pixel, text, textDirty, layout, shake, bg })`，其中 `world` 是 `{ scene, camera, w, h } | null`，`pixel` 是 180×320 的 2D 画布（每帧上传），`text` 是屏幕分辨率的 2D 画布（`textDirty` 为真才上传），`layout` 是 `{ ox, oy, w, h }`（CSS 像素），`shake` 是 `{ x, y }`（只作用于世界层和像素层，文字层的抖动由调用方画进去），`bg` 是四周留边的颜色

- [ ] **Step 1: 写平台层**

`js/platform.js`:

```js
/**
 * 平台适配层：所有 tt.* 调用集中在这里，其余模块不直接依赖抖音 API
 * 迁移到微信小游戏时，把 tt 换成 wx 即可
 */
/* global tt */
const api = typeof tt !== 'undefined' ? tt : null;

const sys = api.getSystemInfoSync();
const width = sys.windowWidth;
const height = sys.windowHeight;
const dpr = sys.pixelRatio || 1;

// 安全区：顶部避开状态栏和右上角胶囊按钮，底部避开 Home Indicator
function computeSafeInsets() {
  let top = sys.safeArea ? sys.safeArea.top : sys.statusBarHeight || 20;
  const bottom = sys.safeArea ? Math.max(0, height - sys.safeArea.bottom) : 0;
  try {
    const menu = api.getMenuButtonBoundingClientRect && api.getMenuButtonBoundingClientRect();
    if (menu && menu.bottom) top = Math.max(top, menu.bottom);
  } catch (e) { /* 部分版本不支持，忽略 */ }
  return { top, bottom };
}

// 第一个 createCanvas 创建的是上屏画布，整块屏幕交给 WebGL（设计文档 10.3）
// 先要 WebGL 2，拿不到再用 WebGL 1（three.js r162 两种都支持）
function createScreenCanvas() {
  const canvas = api.createCanvas();
  canvas.width = width * dpr;
  canvas.height = height * dpr;
  // three.js 会给画布挂 webglcontextlost 监听；小游戏画布没有这两个方法时补上空实现
  if (!canvas.addEventListener) {
    canvas.addEventListener = () => {};
    canvas.removeEventListener = () => {};
  }
  const opts = { antialias: false, alpha: false, depth: true, stencil: false, preserveDrawingBuffer: false };
  const gl = canvas.getContext('webgl2', opts) || canvas.getContext('webgl', opts);
  return { canvas, gl };
}

// 之后的 createCanvas 都是离屏画布：像素层、文字层、精灵缓存
function createOffscreen(w, h) {
  const canvas = api.createCanvas();
  canvas.width = w;
  canvas.height = h;
  return canvas;
}

// 触摸：对本次变化的每根手指回调 fn(id, x, y)
function bindTouches(register, fn) {
  register((res) => {
    const list = res.changedTouches || [];
    for (let i = 0; i < list.length; i++) {
      const t = list[i];
      fn(t.identifier != null ? t.identifier : i, t.clientX, t.clientY);
    }
  });
}

function onTouchStart(fn) { bindTouches((cb) => api.onTouchStart(cb), fn); }
function onTouchMove(fn) { bindTouches((cb) => api.onTouchMove(cb), fn); }
function onTouchEnd(fn) { bindTouches((cb) => api.onTouchEnd(cb), fn); }
function onTouchCancel(fn) {
  if (api.onTouchCancel) bindTouches((cb) => api.onTouchCancel(cb), fn);
}

function onHide(fn) { if (api.onHide) api.onHide(fn); }
function onShow(fn) { if (api.onShow) api.onShow(fn); }

// kind: 'light' 轻震一次；'double' 短震两次；'long' 长震一次
function vibrate(kind = 'light') {
  try {
    if (kind === 'long') {
      if (api.vibrateLong) api.vibrateLong({});
      return;
    }
    if (api.vibrateShort) api.vibrateShort({});
    if (kind === 'double') setTimeout(() => { try { api.vibrateShort({}); } catch (e) { /* 忽略 */ } }, 120);
  } catch (e) { /* 忽略 */ }
}

function getStorage(key) {
  try { return api.getStorageSync(key); } catch (e) { return null; }
}

function setStorage(key, value) {
  try { api.setStorageSync(key, value); } catch (e) { /* 忽略 */ }
}

module.exports = {
  width, height, dpr,
  safe: computeSafeInsets(),
  createScreenCanvas, createOffscreen,
  onTouchStart, onTouchMove, onTouchEnd, onTouchCancel, onHide, onShow,
  vibrate, getStorage, setStorage,
  raf: (fn) => requestAnimationFrame(fn),
  now: () => Date.now(),
};
```

- [ ] **Step 2: 写三层合成**

`js/renderer/presenter-three.js`:

```js
const THREE = require('../libs/three');

/**
 * 上屏（设计文档 10.3）：把三层画面合成到屏幕。整个工程只有这里创建 WebGLRenderer
 *   1. 3D 世界：渲染到 w×h 的渲染目标（最近邻），贴到布局矩形上
 *   2. 2D 像素层：Canvas 2D 画布，每帧上传，贴到同一个布局矩形上
 *   3. 文字层：屏幕分辨率的 Canvas 2D 画布，内容变了才上传，铺满全屏
 * 合成用的正交摄像机以 CSS 像素为单位，原点在屏幕左上角，y 向下为负
 */
class ThreePresenter {
  constructor({ canvas, gl, W, H, dpr }) {
    this.W = W;
    this.H = H;
    const r = new THREE.WebGLRenderer({ canvas, context: gl, antialias: false });
    r.setPixelRatio(dpr);
    r.setSize(W, H, false);
    r.shadowMap.enabled = true;
    r.shadowMap.type = THREE.BasicShadowMap;
    r.autoClear = false;
    this.renderer = r;
    this.target = null;
    this.camera = new THREE.OrthographicCamera(0, W, 0, -H, -10, 10);
    this.scene = new THREE.Scene();
    this.worldQuad = this.quad(0);
    this.pixelQuad = this.quad(1);
    this.textQuad = this.quad(2);
  }

  quad(order) {
    const mat = new THREE.MeshBasicMaterial({ transparent: true, depthTest: false, depthWrite: false });
    const m = new THREE.Mesh(new THREE.PlaneGeometry(1, 1), mat);
    m.renderOrder = order;
    m.visible = false;
    this.scene.add(m);
    return m;
  }

  place(m, x, y, w, h) {
    m.scale.set(w, h, 1);
    m.position.set(x + w / 2, -(y + h / 2), 0);
    m.visible = true;
  }

  // 第一次见到这张画布时建纹理；之后只在 dirty 时重新上传
  bind(m, canvas, filter, dirty) {
    const old = m.material.map;
    if (old && old.image === canvas) {
      if (dirty) old.needsUpdate = true;
      return;
    }
    const t = new THREE.CanvasTexture(canvas);
    t.minFilter = filter;
    t.magFilter = filter;
    t.generateMipmaps = false;
    t.colorSpace = THREE.SRGBColorSpace;
    if (old) old.dispose();
    m.material.map = t;
    m.material.needsUpdate = true;
  }

  present({ world, pixel, text, textDirty, layout, shake, bg }) {
    const r = this.renderer;
    const L = layout;
    if (world) {
      if (!this.target || this.target.width !== world.w || this.target.height !== world.h) {
        if (this.target) this.target.dispose();
        this.target = new THREE.WebGLRenderTarget(world.w, world.h, {
          minFilter: THREE.NearestFilter, magFilter: THREE.NearestFilter, generateMipmaps: false,
        });
        this.worldQuad.material.map = this.target.texture;
        this.worldQuad.material.needsUpdate = true;
      }
      r.setRenderTarget(this.target);
      r.clear();
      r.render(world.scene, world.camera);
      r.setRenderTarget(null);
      this.place(this.worldQuad, L.ox + shake.x, L.oy + shake.y, L.w, L.h);
    } else {
      this.worldQuad.visible = false;
    }
    this.bind(this.pixelQuad, pixel, THREE.NearestFilter, true);
    this.place(this.pixelQuad, L.ox + shake.x, L.oy + shake.y, L.w, L.h);
    this.bind(this.textQuad, text, THREE.LinearFilter, textDirty);
    this.place(this.textQuad, 0, 0, this.W, this.H);
    r.setClearColor(bg, 1);
    r.clear();
    r.render(this.scene, this.camera);
  }
}

module.exports = ThreePresenter;
```

- [ ] **Step 3: 写临时验证程序**

`js/main.js`:

```js
/**
 * 技术验证（M1 Task 3）：在抖音小游戏里跑通 three.js 三层合成
 *   3D：三个贴像素纹理的方块在转，有阴影 → 180×320 渲染目标
 *   2D 像素层：底部一根每帧变化的进度条（验证每帧上传）
 *   文字层：帧率、WebGL 版本、放大倍数（验证中文和高清文字）
 * Task 14 会用正式的 main.js 整个替换这个文件
 */
const THREE = require('./libs/three');
const platform = require('./platform');
const ThreePresenter = require('./renderer/presenter-three');

const VW = 180;
const VH = 320;
const W = platform.width;
const H = platform.height;
const dpr = platform.dpr;
const safe = platform.safe;
const { canvas, gl } = platform.createScreenCanvas();
const presenter = new ThreePresenter({ canvas, gl, W, H, dpr });
const availH = H - safe.top - safe.bottom;
const k = Math.max(1, Math.floor(Math.min((W * dpr) / VW, (availH * dpr) / VH)));
const scale = k / dpr;
const layout = {
  ox: Math.round((W - VW * scale) / 2), oy: Math.round(safe.top + (availH - VH * scale) / 2),
  w: VW * scale, h: VH * scale,
};

const pixel = platform.createOffscreen(VW, VH);
const pg = pixel.getContext('2d');
const text = platform.createOffscreen(Math.round(W * dpr), Math.round(H * dpr));
const tg = text.getContext('2d');

const tile = platform.createOffscreen(8, 8);
const tc = tile.getContext('2d');
for (let y = 0; y < 8; y++) {
  for (let x = 0; x < 8; x++) {
    tc.fillStyle = (x + y) % 2 ? '#c8473a' : '#f2b632';
    tc.fillRect(x, y, 1, 1);
  }
}
const tex = new THREE.CanvasTexture(tile);
tex.minFilter = THREE.NearestFilter;
tex.magFilter = THREE.NearestFilter;
tex.generateMipmaps = false;
tex.colorSpace = THREE.SRGBColorSpace;

const scene = new THREE.Scene();
scene.background = new THREE.Color('#a9cde0');
const a = VW / VH;
const camera = new THREE.OrthographicCamera(-12 * a, 12 * a, 12, -12, 0.1, 200);
camera.position.set(-12, 16, 12);
camera.lookAt(0, 0, 0);
scene.add(new THREE.HemisphereLight('#fff6e0', '#8fb35a', 0.8));
const sun = new THREE.DirectionalLight('#fff1d0', 0.8);
sun.position.set(-8, 16, 6);
sun.castShadow = true;
sun.shadow.mapSize.set(512, 512);
Object.assign(sun.shadow.camera, { left: -20, right: 20, top: 20, bottom: -20 });
scene.add(sun);
const ground = new THREE.Mesh(new THREE.PlaneGeometry(40, 40), new THREE.MeshLambertMaterial({ color: '#8fb35a' }));
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);
const boxes = [];
for (let i = 0; i < 3; i++) {
  const b = new THREE.Mesh(new THREE.BoxGeometry(2, 2, 2), new THREE.MeshLambertMaterial({ map: tex }));
  b.position.set((i - 1) * 4, 1, 0);
  b.castShadow = true;
  scene.add(b);
  boxes.push(b);
}

const webgl = gl && typeof gl.texImage3D === 'function' ? 2 : 1;
let frames = 0;
let fps = 0;
let since = platform.now();
let label = '';
function loop() {
  const t = platform.now();
  frames++;
  if (t - since >= 500) {
    fps = Math.round((frames * 1000) / (t - since));
    frames = 0;
    since = t;
  }
  boxes.forEach((b, i) => { b.rotation.y = t / 600 + i; });
  pg.clearRect(0, 0, VW, VH);
  pg.fillStyle = '#141210';
  pg.fillRect(4, 300, 172, 10);
  pg.fillStyle = '#f2b632';
  pg.fillRect(4, 300, (t / 10) % 172, 10);
  const next = `帧率 ${fps} · WebGL ${webgl} · ${k} 倍`;
  const textDirty = next !== label;
  if (textDirty) {
    label = next;
    tg.clearRect(0, 0, text.width, text.height);
    tg.font = `bold ${Math.round(16 * dpr)}px sans-serif`;
    tg.fillStyle = '#141210';
    tg.textAlign = 'center';
    tg.textBaseline = 'middle';
    tg.fillText(label, (W / 2) * dpr, (layout.oy + 20) * dpr);
  }
  presenter.present({ world: { scene, camera, w: VW, h: VH }, pixel, text, textDirty, layout, shake: { x: 0, y: 0 }, bg: '#000000' });
  platform.raf(loop);
}

loop();
```

- [ ] **Step 4: 写浏览器预览**

`.claude/launch.json`:

```json
{
  "version": "0.0.1",
  "configurations": [
    {
      "name": "retro-run-preview",
      "runtimeExecutable": "python3",
      "runtimeArgs": ["-m", "http.server", "8766", "--bind", "127.0.0.1"],
      "port": 8766
    }
  ]
}
```

`.claude/preview/index.html`:

```html
<!doctype html>
<html lang="zh-CN"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no">
<title>retro_run 预览</title>
<style>html,body{margin:0;height:100%;background:#000;overflow:hidden}canvas{display:block;touch-action:none}</style>
</head><body><canvas id="c"></canvas>
<script>
// 仅供本地预览（服务根目录为工程根目录，访问 /.claude/preview/index.html）：
// 用浏览器模拟 tt 接口，并提供一个同步的 CommonJS 加载器
const cache = {};
function load(path) {
  if (cache[path]) return cache[path].exports;
  const x = new XMLHttpRequest(); x.open('GET', path, false); x.send();
  if (x.status !== 200) throw new Error('load ' + path + ' ' + x.status);
  const module = { exports: {} }; cache[path] = module;
  const dir = path.slice(0, path.lastIndexOf('/') + 1);
  const req = (p) => load(new URL(p.endsWith('.js') ? p : p + '.js', location.origin + dir).pathname);
  new Function('module', 'exports', 'require', x.responseText + '\n//# sourceURL=' + path)(module, module.exports, req);
  return module.exports;
}
const W = innerWidth, H = innerHeight, screenCanvas = document.getElementById('c');
screenCanvas.style.width = W + 'px'; screenCanvas.style.height = H + 'px';
let first = true;
const handlers = {};
function emit(kind, id, e) { (handlers[kind] || []).forEach((fn) => fn({ changedTouches: [{ identifier: id, clientX: e.clientX, clientY: e.clientY }] })); }
let down = false;
screenCanvas.addEventListener('mousedown', (e) => { down = true; emit('start', 0, e); });
screenCanvas.addEventListener('mousemove', (e) => { if (down) emit('move', 0, e); });
addEventListener('mouseup', (e) => { if (down) { down = false; emit('end', 0, e); } });
screenCanvas.addEventListener('touchstart', (e) => { e.preventDefault(); [...e.changedTouches].forEach((t) => emit('start', t.identifier, t)); }, { passive: false });
screenCanvas.addEventListener('touchmove', (e) => { e.preventDefault(); [...e.changedTouches].forEach((t) => emit('move', t.identifier, t)); }, { passive: false });
screenCanvas.addEventListener('touchend', (e) => [...e.changedTouches].forEach((t) => emit('end', t.identifier, t)));
// 键盘：方向键 = 滑动，空格按住 = 冲刺（只为桌面试玩方便）
const KEY = { ArrowLeft: [-40, 0], ArrowRight: [40, 0], ArrowUp: [0, -40], ArrowDown: [0, 40] };
addEventListener('keydown', (e) => {
  if (KEY[e.key]) { const [dx, dy] = KEY[e.key]; emit('start', 9, { clientX: 200, clientY: 400 }); emit('move', 9, { clientX: 200 + dx, clientY: 400 + dy }); emit('end', 9, { clientX: 200 + dx, clientY: 400 + dy }); }
  if (e.key === ' ' && !e.repeat) emit('start', 8, { clientX: 1, clientY: 1 });
});
addEventListener('keyup', (e) => { if (e.key === ' ') emit('end', 8, { clientX: 1, clientY: 1 }); });
const on = (k) => (fn) => (handlers[k] = handlers[k] || []).push(fn);
window.tt = {
  getSystemInfoSync: () => ({ windowWidth: W, windowHeight: H, pixelRatio: devicePixelRatio || 1, safeArea: { top: 44, bottom: H - 34 } }),
  getMenuButtonBoundingClientRect: () => ({ bottom: 80 }),
  createCanvas: () => { if (first) { first = false; return screenCanvas; } return document.createElement('canvas'); },
  onTouchStart: on('start'), onTouchMove: on('move'), onTouchEnd: on('end'), onTouchCancel: on('cancel'),
  onHide: (fn) => { window.__hide = fn; }, onShow: () => {},
  vibrateShort() {}, vibrateLong() {}, getStorageSync: () => null, setStorageSync() {},
};
// 调试钩子：window.E 是引擎，window.R 是渲染器（Task 14 之后才有）
const M = load('/js/main.js');
window.E = M.engine; window.R = M.renderer;
</script></body></html>
```

注意：键盘的空格冲刺用的是 `(1, 1)` 这个点，抬起时会产生一次 tap；在跑步段 tap 不对应任何按钮，所以没有副作用。

- [ ] **Step 5: 浏览器里验证**

用 `mcp__Claude_Browser__preview_start`（name: `retro-run-preview`）启动服务器，打开 `http://127.0.0.1:8766/.claude/preview/index.html`，用 `resize_window` 切到 `mobile` 预设后刷新，截图确认：
- 三个红黄格子方块在转，地面上有颗粒状的硬阴影，边缘是明显的像素锯齿
- 底部进度条在动（像素层每帧上传正常）
- 顶部中文文字清晰不糊，帧率数字在更新（文字层正常）
- 画面在安全区内居中，四周是黑边

用 `read_console_messages`（onlyErrors）确认没有报错，然后把窗口恢复成 `desktop`。

- [ ] **Step 6: 提交**

```bash
git add js/platform.js js/renderer/presenter-three.js js/main.js
git commit -m "spike: render three.js layers through douyin-style canvas"
```

- [ ] **Step 7: 关卡——请 Rainman 在抖音开发者工具和真机上验证**

把工程导入抖音开发者工具，在模拟器里运行，再用"预览"扫码在真机上运行。请 Rainman 反馈：
1. 模拟器和真机上画面是否和浏览器一致（方块、阴影、进度条、中文文字）
2. 真机的帧率：中端机应当稳定在 50 以上，低端机至少 30
3. 显示的 WebGL 版本（1 还是 2）
4. 开发者工具"详情"里的代码包大小
5. 控制台有没有报错

把结论写进设计文档第 1 节末尾（"技术验证结论（日期）"），然后提交：

```bash
git add docs/superpowers/specs/2026-10-13-retro-run-design.md
git commit -m "docs: record three.js spike findings on douyin"
```

**常见问题的处理方向**（遇到再用）：
- 报 `addEventListener` / `document` 未定义：在 `platform.createScreenCanvas` 里补对应的空实现，或在 `game.js` 开头补最小的全局对象
- 像素层或文字层是黑的：说明这个平台不支持把 2D 画布直接当纹理上传；改成在 `presenter-three.js` 里用 `getImageData` 取像素，再用 `THREE.DataTexture` 上传
- 帧率太低：先关阴影（`shadowMap.enabled = false`），再看是不是每帧上传文字层导致的

**如果问题解决不了，停在这里，和 Rainman 讨论是否退回 Canvas 2D 方案**（第 1 版的设计和计划在 git 历史里）。

---

### Task 4: 全局配置和午休时钟

**Files:**
- Create: `js/config.js`, `js/engine/clock.js`
- Test: `test/clock.test.cjs`

**Interfaces:**
- Consumes: `pad2` from `js/util.js`
- Produces:
  - `js/config.js`：见下方完整内容；后续任务用到 `maxDt`、`view`、`clock`、`home`、`leaveSec`、`run`、`score`、`iso`
  - `js/engine/clock.js` 导出 `class Clock`：`new Clock(config.clock)`；属性 `minutes`（游戏分钟，浮点）；`tick(sec) → 本次增加的游戏分钟`；`spend(min)`；getter `late`（`minutes >= bell`）、`left`（距 13:00 的剩余分钟，不小于 0）、`warn`（剩余不到 `warnMin`）；`label() → 'H:MM'`

- [ ] **Step 1: 写失败的测试**

`test/clock.test.cjs`:

```js
const test = require('node:test');
const assert = require('node:assert/strict');
const CONFIG = require('../js/config');
const Clock = require('../js/engine/clock');

test('从 12:25 开始，3 真实秒 = 1 游戏分钟', () => {
  const c = new Clock(CONFIG.clock);
  assert.equal(c.label(), '12:25');
  assert.equal(c.tick(3), 1);
  assert.equal(c.label(), '12:26');
  c.tick(1.5);
  assert.equal(c.label(), '12:26');
});

test('spend 直接加游戏分钟', () => {
  const c = new Clock(CONFIG.clock);
  c.spend(0.8);
  assert.ok(Math.abs(c.minutes - (12 * 60 + 25.8)) < 1e-9);
});

test('13:00 整算迟到，12:59.99 不算；left 和 warn', () => {
  const c = new Clock(CONFIG.clock);
  assert.equal(c.left, 35);
  assert.equal(c.warn, false);
  c.minutes = 12 * 60 + 55;
  assert.equal(c.warn, true);
  c.minutes = 12 * 60 + 59.99;
  assert.equal(c.late, false);
  assert.equal(c.label(), '12:59');
  c.minutes = 13 * 60;
  assert.equal(c.late, true);
  assert.equal(c.left, 0);
  assert.equal(c.label(), '13:00');
  c.minutes = 13 * 60 + 4.5;
  assert.equal(c.left, 0);
  assert.equal(c.label(), '13:04');
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test test/clock.test.cjs`
Expected: FAIL（Cannot find module '../js/config'）

- [ ] **Step 3: 写实现**

`js/config.js`:

```js
/**
 * 全局常量（设计文档 2.1、3.2、3.3、4.2–4.4、9）
 * 时间单位：名字带 Min 的是游戏分钟，带 Sec 的是真实秒
 */
module.exports = {
  maxDt: 0.1,
  view: { w: 180, h: 320 },
  clock: { secPerMin: 3, start: 12 * 60 + 25, bell: 13 * 60, warnMin: 5 },
  home: { addictionPerMin: 10, shutdownMin: [1, 2], hangChance: 0.2, hangExtraMin: 0.5 },
  leaveSec: 2,
  run: {
    lanes: [-3, 0, 3], // 三条道中心的横向位置（米）
    laneSwitchSec: 0.15,
    speed: 10, // 米/真实秒
    sprintMult: 1.5,
    jumpSec: 0.6,
    slideSec: 0.7,
    stumbleSec: 0.6,
    stamina: { max: 100, drain: 25, regen: 8, windedSec: 1.5, windedMult: 0.5, after: 30 },
  },
  // 得分 = 过瘾值 + 到校时剩余的整分钟 × perMinLeft（M1 试玩后再调，见设计文档 6.3）
  score: { perMinLeft: 5 },
  // 斜 45° 像素 3D（设计文档 4.1、9.1）：正交摄像机视野高 viewH 米；学生朝 -z（屏幕左上方）跑
  iso: {
    viewH: 24, camOffset: [-12, 16, 12], camFollowX: 0.3, lookAhead: 3, jumpHeight: 1.4,
    halfWidth: 4.5, walkWidth: 1.5,
    propEvery: 7, propCount: 10, propBehind: 14, dashEvery: 4, dashCount: 20,
    fogNear: 40, fogFar: 70,
  },
};
```

`js/engine/clock.js`:

```js
const { pad2 } = require('../util');

/**
 * 午休时钟（设计文档 2.1、10.3）：全局唯一的时间来源，单位是游戏分钟
 * 家里段和跑步段都用 tick 报告真实秒；摔倒等惩罚用 spend 直接加游戏分钟
 */
class Clock {
  constructor({ secPerMin, start, bell, warnMin }) {
    this.secPerMin = secPerMin;
    this.bell = bell;
    this.warnMin = warnMin;
    this.minutes = start;
  }

  tick(sec) {
    const d = sec / this.secPerMin;
    this.minutes += d;
    return d;
  }

  spend(min) { this.minutes += min; }

  get late() { return this.minutes >= this.bell; }

  get left() { return Math.max(0, this.bell - this.minutes); }

  get warn() { return this.minutes >= this.bell - this.warnMin; }

  label() {
    const m = Math.floor(this.minutes + 1e-9);
    return `${Math.floor(m / 60)}:${pad2(m % 60)}`;
  }
}

module.exports = Clock;
```

- [ ] **Step 4: 运行，确认通过**

Run: `node --test test/clock.test.cjs`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add js/config.js js/engine/clock.js test/clock.test.cjs
git commit -m "feat: add global config and lunch-break clock"
```

---

### Task 5: 周一数据和家里段逻辑

**Files:**
- Create: `js/data/days.js`, `js/engine/home.js`
- Test: `test/home.test.cjs`

**Interfaces:**
- Consumes: `config.home`
- Produces:
  - `js/data/days.js` 导出数组，`[0]` 是周一：`{ id, name, game, seed, temptations: [{ at, dur, bonus, text }], track: { length, firstAt, gapMin, gapMax, weights }, stars: [n1, n2, n3] }`
  - `js/engine/home.js` 导出 `class Home`：`new Home(config.home, day, rng, emit)`；属性 `state`（`'playing' | 'confirm' | 'shutting' | 'hung' | 'off'`）、`addiction`、`temps`（每个诱惑多了 `status: 'pending' | 'active' | 'done' | 'missed'` 和 `progress`）、`wait`；getter `active`（进行中的诱惑或 `null`）；方法 `update(dMin, now)`、`pressPower()`、`cancel()`、`confirm()`、`endTask()`；通过 `emit(ev, payload)` 发 `'temptation' { t, status }`、`'hang' {}`、`'off' {}`

- [ ] **Step 1: 写失败的测试**

`test/home.test.cjs`:

```js
const test = require('node:test');
const assert = require('node:assert/strict');
const CONFIG = require('../js/config');
const DAYS = require('../js/data/days');
const Home = require('../js/engine/home');

const T = (h, m) => h * 60 + m;
const START = T(12, 25);
const DAY = {
  temptations: [
    { at: T(12, 27), dur: 2, bonus: 50, text: 'A' },
    { at: T(12, 30), dur: 1, bonus: 80, text: 'B' },
  ],
};
// 按顺序循环返回给定的数，代替随机数
const seq = (...xs) => { let i = 0; return () => xs[i++ % xs.length]; };
// 以 0.1 游戏分钟为步长推进 minutes 分钟，返回新的"现在"
function step(h, from, minutes) {
  let now = from;
  for (let i = 0; i < Math.round(minutes / 0.1); i++) {
    now += 0.1;
    h.update(0.1, now);
  }
  return now;
}
const near = (a, b) => Math.abs(a - b) < 1e-6;

test('周一的数据完整', () => {
  const mon = DAYS[0];
  assert.equal(mon.id, 'mon');
  assert.equal(mon.temptations.length, 2);
  assert.equal(mon.stars.length, 3);
  assert.ok(mon.track.length > 0);
});

test('电脑开着时每游戏分钟 +10 过瘾值', () => {
  const h = new Home(CONFIG.home, { temptations: [] }, seq(0.5));
  step(h, START, 3);
  assert.ok(near(h.addiction, 30));
});

test('诱惑等到完成才加奖励，进行中的那个可以读到进度', () => {
  const events = [];
  const h = new Home(CONFIG.home, DAY, seq(0.5), (ev, p) => events.push(`${ev}:${p.t ? p.t.text : ''}:${p.status || ''}`));
  step(h, START, 5.5); // 到 12:30.5：A 已完成，B 进行到一半
  assert.ok(near(h.addiction, 55 + 50));
  assert.equal(h.active.text, 'B');
  assert.ok(Math.abs(h.active.progress - 0.5) < 0.06);
  assert.deepEqual(events, ['temptation:A:active', 'temptation:A:done', 'temptation:B:active']);
});

test('确认关机时进行中的诱惑作废，之后不再涨过瘾值', () => {
  const h = new Home(CONFIG.home, DAY, seq(0.5, 0.9));
  const now = step(h, START, 5.5);
  const before = h.addiction;
  h.pressPower();
  assert.equal(h.state, 'confirm');
  h.confirm();
  assert.equal(h.state, 'shutting');
  assert.equal(h.temps[1].status, 'missed');
  assert.equal(h.active, null);
  step(h, now, 1);
  assert.equal(h.addiction, before);
});

test('确认框打开时电脑还开着：过瘾值照涨', () => {
  const h = new Home(CONFIG.home, { temptations: [] }, seq(0.5));
  h.pressPower();
  step(h, START, 1);
  assert.ok(near(h.addiction, 10));
});

test('关机等待 1～2 游戏分钟，然后变成 off', () => {
  const events = [];
  const h = new Home(CONFIG.home, { temptations: [] }, seq(0.5, 0.9), (ev) => events.push(ev));
  h.pressPower();
  h.confirm(); // wait = 1 + 0.5 × 1 = 1.5；0.9 ≥ 0.2，不卡住
  assert.ok(near(h.wait, 1.5));
  const now = step(h, START, 1.4);
  assert.equal(h.state, 'shutting');
  step(h, now, 0.2);
  assert.equal(h.state, 'off');
  assert.deepEqual(events, ['off']);
});

test('20% 概率卡住：一直等也不会关，点结束任务后再等 0.5 分钟', () => {
  const h = new Home(CONFIG.home, { temptations: [] }, seq(0, 0.1));
  h.pressPower();
  h.confirm(); // wait = 1，0.1 < 0.2 → 等到一半时卡住
  let now = step(h, START, 0.6);
  assert.equal(h.state, 'hung');
  now = step(h, now, 5);
  assert.equal(h.state, 'hung');
  h.endTask();
  assert.equal(h.state, 'shutting');
  assert.ok(h.wait >= 0.9 - 1e-6 && h.wait <= 1.0 + 1e-6);
  now = step(h, now, 0.85);
  assert.equal(h.state, 'shutting');
  step(h, now, 0.3);
  assert.equal(h.state, 'off');
});

test('按键只在对应状态生效：取消回到 playing，重复确认不会重新抽等待时间', () => {
  const h = new Home(CONFIG.home, { temptations: [] }, seq(0.5, 0.9, 0.1, 0.1));
  h.confirm();
  assert.equal(h.state, 'playing');
  h.pressPower();
  h.cancel();
  assert.equal(h.state, 'playing');
  h.pressPower();
  h.confirm();
  const wait = h.wait;
  h.confirm();
  h.pressPower();
  h.endTask();
  assert.equal(h.state, 'shutting');
  assert.equal(h.wait, wait);
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test test/home.test.cjs`
Expected: FAIL（Cannot find module '../js/data/days'）

- [ ] **Step 3: 写实现**

`js/data/days.js`:

```js
/**
 * 每天的配置（设计文档 6.1）。M1 只有周一。
 * 时间都是游戏分钟（12:28 = 12 * 60 + 28）；track 的长度单位是米
 */
const T = (h, m) => h * 60 + m;

module.exports = [
  {
    id: 'mon',
    name: '周一',
    game: 'stone',
    seed: 101,
    temptations: [
      { at: T(12, 28), dur: 2, bonus: 50, text: '宠物快抓到了！再扔一次' },
      { at: T(12, 38), dur: 3, bonus: 100, text: '还差 1 场战斗就升级' },
    ],
    track: { length: 330, firstAt: 30, gapMin: 16, gapMax: 26, weights: { bikes: 3, puddle: 2, pole: 2 } },
    stars: [120, 220, 320],
  },
];
```

`js/engine/home.js`:

```js
/**
 * 家里段（设计文档 3.2、3.3）：过瘾值、诱惑节点、关机流程
 * 状态：playing → confirm → shutting ⇄ hung → off；confirm 可以 cancel 回 playing
 * 电脑开着（playing / confirm）时过瘾值才增长、诱惑才推进；确认关机的瞬间，进行中的诱惑作废
 */
class Home {
  constructor(cfg, day, rng, emit = () => {}) {
    this.cfg = cfg;
    this.rng = rng;
    this.emit = emit;
    this.state = 'playing';
    this.addiction = 0;
    this.wait = 0;
    this.hangAt = null;
    this.temps = day.temptations.map((t) => Object.assign({}, t, { status: 'pending', progress: 0 }));
  }

  get computerOn() { return this.state === 'playing' || this.state === 'confirm'; }

  get active() { return this.temps.find((t) => t.status === 'active') || null; }

  // dMin: 本帧经过的游戏分钟；now: 当前时钟读数
  update(dMin, now) {
    if (this.computerOn) {
      this.addiction += dMin * this.cfg.addictionPerMin;
      for (const t of this.temps) {
        if (t.status === 'pending' && now >= t.at) {
          t.status = 'active';
          this.emit('temptation', { t, status: 'active' });
        }
        if (t.status === 'active') {
          t.progress = Math.min(1, (now - t.at) / t.dur);
          if (t.progress >= 1) {
            t.status = 'done';
            this.addiction += t.bonus;
            this.emit('temptation', { t, status: 'done' });
          }
        }
      }
      return;
    }
    if (this.state === 'shutting') {
      this.wait -= dMin;
      if (this.hangAt !== null && this.wait <= this.hangAt) {
        this.hangAt = null;
        this.state = 'hung';
        this.emit('hang', {});
        return;
      }
      if (this.wait <= 0) {
        this.wait = 0;
        this.state = 'off';
        this.emit('off', {});
      }
    }
  }

  pressPower() { if (this.state === 'playing') this.state = 'confirm'; }

  cancel() { if (this.state === 'confirm') this.state = 'playing'; }

  confirm() {
    if (this.state !== 'confirm') return;
    const [lo, hi] = this.cfg.shutdownMin;
    this.wait = lo + this.rng() * (hi - lo);
    this.hangAt = this.rng() < this.cfg.hangChance ? this.wait / 2 : null;
    for (const t of this.temps) {
      if (t.status === 'active') {
        t.status = 'missed';
        this.emit('temptation', { t, status: 'missed' });
      }
    }
    this.state = 'shutting';
  }

  endTask() {
    if (this.state !== 'hung') return;
    this.wait += this.cfg.hangExtraMin;
    this.state = 'shutting';
  }
}

module.exports = Home;
```

- [ ] **Step 4: 运行，确认通过**

Run: `node --test test/home.test.cjs`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add js/data/days.js js/engine/home.js test/home.test.cjs
git commit -m "feat: add monday data and home phase with temptations and shutdown"
```

---

### Task 6: 跑步者

**Files:**
- Create: `js/engine/runner.js`
- Test: `test/runner.test.cjs`

**Interfaces:**
- Consumes: `clamp` from `js/util.js`；`config.run`
- Produces: `js/engine/runner.js` 导出 `class Runner`：`new Runner(config.run)`；属性 `lane`（目标道 0..2）、`x`（当前横向位置，米）、`z`（已跑距离，米）、`stamina`、`sprinting`、`winded`（剩余喘气秒数）、`stumble`（剩余摔倒秒数）、`action`（`null | 'jump' | 'slide'`）；getter `speed`、`laneAt`（离 `x` 最近的道）、`actionProgress`（0..1）；方法 `swipe(dir)`（`'left' | 'right' | 'up' | 'down'`）、`setSprint(on)`、`hit()`、`update(dt)`

- [ ] **Step 1: 写失败的测试**

`test/runner.test.cjs`:

```js
const test = require('node:test');
const assert = require('node:assert/strict');
const CONFIG = require('../js/config');
const Runner = require('../js/engine/runner');

const frames = (r, sec) => { for (let i = 0; i < Math.round(sec * 60); i++) r.update(1 / 60); };
const near = (a, b, eps = 1e-6) => Math.abs(a - b) < eps;

test('默认在中间道，自动以 10 米/秒往前跑', () => {
  const r = new Runner(CONFIG.run);
  assert.equal(r.lane, 1);
  assert.equal(r.laneAt, 1);
  frames(r, 1);
  assert.ok(near(r.z, 10, 1e-6));
});

test('左右滑换道，不会滑出路面；0.15 秒内到位', () => {
  const r = new Runner(CONFIG.run);
  r.swipe('left');
  r.swipe('left');
  assert.equal(r.lane, 0);
  frames(r, 0.1);
  assert.ok(r.x > -3 && r.x < 0);
  frames(r, 0.1);
  assert.equal(r.x, -3);
  assert.equal(r.laneAt, 0);
  r.swipe('right');
  r.swipe('right');
  r.swipe('right');
  assert.equal(r.lane, 2);
});

test('上滑跳 0.6 秒，跳的时候不能钻；结束后可以再动作', () => {
  const r = new Runner(CONFIG.run);
  r.swipe('up');
  assert.equal(r.action, 'jump');
  r.swipe('down');
  assert.equal(r.action, 'jump');
  frames(r, 0.3);
  assert.ok(near(r.actionProgress, 0.5, 0.03));
  frames(r, 0.35);
  assert.equal(r.action, null);
  r.swipe('down');
  assert.equal(r.action, 'slide');
  frames(r, 0.75);
  assert.equal(r.action, null);
});

test('冲刺速度 ×1.5，每秒耗 25 体力；耗尽后喘气 1.5 秒、速度减半，然后体力回到 30', () => {
  const r = new Runner(CONFIG.run);
  r.setSprint(true);
  assert.equal(r.speed, 15);
  frames(r, 2);
  assert.ok(near(r.stamina, 50, 1e-6));
  frames(r, 2.1);
  assert.ok(r.winded > 0);
  assert.equal(r.stamina, 0);
  assert.equal(r.speed, 5);
  r.setSprint(false); // 松开冲刺，方便检查喘气结束时体力正好回到 30
  frames(r, 1.5);
  assert.equal(r.winded, 0);
  assert.ok(r.stamina >= 30 && r.stamina < 31, `stamina ${r.stamina}`);
  r.setSprint(true);
  assert.equal(r.speed, 15);
});

test('不冲刺时每秒恢复 8 体力，不超过 100', () => {
  const r = new Runner(CONFIG.run);
  r.stamina = 50;
  frames(r, 1);
  assert.ok(near(r.stamina, 58, 1e-6));
  frames(r, 10);
  assert.equal(r.stamina, 100);
});

test('摔倒：0.6 秒原地不动，打断跳跃，期间滑动无效', () => {
  const r = new Runner(CONFIG.run);
  r.swipe('up');
  r.hit();
  assert.equal(r.action, null);
  assert.equal(r.speed, 0);
  r.swipe('left');
  r.swipe('up');
  assert.equal(r.lane, 1);
  assert.equal(r.action, null);
  const z = r.z;
  frames(r, 0.5);
  assert.equal(r.z, z);
  frames(r, 0.2);
  assert.equal(r.speed, 10);
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test test/runner.test.cjs`
Expected: FAIL（Cannot find module '../js/engine/runner'）

- [ ] **Step 3: 写实现**

`js/engine/runner.js`:

```js
const { clamp } = require('../util');

/**
 * 跑步者（设计文档 4.2–4.4）：换道、跳、钻、冲刺、体力、喘气、摔倒
 * 只管自己的状态和速度；有没有撞上由引擎判定，撞上后调用 hit()
 */
class Runner {
  constructor(cfg) {
    this.cfg = cfg;
    this.lane = 1;
    this.x = cfg.lanes[1];
    this.z = 0;
    this.stamina = cfg.stamina.max;
    this.sprinting = false;
    this.winded = 0;
    this.stumble = 0;
    this.action = null;
    this.actionT = 0;
  }

  get speed() {
    const c = this.cfg;
    if (this.stumble > 0) return 0;
    if (this.winded > 0) return c.speed * c.stamina.windedMult;
    return this.sprinting ? c.speed * c.sprintMult : c.speed;
  }

  // 跑步者现在所在的道：离 x 最近的那条（换道途中按更近的算）
  get laneAt() {
    const L = this.cfg.lanes;
    let best = 0;
    for (let i = 1; i < L.length; i++) {
      if (Math.abs(L[i] - this.x) < Math.abs(L[best] - this.x)) best = i;
    }
    return best;
  }

  get actionProgress() {
    if (!this.action) return 0;
    const dur = this.action === 'jump' ? this.cfg.jumpSec : this.cfg.slideSec;
    return 1 - this.actionT / dur;
  }

  swipe(dir) {
    if (this.stumble > 0) return;
    const last = this.cfg.lanes.length - 1;
    if (dir === 'left') this.lane = clamp(this.lane - 1, 0, last);
    else if (dir === 'right') this.lane = clamp(this.lane + 1, 0, last);
    else if (!this.action && (dir === 'up' || dir === 'down')) {
      this.action = dir === 'up' ? 'jump' : 'slide';
      this.actionT = dir === 'up' ? this.cfg.jumpSec : this.cfg.slideSec;
    }
  }

  setSprint(on) { this.sprinting = !!on; }

  hit() {
    this.stumble = this.cfg.stumbleSec;
    this.action = null;
    this.actionT = 0;
  }

  update(dt) {
    const c = this.cfg;
    const s = c.stamina;
    const v = this.speed; // 用本帧开始时的速度
    const target = c.lanes[this.lane];
    const step = (Math.abs(c.lanes[1] - c.lanes[0]) / c.laneSwitchSec) * dt;
    this.x = Math.abs(target - this.x) <= step ? target : this.x + Math.sign(target - this.x) * step;
    if (this.action) {
      this.actionT -= dt;
      if (this.actionT <= 0) {
        this.action = null;
        this.actionT = 0;
      }
    }
    if (this.stumble > 0) this.stumble = Math.max(0, this.stumble - dt);
    if (this.winded > 0) {
      this.winded -= dt;
      if (this.winded <= 0) {
        this.winded = 0;
        this.stamina = s.after;
      }
    } else if (this.sprinting && v > 0) {
      this.stamina -= s.drain * dt;
      if (this.stamina <= 0) {
        this.stamina = 0;
        this.winded = s.windedSec;
      }
    } else {
      this.stamina = Math.min(s.max, this.stamina + s.regen * dt);
    }
    this.z += v * dt;
  }
}

module.exports = Runner;
```

- [ ] **Step 4: 运行，确认通过**

Run: `node --test test/runner.test.cjs`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add js/engine/runner.js test/runner.test.cjs
git commit -m "feat: add runner with lanes, jump, slide, sprint and stamina"
```

---

### Task 7: 阻碍数据和轨道

**Files:**
- Create: `js/data/obstacles.js`, `js/engine/track.js`
- Test: `test/track.test.cjs`

**Interfaces:**
- Consumes: `pickWeighted`、`createRng` from `js/util.js`；`day.track`
- Produces:
  - `js/data/obstacles.js` 导出 `{ bikes, puddle, pole }`，每项 `{ name, width, clearBy: string[], penalty }`
  - `js/engine/track.js` 导出 `class Track`：`new Track(day.track, obstacles, rng, { lanes })`；属性 `length`、`items: [{ z, type, lanes: number[], done }]`；方法 `crossed(from, to) → items`（`from < z <= to` 且 `!done`）

- [ ] **Step 1: 写失败的测试**

`test/track.test.cjs`:

```js
const test = require('node:test');
const assert = require('node:assert/strict');
const { createRng } = require('../js/util');
const OBSTACLES = require('../js/data/obstacles');
const DAYS = require('../js/data/days');
const Track = require('../js/engine/track');

const SPEC = DAYS[0].track;
const make = (spec = SPEC, seed = 3) => new Track(spec, OBSTACLES, createRng(seed), { lanes: 3 });

test('阻碍定义：自行车只能换道躲，积水能跳，晾衣杆占满整条路只能钻', () => {
  assert.deepEqual(OBSTACLES.bikes.clearBy, []);
  assert.deepEqual(OBSTACLES.puddle.clearBy, ['jump']);
  assert.equal(OBSTACLES.pole.width, 3);
  assert.deepEqual(OBSTACLES.pole.clearBy, ['slide']);
});

test('同一种子生成相同的轨道', () => {
  assert.deepEqual(make().items, make().items);
  assert.notDeepEqual(make(SPEC, 3).items, make(SPEC, 4).items);
});

test('物件从 firstAt 开始，最后 20 米没有物件，间隔在 gapMin～gapMax 之间', () => {
  const t = make();
  assert.ok(t.items.length > 5);
  assert.equal(t.items[0].z, SPEC.firstAt);
  for (const it of t.items) assert.ok(it.z < SPEC.length - 20);
  for (let i = 1; i < t.items.length; i++) {
    const gap = t.items[i].z - t.items[i - 1].z;
    assert.ok(gap >= SPEC.gapMin && gap <= SPEC.gapMax, `gap ${gap}`);
  }
});

test('晾衣杆占 0、1、2 三条道；其他阻碍只占一条道', () => {
  for (const it of make().items) {
    if (it.type === 'pole') assert.deepEqual(it.lanes, [0, 1, 2]);
    else {
      assert.equal(it.lanes.length, 1);
      assert.ok(it.lanes[0] >= 0 && it.lanes[0] <= 2);
    }
  }
});

test('firstAt 超过终点前 20 米时没有物件', () => {
  const t = make(Object.assign({}, SPEC, { firstAt: 9999 }));
  assert.deepEqual(t.items, []);
});

test('crossed 返回 (from, to] 区间内没处理过的物件', () => {
  const t = make();
  const first = t.items[0];
  assert.deepEqual(t.crossed(0, first.z - 0.01), []);
  assert.deepEqual(t.crossed(first.z - 1, first.z), [first]);
  assert.deepEqual(t.crossed(first.z, first.z + 0.01), []);
  first.done = true;
  assert.deepEqual(t.crossed(first.z - 1, first.z), []);
});

```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test test/track.test.cjs`
Expected: FAIL（Cannot find module '../js/data/obstacles'）

- [ ] **Step 3: 写实现**

`js/data/obstacles.js`:

```js
/**
 * 阻碍定义（设计文档 4.6），M1 只有三种
 *   width: 占几条道（3 = 整条路）
 *   clearBy: 用哪种动作能过去（空数组 = 只能换道躲开）
 *   penalty: 撞上后额外扣的游戏分钟
 */
module.exports = {
  bikes: { name: '自行车流', width: 1, clearBy: [], penalty: 1 },
  puddle: { name: '积水', width: 1, clearBy: ['jump'], penalty: 0.5 },
  pole: { name: '晾衣杆', width: 3, clearBy: ['slide'], penalty: 0.8 },
};
```

`js/engine/track.js`:

```js
const { pickWeighted } = require('../util');

/**
 * 轨道（设计文档 10.3）：把当天的 track 配置和随机种子展开成具体的物件
 * 只提供数据（第 z 米有什么），不知道怎么画；M1 是一条没有路口的直路
 */
class Track {
  constructor(spec, defs, rng, { lanes = 3 } = {}) {
    this.length = spec.length;
    this.items = [];
    let z = spec.firstAt;
    while (z < spec.length - 20) {
      const type = pickWeighted(rng, spec.weights);
      const w = Math.min(defs[type].width, lanes);
      const first = w >= lanes ? 0 : Math.floor(rng() * (lanes - w + 1));
      const covered = [];
      for (let i = 0; i < w; i++) covered.push(first + i);
      this.items.push({ z, type, lanes: covered, done: false });
      z += spec.gapMin + rng() * (spec.gapMax - spec.gapMin);
    }
  }

  crossed(from, to) {
    return this.items.filter((it) => !it.done && it.z > from && it.z <= to);
  }
}

module.exports = Track;
```

- [ ] **Step 4: 运行，确认通过**

Run: `node --test test/track.test.cjs`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add js/data/obstacles.js js/engine/track.js test/track.test.cjs
git commit -m "feat: add obstacle definitions and seeded track generation"
```

---

### Task 8: 单局引擎

**Files:**
- Create: `js/engine/game-engine.js`
- Test: `test/game-engine.test.cjs`

**Interfaces:**
- Consumes: `createRng`；`Clock`（Task 4）；`Home`（Task 5）；`Runner`（Task 6）；`Track`（Task 7）；`config`；`obstacles`
- Produces: `js/engine/game-engine.js` 导出 `{ GameEngine }`：
  - `new GameEngine({ config, obstacles })`；`on(ev, fn) → this`
  - 属性：`state`（`'ready' | 'home' | 'leaving' | 'run' | 'over'`）、`paused`、`clock`、`home`、`runner`、`track`、`result`、`stumbles`
  - 方法：`start(day, { seed })`、`update(dt)`、`pressPower()`、`cancelShutdown()`、`confirmShutdown()`、`endTask()`、`swipe(dir)`、`sprint(on)`、`pause()`、`resume()`、`view()`
  - 事件：`'state' { state }`、`'temptation' { t, status }`、`'hang' {}`、`'off' {}`、`'stumble' { type, penalty }`、`'winded' {}`、`'over' result`
  - `result`：`{ win, onTime, reason: 'gate' | 'home', arrival: 'H:MM', minutesLeft, addiction, score, stars, stumbles, day }`
  - `view()`：`ready` 时返回 `{ state: 'ready' }`；否则返回
    `{ state, paused, day: { id, name, game }, clock: { label, minutes, left, late, warn }, home: { state, addiction, temptation: { text, progress, bonus } | null }, leave: 0..1, run: { z, x, lane, action, actionProgress, stamina, staminaMax, winded, stumble, sprinting } | null, track, result }`

- [ ] **Step 1: 写失败的测试**

`test/game-engine.test.cjs`:

```js
const test = require('node:test');
const assert = require('node:assert/strict');
const CONFIG = require('../js/config');
const OBSTACLES = require('../js/data/obstacles');
const DAYS = require('../js/data/days');
const { GameEngine } = require('../js/engine/game-engine');

const T = (h, m) => h * 60 + m;
// 测试用的一天：默认 100 米、没有物件、没有诱惑
const day = (track = {}) => ({
  id: 'test', name: '测试', game: 'stone', seed: 7, temptations: [],
  track: Object.assign({ length: 100, firstAt: 9999, gapMin: 9999, gapMax: 9999, weights: { pole: 1 } }, track),
  stars: [10, 20, 30],
});
const newEngine = () => new GameEngine({ config: CONFIG, obstacles: OBSTACLES });
function frames(e, sec, each) {
  for (let i = 0; i < Math.round(sec * 60); i++) {
    if (each) each(e);
    e.update(1 / 60);
  }
}
function until(e, pred, maxSec = 300, each) {
  for (let i = 0; i < maxSec * 60; i++) {
    if (pred(e)) return true;
    if (each) each(e);
    e.update(1 / 60);
  }
  return pred(e);
}
// 点关机、确认，卡住就点结束任务，直到离开家
function leaveHome(e) {
  e.pressPower();
  e.confirmShutdown();
  assert.ok(until(e, (x) => x.state !== 'home', 60, (x) => { if (x.home.state === 'hung') x.endTask(); }));
}
function toGate(e, each) {
  assert.ok(until(e, (x) => x.state === 'over', 300, each));
  return e.result;
}
// 让跑步者进入第一个物件所在的道（物件只占一条道时）
function swipeInto(e) {
  const lane = e.track.items[0].lanes[0];
  if (lane === 0) e.swipe('left');
  if (lane === 2) e.swipe('right');
}

test('开局在家里，12:25，ready 时 view 只有 state', () => {
  const e = newEngine();
  assert.deepEqual(e.view(), { state: 'ready' });
  const states = [];
  e.on('state', (p) => states.push(p.state));
  e.start(day());
  assert.equal(e.state, 'home');
  assert.deepEqual(states, ['home']);
  const v = e.view();
  assert.equal(v.clock.label, '12:25');
  assert.equal(v.home.state, 'playing');
  assert.equal(v.run, null);
});

test('出门转场 2 秒，时钟不走，然后进入跑步段', () => {
  const e = newEngine();
  e.start(day());
  leaveHome(e);
  assert.equal(e.state, 'leaving');
  const m = e.clock.minutes;
  frames(e, 1.9);
  assert.equal(e.state, 'leaving');
  assert.equal(e.clock.minutes, m);
  assert.ok(e.view().leave > 0.9);
  frames(e, 0.2);
  assert.equal(e.state, 'run');
  assert.ok(e.view().run);
});

test('一直不关机：到 13:00 立刻结束，算失败', () => {
  const e = newEngine();
  e.start(day());
  let over = null;
  e.on('over', (r) => { over = r; });
  assert.ok(until(e, (x) => x.state === 'over', 120));
  assert.equal(over.reason, 'home');
  assert.equal(over.win, false);
  assert.equal(over.stars, 0);
  assert.equal(over.arrival, '13:00');
  frames(e, 1);
  assert.equal(e.state, 'over');
});

test('早早出门、一路顺利：按时到校，得分 = 过瘾值 + 剩余分钟 × 5', () => {
  const e = newEngine();
  e.start(day());
  leaveHome(e);
  const r = toGate(e);
  assert.equal(r.reason, 'gate');
  assert.equal(r.win, true);
  assert.equal(r.onTime, true);
  assert.equal(r.stumbles, 0);
  assert.equal(r.score, r.addiction + r.minutesLeft * 5);
  assert.equal(r.stars, 3);
});

test('晾衣杆：不钻就摔，多花约 1 分钟（0.8 惩罚 + 0.6 秒趴地）；在杆前下滑就能钻过去', () => {
  const play = (slide) => {
    const e = newEngine();
    e.start(day({ firstAt: 30 }));
    assert.equal(e.track.items[0].type, 'pole');
    const hits = [];
    e.on('stumble', (p) => hits.push(p));
    leaveHome(e);
    let slid = false;
    toGate(e, (x) => {
      if (slide && !slid && x.state === 'run' && x.runner.z >= 26) { x.swipe('down'); slid = true; }
    });
    return { e, hits };
  };
  const hit = play(false);
  const clean = play(true);
  assert.deepEqual(hit.hits, [{ type: 'pole', penalty: 0.8 }]);
  assert.equal(hit.e.result.stumbles, 1);
  assert.equal(clean.e.result.stumbles, 0);
  const diff = hit.e.clock.minutes - clean.e.clock.minutes;
  assert.ok(Math.abs(diff - 1.0) < 0.02, `diff ${diff}`);
});

test('自行车：在它那条道上会撞，换到别的道就躲开', () => {
  const play = (into) => {
    const e = newEngine();
    e.start(day({ firstAt: 30, weights: { bikes: 1 } }));
    leaveHome(e);
    until(e, (x) => x.state === 'run', 5);
    const lane = e.track.items[0].lanes[0];
    if (into) swipeInto(e);
    else if (lane === 1) e.swipe('left');
    return toGate(e).stumbles;
  };
  assert.equal(play(true), 1);
  assert.equal(play(false), 0);
});

test('积水：在它那条道上起跳就不会摔', () => {
  const play = (jump) => {
    const e = newEngine();
    e.start(day({ firstAt: 30, weights: { puddle: 1 } }));
    leaveHome(e);
    until(e, (x) => x.state === 'run', 5);
    swipeInto(e);
    let jumped = false;
    return toGate(e, (x) => {
      if (jump && !jumped && x.runner.z >= 27) { x.swipe('up'); jumped = true; }
    }).stumbles;
  };
  assert.equal(play(false), 1);
  assert.equal(play(true), 0);
});

test('在家待到 12:52 才走，330 米跑不完：迟到，算失败', () => {
  const e = newEngine();
  e.start(day({ length: 330 }));
  assert.ok(until(e, (x) => x.clock.minutes >= T(12, 52), 120));
  leaveHome(e);
  const r = toGate(e);
  assert.equal(r.reason, 'gate');
  assert.equal(r.onTime, false);
  assert.equal(r.win, false);
  assert.equal(r.stars, 0);
  assert.equal(r.score, r.addiction);
});

test('暂停：时钟不走、冲刺松开、滑动无效；继续后恢复', () => {
  const e = newEngine();
  e.start(day());
  leaveHome(e);
  until(e, (x) => x.state === 'run', 5);
  e.sprint(true);
  e.pause();
  assert.equal(e.runner.sprinting, false);
  const m = e.clock.minutes;
  const z = e.runner.z;
  e.swipe('left');
  e.sprint(true);
  frames(e, 1);
  assert.equal(e.clock.minutes, m);
  assert.equal(e.runner.z, z);
  assert.equal(e.runner.lane, 1);
  assert.equal(e.runner.sprinting, false);
  assert.equal(e.view().paused, true);
  e.resume();
  frames(e, 0.5);
  assert.ok(e.runner.z > z);
});

test('单帧 dt 最多算 0.1 秒；连续用最大步长也不会穿过晾衣杆', () => {
  const e = newEngine();
  e.start(day({ firstAt: 30 }));
  const m = e.clock.minutes;
  e.update(5);
  assert.ok(Math.abs(e.clock.minutes - m - 0.1 / 3) < 1e-9);
  leaveHome(e);
  for (let i = 0; i < 3000 && e.state !== 'over'; i++) {
    if (e.home.state === 'hung') e.endTask();
    e.update(5);
  }
  assert.equal(e.result.stumbles, 1);
});

test('体力耗尽时发 winded 事件', () => {
  const e = newEngine();
  e.start(day({ length: 330 }));
  let winded = 0;
  e.on('winded', () => { winded++; });
  leaveHome(e);
  until(e, (x) => x.state === 'run', 5);
  e.sprint(true);
  frames(e, 4.2);
  assert.equal(winded, 1);
  assert.equal(e.view().run.winded, true);
});

test('同一种子、同样的操作，结果完全相同（用周一的真实数据）', () => {
  const play = () => {
    const e = newEngine();
    e.start(DAYS[0]);
    leaveHome(e);
    return toGate(e);
  };
  assert.deepEqual(play(), play());
});

test('家里的诱惑会通过引擎发出来，view 能读到进行中的诱惑', () => {
  const e = newEngine();
  e.start(DAYS[0]);
  const seen = [];
  e.on('temptation', (p) => seen.push(p.status));
  assert.ok(until(e, (x) => x.clock.minutes >= T(12, 29), 30));
  assert.deepEqual(seen, ['active']);
  assert.equal(e.view().home.temptation.text, DAYS[0].temptations[0].text);
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test test/game-engine.test.cjs`
Expected: FAIL（Cannot find module '../js/engine/game-engine'）

- [ ] **Step 3: 写实现**

`js/engine/game-engine.js`:

```js
const { createRng } = require('../util');
const Clock = require('./clock');
const Home = require('./home');
const Runner = require('./runner');
const Track = require('./track');

/**
 * 单局总控（设计文档 2、10.3）
 * 状态：ready → home → leaving → run → over
 * 引擎只用累计的 dt，不碰平台和绘制；用 on/emit 往外发事件：
 *   state { state } / temptation { t, status } / hang / off / stumble { type, penalty } / winded / over（结算）
 */
class GameEngine {
  constructor({ config, obstacles }) {
    this.cfg = config;
    this.defs = obstacles;
    this.handlers = {};
    this.state = 'ready';
    this.paused = false;
  }

  on(ev, fn) {
    (this.handlers[ev] = this.handlers[ev] || []).push(fn);
    return this;
  }

  emit(ev, payload) { (this.handlers[ev] || []).forEach((fn) => fn(payload)); }

  setState(state) {
    this.state = state;
    this.emit('state', { state });
  }

  start(day, { seed } = {}) {
    const cfg = this.cfg;
    const rng = createRng(seed != null ? seed : day.seed);
    this.day = day;
    this.clock = new Clock(cfg.clock);
    this.home = new Home(cfg.home, day, rng, (ev, p) => this.emit(ev, p));
    this.track = new Track(day.track, this.defs, rng, { lanes: cfg.run.lanes.length });
    this.runner = null;
    this.leaveT = 0;
    this.paused = false;
    this.result = null;
    this.stumbles = 0;
    this.setState('home');
  }

  update(dt) {
    if (this.paused || this.state === 'ready' || this.state === 'over') return;
    dt = Math.min(dt, this.cfg.maxDt);
    if (this.state === 'home') {
      const dMin = this.clock.tick(dt);
      this.home.update(dMin, this.clock.minutes);
      if (this.home.state === 'off') {
        this.leaveT = this.cfg.leaveSec;
        this.setState('leaving');
      } else if (this.clock.late) {
        this.finish('home');
      }
    } else if (this.state === 'leaving') {
      this.leaveT -= dt;
      if (this.leaveT <= 0) {
        this.runner = new Runner(this.cfg.run);
        this.setState('run');
      }
    } else if (this.state === 'run') {
      this.clock.tick(dt);
      const r = this.runner;
      const from = r.z;
      const wasWinded = r.winded > 0;
      r.update(dt);
      if (!wasWinded && r.winded > 0) this.emit('winded', {});
      this.collide(from, r.z);
      if (r.z >= this.track.length) this.finish('gate');
    }
  }

  // 本帧跨过的物件：在同一条道上、又没用对动作，就摔倒并扣时间
  collide(from, to) {
    const r = this.runner;
    for (const it of this.track.crossed(from, to)) {
      it.done = true;
      const def = this.defs[it.type];
      if (!it.lanes.includes(r.laneAt)) continue;
      if (r.action && def.clearBy.includes(r.action)) continue;
      r.hit();
      this.clock.spend(def.penalty);
      this.stumbles++;
      this.emit('stumble', { type: it.type, penalty: def.penalty });
    }
  }

  // M1 没有好人好事：只有按时进校门才算过关（设计文档 2.2）
  finish(reason) {
    const c = this.clock;
    const onTime = reason === 'gate' && !c.late;
    const addiction = Math.round(this.home.addiction);
    const minutesLeft = Math.floor(c.left);
    const score = addiction + (onTime ? minutesLeft * this.cfg.score.perMinLeft : 0);
    const stars = onTime ? this.day.stars.filter((s) => score >= s).length : 0;
    this.result = {
      win: onTime, onTime, reason, arrival: c.label(), minutesLeft, addiction, score, stars,
      stumbles: this.stumbles, day: this.day.id,
    };
    if (this.runner) this.runner.setSprint(false);
    this.setState('over');
    this.emit('over', this.result);
  }

  canAct(state) { return this.state === state && !this.paused; }

  pressPower() { if (this.canAct('home')) this.home.pressPower(); }

  cancelShutdown() { if (this.canAct('home')) this.home.cancel(); }

  confirmShutdown() { if (this.canAct('home')) this.home.confirm(); }

  endTask() { if (this.canAct('home')) this.home.endTask(); }

  swipe(dir) { if (this.canAct('run')) this.runner.swipe(dir); }

  sprint(on) { if (this.canAct('run')) this.runner.setSprint(on); }

  pause() {
    if (this.state === 'ready' || this.state === 'over') return;
    this.paused = true;
    if (this.runner) this.runner.setSprint(false);
  }

  resume() { this.paused = false; }

  view() {
    if (this.state === 'ready') return { state: 'ready' };
    const c = this.clock;
    const h = this.home;
    const r = this.runner;
    const t = h.active;
    let leave = 0;
    if (this.state === 'leaving') leave = 1 - this.leaveT / this.cfg.leaveSec;
    else if (r) leave = 1;
    return {
      state: this.state,
      paused: this.paused,
      day: { id: this.day.id, name: this.day.name, game: this.day.game },
      clock: { label: c.label(), minutes: c.minutes, left: c.left, late: c.late, warn: c.warn },
      home: {
        state: h.state,
        addiction: Math.round(h.addiction),
        temptation: t ? { text: t.text, progress: t.progress, bonus: t.bonus } : null,
      },
      leave,
      run: r ? {
        z: r.z, x: r.x, lane: r.lane, action: r.action, actionProgress: r.actionProgress,
        stamina: r.stamina, staminaMax: this.cfg.run.stamina.max,
        winded: r.winded > 0, stumble: r.stumble > 0, sprinting: r.sprinting && r.winded <= 0 && r.stumble <= 0,
      } : null,
      track: this.track,
      result: this.result,
    };
  }
}

module.exports = { GameEngine };
```

- [ ] **Step 4: 运行，确认通过**

Run: `node --test test/game-engine.test.cjs`
Expected: PASS（13 个测试）

- [ ] **Step 5: 提交**

```bash
git add js/engine/game-engine.js test/game-engine.test.cjs
git commit -m "feat: add game engine tying home, leaving and run phases"
```

---

### Task 9: 手势识别

**Files:**
- Create: `js/input.js`
- Test: `test/input.test.cjs`

**Interfaces:**
- Produces: `js/input.js` 导出 `class GestureTracker`：`new GestureTracker({ swipeDist = 24, holdMs = 180 })`；方法都返回命令数组：`down(id, x, y, t)`、`move(id, x, y)`、`up(id, x, y)`、`cancel(id)`、`tick(t)`、`reset()`。命令：`{ action: 'swipe', dir }`、`{ action: 'holdStart' }`、`{ action: 'holdEnd' }`、`{ action: 'tap', x, y }`

- [ ] **Step 1: 写失败的测试**

`test/input.test.cjs`:

```js
const test = require('node:test');
const assert = require('node:assert/strict');
const GestureTracker = require('../js/input');

test('四个方向的滑动，以位移大的方向为准', () => {
  const g = new GestureTracker();
  const swipe = (dx, dy) => {
    g.down(1, 100, 100, 0);
    const out = g.move(1, 100 + dx, 100 + dy);
    g.up(1, 100 + dx, 100 + dy);
    return out;
  };
  assert.deepEqual(swipe(-30, 5), [{ action: 'swipe', dir: 'left' }]);
  assert.deepEqual(swipe(30, -10), [{ action: 'swipe', dir: 'right' }]);
  assert.deepEqual(swipe(5, -30), [{ action: 'swipe', dir: 'up' }]);
  assert.deepEqual(swipe(-10, 30), [{ action: 'swipe', dir: 'down' }]);
});

test('位移不够不算滑动；同一根手指可以连续滑', () => {
  const g = new GestureTracker();
  g.down(1, 0, 0, 0);
  assert.deepEqual(g.move(1, 10, 0), []);
  assert.deepEqual(g.move(1, 30, 0), [{ action: 'swipe', dir: 'right' }]);
  assert.deepEqual(g.move(1, 40, 0), []);
  assert.deepEqual(g.move(1, 60, 0), [{ action: 'swipe', dir: 'right' }]);
  assert.deepEqual(g.up(1, 60, 0), []);
});

test('快速点一下就是 tap，带抬起时的坐标', () => {
  const g = new GestureTracker();
  g.down(1, 50, 60, 0);
  assert.deepEqual(g.tick(100), []);
  assert.deepEqual(g.up(1, 52, 61), [{ action: 'tap', x: 52, y: 61 }]);
});

test('按住 180 毫秒开始长按；松开时结束长按，并且仍然算一次 tap（按钮按得慢也要生效）', () => {
  const g = new GestureTracker();
  g.down(1, 50, 60, 1000);
  assert.deepEqual(g.tick(1170), []);
  assert.deepEqual(g.tick(1180), [{ action: 'holdStart' }]);
  assert.deepEqual(g.tick(1500), []);
  assert.deepEqual(g.up(1, 50, 60), [{ action: 'holdEnd' }, { action: 'tap', x: 50, y: 60 }]);
});

test('先滑动的手指不会再变成长按；长按中滑动照常发 swipe，长按不断', () => {
  const g = new GestureTracker();
  g.down(1, 0, 0, 0);
  g.move(1, 30, 0);
  assert.deepEqual(g.tick(500), []);
  assert.deepEqual(g.up(1, 30, 0), []);
  g.down(2, 0, 0, 0);
  assert.deepEqual(g.tick(200), [{ action: 'holdStart' }]);
  assert.deepEqual(g.move(2, -30, 0), [{ action: 'swipe', dir: 'left' }]);
  assert.deepEqual(g.up(2, -30, 0), [{ action: 'holdEnd' }]);
});

test('两根手指都按住：只在第一根开始时 holdStart、最后一根松开时 holdEnd', () => {
  const g = new GestureTracker();
  g.down(1, 0, 0, 0);
  g.down(2, 100, 0, 50);
  assert.deepEqual(g.tick(200), [{ action: 'holdStart' }]);
  assert.deepEqual(g.tick(260), []);
  assert.deepEqual(g.up(1, 0, 0), [{ action: 'tap', x: 0, y: 0 }]);
  assert.deepEqual(g.up(2, 100, 0), [{ action: 'holdEnd' }, { action: 'tap', x: 100, y: 0 }]);
});

test('被系统取消的触摸：结束长按但不算 tap；未知手指和 reset 之后的事件被忽略', () => {
  const g = new GestureTracker();
  g.down(1, 0, 0, 0);
  g.tick(300);
  assert.deepEqual(g.cancel(1), [{ action: 'holdEnd' }]);
  g.down(2, 0, 0, 0);
  assert.deepEqual(g.cancel(2), []);
  assert.deepEqual(g.move(9, 100, 100), []);
  assert.deepEqual(g.up(9, 0, 0), []);
  g.down(3, 0, 0, 0);
  g.reset();
  assert.deepEqual(g.tick(1000), []);
  assert.deepEqual(g.up(3, 0, 0), []);
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test test/input.test.cjs`
Expected: FAIL（Cannot find module '../js/input'）

- [ ] **Step 3: 写实现**

`js/input.js`:

```js
/**
 * 手势识别（设计文档 4.2）：滑动、长按、点击
 * 每根手指单独跟踪：
 *   - 移动超过 swipeDist 就发一次 swipe，并以当前位置为新起点（同一根手指可以连续滑）
 *   - 按住超过 holdMs 且还没滑过 → 进入长按；任意手指在长按就算"冲刺中"，
 *     holdStart / holdEnd 只在长按手指数 0 ↔ 1 切换时发
 *   - 抬起时没滑过就是 tap（包括按得慢、已经进入长按的情况，保证按钮一定能按）
 * 时间由调用方传入（毫秒），方便测试
 */
class GestureTracker {
  constructor({ swipeDist = 24, holdMs = 180 } = {}) {
    this.swipeDist = swipeDist;
    this.holdMs = holdMs;
    this.reset();
  }

  reset() {
    this.ptrs = new Map();
    this.holding = 0;
  }

  down(id, x, y, t) {
    this.ptrs.set(id, { x0: x, y0: y, t0: t, swiped: false, holding: false });
    return [];
  }

  move(id, x, y) {
    const p = this.ptrs.get(id);
    if (!p) return [];
    const dx = x - p.x0;
    const dy = y - p.y0;
    if (Math.max(Math.abs(dx), Math.abs(dy)) < this.swipeDist) return [];
    p.swiped = true;
    p.x0 = x;
    p.y0 = y;
    const dir = Math.abs(dx) >= Math.abs(dy) ? (dx < 0 ? 'left' : 'right') : (dy < 0 ? 'up' : 'down');
    return [{ action: 'swipe', dir }];
  }

  tick(t) {
    const out = [];
    for (const p of this.ptrs.values()) {
      if (p.holding || p.swiped || t - p.t0 < this.holdMs) continue;
      p.holding = true;
      this.holding++;
      if (this.holding === 1) out.push({ action: 'holdStart' });
    }
    return out;
  }

  up(id, x, y) {
    const p = this.ptrs.get(id);
    if (!p) return [];
    this.ptrs.delete(id);
    const out = [];
    if (p.holding) {
      this.holding--;
      if (this.holding === 0) out.push({ action: 'holdEnd' });
    }
    if (!p.swiped) out.push({ action: 'tap', x, y });
    return out;
  }

  cancel(id) {
    return this.up(id, 0, 0).filter((c) => c.action !== 'tap');
  }
}

module.exports = GestureTracker;
```

- [ ] **Step 4: 运行，确认通过**

Run: `node --test test/input.test.cjs`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add js/input.js test/input.test.cjs
git commit -m "feat: add gesture tracker for swipe, hold and tap"
```

---

### Task 10: 像素素材

**Files:**
- Create: `js/art/palette.js`, `js/art/sprites/student.js`, `js/art/sprites/props.js`, `js/art/sprites/stone.js`, `js/art/sprites.js`
- Test: `test/sprites.test.cjs`

**Interfaces:**
- Produces:
  - `js/art/palette.js` 导出颜色表（键名见下方代码）
  - 每个 `js/art/sprites/*.js` 导出 `{ 素材名: { palette: { 字符: 颜色 | null }, frames: [string[]] } }`
  - `js/art/sprites.js` 导出 `{ Sprites, DATA }`：`new Sprites(createCanvas)`（`createCanvas(w, h)` 返回有 `getContext('2d')` 的画布）；`get(name, frame = 0) → { img, w, h }`（未知名字抛错）；`frames(name) → 帧数`；`draw(g, name, x, y, { frame, scale, ax = 0.5, ay = 1, flip })`（默认以底部中点为锚点）
  - 素材名：`student_run`（2 帧）、`student_jump`、`student_slide`、`student_fall`、`bike`、`puddle`、`shop_red`、`shop_blue`、`tree`、`gate`、`dino`、`beast`

- [ ] **Step 1: 写失败的测试**

`test/sprites.test.cjs`:

```js
const test = require('node:test');
const assert = require('node:assert/strict');
const { Sprites, DATA } = require('../js/art/sprites');

// 假画布：记录 fillRect 次数，供缓存检查
function fakeCanvas(w, h) {
  const calls = [];
  const ctx = new Proxy({ calls }, {
    get: (t, k) => (k in t ? t[k] : (...a) => { if (k === 'fillRect' || k === 'drawImage') calls.push([k, ...a]); }),
    set: (t, k, v) => { t[k] = v; return true; },
  });
  return { width: w, height: h, getContext: () => ctx };
}

const NAMES = ['student_run', 'student_jump', 'student_slide', 'student_fall', 'bike', 'puddle',
  'shop_red', 'shop_blue', 'tree', 'gate', 'dino', 'beast'];

test('所有需要的素材都在', () => {
  for (const n of NAMES) assert.ok(DATA[n], n);
  assert.equal(DATA.student_run.frames.length, 2);
});

test('每个素材：各帧尺寸一致、每行等宽、用到的字符都在调色板里', () => {
  for (const [name, s] of Object.entries(DATA)) {
    const h = s.frames[0].length;
    const w = s.frames[0][0].length;
    for (const rows of s.frames) {
      assert.equal(rows.length, h, `${name} 行数`);
      rows.forEach((row, i) => {
        assert.equal(row.length, w, `${name} 第 ${i} 行宽度`);
        for (const ch of row) assert.ok(ch in s.palette, `${name} 字符 ${ch}`);
      });
    }
  }
});

test('Sprites 启动时把每一帧缓存成画布，get 返回尺寸；未知名字报错', () => {
  const sp = new Sprites(fakeCanvas);
  const s = sp.get('student_run', 1);
  assert.equal(s.w, 12);
  assert.equal(s.h, 18);
  assert.ok(s.img.getContext('2d').calls.length > 0);
  assert.equal(sp.frames('student_run'), 2);
  assert.equal(sp.get('student_run', 3), sp.get('student_run', 1));
  assert.throws(() => sp.get('nope'));
});

test('draw 默认以底部中点为锚点，按比例缩放并取整', () => {
  const sp = new Sprites(fakeCanvas);
  const g = fakeCanvas(180, 320).getContext('2d');
  sp.draw(g, 'student_run', 90, 200, { scale: 2 });
  const [, img, x, y, w, h] = g.calls[g.calls.length - 1];
  assert.equal(img, sp.get('student_run').img);
  assert.deepEqual([x, y, w, h], [78, 164, 24, 36]);
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test test/sprites.test.cjs`
Expected: FAIL（Cannot find module '../js/art/sprites'）

- [ ] **Step 3: 写实现**

`js/art/palette.js`:

```js
/**
 * 全局调色板（设计文档 9）：偏暖、略微褪色，像旧照片
 * 电脑屏幕里的游戏画面另用更饱和的颜色（见 sprites/stone.js）
 */
module.exports = {
  ink: '#2b2622', skin: '#f2c3a0', shirt: '#efe8d8', blue: '#3d6aa8', red: '#c8473a', shoe: '#4a4038', bag: '#5a7d43',
  sky: '#a9cde0', far: '#93a7ad', haze: '#c9d6d0',
  grass: '#8fb35a', grass2: '#7fa24d', walk: '#cbbfa8', walk2: '#bfb39b', road: '#6f6a64', road2: '#686360', line: '#e8e0c8',
  wall: '#e6d6b8', wall2: '#d9c7a6', desk: '#8a5a3a', desk2: '#744a2e',
  crt: '#d8d2c0', crt2: '#b8b19c', win: '#1f6f8b', winBar: '#c0c0c0', winTitle: '#0a246a',
  gold: '#f2b632', warn: '#e0483c', white: '#fffaf0', black: '#141210', water: '#6fa8c8',
};
```

`js/art/sprites/student.js`:

```js
const P = require('../palette');

// 学生（背影）：黑头发、白校服、红领巾、绿书包、蓝校裤（设计文档 4.1）
const pal = { '.': null, k: P.ink, s: P.skin, w: P.shirt, b: P.blue, r: P.red, d: P.shoe, g: P.bag };
const mirror = (rows) => rows.map((r) => r.split('').reverse().join(''));

const RUN = [
  '....kkkk....',
  '...kkkkkk...',
  '...kkkkkk...',
  '...kkkkkk...',
  '....ssss....',
  '..wwrrrrww..',
  '.wwggggggww.',
  '.sggggggggs.',
  '.sggggggggs.',
  '..gggggggg..',
  '..wwwwwwww..',
  '..bbbbbbbb..',
  '..bbb..bbb..',
  '..bbb..bbb..',
  '..bbb...bb..',
  '..bb....bb..',
  '.ddd....bb..',
  '........dd..',
];

const JUMP = [
  '....kkkk....',
  '...kkkkkk...',
  '...kkkkkk...',
  '...kkkkkk...',
  's...ssss...s',
  'ss.wrrrrw.ss',
  '.wwggggggww.',
  '..gggggggg..',
  '..gggggggg..',
  '..gggggggg..',
  '..wwwwwwww..',
  '..bbbbbbbb..',
  '.bbbb..bbbb.',
  '.dbb....bbd.',
  '.dd......dd.',
  '............',
  '............',
  '............',
];

const SLIDE = [
  '............',
  '............',
  '............',
  '............',
  '............',
  '............',
  '............',
  '............',
  '....kkkk....',
  '...kkkkkk...',
  '...kkkkkk...',
  '.ssggggggss.',
  '.wggggggggw.',
  '..gggggggg..',
  '..wwwwwwww..',
  'bbbbbbbbbbbb',
  'dbb......bbd',
  'dd........dd',
];

const FALL = [
  '............',
  '............',
  '............',
  '............',
  '............',
  '............',
  '............',
  '............',
  '............',
  '............',
  '............',
  '............',
  '..kkkk......',
  '.kkkkkssww..',
  '.kkkkgggggbb',
  '..ssgggggbbd',
  '....wwwwbbbd',
  '............',
];

module.exports = {
  student_run: { palette: pal, frames: [RUN, mirror(RUN)] },
  student_jump: { palette: pal, frames: [JUMP] },
  student_slide: { palette: pal, frames: [SLIDE] },
  student_fall: { palette: pal, frames: [FALL] },
};
```

`js/art/sprites/props.js`:

```js
const P = require('../palette');

// 路上的东西和街景（设计文档 4.1、4.6）
const BIKE = [
  '...kkkk...',
  '...kkkk...',
  '...ssss...',
  '..rrrrrr..',
  '.rrrrrrrr.',
  '.srrrrrrs.',
  'kkkkkkkkkk',
  '...bbbb...',
  '...bbbb...',
  '....kk....',
  '....kk....',
  '...kkkk...',
  '...k..k...',
  '...k..k...',
  '...kkkk...',
  '....kk....',
];

const PUDDLE = [
  '....wwwwwwww....',
  '..wwwwlwwwwwww..',
  '.wwwwwwwwwlwwww.',
  '...wwwwwwwwww...',
];

const WINDOW = (glass) => 'm' + glass + 'm'.repeat(7) + 'k'.repeat(6) + 'mm';
const SHOP = [
  'r'.repeat(24),
  'r' + 'yyyyrr'.repeat(3) + 'yyyyr',
  'r'.repeat(24),
  'm'.repeat(24),
  WINDOW('g'.repeat(8)),
  WINDOW('gllggggg'),
  WINDOW('g'.repeat(8)),
  WINDOW('g'.repeat(8)),
  'm'.repeat(16) + 'k'.repeat(6) + 'mm',
  'm'.repeat(16) + 'k'.repeat(6) + 'mm',
  'n'.repeat(24),
];
const shop = (sign) => ({
  palette: { '.': null, r: sign, y: P.gold, m: P.wall, g: P.water, l: P.white, k: P.ink, n: P.wall2 },
  frames: [SHOP],
});

const TREE = ['..gggg..', '.gggggg.', 'gggggggg', 'gggggggg', '.gggggg.', '..gggg..'].concat(new Array(10).fill('...tt...'));

const GATE_ROW = (mid) => 'pppp' + mid + 'pppp';
const GATE = [
  GATE_ROW('r'.repeat(32)),
  GATE_ROW('r' + 'y'.repeat(30) + 'r'),
  GATE_ROW('r'.repeat(32)),
]
  .concat(new Array(10).fill(GATE_ROW('.'.repeat(32))))
  .concat([GATE_ROW('k'.repeat(32))])
  .concat(new Array(9).fill(GATE_ROW('k..'.repeat(10) + 'k.')))
  .concat([GATE_ROW('k'.repeat(32))]);

module.exports = {
  bike: { palette: { '.': null, k: P.ink, s: P.skin, r: P.red, b: P.blue }, frames: [BIKE] },
  puddle: { palette: { '.': null, w: P.water, l: P.white }, frames: [PUDDLE] },
  shop_red: shop(P.red),
  shop_blue: shop(P.blue),
  tree: { palette: { '.': null, g: P.bag, t: P.desk2 }, frames: [TREE] },
  gate: { palette: { '.': null, p: P.wall2, r: P.red, y: P.gold, k: P.ink }, frames: [GATE] },
};
```

`js/art/sprites/stone.js`:

```js
const P = require('../palette');

// 电脑屏幕里的"恐龙回合制"占位画（设计文档 3.1、7）：只致敬画面气质，不用原作素材和名字
const DINO = [
  '..........ggg...',
  '.........ggkgg..',
  '.........ggggg..',
  '....ss...gggg...',
  '...skks..ggg....',
  '....ss..gggg....',
  '..gggggggggg....',
  '.ggggggggggg....',
  'gggggggggggg....',
  'g.gggggggggg....',
  '...gg....gg.....',
  '...gg....gg.....',
];

const BEAST = [
  '..oooooo..',
  '.oooooooo.',
  'ookooookoo',
  'oooooooooo',
  'oowwwwwwoo',
  '.oooooooo.',
  '.oo....oo.',
  '.oo....oo.',
];

module.exports = {
  dino: { palette: { '.': null, g: '#58b848', k: P.ink, s: P.skin }, frames: [DINO] },
  beast: { palette: { '.': null, o: '#d07a3a', k: P.ink, w: P.white }, frames: [BEAST] },
};
```

`js/art/sprites.js`:

```js
const DATA = Object.assign({},
  require('./sprites/student'),
  require('./sprites/props'),
  require('./sprites/stone'));

// 把一帧字符矩阵画到一张离屏画布上
function bake(createCanvas, palette, rows) {
  const h = rows.length;
  const w = rows[0].length;
  const img = createCanvas(w, h);
  const g = img.getContext('2d');
  for (let y = 0; y < h; y++) {
    for (let x = 0; x < w; x++) {
      const col = palette[rows[y][x]];
      if (!col) continue;
      g.fillStyle = col;
      g.fillRect(x, y, 1, 1);
    }
  }
  return { img, w, h };
}

/**
 * 素材表（设计文档 9）：名字 → { palette, frames }。启动时把每一帧画到离屏画布上缓存
 * 以后换成美术画的图片时，只需要让 get() 对这个名字返回图片，渲染代码不用改
 */
class Sprites {
  constructor(createCanvas) {
    this.cache = {};
    for (const name of Object.keys(DATA)) {
      const s = DATA[name];
      this.cache[name] = s.frames.map((rows) => bake(createCanvas, s.palette, rows));
    }
  }

  get(name, frame = 0) {
    const frames = this.cache[name];
    if (!frames) throw new Error(`unknown sprite: ${name}`);
    return frames[frame % frames.length];
  }

  frames(name) { return this.get(name) && this.cache[name].length; }

  // 以 (x, y) 为锚点画精灵；ax / ay 是锚点在精灵里的比例，默认底部中点
  draw(g, name, x, y, { frame = 0, scale = 1, ax = 0.5, ay = 1, flip = false } = {}) {
    const s = this.get(name, frame);
    const w = Math.max(1, Math.round(s.w * scale));
    const h = Math.max(1, Math.round(s.h * scale));
    const dx = Math.round(x - w * ax);
    const dy = Math.round(y - h * ay);
    if (!flip) {
      g.drawImage(s.img, dx, dy, w, h);
      return;
    }
    g.save();
    g.translate(dx + w, dy);
    g.scale(-1, 1);
    g.drawImage(s.img, 0, 0, w, h);
    g.restore();
  }
}

module.exports = { Sprites, DATA };
```

- [ ] **Step 4: 运行，确认通过**

Run: `node --test test/sprites.test.cjs`
Expected: PASS。如果"每行等宽"失败，按报错的素材名和行号修正那一行的字符数。

- [ ] **Step 5: 提交**

```bash
git add js/art test/sprites.test.cjs
git commit -m "feat: add code-defined pixel sprites with cached rendering"
```

---

### Task 11: 渲染器核心、标题页和结算页

**Files:**
- Create: `js/renderer/pixel.js`, `js/renderer/renderer.js`, `js/renderer/scene-title.js`, `js/renderer/scene-result.js`
- Create（占位，Task 12、13 会替换成完整实现）: `js/renderer/scene-home.js`, `js/renderer/scene-leaving.js`, `js/renderer/scene-run.js`
- Test: `test/render-smoke.test.cjs`

**Interfaces:**
- Consumes: `config.view`；`Sprites`（Task 10）；`palette`；presenter 接口（Task 3 的 `ThreePresenter.present`）
- Produces:
  - `js/renderer/pixel.js` 导出 `{ rect(g, x, y, w, h, color), bar(g, x, y, w, h, p, fg, bg), digits(g, str, x, y, color, size) → 宽度, clockPlate(g, clock, now) }`
  - `js/renderer/renderer.js` 导出 `class Renderer`：`new Renderer(W, H, safe, { dpr, createCanvas, presenter })`；属性 `w`、`h`（180、320）、`scale`（每个像素占多少 CSS 像素）、`ox`、`oy`、`g`（像素画布的 context）、`buf`（像素画布）、`textCanvas`、`sprites`、`scene`、`sceneData`、`hits`、`texts`、`world`（本帧的 3D 世界，场景可以设置）；方法 `go(name, data)`、`text(str, x, y, opts)`、`hit(action, x, y, w, h)`、`hitTest(sx, sy) → { action } | null`、`onStumble({ penalty }, now)`、`draw(view, now)`
  - 每个场景：`class XxxScene`，`constructor(renderer)`，属性 `bg`（上屏时四周留边的颜色），方法 `draw(g, view, now, data)`；像素画布每帧先清成透明再交给场景画
  - 按钮 action：标题页 `'start'`；结算页 `'restart'`、`'title'`

- [ ] **Step 1: 写失败的测试**

`test/render-smoke.test.cjs`（这一步先写标题页、结算页、缩放和上屏部分；Task 12、13 会往这个文件追加测试）:

```js
const test = require('node:test');
const assert = require('node:assert/strict');
const Renderer = require('../js/renderer/renderer');

// 假的 Canvas 2D 上下文：吞掉所有绘图调用，只实现有返回值的方法
function fakeCtx() {
  const base = { measureText: (s) => ({ width: String(s).length * 10 }) };
  return new Proxy(base, {
    get: (t, k) => (k in t ? t[k] : () => {}),
    set: (t, k, v) => { t[k] = v; return true; },
  });
}
const fakeCanvas = (w, h) => ({ width: w, height: h, getContext: () => fakeCtx() });
// 假的 presenter：只记录每帧交给它的内容
const fakePresenter = () => ({ frames: [], present(f) { this.frames.push(f); } });
const SAFE = { top: 80, bottom: 34 };
const make = (W = 375, H = 812, dpr = 2) => {
  const presenter = fakePresenter();
  const r = new Renderer(W, H, SAFE, { dpr, createCanvas: fakeCanvas, presenter });
  r.presenter = presenter;
  return r;
};
// 低分辨率按钮的中心 → 屏幕坐标
const center = (r, b) => [r.ox + (b.x + b.w / 2) * r.scale, r.oy + (b.y + b.h / 2) * r.scale];
const button = (r, action) => r.hits.find((b) => b.action === action);

test('不同屏幕：按设备像素整数倍放大，居中且在安全区内', () => {
  for (const [W, H, dpr] of [[375, 812, 2], [768, 1024, 2], [360, 640, 3], [414, 896, 3]]) {
    const r = make(W, H, dpr);
    assert.ok(Number.isInteger(Math.round(r.scale * dpr * 1e6) / 1e6), `${W}x${H}`);
    assert.ok(r.scale * dpr >= 1);
    assert.ok(r.ox >= 0 && r.ox + r.w * r.scale <= W + 0.5, `${W}x${H} 横向`);
    assert.ok(r.oy >= SAFE.top - 0.5, `${W}x${H} 顶部`);
    assert.ok(r.oy + r.h * r.scale <= H - SAFE.bottom + 0.5, `${W}x${H} 底部`);
  }
});

test('标题页可绘制，开始按钮的屏幕坐标能点中；按钮外点不中', () => {
  for (const [W, H, dpr] of [[375, 812, 2], [768, 1024, 2]]) {
    const r = make(W, H, dpr);
    r.draw({ state: 'ready' }, 0);
    const b = button(r, 'start');
    assert.ok(b);
    assert.deepEqual(r.hitTest(...center(r, b)), { action: 'start' });
    assert.equal(r.hitTest(r.ox + 1, r.oy + 1), null);
  }
});

test('每帧交给 presenter：像素画布、文字画布、布局；文字没变就不重画文字层', () => {
  const r = make();
  r.draw({ state: 'ready' }, 0);
  r.draw({ state: 'ready' }, 0);
  const [a, b] = r.presenter.frames;
  assert.equal(a.pixel, r.buf);
  assert.equal(a.text, r.textCanvas);
  assert.equal(a.world, null);
  assert.deepEqual(a.layout, { ox: r.ox, oy: r.oy, w: r.w * r.scale, h: r.h * r.scale });
  assert.deepEqual(a.shake, { x: 0, y: 0 });
  assert.equal(a.textDirty, true);
  assert.equal(b.textDirty, false);
});

test('结算页三种结局都能绘制，有再来一次和回标题两个按钮', () => {
  const r = make();
  const results = [
    { win: true, onTime: true, reason: 'gate', arrival: '12:51', minutesLeft: 8, addiction: 250, score: 290, stars: 2, stumbles: 1 },
    { win: false, onTime: false, reason: 'gate', arrival: '13:03', minutesLeft: 0, addiction: 300, score: 300, stars: 0, stumbles: 3 },
    { win: false, onTime: false, reason: 'home', arrival: '13:00', minutesLeft: 0, addiction: 400, score: 400, stars: 0, stumbles: 0 },
  ];
  for (const res of results) {
    r.go('result', res);
    r.draw({ state: 'over' }, 1000);
    assert.ok(button(r, 'restart'));
    assert.ok(button(r, 'title'));
  }
});

test('摔倒时抖动、出现浮字；暂停时画遮罩', () => {
  const r = make();
  r.onStumble({ penalty: 0.8 }, 0);
  r.draw({ state: 'ready' }, 100);
  assert.ok(r.texts.some((t) => t.str === '-0.8 分钟'));
  r.draw({ state: 'ready', paused: true }, 2000);
  assert.ok(r.texts.some((t) => t.str.includes('继续')));
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test test/render-smoke.test.cjs`
Expected: FAIL（Cannot find module '../js/renderer/renderer'）

- [ ] **Step 3: 写实现**

`js/renderer/pixel.js`:

```js
const P = require('../art/palette');

// 3×5 像素数字（时钟用），1 = 有像素
const FONT = {
  0: ['111', '101', '101', '101', '111'],
  1: ['010', '110', '010', '010', '111'],
  2: ['111', '001', '111', '100', '111'],
  3: ['111', '001', '111', '001', '111'],
  4: ['101', '101', '111', '001', '001'],
  5: ['111', '100', '111', '001', '111'],
  6: ['111', '100', '111', '101', '111'],
  7: ['111', '001', '001', '001', '001'],
  8: ['111', '101', '111', '101', '111'],
  9: ['111', '101', '111', '001', '111'],
  ':': ['0', '1', '0', '1', '0'],
};

function rect(g, x, y, w, h, color) {
  g.fillStyle = color;
  g.fillRect(Math.round(x), Math.round(y), Math.round(w), Math.round(h));
}

// 进度条：p 为 0..1
function bar(g, x, y, w, h, p, fg, bg) {
  rect(g, x, y, w, h, bg);
  rect(g, x, y, Math.max(0, Math.min(1, p)) * w, h, fg);
}

// 画像素数字，返回总宽度
function digits(g, str, x, y, color, size = 1) {
  let cx = x;
  g.fillStyle = color;
  for (const ch of String(str)) {
    const glyph = FONT[ch];
    if (!glyph) { cx += 4 * size; continue; }
    glyph.forEach((row, ry) => {
      for (let rx = 0; rx < row.length; rx++) {
        if (row[rx] === '1') g.fillRect(cx + rx * size, y + ry * size, size, size);
      }
    });
    cx += (glyph[0].length + 1) * size;
  }
  return cx - x;
}

// 左上角的电子表（设计文档 2.1）：12:55 之后红白闪烁
function clockPlate(g, clock, now) {
  rect(g, 3, 3, 42, 14, P.black);
  const color = clock.warn ? (Math.floor(now / 250) % 2 ? P.warn : P.white) : P.gold;
  digits(g, clock.label, 6, 5, color, 2);
}

module.exports = { rect, bar, digits, clockPlate };
```

`js/renderer/renderer.js`:

```js
const CONFIG = require('../config');
const P = require('../art/palette');
const { Sprites } = require('../art/sprites');
const { rect } = require('./pixel');
const TitleScene = require('./scene-title');
const HomeScene = require('./scene-home');
const LeavingScene = require('./scene-leaving');
const RunScene = require('./scene-run');
const ResultScene = require('./scene-result');

const SHAKE_MS = 300;
const FLOAT_MS = 900;

/**
 * 渲染器（设计文档 9、10.3）
 * 2D 画面画到 180×320 的像素画布上；中文在像素画布上会糊，所以单独画到屏幕分辨率的文字画布上（内容变了才重画）；
 * 跑步场景另外通过 this.world 提供 3D 世界。三层交给 presenter 合成上屏
 * 每帧根据引擎快照绘制，不修改游戏状态；hitTest 只把坐标翻译成 { action }
 */
class Renderer {
  constructor(W, H, safe, { dpr = 1, createCanvas, presenter }) {
    this.W = W;
    this.H = H;
    this.dpr = dpr;
    this.presenter = presenter;
    this.w = CONFIG.view.w;
    this.h = CONFIG.view.h;
    this.buf = createCanvas(this.w, this.h);
    this.g = this.buf.getContext('2d');
    this.g.imageSmoothingEnabled = false;
    this.textCanvas = createCanvas(Math.round(W * dpr), Math.round(H * dpr));
    this.tg = this.textCanvas.getContext('2d');
    const availH = H - safe.top - safe.bottom;
    const k = Math.max(1, Math.floor(Math.min((W * dpr) / this.w, (availH * dpr) / this.h)));
    this.scale = k / dpr;
    this.ox = Math.round((W - this.w * this.scale) / 2);
    this.oy = Math.round(safe.top + (availH - this.h * this.scale) / 2);
    this.sprites = new Sprites(createCanvas);
    this.texts = [];
    this.hits = [];
    this.floats = [];
    this.world = null;
    this.textKey = null;
    this.shakeAt = -Infinity;
    this.scenes = {
      title: new TitleScene(this),
      home: new HomeScene(this),
      leaving: new LeavingScene(this),
      run: new RunScene(this),
      result: new ResultScene(this),
    };
    this.go('title');
  }

  go(name, data) {
    this.scene = name;
    this.sceneData = data || null;
    this.floats = [];
  }

  // 文字：坐标是像素画布坐标，size 是像素画布上的字高
  text(str, x, y, { size = 8, color = P.ink, align = 'center' } = {}) {
    this.texts.push({ str: String(str), x, y, size, color, align });
  }

  hit(action, x, y, w, h) { this.hits.push({ action, x, y, w, h }); }

  hitTest(sx, sy) {
    const x = (sx - this.ox) / this.scale;
    const y = (sy - this.oy) / this.scale;
    const b = this.hits.find((r) => x >= r.x && x <= r.x + r.w && y >= r.y && y <= r.y + r.h);
    return b ? { action: b.action } : null;
  }

  onStumble({ penalty }, now) {
    this.shakeAt = now;
    this.floats.push({ str: `-${penalty} 分钟`, at: now });
  }

  draw(view, now) {
    this.texts = [];
    this.hits = [];
    this.world = null;
    const scene = this.scenes[this.scene];
    const g = this.g;
    g.clearRect(0, 0, this.w, this.h);
    scene.draw(g, view, now, this.sceneData);
    this.floats = this.floats.filter((f) => now - f.at < FLOAT_MS);
    for (const f of this.floats) {
      const t = (now - f.at) / FLOAT_MS;
      this.text(f.str, this.w / 2, 150 - t * 30, { size: 10, color: P.warn });
    }
    if (view && view.paused) {
      g.globalAlpha = 0.6;
      rect(g, 0, 0, this.w, this.h, P.black);
      g.globalAlpha = 1;
      this.text('已暂停', this.w / 2, 150, { size: 12, color: P.white });
      this.text('点屏幕继续', this.w / 2, 170, { size: 8, color: P.white });
    }
    let sx = 0;
    let sy = 0;
    if (now - this.shakeAt < SHAKE_MS) {
      sx = Math.round((Math.random() * 2 - 1) * 2) * this.scale;
      sy = Math.round((Math.random() * 2 - 1) * 2) * this.scale;
    }
    const textDirty = this.paintTexts(sx, sy);
    this.presenter.present({
      world: this.world,
      pixel: this.buf,
      text: this.textCanvas,
      textDirty,
      layout: { ox: this.ox, oy: this.oy, w: this.w * this.scale, h: this.h * this.scale },
      shake: { x: sx, y: sy },
      bg: scene.bg || P.black,
    });
  }

  // 文字层只在内容（含抖动偏移）变化时重画，返回是否重画了
  paintTexts(sx, sy) {
    const key = `${sx},${sy}|${this.texts.map((t) => `${t.str}/${t.x}/${t.y}/${t.size}/${t.color}/${t.align}`).join('|')}`;
    if (key === this.textKey) return false;
    this.textKey = key;
    const tg = this.tg;
    const s = this.scale * this.dpr;
    tg.clearRect(0, 0, this.textCanvas.width, this.textCanvas.height);
    tg.textBaseline = 'middle';
    for (const t of this.texts) {
      tg.font = `bold ${Math.round(t.size * s)}px "PingFang SC", "Microsoft YaHei", sans-serif`;
      tg.fillStyle = t.color;
      tg.textAlign = t.align;
      tg.fillText(t.str, (this.ox + sx + t.x * this.scale) * this.dpr, (this.oy + sy + t.y * this.scale) * this.dpr);
    }
    return true;
  }
}

module.exports = Renderer;
```

`js/renderer/scene-title.js`:

```js
const P = require('../art/palette');
const { rect } = require('./pixel');

// 标题页（设计文档 8）：初中校门、标题、开始按钮
class TitleScene {
  constructor(r) {
    this.r = r;
    this.bg = P.sky;
  }

  draw(g, view, now) {
    const r = this.r;
    rect(g, 0, 0, r.w, 200, P.sky);
    rect(g, 0, 200, r.w, r.h - 200, P.walk);
    r.sprites.draw(g, 'gate', r.w / 2, 205, { scale: 3 });
    r.text('午休快跑', r.w / 2, 46, { size: 20 });
    r.text('2002 年 · 初二（3）班', r.w / 2, 72, { size: 8 });
    const bob = Math.round(Math.sin(now / 300) * 1.5);
    rect(g, 50, 250 + bob, 80, 24, P.red);
    r.text('开始', 90, 262 + bob, { size: 10, color: P.white });
    r.hit('start', 50, 250, 80, 24);
  }
}

module.exports = TitleScene;
```

`js/renderer/scene-result.js`:

```js
const P = require('../art/palette');
const { rect } = require('./pixel');

// 结算页（设计文档 2.2、8）：按时到校 / 迟到 / 没出门
class ResultScene {
  constructor(r) {
    this.r = r;
    this.bg = P.sky;
  }

  draw(g, view, now, res) {
    const r = this.r;
    rect(g, 0, 0, r.w, r.h, P.sky);
    rect(g, 0, 150, r.w, r.h - 150, P.walk);
    r.sprites.draw(g, 'gate', r.w / 2, 150, { scale: 2.5 });
    if (res.win) {
      r.sprites.draw(g, 'student_run', r.w / 2, 150, { frame: Math.floor(now / 150), scale: 2 });
      r.text('按时到校！', r.w / 2, 26, { size: 16 });
    } else if (res.reason === 'home') {
      r.text('13:00 了……', r.w / 2, 26, { size: 16, color: P.warn });
      r.text('你还坐在电脑前', r.w / 2, 46, { size: 9 });
    } else {
      r.sprites.draw(g, 'student_fall', r.w / 2, 150, { scale: 2 });
      r.text('迟到了', r.w / 2, 26, { size: 16, color: P.warn });
      r.text('被值周老师拦在门口罚站', r.w / 2, 46, { size: 9 });
    }
    rect(g, 20, 166, 140, 104, P.white);
    const stars = '★'.repeat(res.stars) + '☆'.repeat(3 - res.stars);
    r.text(stars, r.w / 2, 182, { size: 14, color: P.gold });
    r.text(`到校时间  ${res.reason === 'home' ? '没出门' : res.arrival}`, r.w / 2, 204, { size: 8 });
    r.text(`过瘾值  ${res.addiction}`, r.w / 2, 220, { size: 8 });
    r.text(`摔倒  ${res.stumbles} 次`, r.w / 2, 236, { size: 8 });
    r.text(`得分  ${res.score}`, r.w / 2, 254, { size: 10 });
    rect(g, 20, 282, 66, 22, P.red);
    r.text('再来一次', 53, 293, { size: 9, color: P.white });
    r.hit('restart', 20, 282, 66, 22);
    rect(g, 94, 282, 66, 22, P.crt2);
    r.text('回标题', 127, 293, { size: 9 });
    r.hit('title', 94, 282, 66, 22);
  }
}

module.exports = ResultScene;
```

占位场景（Task 12、13 替换）。`js/renderer/scene-home.js`、`js/renderer/scene-leaving.js`、`js/renderer/scene-run.js` 三个文件内容相同，只是类名不同（分别为 `HomeScene`、`LeavingScene`、`RunScene`）:

```js
const P = require('../art/palette');

// 占位：完整实现见后续任务
class HomeScene {
  constructor(r) {
    this.r = r;
    this.bg = P.black;
  }

  draw() {}
}

module.exports = HomeScene;
```

- [ ] **Step 4: 运行，确认通过**

Run: `node --test test/render-smoke.test.cjs`
Expected: PASS（5 个测试）

- [ ] **Step 5: 提交**

```bash
git add js/renderer test/render-smoke.test.cjs
git commit -m "feat: add layered renderer core with title and result scenes"
```

---

### Task 12: 家里场景（CRT 电脑）

**Files:**
- Modify: `js/renderer/scene-home.js`（整个替换）
- Test: `test/render-smoke.test.cjs`（追加）

**Interfaces:**
- Consumes: `view.clock`、`view.home`（Task 8）；`rect`、`bar`、`clockPlate`（Task 11）；素材 `dino`、`beast`
- Produces: 按钮 action：`'power'`（仅 `playing`）、`'confirm'` 和 `'cancel'`（仅 `confirm`）、`'endTask'`（仅 `hung`）

- [ ] **Step 1: 写失败的测试**

在 `test/render-smoke.test.cjs` 末尾追加:

```js
const CONFIG = require('../js/config');
const OBSTACLES = require('../js/data/obstacles');
const DAYS = require('../js/data/days');
const { GameEngine } = require('../js/engine/game-engine');

const newEngine = () => new GameEngine({ config: CONFIG, obstacles: OBSTACLES });
const actions = (r) => r.hits.map((b) => b.action).sort();

test('家里场景：每个关机阶段都能绘制，按钮和状态对应', () => {
  const r = make();
  const e = newEngine();
  e.start(DAYS[0]);
  r.go('home');
  let now = 0;
  const draw = () => { now += 16; r.draw(e.view(), now, 1 / 60); };

  draw();
  assert.deepEqual(actions(r), ['power']);
  assert.deepEqual(r.hitTest(...center(r, button(r, 'power'))), { action: 'power' });
  assert.ok(r.texts.some((t) => t.str.includes('过瘾值')));

  for (let i = 0; i < 60 * 12; i++) e.update(1 / 60); // 到 12:29，第一个诱惑进行中
  draw();
  assert.ok(r.texts.some((t) => t.str === DAYS[0].temptations[0].text));

  e.pressPower();
  draw();
  assert.deepEqual(actions(r), ['cancel', 'confirm']);

  e.confirmShutdown();
  e.home.hangAt = null; // 让这次关机不卡住，下面单独画卡住的状态
  draw();
  assert.deepEqual(actions(r), []);
  assert.ok(r.texts.some((t) => t.str.includes('正在关闭')));

  e.home.state = 'hung';
  draw();
  assert.deepEqual(actions(r), ['endTask']);

  e.home.state = 'shutting';
  e.clock.minutes = 12 * 60 + 56; // 打铃前：时钟进入警告状态
  draw();
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test test/render-smoke.test.cjs`
Expected: FAIL（家里场景还是占位，`button(r, 'power')` 为 undefined）

- [ ] **Step 3: 写实现**

`js/renderer/scene-home.js`:

```js
const P = require('../art/palette');
const { rect, bar, digits, clockPlate } = require('./pixel');

// CRT 屏幕在低分辨率画布上的位置（4:3）
const S = { x: 24, y: 50, w: 132, h: 99 };
const POWER = { x: 60, y: 262, w: 60, h: 30 };

/**
 * 家里：第一人称看电脑（设计文档 3.1–3.3）
 * 上部是大头显示器，屏幕里放当天的致敬游戏；中部是桌子和键盘；下部是关机按钮
 */
class HomeScene {
  constructor(r) {
    this.r = r;
    this.bg = P.wall;
  }

  draw(g, view, now) {
    const r = this.r;
    const h = view.home;
    this.room(g);
    this.monitor(g);
    if (h.state === 'playing' || h.state === 'confirm') {
      this.game(g, now);
      this.taskbar(g, view.clock);
      if (h.temptation) this.temptation(g, h.temptation);
      if (h.state === 'confirm') this.confirmDialog(g);
    } else if (h.state === 'shutting') {
      rect(g, S.x, S.y, S.w, S.h, P.win);
      r.text('正在关闭计算机……', S.x + S.w / 2, S.y + S.h / 2, { size: 8, color: P.white });
    } else if (h.state === 'hung') {
      rect(g, S.x, S.y, S.w, S.h, P.win);
      this.dialog(g, '结束程序', '程序没有响应', [{ label: '结束任务', action: 'endTask', x: S.x + 46, w: 40 }]);
    } else {
      rect(g, S.x, S.y, S.w, S.h, P.black);
    }
    const on = h.state === 'playing';
    rect(g, POWER.x, POWER.y, POWER.w, POWER.h, on ? P.red : P.crt2);
    rect(g, POWER.x, POWER.y + POWER.h - 3, POWER.w, 3, P.desk2);
    r.text('关机', POWER.x + POWER.w / 2, POWER.y + POWER.h / 2 - 1, { size: 10, color: on ? P.white : P.ink });
    if (on) r.hit('power', POWER.x, POWER.y, POWER.w, POWER.h);
    clockPlate(g, view.clock, now);
    rect(g, 104, 3, 73, 14, P.black);
    r.text(`过瘾值 ${h.addiction}`, 140, 10, { size: 7, color: P.gold });
  }

  room(g) {
    const r = this.r;
    rect(g, 0, 0, r.w, r.h, P.wall);
    for (let x = 0; x < r.w; x += 12) rect(g, x, 0, 1, 176, P.wall2);
    rect(g, 160, 22, 16, 20, P.white); // 墙上的日历
    rect(g, 160, 22, 16, 5, P.red);
    rect(g, 0, 176, r.w, 76, P.desk);
    rect(g, 0, 176, r.w, 3, P.desk2);
    rect(g, 0, 252, r.w, r.h - 252, P.wall2);
    rect(g, 30, 192, 100, 18, P.crt); // 键盘
    for (let row = 0; row < 3; row++) {
      for (let k = 0; k < 12; k++) rect(g, 33 + k * 8, 195 + row * 5, 6, 3, P.crt2);
    }
    rect(g, 142, 196, 10, 14, P.crt); // 鼠标
    rect(g, 146, 196, 2, 5, P.crt2);
    rect(g, 10, 186, 8, 14, P.red); // 喝了一半的汽水
    rect(g, 10, 186, 8, 3, P.crt2);
  }

  monitor(g) {
    rect(g, 14, 38, 152, 124, P.crt);
    rect(g, 18, 44, 144, 112, P.crt2);
    rect(g, 70, 162, 40, 8, P.crt2);
    rect(g, 50, 170, 80, 6, P.crt);
    rect(g, 148, 150, 6, 3, '#58b848'); // 电源指示灯
  }

  // 屏幕里的"恐龙回合制"占位画（M3 换成五天各自的致敬画面）
  game(g, now) {
    const sp = this.r.sprites;
    rect(g, S.x, S.y, S.w, 40, '#7fd0ff');
    rect(g, S.x, S.y + 40, S.w, S.h - 40, '#7cc860');
    rect(g, S.x, S.y, S.w, 10, P.black);
    this.r.text('攻击  捕捉  道具  逃跑', S.x + S.w / 2, S.y + 5, { size: 6, color: P.white });
    const bob = Math.round(Math.sin(now / 200) * 1.5);
    sp.draw(g, 'dino', S.x + 36, S.y + 78 + bob, { scale: 2 });
    sp.draw(g, 'beast', S.x + 100, S.y + 76, { scale: 2, flip: true });
    if (Math.floor(now / 600) % 2 === 0) rect(g, S.x + 90, S.y + 60, 20, 16, 'rgba(255,255,255,0.5)');
  }

  taskbar(g, clock) {
    const y = S.y + S.h - 8;
    rect(g, S.x, y, S.w, 8, P.winBar);
    rect(g, S.x + 1, y + 1, 18, 6, P.crt2);
    digits(g, clock.label, S.x + S.w - 22, y + 2, P.ink, 1);
  }

  temptation(g, t) {
    const x = S.x + 4;
    const y = S.y + S.h - 34;
    const w = S.w - 8;
    rect(g, x, y, w, 24, 'rgba(0,0,0,0.7)');
    this.r.text(t.text, x + w / 2, y + 7, { size: 6, color: P.white });
    bar(g, x + 6, y + 15, w - 12, 4, t.progress, P.gold, P.ink);
  }

  confirmDialog(g) {
    this.dialog(g, '关闭计算机', '确定要关机吗？', [
      { label: '确定', action: 'confirm', x: S.x + 26, w: 34 },
      { label: '取消', action: 'cancel', x: S.x + 72, w: 34 },
    ]);
  }

  // Win98 风格对话框（重画，不用原版素材）
  dialog(g, title, msg, buttons) {
    const r = this.r;
    const x = S.x + 12;
    const y = S.y + 22;
    const w = S.w - 24;
    rect(g, x, y, w, 52, P.winBar);
    rect(g, x, y, w, 10, P.winTitle);
    r.text(title, x + 4, y + 5, { size: 6, color: P.white, align: 'left' });
    r.text(msg, x + w / 2, y + 22, { size: 7 });
    for (const b of buttons) {
      rect(g, b.x, y + 34, b.w, 13, P.crt);
      rect(g, b.x, y + 46, b.w, 1, P.ink);
      r.text(b.label, b.x + b.w / 2, y + 40, { size: 7 });
      r.hit(b.action, b.x, y + 34, b.w, 13);
    }
  }
}

module.exports = HomeScene;
```

- [ ] **Step 4: 运行，确认通过**

Run: `node --test test/render-smoke.test.cjs`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add js/renderer/scene-home.js test/render-smoke.test.cjs
git commit -m "feat: draw first-person home scene with crt and shutdown dialogs"
```

---

### Task 13: 跑步段的 3D 场景、出门转场和跑步场景

**Files:**
- Create: `js/renderer/world3d.js`
- Modify: `js/renderer/scene-leaving.js`、`js/renderer/scene-run.js`（整个替换占位）
- Test: `test/render-smoke.test.cjs`（追加）

**Interfaces:**
- Consumes: `js/libs/three.js`（Task 2）；`config.iso`、`config.run.lanes`、`config.view`（Task 4）；`Sprites`（Task 10）；`view.leave`、`view.run`、`view.track`、`view.clock`（Task 8）；`track.items`、`track.length`（Task 7）；`rect`、`bar`、`clockPlate`（Task 11）；渲染器的 `r.world`（Task 11）
- Produces: `js/renderer/world3d.js` 导出 `class World3D`：`new World3D(sprites, config?)`；属性 `scene`、`camera`、`kid`（学生纸片）、`items`（放阻碍和校门的 Group）、`props`（循环使用的店铺和树）、`dashes`、`present`（`{ scene, camera, w, h }`，交给 presenter）；方法 `tex(name, frame) → CanvasTexture`（按名字和帧缓存）、`setTrack(track)`、`update(run, now)`

- [ ] **Step 1: 写失败的测试**

在 `test/render-smoke.test.cjs` 末尾追加:

```js
const World3D = require('../js/renderer/world3d');
const { Sprites } = require('../js/art/sprites');
const Track = require('../js/engine/track');
const { createRng } = require('../js/util');

const newWorld = () => new World3D(new Sprites(fakeCanvas));
const runView = (o) => Object.assign({
  z: 0, x: 0, lane: 1, action: null, actionProgress: 0, stamina: 100, staminaMax: 100,
  winded: false, stumble: false, sprinting: false,
}, o);
const trackOf = (seed, spec = DAYS[0].track) => new Track(spec, OBSTACLES, createRng(seed), { lanes: 3 });
const ISO = CONFIG.iso;

test('3D 场景：每个阻碍一个对象，外加校门；换轨道时旧对象被清掉', () => {
  const w = newWorld();
  const t1 = trackOf(1);
  w.setTrack(t1);
  assert.equal(w.items.children.length, t1.items.length + 1);
  const t2 = trackOf(2, Object.assign({}, DAYS[0].track, { length: 100 }));
  w.setTrack(t2);
  assert.equal(w.items.children.length, t2.items.length + 1);
});

test('学生和摄像机跟着跑步者：轨道第 z 米在 three.js 的 -z', () => {
  const w = newWorld();
  w.setTrack(trackOf(1));
  w.update(runView({ z: 100, x: 3 }), 0);
  assert.equal(w.kid.position.x, 3);
  assert.equal(w.kid.position.y, 0);
  assert.equal(w.kid.position.z, -100);
  const o = ISO.camOffset;
  assert.ok(Math.abs(w.camera.position.x - (3 * ISO.camFollowX + o[0])) < 1e-9);
  assert.equal(w.camera.position.y, o[1]);
  assert.ok(Math.abs(w.camera.position.z - (-100 + o[2])) < 1e-9);
});

test('跳到一半离地最高；钻和摔倒换成对应的纹理；跑步两帧交替', () => {
  const w = newWorld();
  w.setTrack(trackOf(1));
  w.update(runView({ action: 'jump', actionProgress: 0.5 }), 0);
  assert.ok(Math.abs(w.kid.position.y - ISO.jumpHeight) < 1e-9);
  w.update(runView({ action: 'slide', actionProgress: 0.5 }), 0);
  assert.equal(w.kid.material.map, w.tex('student_slide'));
  w.update(runView({ stumble: true }), 0);
  assert.equal(w.kid.material.map, w.tex('student_fall'));
  w.update(runView(), 0);
  const a = w.kid.material.map;
  w.update(runView(), 140);
  assert.notEqual(w.kid.material.map, a);
  assert.equal(w.tex('student_run', 2), w.tex('student_run', 0));
});

test('街景循环使用：跑到哪里，店铺和树都在学生附近的固定窗口里', () => {
  const w = newWorld();
  w.setTrack(trackOf(1));
  for (const z of [0, 50, 333, 1000]) {
    w.update(runView({ z }), 0);
    const top = -z + ISO.propBehind;
    for (const p of w.props) {
      assert.ok(p.position.z <= top + 1e-9, `z=${z}`);
      assert.ok(p.position.z > top - ISO.propEvery * ISO.propCount - 1e-9, `z=${z}`);
    }
  }
});

test('出门转场和跑步：跳、钻、冲刺、摔倒、喘气、看到校门、迟到警告都能绘制，跑步时交出 3D 世界', () => {
  const r = make();
  const e = newEngine();
  e.start(DAYS[0]);
  e.on('stumble', (p) => r.onStumble(p, 0));
  e.pressPower();
  e.confirmShutdown();
  for (let i = 0; i < 60 * 30 && e.state === 'home'; i++) {
    if (e.home.state === 'hung') e.endTask();
    e.update(1 / 60);
  }
  r.go('leaving');
  let now = 0;
  while (e.state === 'leaving') {
    e.update(1 / 60);
    now += 16;
    r.draw(e.view(), now);
  }
  assert.equal(e.state, 'run');
  r.go('run');
  const script = { 30: () => e.swipe('up'), 90: () => e.swipe('down'), 150: () => e.swipe('left'), 200: () => e.sprint(true) };
  for (let i = 0; i < 60 * 60 && e.state === 'run'; i++) {
    if (script[i]) script[i]();
    if (i === 600) e.clock.minutes = 12 * 60 + 58;
    e.update(1 / 60);
    now += 16;
    r.draw(e.view(), now);
  }
  assert.equal(e.state, 'over');
  assert.ok(r.texts.some((t) => t.str.includes('体力')));
  const last = r.presenter.frames[r.presenter.frames.length - 1];
  assert.ok(last.world && last.world.scene && last.world.camera);
  assert.equal(last.world.w, 180);
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test test/render-smoke.test.cjs`
Expected: FAIL（Cannot find module '../js/renderer/world3d'）

- [ ] **Step 3: 写实现**

`js/renderer/world3d.js`:

```js
const THREE = require('../libs/three');
const CONFIG = require('../config');
const P = require('../art/palette');

/**
 * 跑步段的 3D 场景（设计文档 4.1、9.1）：只创建 three.js 场景对象，不碰 WebGL
 * 斜 45° 正交摄像机跟着学生走，学生朝屏幕左上方跑；轨道第 z 米对应 three.js 的 -z
 * 店铺放在右边（正面朝路、对着镜头），树放在左边；街景和车道线是固定数量的对象，跑过去就挪到前面循环使用
 */
class World3D {
  constructor(sprites, cfg = CONFIG) {
    const iso = cfg.iso;
    this.sp = sprites;
    this.cfg = cfg;
    this.iso = iso;
    this.texCache = {};
    const s = new THREE.Scene();
    s.background = new THREE.Color(P.sky);
    s.fog = new THREE.Fog(P.haze, iso.fogNear, iso.fogFar);
    this.scene = s;
    const a = cfg.view.w / cfg.view.h;
    this.camera = new THREE.OrthographicCamera((-iso.viewH * a) / 2, (iso.viewH * a) / 2, iso.viewH / 2, -iso.viewH / 2, 0.1, 200);
    s.add(new THREE.HemisphereLight('#fff6e0', P.grass, 0.8));
    const sun = new THREE.DirectionalLight('#fff1d0', 0.8);
    sun.castShadow = true;
    sun.shadow.mapSize.set(512, 512);
    Object.assign(sun.shadow.camera, { left: -20, right: 20, top: 20, bottom: -20, near: 1, far: 60 });
    s.add(sun);
    s.add(sun.target);
    this.sun = sun;
    this.buildGround();
    this.buildDashes();
    this.buildProps();
    this.items = new THREE.Group();
    s.add(this.items);
    this.kid = new THREE.Sprite(new THREE.SpriteMaterial({ map: this.tex('student_run') }));
    this.kid.center.set(0.5, 0);
    this.kid.scale.set(1.2, 1.8, 1);
    s.add(this.kid);
    this.blob = new THREE.Mesh(new THREE.CircleGeometry(0.5, 8), new THREE.MeshBasicMaterial({ color: P.black, transparent: true, opacity: 0.25 }));
    this.blob.rotation.x = -Math.PI / 2;
    s.add(this.blob);
    this.present = { scene: s, camera: this.camera, w: cfg.view.w, h: cfg.view.h };
  }

  // 像素精灵 → 纹理（最近邻、无 mipmap），按名字和帧缓存
  tex(name, frame = 0) {
    const f = frame % this.sp.frames(name);
    const key = `${name}:${f}`;
    if (!this.texCache[key]) {
      const t = new THREE.CanvasTexture(this.sp.get(name, f).img);
      t.minFilter = THREE.NearestFilter;
      t.magFilter = THREE.NearestFilter;
      t.generateMipmaps = false;
      t.colorSpace = THREE.SRGBColorSpace;
      this.texCache[key] = t;
    }
    return this.texCache[key];
  }

  lambert(color) { return new THREE.MeshLambertMaterial({ color }); }

  box(w, h, d, mat, x, y, z) {
    const m = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), mat);
    m.position.set(x, y, z);
    m.castShadow = true;
    m.receiveShadow = true;
    return m;
  }

  // 地面、路面、人行道：一组很长的平板，跟着学生一起平移（颜色均匀，看不出在动）
  buildGround() {
    const iso = this.iso;
    const hw = iso.halfWidth;
    this.ground = new THREE.Group();
    const flat = (w, color, x, y) => {
      const m = new THREE.Mesh(new THREE.PlaneGeometry(w, 200), this.lambert(color));
      m.rotation.x = -Math.PI / 2;
      m.position.set(x, y, 0);
      m.receiveShadow = true;
      this.ground.add(m);
    };
    flat(80, P.grass, 0, 0);
    flat(hw * 2, P.road, 0, 0.01);
    flat(iso.walkWidth, P.walk, -(hw + iso.walkWidth / 2), 0.02);
    flat(iso.walkWidth, P.walk, hw + iso.walkWidth / 2, 0.02);
    this.scene.add(this.ground);
  }

  // 车道线：真的往后退，提供速度感
  buildDashes() {
    const iso = this.iso;
    const geo = new THREE.PlaneGeometry(0.2, 1.2);
    const mat = this.lambert(P.line);
    this.dashes = [];
    for (let i = 0; i < iso.dashCount; i++) {
      for (const x of [-1.5, 1.5]) {
        const d = new THREE.Mesh(geo, mat);
        d.rotation.x = -Math.PI / 2;
        d.position.set(x, 0.03, iso.propBehind - (i + 0.5) * iso.dashEvery);
        this.scene.add(d);
        this.dashes.push(d);
      }
    }
  }

  buildProps() {
    const iso = this.iso;
    const edge = iso.halfWidth + iso.walkWidth;
    this.props = [];
    for (let i = 0; i < iso.propCount; i++) {
      const shop = this.shop(i % 2 ? 'shop_red' : 'shop_blue');
      shop.position.set(edge + 2, 0, iso.propBehind - (i + 0.5) * iso.propEvery);
      const tree = this.tree();
      tree.position.set(-(edge + 1.5), 0, iso.propBehind - i * iso.propEvery);
      this.scene.add(shop, tree);
      this.props.push(shop, tree);
    }
  }

  // 店铺：-x 面朝路、对着镜头，贴店面纹理，宽高比和像素图一致
  shop(name) {
    const g = new THREE.Group();
    const s = this.sp.get(name);
    const len = 6.5;
    const h = (len * s.h) / s.w;
    const wall = this.lambert(P.wall);
    const front = new THREE.MeshLambertMaterial({ map: this.tex(name) });
    g.add(this.box(4, h, len, [wall, front, this.lambert(P.wall2), wall, wall, wall], 0, h / 2, 0));
    g.add(this.box(4.4, 0.4, len + 0.4, this.lambert(P.desk2), 0, h + 0.2, 0));
    return g;
  }

  tree() {
    const g = new THREE.Group();
    g.add(this.box(0.5, 1.6, 0.5, this.lambert(P.desk2), 0, 0.8, 0));
    g.add(this.box(2, 2, 2, this.lambert(P.bag), 0, 2.4, 0));
    g.add(this.box(1.2, 1.2, 1.2, this.lambert(P.grass2), 0, 3.8, 0));
    return g;
  }

  // 每种阻碍一个 Group，局部坐标里摆好，外面只设位置
  item(type) {
    const g = new THREE.Group();
    if (type === 'bikes') {
      const sp = new THREE.Sprite(new THREE.SpriteMaterial({ map: this.tex('bike') }));
      sp.center.set(0.5, 0);
      sp.scale.set(1.1, 1.8, 1);
      g.add(sp);
    } else if (type === 'puddle') {
      const m = new THREE.Mesh(new THREE.PlaneGeometry(2.4, 0.9),
        new THREE.MeshLambertMaterial({ map: this.tex('puddle'), transparent: true, alphaTest: 0.5 }));
      m.rotation.x = -Math.PI / 2;
      m.position.y = 0.04;
      g.add(m);
    } else {
      // 晾衣杆：两根柱子、横杆高 1.55 米、挂着衣服，只能钻过去
      const hw = this.iso.halfWidth + 0.3;
      for (const x of [-hw, hw]) g.add(this.box(0.2, 1.6, 0.2, this.lambert(P.desk2), x, 0.8, 0));
      g.add(this.box(hw * 2, 0.08, 0.08, this.lambert(P.ink), 0, 1.55, 0));
      for (let c = 0; c < 5; c++) g.add(this.box(0.9, 0.6, 0.05, this.lambert(c % 2 ? P.red : P.shirt), -3.6 + c * 1.8, 1.2, 0));
    }
    return g;
  }

  // 换一条轨道（新开一局）：清掉旧的阻碍和校门，按新轨道重建
  setTrack(track) {
    for (const c of this.items.children.slice()) {
      this.items.remove(c);
      c.traverse((o) => {
        if (o.geometry) o.geometry.dispose();
        if (o.material && !o.material.map) o.material.dispose();
      });
    }
    const lanes = this.cfg.run.lanes;
    for (const it of track.items) {
      const obj = this.item(it.type);
      obj.position.set(it.type === 'pole' ? 0 : lanes[it.lanes[0]], 0, -it.z);
      this.items.add(obj);
    }
    const gate = new THREE.Mesh(new THREE.PlaneGeometry(10, 6),
      new THREE.MeshLambertMaterial({ map: this.tex('gate'), transparent: true, alphaTest: 0.5 }));
    gate.position.set(0, 3, -track.length);
    this.items.add(gate);
  }

  update(run, now) {
    const iso = this.iso;
    const zp = -run.z;
    let name = 'student_run';
    let frame = Math.floor(now / (run.sprinting ? 90 : 140));
    let y = 0;
    if (run.stumble) {
      name = 'student_fall';
      frame = 0;
    } else if (run.action === 'jump') {
      name = 'student_jump';
      frame = 0;
      y = Math.sin(Math.PI * run.actionProgress) * iso.jumpHeight;
    } else if (run.action === 'slide') {
      name = 'student_slide';
      frame = 0;
    }
    this.kid.material.map = this.tex(name, frame);
    this.kid.position.set(run.x, y, zp);
    this.blob.position.set(run.x, 0.04, zp);
    this.ground.position.z = zp - 60;
    const top = zp + iso.propBehind;
    for (const p of this.props) while (p.position.z > top) p.position.z -= iso.propEvery * iso.propCount;
    for (const d of this.dashes) while (d.position.z > top) d.position.z -= iso.dashEvery * iso.dashCount;
    const o = iso.camOffset;
    const cx = run.x * iso.camFollowX;
    this.camera.position.set(cx + o[0], o[1], zp + o[2]);
    this.camera.lookAt(cx, 0, zp - iso.lookAhead);
    this.sun.position.set(cx - 8, 16, zp + 6);
    this.sun.target.position.set(cx, 0, zp - 4);
  }
}

module.exports = World3D;
```

`js/renderer/scene-leaving.js`:

```js
const P = require('../art/palette');
const { rect } = require('./pixel');

// 出门转场（设计文档 3.4）：换鞋 → 锁门 → 下楼，三格快剪，共 2 秒
class LeavingScene {
  constructor(r) {
    this.r = r;
    this.bg = P.black;
  }

  draw(g, view) {
    const r = this.r;
    const i = Math.min(2, Math.floor(view.leave * 3));
    rect(g, 0, 0, r.w, r.h, P.black);
    rect(g, 20, 100, 140, 100, P.wall);
    if (i === 0) {
      rect(g, 60, 160, 26, 10, P.shoe); // 一双鞋
      rect(g, 94, 160, 26, 10, P.shoe);
      rect(g, 60, 156, 10, 4, P.shoe);
      rect(g, 94, 156, 10, 4, P.shoe);
    } else if (i === 1) {
      rect(g, 70, 112, 40, 70, P.desk); // 门和钥匙
      rect(g, 100, 145, 4, 6, P.gold);
      rect(g, 104, 147, 10, 2, P.gold);
    } else {
      for (let k = 0; k < 5; k++) rect(g, 40 + k * 20, 120 + k * 14, 100 - k * 20, 14, P.wall2); // 楼梯
    }
    r.text(['换鞋……', '锁门……', '下楼！'][i], r.w / 2, 220, { size: 12, color: P.white });
  }
}

module.exports = LeavingScene;
```

`js/renderer/scene-run.js`:

```js
const P = require('../art/palette');
const World3D = require('./world3d');
const { rect, bar, clockPlate } = require('./pixel');

/**
 * 跑步段（设计文档 4.1）：3D 世界交给 World3D，这里只在像素画布上画 HUD
 * World3D 第一次进入跑步段时才创建；轨道换了（重开一局）就重建阻碍
 */
class RunScene {
  constructor(r) {
    this.r = r;
    this.bg = P.sky;
    this.world = null;
    this.track = null;
  }

  draw(g, view, now) {
    if (!this.world) this.world = new World3D(this.r.sprites);
    if (view.track !== this.track) {
      this.world.setTrack(view.track);
      this.track = view.track;
    }
    this.world.update(view.run, now);
    this.r.world = this.world.present;
    this.hud(g, view, now);
  }

  hud(g, view, now) {
    const r = this.r;
    const run = view.run;
    clockPlate(g, view.clock, now);
    rect(g, 120, 3, 57, 14, P.black);
    r.text(`学校 ${Math.max(0, Math.ceil(view.track.length - run.z))}m`, 148, 10, { size: 7, color: P.white });
    rect(g, 4, 298, 172, 16, 'rgba(20,18,16,0.6)');
    r.text('体力', 18, 306, { size: 7, color: P.white });
    bar(g, 32, 303, 120, 6, run.stamina / run.staminaMax, run.winded ? P.warn : P.gold, P.ink);
    if (run.winded) r.text('喘不过气了……', r.w / 2, 286, { size: 8, color: P.warn });
    if (run.sprinting) {
      for (let k = 0; k < 4; k++) rect(g, k % 2 ? 6 : r.w - 8, 140 + ((now / 4 + k * 37) % 140), 2, 10, P.white);
    }
  }
}

module.exports = RunScene;
```

- [ ] **Step 4: 运行，确认通过**

Run: `npm test`
Expected: PASS（全部测试）

- [ ] **Step 5: 提交**

```bash
git add js/renderer/world3d.js js/renderer/scene-leaving.js js/renderer/scene-run.js test/render-smoke.test.cjs
git commit -m "feat: build isometric pixel 3d run scene with three.js"
```

---

### Task 14: 正式主循环和接线

**Files:**
- Modify: `js/main.js`（整个替换 Task 3 的验证程序）
- Test: `test/main-wiring.test.cjs`

**Interfaces:**
- Consumes: 前面所有模块；`ThreePresenter`（Task 3）
- Produces: `js/main.js` 导出 `{ engine, renderer }`（只给测试和浏览器预览用）

- [ ] **Step 1: 写失败的测试**

`test/main-wiring.test.cjs`:

```js
const test = require('node:test');
const assert = require('node:assert/strict');

/**
 * main.js 接线测试：用假的 tt 全局把整条链路跑起来
 *   触摸 → GestureTracker → 引擎 → 渲染器 → presenter，以及切后台、结算、重开
 * main.js 加载时就会读取 tt，所以 tt 必须在 require 之前装好；
 * Node 里没有 WebGL，所以把 presenter-three 换成只记录帧的假模块
 */
const W = 375;
const H = 812;
let t = 1000000;
const cbs = {};
let rafFn = null;
const ctx = new Proxy({ measureText: (s) => ({ width: String(s).length * 10 }) }, {
  get: (o, k) => (k in o ? o[k] : () => {}),
  set: (o, k, v) => { o[k] = v; return true; },
});

Date.now = () => t;
global.requestAnimationFrame = (fn) => { rafFn = fn; };
global.tt = {
  getSystemInfoSync: () => ({ windowWidth: W, windowHeight: H, pixelRatio: 2, safeArea: { top: 44, bottom: H - 34 } }),
  getMenuButtonBoundingClientRect: () => ({ bottom: 80 }),
  createCanvas: () => ({ width: 0, height: 0, getContext: () => ctx }),
  vibrateShort() {},
  vibrateLong() {},
  getStorageSync() { return null; },
  setStorageSync() {},
};
['onTouchStart', 'onTouchMove', 'onTouchEnd', 'onTouchCancel', 'onHide', 'onShow']
  .forEach((k) => { global.tt[k] = (fn) => { cbs[k] = fn; }; });

const frames3 = [];
const presenterPath = require.resolve('../js/renderer/presenter-three');
require.cache[presenterPath] = {
  id: presenterPath, filename: presenterPath, loaded: true,
  exports: class FakePresenter { present(f) { frames3.push(f); } },
};

const { engine, renderer } = require('../js/main');

const touch = (k, id, x, y) => cbs[k]({ changedTouches: [{ identifier: id, clientX: x, clientY: y }] });
const frames = (sec) => {
  for (let i = 0; i < Math.round(sec * 60); i++) {
    t += 1000 / 60;
    rafFn();
  }
};
const tap = (x, y) => { touch('onTouchStart', 1, x, y); frames(0.05); touch('onTouchEnd', 1, x, y); frames(0.05); };
const tapButton = (action) => {
  const b = renderer.hits.find((h) => h.action === action);
  assert.ok(b, `找不到按钮 ${action}`);
  tap(renderer.ox + (b.x + b.w / 2) * renderer.scale, renderer.oy + (b.y + b.h / 2) * renderer.scale);
};
const swipe = (dx, dy) => {
  touch('onTouchStart', 2, 200, 400);
  touch('onTouchMove', 2, 200 + dx, 400 + dy);
  touch('onTouchEnd', 2, 200 + dx, 400 + dy);
};

test('完整一局：开始 → 关机 → 转场 → 跑步（滑动、长按冲刺）→ 切后台暂停 → 结算 → 重开', () => {
  frames(0.1);
  assert.equal(renderer.scene, 'title');
  assert.equal(frames3[frames3.length - 1].world, null);
  tapButton('start');
  assert.equal(engine.state, 'home');
  assert.equal(renderer.scene, 'home');

  tapButton('power');
  assert.equal(engine.home.state, 'confirm');
  tapButton('cancel');
  assert.equal(engine.home.state, 'playing');
  tapButton('power');
  tapButton('confirm');
  assert.equal(engine.home.state, 'shutting');
  for (let i = 0; i < 60 && engine.state === 'home'; i++) {
    frames(0.5);
    if (engine.home.state === 'hung') tapButton('endTask');
  }
  assert.equal(engine.state, 'leaving');
  assert.equal(renderer.scene, 'leaving');
  frames(2.1);
  assert.equal(engine.state, 'run');
  assert.equal(renderer.scene, 'run');
  assert.ok(frames3[frames3.length - 1].world);

  swipe(-40, 0);
  frames(0.1);
  assert.equal(engine.runner.lane, 0);
  swipe(0, -40);
  assert.equal(engine.runner.action, 'jump');

  touch('onTouchStart', 3, 200, 500);
  frames(0.3);
  assert.equal(engine.runner.sprinting, true);
  touch('onTouchEnd', 3, 200, 500);
  frames(0.05);
  assert.equal(engine.runner.sprinting, false);

  touch('onTouchStart', 3, 200, 500);
  frames(0.3);
  cbs.onHide();
  assert.equal(engine.paused, true);
  assert.equal(engine.runner.sprinting, false);
  const z = engine.runner.z;
  frames(1);
  assert.equal(engine.runner.z, z);
  tap(200, 500);
  assert.equal(engine.paused, false);
  frames(0.5);
  assert.ok(engine.runner.z > z);

  for (let i = 0; i < 120 && engine.state !== 'over'; i++) frames(0.5);
  assert.equal(engine.state, 'over');
  assert.equal(renderer.scene, 'result');
  assert.ok(renderer.sceneData);

  tapButton('restart');
  assert.equal(engine.state, 'home');
  assert.equal(renderer.scene, 'home');
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node --test test/main-wiring.test.cjs`
Expected: FAIL（Task 3 的验证程序没有导出 `engine` 和 `renderer`）

- [ ] **Step 3: 写实现**

`js/main.js`（整个替换）:

```js
const CONFIG = require('./config');
const DAYS = require('./data/days');
const OBSTACLES = require('./data/obstacles');
const platform = require('./platform');
const GestureTracker = require('./input');
const { GameEngine } = require('./engine/game-engine');
const Renderer = require('./renderer/renderer');
const ThreePresenter = require('./renderer/presenter-three');

/**
 * 入口：组装模块、绑定事件、驱动主循环
 *   触摸 → GestureTracker → 引擎（逻辑） → 事件 → 渲染器 → presenter 上屏 / 震动
 */
const { canvas, gl } = platform.createScreenCanvas();
const presenter = new ThreePresenter({ canvas, gl, W: platform.width, H: platform.height, dpr: platform.dpr });
const engine = new GameEngine({ config: CONFIG, obstacles: OBSTACLES });
const renderer = new Renderer(platform.width, platform.height, platform.safe, {
  dpr: platform.dpr,
  createCanvas: platform.createOffscreen,
  presenter,
});
const gestures = new GestureTracker();
const SCENES = { home: 'home', leaving: 'leaving', run: 'run' };

/* ---------- 引擎事件 → 表现层 ---------- */
engine
  .on('state', ({ state }) => { if (SCENES[state]) renderer.go(SCENES[state]); })
  .on('stumble', (p) => {
    renderer.onStumble(p, platform.now());
    platform.vibrate('light');
  })
  .on('over', (result) => {
    gestures.reset();
    renderer.go('result', result);
    platform.vibrate(result.win ? 'double' : 'long');
  });

/* ---------- 输入 → 引擎 ---------- */
function startDay() {
  gestures.reset();
  engine.start(DAYS[0]);
}

function act(hit) {
  if (!hit) return;
  switch (hit.action) {
    case 'start':
    case 'restart': startDay(); break;
    case 'title': renderer.go('title'); break;
    case 'power': engine.pressPower(); break;
    case 'confirm': engine.confirmShutdown(); break;
    case 'cancel': engine.cancelShutdown(); break;
    case 'endTask': engine.endTask(); break;
    default: break;
  }
}

function dispatch(cmds) {
  for (const c of cmds) {
    if (c.action === 'tap') {
      if (engine.paused) engine.resume();
      else act(renderer.hitTest(c.x, c.y));
    } else if (c.action === 'swipe') engine.swipe(c.dir);
    else if (c.action === 'holdStart') engine.sprint(true);
    else if (c.action === 'holdEnd') engine.sprint(false);
  }
}

platform.onTouchStart((id, x, y) => dispatch(gestures.down(id, x, y, platform.now())));
platform.onTouchMove((id, x, y) => dispatch(gestures.move(id, x, y)));
platform.onTouchEnd((id, x, y) => dispatch(gestures.up(id, x, y)));
platform.onTouchCancel((id) => dispatch(gestures.cancel(id)));

// 切到后台：暂停（时钟停、冲刺松开），丢掉手指状态，回来后点屏幕继续
platform.onHide(() => {
  engine.pause();
  gestures.reset();
});

/* ---------- 主循环 ---------- */
let last = platform.now();
function loop() {
  const t = platform.now();
  const dt = (t - last) / 1000; // 引擎内部会把单帧步长限制在 maxDt
  last = t;
  dispatch(gestures.tick(t));
  engine.update(dt);
  renderer.draw(engine.view(), t);
  platform.raf(loop);
}

loop();

module.exports = { engine, renderer };
```

- [ ] **Step 4: 运行，确认通过**

Run: `npm test`
Expected: PASS（全部测试）

- [ ] **Step 5: 提交**

```bash
git add js/main.js test/main-wiring.test.cjs
git commit -m "feat: wire gestures, engine, renderer and three presenter in main loop"
```

---

### Task 15: 试玩验证

**Files:** 无新增（用 Task 3 的浏览器预览）

- [ ] **Step 1: 浏览器里按手机尺寸检查**

用 `mcp__Claude_Browser__preview_start`（name: `retro-run-preview`）启动服务器，打开 `http://127.0.0.1:8766/.claude/preview/index.html`，用 `resize_window` 切到 `mobile` 预设后刷新。依次截图确认：

1. 标题页：校门、"午休快跑"、开始按钮
2. 家里：显示器里有恐龙、任务栏时钟；左上角电子表在走；右上角过瘾值在涨；12:28 出现诱惑框和进度条
3. 点"关机"出现对话框；"取消"能回去；"确定"后屏幕变蓝显示"正在关闭计算机……"（卡住时出现"结束任务"按钮）
4. 转场三格：换鞋、锁门、下楼
5. 跑步：斜 45° 俯视，学生朝左上方跑；右边店铺、左边树循环出现；车道线在后退；有颗粒状阴影；方向键换道、跳、钻；按住空格冲刺时体力条下降；撞到障碍时屏幕抖动并出现"-x 分钟"；跑到终点看到校门
6. 到校后显示结算；迟到时显示"迟到了"；在家挂到 13:00 显示"你还坐在电脑前"（可以在控制台执行 `E.clock.minutes = 12*60+59.9` 快速验证）

用 `read_console_messages`（onlyErrors）确认没有报错。检查完后把窗口恢复成 `desktop`。

- [ ] **Step 2: 抖音开发者工具和真机**

在抖音开发者工具里编译运行一局，再扫码在真机上玩一局，确认和浏览器一致、帧率和 Task 3 的验证结果相近。

- [ ] **Step 3: 交给 Rainman 试玩，收集反馈**

请 Rainman 在真机上玩几局，重点回答 M1 要验证的问题（设计文档第 12 节）：
- "再玩一会儿还是现在走"有没有纠结？
- 跑步的手感爽不爽（换道、跳、钻的响应，冲刺和体力的节奏，速度感）？
- 斜 45° 像素 3D 的画面有没有年代感？

把试玩结论写进设计文档第 1 节末尾（参照 `nine_to_nope` 的"M1 试玩结论"写法），然后提交并推送：

```bash
git add docs/superpowers/specs/2026-10-13-retro-run-design.md
git commit -m "docs: record m1 playtest findings"
git push
```
