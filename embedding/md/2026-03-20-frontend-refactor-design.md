# 前端重构设计文档：Vite 迁移 + 国际化 + 用户认证

## 概述

对 Embedding Check 项目进行三项改动：
1. 前端从 Next.js 迁移到 Vite + React + React Router v6（纯前端 SPA）
2. 接入 react-i18next 国际化（中文 + 英文）
3. 新增用户认证（JWT）+ 多用户数据隔离

## 变更范围

### 后端新增

#### User 模型

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer PK | |
| username | String(100), unique | 用户名 |
| password_hash | String(255) | bcrypt 哈希 |
| must_change_password | Boolean | 首次登录强制改密码（默认 True） |
| language | String(10) | 偏好语言（zh/en，默认 zh） |
| created_at | DateTime | |
| updated_at | DateTime | |

#### 认证 API

- `POST /api/auth/register` — 注册（username, password），返回 User 信息
- `POST /api/auth/login` — 登录，返回 `{access_token, must_change_password, language}`
- `POST /api/auth/change-password` — 改密码（old_password, new_password），设 must_change_password=false
- `PUT /api/auth/profile` — 更新用户偏好（language）

#### JWT 中间件

- 所有 `/api/*` 路由（除 `/api/auth/register` 和 `/api/auth/login`）需要 Bearer Token
- Token payload: `{user_id, username, exp}`，过期时间 24 小时
- FastAPI Depends 函数 `get_current_user` 解析 Token 返回 User 对象
- 新增配置项 `JWT_SECRET`（自动生成并持久化，同 ENCRYPTION_KEY 方案）

#### 数据隔离

以下表新增 `user_id` 字段（Integer FK → users.id, nullable=False）：
- `api_configs`
- `documents`（chunks 通过 document_id 间接隔离）
- `test_cases`
- `test_tasks`（test_results 通过 task_id 间接隔离）
- `scoring_dimensions`（is_default=True 的记录 user_id 可为 null，全局共享）

所有现有 API 查询增加 `filter(Model.user_id == current_user.id)` 条件。创建记录时自动设置 user_id。

对于 scoring_dimensions，查询时返回：用户自己创建的 + 全局默认的（user_id is null）。

#### 初始化

启动时自动创建默认账户：
- username: admin
- password: admin（bcrypt 哈希）
- must_change_password: True
- 如果 admin 用户已存在则跳过

#### 新增依赖

- `passlib[bcrypt]` — 密码哈希
- `PyJWT` — JWT 签发和验证

### 前端重建

#### 技术栈

| 项 | 选择 |
|----|------|
| 构建工具 | Vite |
| 框架 | React 19 + TypeScript |
| 路由 | React Router v6 |
| UI 组件 | Ant Design 6 |
| HTTP 客户端 | Axios |
| 国际化 | react-i18next + i18next |

#### 项目结构

```
frontend/
├── index.html
├── vite.config.ts
├── tsconfig.json
├── package.json
├── .env                          # VITE_API_URL=http://localhost:8000
└── src/
    ├── main.tsx                  # React 入口
    ├── App.tsx                   # 路由定义
    ├── router/
    │   ├── index.tsx             # 路由配置
    │   └── AuthGuard.tsx         # 路由守卫
    ├── pages/
    │   ├── Login.tsx
    │   ├── Register.tsx
    │   ├── ChangePassword.tsx
    │   ├── Dashboard.tsx
    │   ├── Settings.tsx
    │   ├── Documents.tsx
    │   ├── DocumentDetail.tsx
    │   ├── TestCases.tsx
    │   ├── Tests.tsx
    │   ├── TaskDetail.tsx
    │   ├── Results.tsx
    │   └── Scoring.tsx
    ├── components/
    │   └── AppLayout.tsx         # 侧边栏 + 顶栏（语言切换、用户菜单）
    ├── lib/
    │   ├── api.ts                # Axios + Token 拦截器 + 401 跳转
    │   ├── types.ts
    │   └── auth.ts               # Token 存取、登录状态
    ├── i18n/
    │   ├── index.ts              # i18next 初始化
    │   ├── zh.json               # 中文翻译
    │   └── en.json               # 英文翻译
    └── types/
        └── index.ts
```

#### 路由

| 路由 | 页面 | 认证 |
|------|------|------|
| `/login` | 登录 | 无需 |
| `/register` | 注册 | 无需 |
| `/change-password` | 改密码 | 需要 |
| `/` | 仪表盘 | 需要 |
| `/settings` | API 配置 | 需要 |
| `/documents` | 文档列表 | 需要 |
| `/documents/:id` | 文档详情 | 需要 |
| `/test-cases` | 测试用例 | 需要 |
| `/tests` | 测试 | 需要 |
| `/tests/:id` | 任务详情 | 需要 |
| `/results/:taskId` | 结果 | 需要 |
| `/scoring` | 评分维度 | 需要 |

#### AuthGuard 逻辑

1. 无 Token → 跳 `/login`
2. 有 Token 但 `must_change_password=true` → 跳 `/change-password`
3. 都通过 → 渲染子路由

#### API 层

- Axios 请求拦截器：自动带 `Authorization: Bearer <token>`
- Axios 响应拦截器：401 时清除 Token 并跳转 `/login`
- 新增 auth 相关 API 函数（login, register, changePassword, updateProfile）

#### 国际化

- react-i18next 初始化，语言检测顺序：用户偏好 → localStorage → 浏览器语言 → zh
- 翻译文件：扁平 JSON，按模块前缀分组（common.*, nav.*, auth.*, settings.*, documents.*, tests.*, scoring.*, results.*, dashboard.*）
- 语言切换：顶栏右侧下拉，切换时调用 `PUT /api/auth/profile` 保存偏好
- Ant Design locale 同步切换（zhCN / enUS）

#### 页面迁移

所有现有页面逻辑保持不变，仅做以下调整：
- 移除 `"use client"` 指令
- Next.js `useRouter` → React Router `useNavigate`
- Next.js `useParams` → React Router `useParams`
- Next.js `Link` → React Router `Link`
- 所有硬编码文案替换为 `t('key')` 调用

### 数据库迁移

通过 Alembic 生成迁移：
1. 创建 users 表
2. 为 api_configs, documents, test_cases, test_tasks, scoring_dimensions 添加 user_id 列
3. 迁移策略：新增列时 user_id 先设为 nullable，迁移后将现有数据的 user_id 设为 admin 用户的 id，最后改为 non-nullable（scoring_dimensions 的 is_default=True 记录保持 nullable）

## 补充说明

### 与原始设计的关系

本文档取代原始设计文档中"单用户本地工具，无认证层"的约束。项目现在是多用户应用。

### CORS 配置

Vite 开发服务器默认端口为 5173，后端 `CORS_ORIGINS` 默认值需更新为 `http://localhost:5173`。同时在 `vite.config.ts` 中配置开发代理 `/api → http://localhost:8000`。

### 全局默认评分维度行为

- 全局默认维度（user_id=null, is_default=True）对所有用户只读
- 用户不可编辑或删除全局默认维度
- 用户可创建自己的自定义维度（user_id 绑定到自己）

### Token 过期处理

JWT 过期时间 24 小时。长时间运行的批量任务不受 Token 过期影响（任务在后端线程池执行，不依赖前端 Token）。前端 Token 过期后，401 拦截器自动跳转登录页，用户重新登录即可继续查看结果。

### 文件访问隔离

上传文件以 UUID 命名存储，通过 DB 层面的 user_id 过滤控制访问。文档下载/查看 API 已有 user_id 校验，无法通过猜测 UUID 跨用户访问。

### 404 处理

路由表增加 `*` 通配路由，渲染 404 页面。

### 清理项

迁移时需删除的 Next.js 相关文件和依赖：
- `next.config.js`, `@ant-design/nextjs-registry`, `eslint-config-next`, `tailwindcss` 相关配置
- `src/app/` 目录结构（替换为 `src/pages/`）
- 移除 `src/types/` 目录（统一使用 `src/lib/types.ts`）
