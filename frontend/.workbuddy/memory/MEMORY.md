# MEMORY.md - 股票投资助手前端项目

## 项目基础信息
- **项目路径**：`c:/develop/workspace/stock-assistant/frontend`
- **技术栈**：Vue 3 + Vite 5 + SCSS + vue-router 4 + Pinia
- **主题**：深色主题，涨红跌绿（中国A股规范）

## 启动方式
```
cd c:/develop/workspace/stock-assistant/frontend
node ./node_modules/vite/bin/vite.js --port 5173 --host
```
注意：Windows 下不能直接用 `node .\node_modules\.bin\vite`（是 bash 脚本），需用完整 JS 路径。

## 已实现的页面
| 路由 | 文件 | 功能 |
|------|------|------|
| /chat | ChatView.vue | AI 对话、历史会话 |
| /market | MarketView.vue | 行情列表、K线图 |
| /watchlist | WatchlistView.vue | 自选股卡片 |
| /portfolio | PortfolioView.vue | 投资组合管理 |
| /analysis | AnalysisView.vue | 技术分析K线 |

## 设计规范
- 背景：#0d1117 / 卡片：#161b22 / 边框：#30363d
- 涨：#f5222d（红）/ 跌：#52c41a（绿）/ 主色：#1677ff
- 字体：-apple-system / PingFang SC / Microsoft YaHei

## 后续计划
- 对接真实后端 API（/api 代理已配置，目标 localhost:8080）
- 集成 ECharts 替换 SVG 手写图表
- 添加 Pinia 状态管理 store
