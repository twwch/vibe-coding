# vibe-coding

## 社区支持
[https://linux.do/](https://linux.do/)

本项目用于记录我的 AI 编程过程

- 使用的工具 claude code

AI 时代下的编程：**文档为王**

## 什么是 Vibe Coding？

**Vibe Coding**（氛围编程）是由 Andrej Karpathy（前特斯拉 AI 总监、OpenAI 联合创始人）在 2025 年 2 月提出的概念。核心思想是：

> 你不再真正"写"代码，而是用自然语言描述你想要什么，让 AI 来生成代码。你只需要"感受氛围"（vibe），看看结果对不对，不对就继续用自然语言调整。

Karpathy 原话：

> *"There's a new kind of coding I call 'vibe coding', where you fully give in to the vibes, embrace exponentials, and forget that the code even exists."*

### Vibe Coding 的特点

- **自然语言驱动**：用人话描述需求，AI 生成代码
- **迭代式开发**：看效果 → 反馈 → 再调整，循环往复
- **关注"做什么"而非"怎么做"**：开发者更像产品经理，而非传统程序员
- **文档即代码**：好的需求描述和技术文档比写代码本身更重要

### SPEC开发规范

简单来说，SPEC 规范就是一套 “结构化的标准语言”，用来告诉机器（或 AI）一个功能长什么样、怎么调用、输入什么、输出什么。

- **核心定义**
  - 非执行代码： 它通常是一份 YAML 或 JSON 文件，描述了系统的行为。
  - 标准化格式： 最常用的是 OpenAPI Specification (OAS)。
  - 机器可理解： 因为有了规范，AI 才能在没有人工干预的情况下，读懂你的接口并自动调用。

- SPEC 的典型工作流 “先写规矩，再写实现。”
    - Drafting (起草)： 使用 YAML 编写一份符合 OpenAPI 规范的文档。
    - Validation (验证)： 使用工具（如 Swagger Editor）检查语法是否正确。
    - Binding (绑定)： 将这个 SPEC 导入到你的 Skill 平台。
    - Implementation (实现)： 编写后端逻辑代码，确保它返回的数据格式跟 SPEC 描述的一模一样。

### 本项目的实践方式

本项目使用 **Claude Code** 作为 AI 编程工具，通过以下方式实践 Vibe Coding：

1. **技术选型阶段**：用自然语言描述项目需求，让 AI 生成技术方案文档
2. **开发阶段**：通过 Skills 和 Subagent 驱动开发，AI 自动完成代码编写和 Git 提交
3. **迭代阶段**：截图 + 自然语言反馈，AI 理解上下文后自动修复和优化
4. **UI 优化阶段**：一句话指令触发 UI/UX 重新设计

本次实践的代码地址：https://github.com/twwch/AIComicBuilder


## 需要安装的 Skills

### 1. superpowers

- **GitHub**: https://github.com/obra/superpowers
- **介绍**: AI 编程代理的完整软件开发工作流框架，包含 14 个可组合技能（brainstorming、test-driven-development、systematic-debugging、subagent-driven-development、code-review 等）。强制执行"红-绿-重构" TDD 循环、四阶段调试方法论和苏格拉底式头脑风暴等最佳实践。
- **安装** (通过 Claude Code 官方插件市场，一键安装全部 14 个技能):
  ```
  /plugin install superpowers@claude-plugins-official
  ```

### 2. frontend-design

- **GitHub**: https://github.com/anthropics/skills/tree/main/skills/frontend-design
- **介绍**: Anthropic 官方出品的前端设计技能，用于创建独特的、生产级的前端界面。指导 Claude 避免"AI 通用美学"，做出大胆的设计决策，注重排版、配色主题和动效微交互，特别适合 React + Tailwind 技术栈。
- **安装**:
  ```
  npx skills add https://github.com/anthropics/skills --skill frontend-design
  ```

### 3. planning-with-files

- **GitHub**: https://github.com/OthmanAdi/planning-with-files
- **介绍**: 实现 Manus 风格的持久化 Markdown 规划工作流。将 Markdown 文件作为磁盘上的"工作记忆"，创建 `task_plan.md`（追踪阶段和进度）、`findings.md`（存储研究发现）和 `progress.md`（会话日志和测试结果），适合需要超过 5 次工具调用的复杂多步骤任务。
- **安装**:
  ```
  npx skills add https://github.com/OthmanAdi/planning-with-files --skill planning-with-files
  ```

### 4. skill-creator

- **GitHub**: https://github.com/anthropics/skills/tree/main/skills/skill-creator
- **介绍**: Anthropic 官方出品的技能创建元技能。提供 Create、Eval、Improve 和 Benchmark 四种操作模式，指导你完成 Claude Code 技能的完整开发生命周期，包括初始化、验证、打包和最佳实践。
- **安装**:
  ```
  npx skills add https://github.com/anthropics/skills --skill skill-creator
  ```

### 5. notebooklm

- **GitHub**: https://github.com/PleasePrompto/notebooklm-skill
- **介绍**: 让 Claude Code 直接与 Google NotebookLM 笔记本通信的技能。可以查询上传到 NotebookLM 的文档，获取基于来源的、带引用的回答。支持浏览器自动化、笔记本库管理和持久化身份认证。⚠️ 仅支持本地 Claude Code 安装，不支持 Web UI。
- **安装**:
  ```bash
  mkdir -p ~/.claude/skills
  cd ~/.claude/skills
  git clone https://github.com/PleasePrompto/notebooklm-skill notebooklm
  ```

### 6. best-minds

- **GitHub**: https://github.com/brucexo/skills-collection/tree/main/skills/best-minds
- **介绍**: 实现"模拟器思维"方法的技能——不再问"你怎么想"，而是问"世界上谁最了解这个问题？他们会怎么说？"。让 Claude 采用世界级专家的视角来回答问题或解决问题，提供更权威和深入的回答。
- **安装**:
  ```
  npx skills add https://github.com/brucexo/skills-collection --skill best-minds
  ```

### 7. find-skills

- **GitHub**: https://github.com/vercel-labs/skills
- **介绍**: Vercel 出品的技能发现工具。帮助用户发现和安装 Agent Skills，当你询问"如何做 X"、"有没有技能可以..."等问题时，会搜索可用的技能并推荐合适的选项。
- **安装**:
  ```
  npx skills add https://github.com/vercel-labs/skills --skill find-skills
  ```

### 8. ui-ux-pro-max

- **GitHub**: https://github.com/nextlevelbuilder/ui-ux-pro-max-skill
- **介绍**: AI 设计智能技能，用于构建专业 UI/UX。包含 67+ UI 风格、161 配色方案、57 字体搭配和 161 条行业特定设计规则，支持 13 个技术栈（React、Next.js、Vue、SwiftUI、Flutter 等）。会根据项目类型自动生成定制化的设计系统。
- **安装**:
  ```
  npx skills add https://github.com/nextlevelbuilder/ui-ux-pro-max-skill --skill ui-ux-pro-max
  ```

### 9. simplify (内置)

- **介绍**: Claude Code 内置技能，无需安装。审查最近修改的代码，从代码复用、代码质量和效率三个维度并行分析，汇总发现并自动修复问题。使用 `/simplify` 即可调用。


## 安装 Skills

![](images/skills/install.png)

![](images/skills/installed.png)


## 开启agent team模式

![](images/project/agent-teams.png)


## 项目初始化

本次演示， 从 0 开发一个 AI 漫剧生成平台

### 技术选型

对于从 0 开始的项目， 我们自己需要指定一下我们的技术选型，大概规划一下前后端，数据库的要求。然后让 AI 帮我进行技术选型。

以下是我给到 AI 的提示词

```
我想做一个 AI 漫剧生成平台， 这是我大概的技术选型：

使用 nextjs全栈开发， 数据库使用 sqlite 快速完成集成

请你帮我完成技术选型， 生成一个文档我 check 一下
```

![](./images/project/技术选型%201.png)

![](./images/project/优化%20AI%20生成的%20plan.png)

我们根据 AI 的交互提示， 根据自己的需要选择，中途遇到需要修改的地方，直接在终端给他说就行

![](./images/project/技术选型和系统设计.png)

最终生成的文件

[技术选型和系统设计](https://github.com/twwch/AIComicBuilder/blob/main/docs/superpowers/specs/2026-03-10-ai-comic-builder-design.md)


### AI开发

我们直接@刚刚生成的文件， 让 AI 帮我们开发

![](./images/project/启动开发.png)

AI 会帮我们生成一个很完整的执行计划

![](./images/project/plan.png)

这里 AI 会自动对他刚刚生成的 plan 进行 review。有不合理的地方，他会自己进行二次修改。

![](./images/project/edit-plan.png)


修改完成后， 我们可以开始执行开发了， 这里我预留了一个模型可以在界面上配置的需求没做， 后面演示如何在已有项目上做功能迭代。

![](./images/project/plan-ready.png)

这是他最后生成的执行计划，里面包含代码

[执行计划](https://github.com/twwch/AIComicBuilder/blob/main/docs/superpowers/plans/2026-03-10-ai-comic-builder.md)


加载`superpowers:subagent-driven-development`开始开发

![](./images/project/under-devlepment.png)

他会安装计划去完整整个项目的初始化和功能开发，过程中遇到我们需要确定的，就直接点就行。

![](./images/project/exec-confirm.png)


项目结构开发完成后， 他自动开始往我们的项目里面补功能， 比如，队列处理和数据库的集成

![](./images/project/queue-db.png)

前端页面开发
![](./images/project/frontend.png)


使用 `superpowers skill` 开发有一个好处， 他每一次开发完一个功能会 commit 一次，这样即使 ai 把代码改坏了，我们也可以方便点回退

![](./images/project/git-logs.png)

这个过程我们只需要等待。全程用了将近 45 分钟， 初版系统就开发完成了

![](./images/project/final-code-v1.png)


### 运行测试

启动就遇到个错误，不要慌， 修复他就好了

![](./images/project/bug1.png)

修复方案也很简单，截图发送
![](./images/project/fix-bug1.png)


修复成功后，页面正常了

![](./images/project/project-v1.png)

发现 UI 非常的丑。接下来优化 UI 设计， 引入一个新的`skill`, 输入如下指令

```
UI 比较丑， 请你使用 ui-ux-pro-max 重新设计各个环节的 UI和交互逻辑， 补齐一下切换语言的地方
```
这一步也可以在最开始的指令中加入

![](./images/project/fix-ui.png)

经过 3 轮优化， 改完后的样子

![](./images/project/ui.gif)


## 项目迭代


### 设置页开发

上面说到， 模型配置现在是通过环境变量进行配置的。 我们要改成用户自己配置自己的 key。 支持 openai 协议模型， gemini 协议模型， 还是 字节 seedance api 和 google 的视频 API，图像生成模型也支持 openai 协议和 gemini 协议 用户可以添加多个模型供应商。 在生成剧本调用模型的地方，可以自己选择配置的模型。 这里我给的提示词是

```
现在我们对模型配置进行改造， 有一个设置页面， 点击进入配置模型，文本模型支持 openai 协议， gemini 协议， 图像模型支持 openai 协议， gemini 协议。 视频模型支持字节 seedance 协议个 google 协议。用户可以配置多个模型供应商，默认通过 /model/list api 获取模型， 获取失败的时候，用户可以自己填写 model_id, 将 key 保存在本地. 在使用模型的时候， 用户可以选择自己的勾选的模型。 请你计划整个开发流程，完成开发，界面的风格需要和现在的保持一致
```

![](./images/project/settings.png)

模型对话功能补齐
![](./images/project/model-chat.png)


发现流程有一些问题， 我们来做一些调整

![](./images/project/流程优化.png)

AI 经过理解生成的执行计划

![](./images/project/优化路径.png)


图片生成 api 有问题
![](./images/project/image-generate-fix.png)


分镜页面需要一个子流程, 流程逻辑是分镜描述生成必须-- 首尾帧生成(可跳过, 需要结合分镜描述和人物三维图生成) --  视频生成（根据首尾帧图片 + 场景描述）生成视频, 每一个分镜都有单独生成的控制, 然后最上方都需要配置批量生成

![](./images/project/分镜流程修复.png)

AI 优化后的页面
![](./images/project/分镜流程优化后.png)

优化一些细节
![](./images/project/细节优化.png)


### 功能迭代

用户隔离
![](./images/project/功能迭代1-用户隔离.png)

模型拦截
![](./images/project/功能迭代2-模型点击拦截.png)

Veo 模型接入
![](./images/project/功能迭代3-Veo模型接入.png)


### 完整的系统截图

![列表页](./images/demo/list.png)

![剧本生成](./images/demo/剧本生成.png)

![角色解析](./images/demo/角色解析.png)

![分镜](./images/demo/分镜.png)

![预览](./images/demo/预览.png)

![模型配置](./images/demo/模型配置.png)


### Demo
https://www.bilibili.com/video/BV19rwVzUEeD/

https://www.bilibili.com/video/BV1RrwVzUE3x/

https://www.bilibili.com/video/BV15rwVzSEKZ/

https://www.bilibili.com/video/BV15kwiz7E6Q/

https://www.bilibili.com/video/BV1hTw1zAEgY/

最新版生成

[《拳魂·最后一回合》-seedance1.5](https://www.bilibili.com/video/BV1WGAPzrEs1/)

[《拳魂·最后一回合》-seedance2](https://www.bilibili.com/video/BV1fVAuzLEAX/)

