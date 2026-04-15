# ExifVisualizer 代码审查报告

日期：2026-04-15
范围：`index.html`

## 结论
当前实现可用，但存在 4 个需要优先处理的问题：
1. DOM XSS 风险（高）
2. 文件夹扫描可能漏文件（高）
3. EXIF 段读取缺边界检查（中）
4. 时间格式化策略不稳（低）

## 详细问题

### 1) DOM XSS 风险（高）
- 位置：`index.html:888`
- 现象：`row.innerHTML` 直接拼接 `fileName / folder / camera`。
- 风险：恶意文件名可注入 HTML/脚本并执行。
- 建议：改为 `createElement + textContent` 填充单元格，避免字符串拼接到 `innerHTML`。

### 2) 文件夹扫描可能漏文件（高）
- 位置：`index.html:452`
- 现象：`reader.readEntries()` 只调用一次。
- 风险：目录较大时只拿到第一批条目，导致图片漏读、统计偏差。
- 建议：循环调用 `readEntries`，直到返回空数组后结束。

### 3) EXIF 读取缺边界检查（中）
- 位置：`index.html:570` 附近
- 现象：读取 APP1/TIFF/IFD 偏移时未统一校验 `byteLength` 边界。
- 风险：损坏或截断图片可能触发 `DataView` 越界异常。
- 建议：在每次按偏移读取前做长度校验，不满足时提前返回空结果。

### 4) 时间格式化策略不稳（低）
- 位置：`index.html:905`
- 现象：`replace` 链依赖字符串形态，非标准 EXIF 时间可能被错误转换。
- 风险：表格和 CSV 的时间显示异常。
- 建议：使用正则匹配 `YYYY:MM:DD HH:MM:SS`，匹配失败返回原值或 `-`。

## 正向评价
- `index.html:514` 只读取头部 64KB，性能设计合理。
- `index.html:501` 并发处理文件，吞吐能力好。
- `index.html:809` 处理 DPR，Canvas 清晰度到位。

## 修复优先级建议
1. 先修 XSS 与文件夹漏读（影响正确性与安全性）
2. 再补 EXIF 边界检查（提升稳定性）
3. 最后优化时间格式化（提升展示一致性）
