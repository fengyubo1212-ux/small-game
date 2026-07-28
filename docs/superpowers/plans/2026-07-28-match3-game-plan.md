# 图片消消乐（三消型）实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建手机端三消小游戏，单 HTML 文件，6×6 棋盘，3 类图片方块，触摸拖拽交换，连击计分，目标分数闯关。

**Architecture:** 单文件 `index.html`，内联 CSS + JS。CSS Grid 渲染棋盘，touch 事件处理拖拽交换，async/await 编排消除→下落→填充动画链，combo 循环自动检测连锁消除。

**Tech Stack:** 纯 HTML/CSS/JS，零依赖。三张本地图片与 HTML 同级目录。

---

## File Structure

| 文件 | 职责 |
|------|------|
| `index.html` | 全部代码：HTML 结构 + CSS 样式 + JS 游戏逻辑 |

---

### Task 1: HTML 骨架 + CSS 布局 + 初始状态

**Files:**
- Create: `index.html`

- [ ] **Step 1: 创建 HTML 结构与 CSS 样式**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>图片消消乐</title>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background: linear-gradient(135deg, #e8e0f0 0%, #d0e0f0 100%);
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  touch-action: none;
  -webkit-user-select: none;
  user-select: none;
}

.game-container {
  width: 100%;
  max-width: 400px;
  padding: 12px;
}

.status-bar {
  background: white;
  border-radius: 16px;
  padding: 16px 20px;
  margin-bottom: 12px;
  box-shadow: 0 2px 12px rgba(0,0,0,0.08);
}

.status-bar .level-title {
  font-size: 20px;
  font-weight: 700;
  color: #333;
  text-align: center;
  margin-bottom: 8px;
}

.status-bar .info-row {
  display: flex;
  justify-content: space-between;
  font-size: 14px;
  color: #666;
}

.status-bar .steps-bar {
  height: 6px;
  background: #eee;
  border-radius: 3px;
  margin-top: 8px;
  overflow: hidden;
}

.status-bar .steps-fill {
  height: 100%;
  background: linear-gradient(90deg, #ff6b6b, #ffa94d);
  border-radius: 3px;
  transition: width 0.3s;
}

.board {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  grid-template-rows: repeat(6, 1fr);
  gap: 3px;
  background: white;
  border-radius: 16px;
  padding: 6px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
}

.cell {
  aspect-ratio: 1;
  border-radius: 10px;
  overflow: hidden;
  position: relative;
  transition: transform 0.2s, opacity 0.2s;
  box-shadow: 0 1px 4px rgba(0,0,0,0.12);
}

.cell img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

.cell.swapping {
  opacity: 0.5;
  transform: scale(0.9);
}

.cell.removing {
  opacity: 0;
  transform: scale(0.5);
}

.cell.new-cell {
  animation: dropIn 0.3s ease-in;
}

@keyframes dropIn {
  from { transform: translateY(-60px); opacity: 0; }
  to { transform: translateY(0); opacity: 1; }
}

.combo-popup {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 32px;
  font-weight: 900;
  color: #ff6b6b;
  text-shadow: 0 2px 8px rgba(255,107,107,0.4);
  pointer-events: none;
  z-index: 10;
  animation: comboBounce 0.6s ease-out forwards;
}

@keyframes comboBounce {
  0% { opacity: 0; transform: translate(-50%, -50%) scale(0.5); }
  50% { opacity: 1; transform: translate(-50%, -50%) scale(1.2); }
  100% { opacity: 0; transform: translate(-50%, -60%) scale(1); }
}

.overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100;
}

.modal {
  background: white;
  border-radius: 20px;
  padding: 32px 24px;
  text-align: center;
  width: 280px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.2);
}

.modal .icon { font-size: 48px; margin-bottom: 12px; }
.modal .title { font-size: 22px; font-weight: 700; color: #333; margin-bottom: 8px; }
.modal .subtitle { font-size: 14px; color: #888; margin-bottom: 20px; }
.modal button {
  width: 100%;
  padding: 12px;
  border: none;
  border-radius: 12px;
  font-size: 16px;
  font-weight: 600;
  color: white;
  background: linear-gradient(135deg, #667eea, #764ba2);
  cursor: pointer;
}
</style>
</head>
<body>
<div class="game-container">
  <div class="status-bar">
    <div class="level-title" id="levelTitle">第 1 关</div>
    <div class="info-row">
      <span id="scoreText">分数: 0 / 100</span>
      <span id="stepsText">步数: 20</span>
    </div>
    <div class="steps-bar">
      <div class="steps-fill" id="stepsFill" style="width:100%"></div>
    </div>
  </div>
  <div class="board" id="board"></div>
</div>
<div id="overlayContainer"></div>
<div id="comboContainer"></div>

<script>
// ===== 游戏状态 =====
const ROWS = 6, COLS = 6;
const IMAGES = ['test1.jpg', 'test2.jpg', 'test3.jpg'];

let board = [];
let state = {
  level: 1,
  score: 0,
  target: 100,
  stepsLeft: 20,
  totalSteps: 20,
  comboCount: 0,
  gameStatus: 'idle',
};

// ===== 关卡配置 =====
function getLevelConfig(n) {
  if (n <= 9) {
    const targets = [100, 200, 350, 500, 700, 900, 1200, 1500, 1800];
    const steps = [20, 20, 18, 18, 16, 16, 15, 15, 14];
    return { target: targets[n - 1], steps: steps[n - 1] };
  }
  const target = 2000 + (n - 10) * 300;
  const steps = Math.max(10, 14 - Math.floor(n / 5));
  return { target, steps };
}

// ===== 棋盘初始化 =====
function initBoard() {
  board = [];
  for (let r = 0; r < ROWS; r++) {
    board[r] = [];
    for (let c = 0; c < COLS; c++) {
      let type;
      do {
        type = Math.floor(Math.random() * 3);
      } while (
        (c >= 2 && board[r][c - 1] === type && board[r][c - 2] === type) ||
        (r >= 2 && board[r - 1][c] === type && board[r - 2][c] === type)
      );
      board[r][c] = type;
    }
  }
}

function resetLevel() {
  const cfg = getLevelConfig(state.level);
  state.score = 0;
  state.target = cfg.target;
  state.stepsLeft = cfg.steps;
  state.totalSteps = cfg.steps;
  state.comboCount = 0;
  state.gameStatus = 'idle';
  initBoard();
  updateUI();
  renderBoard();
}
</script>
</body>
</html>
```

- [ ] **Step 2: 用浏览器打开验证**

在项目目录打开 `index.html`，确认：棋盘 6×6 显示（无图片，因为 renderBoard 还没写）、顶部状态栏显示"第 1 关 / 分数: 0 / 100 / 步数: 20"。

---

### Task 2: 棋盘渲染

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 在 `<script>` 末尾（`</script>` 前）添加 renderBoard 和 updateUI**

```js
function renderBoard() {
  const boardEl = document.getElementById('board');
  boardEl.innerHTML = '';
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      const cell = document.createElement('div');
      cell.className = 'cell';
      cell.dataset.row = r;
      cell.dataset.col = c;
      const img = document.createElement('img');
      img.src = IMAGES[board[r][c]];
      img.draggable = false;
      cell.appendChild(img);
      boardEl.appendChild(cell);
    }
  }
}

function updateUI() {
  document.getElementById('levelTitle').textContent = `第 ${state.level} 关`;
  document.getElementById('scoreText').textContent = `分数: ${state.score} / ${state.target}`;
  document.getElementById('stepsText').textContent = `步数: ${state.stepsLeft}`;
  const pct = (state.stepsLeft / state.totalSteps) * 100;
  document.getElementById('stepsFill').style.width = pct + '%';
}

function updateScore(add) {
  state.score += add;
  updateUI();
}
```

- [ ] **Step 2: 在 `<script>` 末尾添加启动代码**

```js
// ===== 启动 =====
resetLevel();
```

- [ ] **Step 3: 浏览器刷新验证**

确认棋盘显示 6×6 格子，三张照片随机填充，无初始三连。状态栏正常显示。

---

### Task 3: 触摸拖拽交换

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 在 renderBoard 函数之后、updateUI 之前，添加触摸处理代码**

```js
let touchStartRow = -1, touchStartCol = -1;
let touchStartX = 0, touchStartY = 0;
let isDragging = false;

function getCellFromTouch(touch) {
  const el = document.elementFromPoint(touch.clientX, touch.clientY);
  if (!el) return null;
  const cell = el.closest('.cell');
  if (!cell) return null;
  return { r: +cell.dataset.row, c: +cell.dataset.col };
}

function handleTouchStart(e) {
  if (state.gameStatus !== 'idle') return;
  const touch = e.touches[0];
  const cell = getCellFromTouch(touch);
  if (!cell) return;
  touchStartRow = cell.r;
  touchStartCol = cell.c;
  touchStartX = touch.clientX;
  touchStartY = touch.clientY;
  isDragging = true;
}

function handleTouchMove(e) {
  e.preventDefault();
}

function handleTouchEnd(e) {
  if (!isDragging || state.gameStatus !== 'idle') return;
  isDragging = false;
  const touch = e.changedTouches[0];
  const dx = touch.clientX - touchStartX;
  const dy = touch.clientY - touchStartY;
  const threshold = 20;

  if (Math.abs(dx) < threshold && Math.abs(dy) < threshold) return;

  let dr = 0, dc = 0;
  if (Math.abs(dx) > Math.abs(dy)) {
    dc = dx > 0 ? 1 : -1;
  } else {
    dr = dy > 0 ? 1 : -1;
  }

  const r1 = touchStartRow, c1 = touchStartCol;
  const r2 = r1 + dr, c2 = c1 + dc;
  if (r2 < 0 || r2 >= ROWS || c2 < 0 || c2 >= COLS) return;

  executeSwap(r1, c1, r2, c2);
}

function swap(r1, c1, r2, c2) {
  const tmp = board[r1][c1];
  board[r1][c1] = board[r2][c2];
  board[r2][c2] = tmp;
}
```

- [ ] **Step 2: 在 renderBoard 的 cell 循环中，给 cell 添加事件监听**

找到 `boardEl.appendChild(cell);`，在其后添加：

```js
      cell.addEventListener('touchstart', handleTouchStart, { passive: true });
```

- [ ] **Step 3: 在 board 元素上绑定 touchmove 和 touchend**

在 `renderBoard` 末尾的 `}` 之后添加：

```js
  document.getElementById('board').addEventListener('touchmove', handleTouchMove, { passive: false });
  document.getElementById('board').addEventListener('touchend', handleTouchEnd);
```

**注意：** touchmove/touchend 只需绑定一次，不要每次 renderBoard 都重复绑定。应该移到 renderBoard 外面，或用标志位控制。修改如下——将这两行移到启动代码之前：

```js
// ===== 启动 =====
document.getElementById('board').addEventListener('touchmove', handleTouchMove, { passive: false });
document.getElementById('board').addEventListener('touchend', handleTouchEnd);
resetLevel();
```

- [ ] **Step 4: 添加 executeSwap 入口函数（占位，后续任务完善）**

```js
async function executeSwap(r1, c1, r2, c2) {
  state.gameStatus = 'swapping';
  swap(r1, c1, r2, c2);
  renderBoard();
  // Task 4-6 将在此处添加消除判定逻辑
  state.gameStatus = 'idle';
}
```

- [ ] **Step 5: 浏览器验证**

在手机上或用 Chrome DevTools 模拟触摸，滑动方块应能交换位置。

---

### Task 4: 消除检测 + 计分 + 重力 + 填充

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 在 swap 函数之后添加 findMatches**

```js
function findMatches() {
  const matched = new Set();

  // 横向扫描
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS - 2; c++) {
      const type = board[r][c];
      if (type === -1) continue;
      let len = 1;
      while (c + len < COLS && board[r][c + len] === type) len++;
      if (len >= 3) {
        for (let i = 0; i < len; i++) matched.add(r * COLS + (c + i));
        c += len - 1;
      }
    }
  }

  // 纵向扫描
  for (let c = 0; c < COLS; c++) {
    for (let r = 0; r < ROWS - 2; r++) {
      const type = board[r][c];
      if (type === -1) continue;
      let len = 1;
      while (r + len < ROWS && board[r + len][c] === type) len++;
      if (len >= 3) {
        for (let i = 0; i < len; i++) matched.add((r + i) * COLS + c);
        r += len - 1;
      }
    }
  }

  return [...matched].map(idx => ({ r: Math.floor(idx / COLS), c: idx % COLS }));
}
```

- [ ] **Step 2: 添加 groupMatches 函数**

```js
function groupMatches(matchedCells) {
  const groups = [];
  const visited = new Set();

  for (const { r, c } of matchedCells) {
    const key = r * COLS + c;
    if (visited.has(key)) continue;

    const type = board[r][c];
    const group = [];
    const queue = [{ r, c }];
    visited.add(key);

    while (queue.length > 0) {
      const curr = queue.shift();
      group.push(curr);
      for (const [dr, dc] of [[-1, 0], [1, 0], [0, -1], [0, 1]]) {
        const nr = curr.r + dr, nc = curr.c + dc;
        const nk = nr * COLS + nc;
        if (nr >= 0 && nr < ROWS && nc >= 0 && nc < COLS &&
            !visited.has(nk) && board[nr][nc] === type &&
            matchedCells.some(m => m.r === nr && m.c === nc)) {
          visited.add(nk);
          queue.push({ r: nr, c: nc });
        }
      }
    }
    groups.push(group);
  }
  return groups;
}
```

- [ ] **Step 3: 添加 applyGravity 和 fillBoard**

```js
function applyGravity() {
  for (let c = 0; c < COLS; c++) {
    const column = [];
    for (let r = ROWS - 1; r >= 0; r--) {
      if (board[r][c] !== -1) column.push(board[r][c]);
    }
    for (let r = ROWS - 1; r >= 0; r--) {
      const idx = ROWS - 1 - r;
      board[r][c] = idx < column.length ? column[idx] : -1;
    }
  }
}

function fillBoard() {
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      if (board[r][c] === -1) {
        board[r][c] = Math.floor(Math.random() * 3);
      }
    }
  }
}
```

- [ ] **Step 4: 添加 sleep 工具函数和 showCombo 动画**

```js
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

function showCombo(n) {
  const container = document.getElementById('comboContainer');
  const el = document.createElement('div');
  el.className = 'combo-popup';
  el.textContent = `COMBO ×${n}!`;
  container.appendChild(el);
  setTimeout(() => el.remove(), 600);
}
```

- [ ] **Step 5: 重写 executeSwap 为完整消除流程**

替换之前的占位 `executeSwap`：

```js
function getScoreForGroup(group) {
  const size = group.length;
  if (size >= 6) return 80;
  if (size === 5) return 40;
  if (size === 4) return 20;
  return 10;
}

async function executeSwap(r1, c1, r2, c2) {
  state.gameStatus = 'swapping';
  swap(r1, c1, r2, c2);
  renderBoard();

  const firstMatches = findMatches();
  if (firstMatches.length === 0) {
    // 无消除 → 换回
    await sleep(150);
    swap(r1, c1, r2, c2);
    renderBoard();
    state.stepsLeft--;
    updateUI();
    state.gameStatus = 'idle';
    checkGameState();
    return;
  }

  // 连击循环
  state.stepsLeft--;
  let combo = 0;

  while (true) {
    const matches = findMatches();
    if (matches.length === 0) break;

    combo++;
    const groups = groupMatches(matches);
    let turnScore = 0;
    for (const g of groups) turnScore += getScoreForGroup(g);
    turnScore *= combo;
    updateScore(turnScore);

    if (combo > 1) showCombo(combo);

    // 消除动画
    markRemovingCells(matches);
    await sleep(200);

    // 清除
    for (const { r, c } of matches) board[r][c] = -1;

    applyGravity();
    fillBoard();
    renderBoard();
    markNewCells();
    await sleep(300);
  }

  updateUI();
  state.gameStatus = 'idle';
  checkGameState();
}

function markRemovingCells(matches) {
  const cells = document.querySelectorAll('.cell');
  for (const { r, c } of matches) {
    const idx = r * COLS + c;
    if (idx < cells.length) cells[idx].classList.add('removing');
  }
}

function markNewCells() {
  // 所有格子标记为新（简化处理：顶部填充的看起来都是从上方落入）
  // 这里只需去掉 removing 类
  const cells = document.querySelectorAll('.cell');
  cells.forEach(c => c.classList.remove('removing'));
}
```

- [ ] **Step 6: 浏览器验证**

滑动交换两个相邻方块，应看到：消除动画 → 下落填充 → 分数更新 → 步数减1。如果首次交换无消除，方块应自动换回。

---

### Task 5: 死局检测 + 游戏胜负判定

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 添加 checkDeadlock 函数**

```js
function checkDeadlock() {
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      // 尝试向右交换
      if (c < COLS - 1) {
        swap(r, c, r, c + 1);
        const m = findMatches();
        swap(r, c, r, c + 1);
        if (m.length > 0) return false;
      }
      // 尝试向下交换
      if (r < ROWS - 1) {
        swap(r, c, r + 1, c);
        const m = findMatches();
        swap(r, c, r + 1, c);
        if (m.length > 0) return false;
      }
    }
  }
  return true;
}
```

- [ ] **Step 2: 添加 checkGameState 和弹窗函数**

```js
function showOverlay(icon, title, subtitle, btnText, onClick) {
  const container = document.getElementById('overlayContainer');
  container.innerHTML = `
    <div class="overlay">
      <div class="modal">
        <div class="icon">${icon}</div>
        <div class="title">${title}</div>
        <div class="subtitle">${subtitle}</div>
        <button id="modalBtn">${btnText}</button>
      </div>
    </div>
  `;
  document.getElementById('modalBtn').addEventListener('click', () => {
    container.innerHTML = '';
    onClick();
  });
}

function checkGameState() {
  if (state.score >= state.target) {
    state.gameStatus = 'win';
    const nextLevel = state.level + 1;
    showOverlay('🎉', '恭喜过关！', `第 ${state.level} 关完成`, '下一关', () => {
      state.level = nextLevel;
      resetLevel();
    });
    return;
  }

  if (state.stepsLeft <= 0) {
    state.gameStatus = 'lose';
    showOverlay('😢', '步数用完', `得分: ${state.score} / ${state.target}`, '重新开始', () => {
      state.level = 1;
      resetLevel();
    });
    return;
  }

  if (checkDeadlock()) {
    state.gameStatus = 'lose';
    showOverlay('🔒', '死局！', `无可消除的方块，得分: ${state.score}`, '重新开始', () => {
      state.level = 1;
      resetLevel();
    });
    return;
  }
}
```

- [ ] **Step 3: 修改 executeSwap 中的无消除分支**

确保无消除交换换回后也调用 `checkGameState`（已在上一步代码中包含）。

- [ ] **Step 4: 浏览器验证**

1. 正常消除直到过关 → 弹窗显示"恭喜过关"，点"下一关"进入第2关
2. 步数用完未达标 → 弹窗显示"步数用完"，点"重新开始"回到第1关
3. 出现死局时 → 弹窗显示"死局"

---

### Task 6: 动画打磨 + 最终验收

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 添加拖拽选中高亮效果**

修改 `handleTouchStart`，在函数末尾添加选中高亮：

```js
  document.querySelectorAll('.cell').forEach(c => c.classList.remove('selected'));
  const cellEl = document.querySelector(`.cell[data-row="${cell.r}"][data-col="${cell.c}"]`);
  if (cellEl) cellEl.classList.add('selected');
```

在 CSS 的 `.cell` 规则之后添加：

```css
.cell.selected {
  transform: scale(1.08);
  box-shadow: 0 0 0 3px #667eea, 0 2px 8px rgba(0,0,0,0.15);
  z-index: 2;
}
```

- [ ] **Step 2: 动画细节调整**

在 CSS 中将 `.cell.removing` 和 `.cell` 的 transition 优化：

```css
.cell {
  /* ...原有属性... */
  transition: transform 0.2s ease, opacity 0.2s ease, box-shadow 0.2s ease;
}
```

- [ ] **Step 3: 添加防止双击缩放和长按菜单**

在 CSS body 规则中确认已有 `touch-action: none` 和 `user-select: none`。

在 board 上添加：

```css
.board {
  touch-action: none;
}
```

- [ ] **Step 4: 全面验收测试**

在手机上打开 `index.html`（可通过微信发送或本地浏览器），逐项验证：

| 测试项 | 预期结果 |
|--------|---------|
| 棋盘初始 | 6×6，3种图片随机分布，无初始三连 |
| 手指滑动交换 | 相邻方块互换位置 |
| 无效交换 | 无消除时自动换回，扣1步 |
| 三连消 | 3个相同水平/垂直排列消除，得10分 |
| 连击 | 消除后下落自动形成新消除，弹出 COMBO 文字，分数倍增 |
| 过关 | 达到目标分 → 弹窗 → 点按钮进下一关 |
| 步数耗尽 | 步数用完未达标 → 失败弹窗 → 回到第1关 |
| 死局 | 无相邻可消除对 → 死局弹窗 |
| 关卡递增 | 第2关目标200/20步，第10关后自动生成 |
