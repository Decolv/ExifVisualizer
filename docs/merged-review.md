# ExifVisualizer 合并审查报告（KIMI + Clod）

日期：2026-04-15  
范围：[index.html](../index.html)

## 来源映射

- KIMI：[docs/review.md](review.md)
- Clod：[docs/code-review.md](code-review.md)

## 合并结论

当前实现可用，但存在安全性、解析正确性与稳定性问题。两份报告互补明显：

- KIMI 侧重 EXIF 解析正确性、性能与兼容性。
- Clod 侧重安全性与边界健壮性。

## 合并问题清单（按优先级）

| ID | 优先级 | 问题 | 代码位置 | 来源 |
|---|---|---|---|---|
| 1 | 高 | DOM XSS 风险：表格行使用 innerHTML 直接拼接文件名等字段 | [index.html#L888](../index.html#L888) | Clod（[code-review.md#L15](code-review.md#L15)） |
| 2 | 高 | EXIF IFD 解析错误：未统一按 typeSize × count 判断值内联或偏移，存在多值读取错误 | [index.html#L613](../index.html#L613) | KIMI（[review.md#L32](review.md#L32)） |
| 3 | 高 | 目录读取可能漏文件：readEntries 仅调用一次，大目录会被截断 | [index.html#L452](../index.html#L452) | KIMI + Clod（[review.md#L59](review.md#L59), [code-review.md#L21](code-review.md#L21)） |
| 4 | 中 | EXIF 读取缺少统一边界校验：损坏文件可能触发 DataView 越界 | [index.html#L570](../index.html#L570) | Clod（[code-review.md#L27](code-review.md#L27)） |
| 5 | 中 | 全量 Promise.all 无并发上限：大批量文件可能导致 UI 卡顿 | [index.html#L502](../index.html#L502) | KIMI（[review.md#L71](review.md#L71)） |
| 6 | 中 | 时间格式化策略不稳：当前 replace 链会产生异常格式 | [index.html#L903](../index.html#L903) | KIMI + Clod（[review.md#L47](review.md#L47), [code-review.md#L33](code-review.md#L33)） |
| 7 | 低 | Canvas roundRect 兼容性：旧版浏览器可能不支持 | [index.html#L860](../index.html#L860), [index.html#L866](../index.html#L866) | KIMI（[review.md#L83](review.md#L83)） |

## 差异与取舍说明

1. 关于时间格式化问题
- KIMI 定位为高优先级，Clod 定位为低优先级。
- 合并后建议定为中优先级：问题稳定复现且影响展示/导出正确性，但不直接导致安全或崩溃。

2. 关于并发处理
- Clod 给出正向评价，认为并发有吞吐优势。
- KIMI 指出无并发上限的风险。
- 合并结论：保留并发，但增加限流（例如 10 到 20 并发）以平衡吞吐和可用性。

## 统一修复顺序建议

1. 先修安全与正确性
- XSS（ID 1）
- readIFD 解析逻辑（ID 2）
- 目录漏读（ID 3）

2. 再补稳定性与性能
- EXIF 边界校验（ID 4）
- 并发限流（ID 5）

3. 最后处理展示与兼容性
- 时间格式化（ID 6）
- roundRect fallback（ID 7）

## 可直接落地的修复原则

- DOM 输出统一使用 textContent，不拼接不可信 HTML。
- EXIF 读取前后统一做偏移与长度校验，越界立即返回空值。
- 目录读取循环 readEntries，直到返回空数组。
- 文件处理使用并发池而非一次性 Promise.all。
- 时间格式化改为正则解析 YYYY:MM:DD HH:MM:SS，不匹配时返回原值或空值。