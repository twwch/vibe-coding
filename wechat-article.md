# 我用 4 小时，让 AI 帮我从 0 开发了一个 AI 漫剧生成平台

---

## 什么是 Vibe Coding？

2025 年 2 月，前特斯拉 AI 总监、OpenAI 联合创始人 **Andrej Karpathy** 提出了一个颠覆性的编程理念——**Vibe Coding（氛围编程）**。

> *"There's a new kind of coding I call 'vibe coding', where you fully give in to the vibes, embrace exponentials, and forget that the code even exists."*

翻译过来就是：**你不再真正"写"代码，而是用自然语言描述你想要什么，让 AI 来生成代码。你只需要"感受氛围"，看看结果对不对，不对就继续用自然语言调整。**

这听起来很玄乎，但我真的用这种方式做到了。本文记录我从 0 开发一个 **AI 漫剧生成平台** 的完整过程。

---

## Vibe Coding 的核心特点

| 特点 | 传统编程 | Vibe Coding |
|------|----------|-------------|
| 输入方式 | 写代码 | 写自然语言 |
| 开发节奏 | 设计→编码→测试 | 描述→生成→看效果→调整 |
| 开发者角色 | 程序员 | 更像产品经理 |
| 最重要的东西 | 代码能力 | **文档和需求描述** |

**AI 时代下的编程：文档为王。**

---

## 工具选择：Claude Code

本项目使用 **Claude Code** 作为 AI 编程工具。配合以下 Skills 插件体系，实现了完整的 Vibe Coding 工作流。

---

## 必装 Skills 插件

### 1. superpowers —— 完整的 AI 开发工作流框架

这是整个实践的核心。superpowers 包含 14 个可组合技能，包括：
- `brainstorming`：头脑风暴
- `test-driven-development`：TDD 测试驱动开发
- `systematic-debugging`：系统化调试
- `subagent-driven-development`：子 Agent 驱动开发
- `code-review`：代码审查

强制执行"红-绿-重构"TDD 循环、四阶段调试方法论。

```
/plugin install superpowers@claude-plugins-official
```

---

### 2. frontend-design —— Anthropic 官方前端设计技能

专门指导 Claude 避免"AI 通用美学"，做出大胆的设计决策，注重排版、配色主题和动效微交互。特别适合 **React + Tailwind** 技术栈。

```
npx skills add https://github.com/anthropics/skills --skill frontend-design
```

---

### 3. planning-with-files —— Manus 风格持久化规划

将 Markdown 文件作为磁盘上的"工作记忆"，创建三个核心文件：
- `task_plan.md`：追踪阶段和进度
- `findings.md`：存储研究发现
- `progress.md`：会话日志和测试结果

适合需要超过 5 次工具调用的复杂多步骤任务。

```
npx skills add https://github.com/OthmanAdi/planning-with-files --skill planning-with-files
```

---

### 4. ui-ux-pro-max —— AI 设计智能技能

包含：
- 67+ UI 风格
- 161 配色方案
- 57 字体搭配
- 161 条行业特定设计规则
- 支持 13 个技术栈（React、Next.js、Vue、SwiftUI、Flutter 等）

```
npx skills add https://github.com/nextlevelbuilder/ui-ux-pro-max-skill --skill ui-ux-pro-max
```

---

### 5. best-minds —— 模拟顶级专家思维

不再问"你怎么想"，而是问"世界上谁最了解这个问题？他们会怎么说？"让 Claude 采用世界级专家视角来解决问题。

```
npx skills add https://github.com/brucexo/skills-collection --skill best-minds
```

---

### 6. notebooklm —— 直连 Google NotebookLM

让 Claude Code 直接与 Google NotebookLM 笔记本通信，获取基于来源的、带引用的回答。

```bash
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/PleasePrompto/notebooklm-skill notebooklm
```

---

### 安装效果

安装 Skills 后的效果：

![安装 Skills](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/skills/install.png)

![Skills 已安装](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/skills/installed.png)

---

## 项目实战：从 0 开发 AI 漫剧生成平台

### 第一步：技术选型（让 AI 帮你做决定）

对于从 0 开始的项目，我先大概想好了技术方向，然后让 AI 来生成完整的技术方案文档。

**我给 AI 的提示词：**

```
我想做一个 AI 漫剧生成平台，这是我大概的技术选型：
使用 nextjs 全栈开发，数据库使用 sqlite 快速完成集成
请你帮我完成技术选型，生成一个文档我 check 一下
```

AI 立刻开始生成技术选型方案：

![技术选型过程](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/%E6%8A%80%E6%9C%AF%E9%80%89%E5%9E%8B%201.png)

中途我根据自己的需要，直接在终端给 AI 反馈，让它调整方案：

![优化 AI 生成的 Plan](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/%E4%BC%98%E5%8C%96%20AI%20%E7%94%9F%E6%88%90%E7%9A%84%20plan.png)

最终生成完整的技术选型和系统设计文档：

![技术选型和系统设计](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/%E6%8A%80%E6%9C%AF%E9%80%89%E5%9E%8B%E5%92%8C%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1.png)

---

### 第二步：启动 AI 开发

技术文档写好后，直接 `@` 引用文档，让 AI 开始开发：

![启动开发](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/%E5%90%AF%E5%8A%A8%E5%BC%80%E5%8F%91.png)

AI 会先生成一个完整的执行计划：

![执行计划](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/plan.png)

然后 AI 会自动对刚生成的 Plan 进行 Review，有不合理的地方会自己进行二次修改：

![AI 自我修改 Plan](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/edit-plan.png)

确认 Plan 没问题后，开始执行：

![Plan 准备就绪](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/plan-ready.png)

---

### 第三步：Subagent 全自动开发

加载 `superpowers:subagent-driven-development` 技能，AI 开始全自动开发：

![开发中](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/under-devlepment.png)

遇到需要确认的操作，直接点击确认即可：

![确认执行](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/exec-confirm.png)

项目结构完成后，AI 自动开始补充功能，比如队列处理和数据库集成：

![队列和数据库](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/queue-db.png)

前端页面也全自动开发：

![前端开发](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/frontend.png)

**superpowers skill 的一个重要优势**：每完成一个功能就自动 commit 一次，即使 AI 把代码改坏了，也可以方便地回退：

![Git 提交记录](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/git-logs.png)

**全程约 45 分钟，初版系统开发完成：**

![初版代码完成](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/final-code-v1.png)

---

### 第四步：调试和 UI 优化

启动时遇到报错，不要慌，截图发给 AI 就行：

![Bug 截图](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/bug1.png)

![修复 Bug](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/fix-bug1.png)

修复成功，页面正常运行，但 UI 比较丑。一句话触发 UI 重设计：

```
UI 比较丑，请你使用 ui-ux-pro-max 重新设计各个环节的 UI 和交互逻辑，补齐一下切换语言的地方
```

![UI 优化过程](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/fix-ui.png)

经过 3 轮迭代优化，UI 焕然一新：

![优化后 UI 动图](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/ui.gif)

---

## 项目迭代：继续添加功能

### 模型配置设置页

初版用环境变量配置模型，太麻烦了。我给 AI 的提示词：

```
现在我们对模型配置进行改造，有一个设置页面，点击进入配置模型，
文本模型支持 openai 协议、gemini 协议，
图像模型支持 openai 协议、gemini 协议。
视频模型支持字节 seedance 协议和 google 协议。
用户可以配置多个模型供应商，默认通过 /model/list api 获取模型，
获取失败的时候，用户可以自己填写 model_id，将 key 保存在本地。
在使用模型的时候，用户可以选择自己勾选的模型。
请你计划整个开发流程，完成开发，界面的风格需要和现在的保持一致
```

![设置页](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/settings.png)

模型对话功能补齐：

![模型对话](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/model-chat.png)

---

### 流程优化

发现生成流程有些问题，给 AI 描述后，它生成优化路径：

![流程优化](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/%E6%B5%81%E7%A8%8B%E4%BC%98%E5%8C%96.png)

![优化路径](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/%E4%BC%98%E5%8C%96%E8%B7%AF%E5%BE%84.png)

图片生成 API 有问题，继续修复：

![图片生成修复](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/image-generate-fix.png)

分镜页面流程修复和优化后的效果：

![分镜流程修复](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/%E5%88%86%E9%95%9C%E6%B5%81%E7%A8%8B%E4%BF%AE%E5%A4%8D.png)

![分镜流程优化后](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/project/%E5%88%86%E9%95%9C%E6%B5%81%E7%A8%8B%E4%BC%98%E5%8C%96%E5%90%8E.png)

---

## 最终成品展示

### 列表页

![列表页](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/demo/list.png)

### 剧本生成

![剧本生成](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/demo/%E5%89%A7%E6%9C%AC%E7%94%9F%E6%88%90.png)

### 角色解析

![角色解析](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/demo/%E8%A7%92%E8%89%B2%E8%A7%A3%E6%9E%90.png)

### 分镜

![分镜](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/demo/%E5%88%86%E9%95%9C.png)

### 预览

![预览](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/demo/%E9%A2%84%E8%A7%88.png)

### 模型配置

![模型配置](https://ladr-1258957911.cos.ap-guangzhou.myqcloud.com/vibe-coding/images/demo/%E6%A8%A1%E5%9E%8B%E9%85%8D%E7%BD%AE.png)

---

## 总结：Vibe Coding 的正确姿势

经过这次实践，总结出 Vibe Coding 的几个关键点：

**1. 文档为王**：描述越清晰，AI 生成的代码质量越高。前期花时间写好需求文档，远比后期反复修 bug 划算。

**2. Skills 是倍增器**：不同的 Skills 组合能显著提升开发效率。superpowers 负责工程化，ui-ux-pro-max 负责设计，各司其职。

**3. 迭代式反馈**：遇到问题，截图 + 一句话描述，AI 能理解上下文并修复。不需要自己懂技术细节。

**4. 自动 Git 保护**：使用 subagent-driven-development 每个功能自动 commit，提供安全网，不怕 AI 改坏代码。

**5. 开发者角色转变**：从"写代码的人"变成"定义需求的人"。核心竞争力从语法记忆转向对产品和业务的深刻理解。

---

**整个项目代码已开源：**
https://github.com/twwch/AIComicBuilder

后续我的编程实践技巧会更新到这里：
https://github.com/twwch/vibe-coding

---

*你也想尝试 Vibe Coding？欢迎点赞转发，让更多人看到这种新的编程方式。*
