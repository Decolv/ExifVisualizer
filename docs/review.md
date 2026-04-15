# 仓库检查报告

> 检查日期：2026/04/15  
> 仓库：ExifVisualizer

---

## 仓库概况

| 项目 | 内容 |
|------|------|
| 文件结构 | 极简，仅 `index.html` + `README.md` |
| 提交历史 | 3 个 commit（`Initial commit` → `init: 第一版` → `feat： readme加入`） |
| 分支状态 | `main`，干净，无未提交更改 |
| 技术栈 | 零依赖，纯原生 HTML/CSS/JS + Canvas API |

---

## 功能定位

一个**离线可用**的 JPG EXIF 批量读取与可视化工具，支持：

- 文件/文件夹选择、拖拽上传
- 读取光圈、快门、焦距、ISO、相机型号、拍摄时间
- Canvas 绘制分布柱状图
- 统计摘要 + CSV 导出

---

## 发现的问题

### 1. EXIF 解析有严重 bug（`readIFD`）

**位置**：`index.html:624-668`

TIFF/EXIF 规范规定：**当 `数据类型大小 × 数量 > 4 字节` 时，`value` 字段存储的是偏移量，而不是直接值**。但当前代码只在部分类型里做了 `numValues > 4` 或类似判断，且没有统一按 `typeSize * count` 计算。

**具体影响**：
- **SHORT (case 3)**：如果某个 SHORT 标签有 3 个值（共 6 字节），代码会直接把这 6 字节的前 2 字节当成值，导致读取错误。
- **LONG (case 4)**：如果有多个 LONG 值，代码只读第一个，把本应是偏移量的位置当成值。
- **BYTE (case 1)**：只读了 1 个字节，忽略多值情况。

> 日常常用标签（光圈、快门、ISO）多为单值，所以轻度使用可能碰不到，但解析器本身不严谨。

---

### 2. `formatDateTime` 逻辑错误

**位置**：`index.html:903-906`

```javascript
return dateTime.replace(/:/g, '-').replace(/-/g, ':', 2);
```

EXIF 日期格式为 `2024:01:15 10:30:00`。这段代码先把所有 `:` 换成 `-`，再把前 2 个 `-` 换回 `:`，最终得到 `2024:01:15 10-30-00`，格式是错误的。

---

### 3. 拖拽读取大文件夹可能丢文件（`readDirectory`）

**位置**：`index.html:442-460`

```javascript
const entries = await new Promise(resolve => reader.readEntries(resolve));
```

在 Chrome 中，`readEntries()` 一次最多返回 100 条。目录条目过多时需要**循环读取**直到为空，当前代码只读一次，大文件夹会丢文件。

---

### 4. 同时并行读取所有文件可能导致浏览器卡顿

**位置**：`index.html:480-511`

```javascript
const results = await Promise.all(files.map(processFile));
```

如果用户一次拖入几千张图，会同时发起几千个 `FileReader`，UI 可能卡死。建议做**分批并发控制**（比如每次 10~20 个）。

---

### 5. `ctx.roundRect` 兼容性

**位置**：`index.html:860,866`

Canvas `roundRect` 在部分旧版 Safari / Firefox 中不支持。现代浏览器基本 OK，但如果要更广兼容，建议做 fallback 或改用 `rect`。

---

## 建议修复优先级

| 优先级 | 问题 | 位置 |
|--------|------|------|
| 高 | `readIFD` 多值/偏移量判断错误 | `index.html:624-668` |
| 高 | `formatDateTime` 格式化错误 | `index.html:903-906` |
| 中 | `readDirectory` 大文件夹截断 | `index.html:442-460` |
| 中 | 并行读取无并发控制 | `index.html:480-511` |
| 低 | `roundRect` 兼容性 | `index.html:860,866` |
