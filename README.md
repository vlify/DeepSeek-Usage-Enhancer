# DeepSeek Usage Enhancer

> **本仓库为 fork 修改版** —— 原项目: [Jmkwang/DeepSeek-Usage-Enhancer](https://github.com/Jmkwang/DeepSeek-Usage-Enhancer) (MIT)
> 修改者: vlify | 已适配新版平台 API 与页面, 发布至 GreasyFork
> 许可证: MIT (完整文本见 LICENSE 文件)

---

## 安装

1. 安装浏览器用户脚本管理器: [Tampermonkey](https://www.tampermonkey.net/) (Chrome / Edge / Firefox)
2. 二选一:
   - **GreasyFork 一键安装** (推荐): 在 [GreasyFork 搜索「DeepSeek Usage Enhancer」](https://greasyfork.org/zh-CN/scripts?q=DeepSeek+Usage+Enhancer) 或 [搜索「DeepSeek Daily Monitor」](https://greasyfork.org/zh-CN/scripts?q=DeepSeek+Daily+Monitor), 打开脚本页点「安装」
   - **GitHub 手动安装**: 打开 `DeepSeek-Usage-Enhancer.js` / `deepseek-usage-monitor.js`, 全选复制; 在管理器「添加新脚本」中粘贴保存
3. 打开 [DeepSeek 用量页面](https://platform.deepseek.com/usage), 脚本自动生效

---

## 新版平台适配

DeepSeek 开放平台于 2026-07 改版, 用量接口从 `/api/v0/usage/amount|cost` 改为
`/api/v0/usage/by_api_key/amount|cost`(按 API Key 拆分, `series[].buckets[]` 结构,
金额为字符串)。本 fork 已适配:

- 拦截新端点, 同时兼容旧端点
- 解析新结构 (`series[].buckets[]`, usage 对象字段), 同时兼容旧结构 (`days[].data[]`)
- 支持 `/api/v0/users/{id}` 形式的账户汇总, 兼容 `total_balance` / `topped_up_balance` / `granted_balance` 新字段
- "今日" 按本地时区计算 (新版桶的 `time` 为 Unix 秒)
- 悬浮面板版 (`deepseek-usage-monitor.js`) 完全自包含, 推荐使用
- 页面注入版 (`DeepSeek-Usage-Enhancer.js`) 自动识别新版/旧版页面布局

---

在 DeepSeek 开放平台用量页面补全**今日数据**——今日消费、请求次数、Token 用量、缓存命中率，并提供图表数字千分位格式化。

---

## 背景

打开 [platform.deepseek.com/usage](https://platform.deepseek.com/usage)，官方页面默认只展示**所选时间范围**的累计数据。想看"今天花了多少钱""今天请求了多少次""缓存命中率是多少"，需要手动点图表柱子、切换日期、自己算。

本项目的脚本帮你自动补齐这些信息。

---

## 两个版本

| 版本 | 文件 | 方式 |
|------|------|------|
| **页面注入版** | `DeepSeek-Usage-Enhancer.js` | 将今日数据注入页面布局 |
| **悬浮面板版** | `deepseek-usage-monitor.js` | 独立浮窗展示，可拖拽可折叠 |

两个版本功能相同，选一个安装即可。**功能比较**：

| 特性 | 页面注入版 | 悬浮面板版 |
|------|:---:|:---:|
| 今日消费 | ✓ | ✓ |
| 各模型请求数 / Token / 缓存命中率 | ✓ | ✓ |
| 图表悬停数字千分位 | ✓ | — |
| 自由拖拽 / 折叠 | — | ✓ |
| 数值点击切换短格式↔原始值 | — | ✓ |
| 不受页面 React 重绘影响 | 自动重注入 | 独立 DOM，天然不受影响 |

---

## 功能详情

### 页面注入版（`DeepSeek-Usage-Enhancer.js`）

安装后打开用量页面，脚本自动识别新版/旧版布局：

- **新版页面 (2026-07 改版后)**:
  - 统计行下方插入「今日消费 / 今日请求 / 今日Tokens / 缓存命中率」数据条
  - 每个模型名下插入今日明细行 (请求数 / Tokens / 缓存命中率 / 消费金额)
- **旧版页面**:
  - 「今日消费」卡片 — 紧挨着官方「本月消费」卡片
  - 今日/昨日请求次数 — 每个模型下追加
  - 今日总 Tokens + 缓存命中率
- **图表悬停数字千分位** — 鼠标悬停图表时，4 位以上数字自动加逗号分隔（如 `1234567` → `1,234,567`）。

> 页面是 React 渲染的，脚本持续监听 DOM 变化，重绘后自动重新注入。

### 悬浮面板版（`deepseek-usage-monitor.js`）

安装后页面右下角会出现一个深色浮窗面板：

- **账户余额** — 充值余额、本月消费、今日消费。
- **各模型详情块** — `deepseek-v4-pro` 和 `deepseek-v4-flash` 的请求数、总 Token、缓存命中/未命中/输出 Token、缓存命中率。
- **底部状态栏** — 汇总总 Token 和总请求数。
- **拖拽移动** — 拖动标题栏即可移动面板，位置自动保存到 localStorage。
- **折叠/展开** — 点击标题栏按钮收起面板。
- **数值切换** — 点击任意数字在短格式（`1.5K`）和原始值（`1,500`）之间切换。

<img width="2033" height="1754" alt="image" src="https://github.com/user-attachments/assets/105d5f99-ff93-47b4-b490-780269ed8a11" />

---

## 原理简介

两个脚本通过拦截页面发起的 API 请求获取原始数据：

| API | 用途 |
|-----|------|
| `GET /api/v0/users/get_user_summary` (或 `/api/v0/users/{id}`) | 账户余额与本月消费 |
| `GET /api/v0/usage/by_api_key/amount` | 各模型的 Token/请求用量 (新版平台) |
| `GET /api/v0/usage/by_api_key/cost` | 每日费用明细 (新版平台) |
| `GET /api/v0/usage/amount` / `usage/cost` | 旧版平台兼容 |

拦截方式：`window.fetch` 和 `XMLHttpRequest` 的双重 monkey-patch，确保覆盖所有请求方式。

数据完全在浏览器本地处理，不向任何第三方发送。

---

## 安全说明

- 纯本地运行，无远程通信，`@grant none`。
- 源代码未压缩混淆，可逐行审查。

---

## 更新日志 (本 fork)

### v1.5.0 (2026-08)
- 适配新版平台 by_api_key 用量接口 (`series[].buckets[]`, 金额为字符串), 兼容旧接口
- 新版页面 DOM 注入 (今日数据条 + 模型今日明细行), 自动识别新版/旧版布局
- 支持 `/api/v0/users/{id}` 账户汇总, 兼容 `total_balance` 等新字段
- 模型费用合并展示

### v1.3.0 (2026-08)
- 元数据规范化 (MIT 许可证、唯一 namespace), 为 GreasyFork 托管做准备

---

## 更新日志 (原项目)

### v1.2.0
- 页面注入版：长轮询间隔从 10s 缩短至 3s，页面重绘后响应更快。

### v1.1.0
- 新增悬浮面板版 `deepseek-usage-monitor.js`，支持拖拽、折叠、数值格式切换。

### v1.0.0
- 首次发布页面注入版：今日消费卡片、模型请求/Token/缓存命中率、图表千分位。

---

## 许可

MIT
