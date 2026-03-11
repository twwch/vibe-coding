# Claude Code AI 编排玩法合集

借助 AI 工具（Claude Code / Trae 2.0 Solo）大幅提升开发效率的真实案例整理。

---

## 文章列表

### [Claude Code AI 编排玩法：用 AI 移植 Python 项目到 Spring AI](claude-code-ai-编排玩法.md)

用 Claude Code 将一个 Python AI 服务（FastAPI）完整移植到 Java（Spring AI），历时约 2 天（投入 ~8h），完成 46 个 API 开发，自测 1 次通过率 100%，联调一次通过率 95.65%。

涵盖 10 种玩法：CLAUDE.md 注入上下文、需求文档驱动开发、IDE 报错直接粘贴、context7 查文档、SSE 协议适配、MCP 动态工具加载、批量改写 API、多项目并行开发等。

---

### [Claude Code 开发 my-mcp-server](claude-code-开发mcp-server.md)

用 Claude Code 从零开发一个完整的 MCP Server（Python），预估 2 天工作量，实际只用了 **36 分钟**完成开发，成功加载 40 余个工具，支持 Cursor、Cherry Studio 等客户端接入。

涵盖：需求文档编写方式、Claude Code 自动拆解任务、两轮 BUG 修复迭代、测试用例自动生成。

---

### [借助 Trae 2.0 Solo 实现需求生产 100%+ 提效](trae-solo-需求生产提效.md)

用 Trae 2.0 Solo 辅助产品经理生产需求文档，将需求生产效率提升 100%+。核心方法：让 AI 生成"给 AI 看的需求文档"，再交给 AI 开发。

涵盖：需求文档模板（基础版/完整版）、两个真实案例、批量出需求、组件级/页面级调整技巧。

---

### [AI Coding 耗时预估 — 各需求工时对比](ai-coding-耗时预估.md)

真实项目数据汇总，记录借助 AI 开发各需求的预估 vs 实际人天，量化 AI 提效效果。

| 角色 | 提效幅度 |
|------|---------|
| 产品经理 | **-66%** |
| 后端开发 | **-45%** |
| 前端开发 | **-30%** |
| 综合 | **-37%**（节省 36 人天）|
