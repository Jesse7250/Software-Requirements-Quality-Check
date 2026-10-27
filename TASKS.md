# 软件需求质量审查系统 · 实现级任务清单

> 周期 2026-10-28 ~ 2026-11-19（约三周）｜ 4 人 ｜ 目标：8 大功能可运行 + 14 份文档

---

## 0. 技术选型与关键决策

| 项 | 选型 | 说明 |
|---|---|---|
| 后端框架 | FastAPI + Pydantic v2 | 异步，自动生成 OpenAPI 文档 |
| ORM / 迁移 | SQLAlchemy 2.0 + Alembic | 版本化建表 |
| 数据库 | MySQL 8 + Redis | Redis 存审查任务队列/缓存 |
| 向量库 | ChromaDB（嵌入式） | 轻量，Docker 内跑，存文本向量 |
| 向量化 | DeepSeek Embedding（备选 sentence-transformers `bge-small-zh`） | 中文语义 |
| LLM | DeepSeek（用 `openai` SDK，`base_url=https://api.deepseek.com`） | 兼容 OpenAI 接口 |
| 文档解析 | python-docx / openpyxl / markdown / pypdf | 多格式导入 |
| 前端 | Vue 3 + Vite 5 + TypeScript + Element Plus + Pinia + ECharts | |
| 测试 | pytest + pytest-asyncio + httpx | |
| 部署 | Docker Compose（mysql / redis / backend / frontend / chroma） | |

**关键决策（写代码前先定死）：**

1. **需求编号与版本分离**：编号 `REQ-001` 永久不变；每次「确认修改」往 `requirement_versions` 写一条新版本 `v1/v2/...`，支持 diff 与回滚。
2. **澄清问题与需求解耦**：`clarification_questions` 是独立表，**绝不直接改需求**；只有用户确认后才写回 `requirement`。
3. **审查分两层**：确定性规则（正则/词典/余弦相似度，快且可解释）先跑，LLM 语义审查（冲突/歧义/缺口）后跑，结果合并去重后落 `review_findings`。
4. **LLM Provider 抽象**：定义 `BaseLLM` 接口，`DeepSeekClient` 实现，带结果缓存与成本统计，避免重复调用烧 token。
5. **批量审查走异步**：先用 FastAPI `BackgroundTasks` 起步（三周内够用），Celery 列为可选增强。

---

## 1. 目录结构

```
backend/
├── app/
│   ├── main.py                 # FastAPI 入口，注册路由/中间件/CORS
│   ├── config.py               # pydantic-settings 读 .env
│   ├── database.py             # engine / SessionLocal / get_db 依赖
│   ├── models/                 # SQLAlchemy 模型（见第 2 节）
│   ├── schemas/                # Pydantic 请求/响应模型
│   ├── api/                    # 路由（见第 3 节）
│   ├── services/               # 业务逻辑
│   │   ├── requirement_service.py
│   │   ├── parser_service.py   # 文档解析
│   │   ├── rule_engine.py      # 确定性规则
│   │   ├── review_service.py   # AI 审查流水线
│   │   ├── gap_service.py      # 信息缺口
│   │   ├── testcase_service.py
│   │   ├── llm_client.py       # DeepSeek 封装
│   │   ├── embedding.py        # 向量化
│   │   ├── vector_store.py     # ChromaDB 封装
│   │   └── report_service.py   # 报告导出
│   ├── core/security.py        # JWT / bcrypt / 权限依赖
│   └── utils/
├── alembic/                    # 迁移脚本
├── data/ambiguous_words.txt    # 歧义词表
├── prompts/review_prompt.txt   # 审查 Prompt 模板
├── tests/                      # pytest
├── Dockerfile
└── requirements.txt

frontend/
├── src/
│   ├── main.ts / App.vue
│   ├── router/index.ts
│   ├── stores/                 # auth / project / requirement / review ...
│   ├── api/request.ts          # axios 封装（拦截器注入 JWT）
│   ├── api/modules/            # 各模块接口
│   ├── views/                  # 见第 5 节
│   └── components/
├── Dockerfile
└── package.json
```

---

## 2. 数据库表（17 张，含关键字段）

| 表 | 关键字段 |
|---|---|
| `users` | id, username(unique), password_hash, role(enum: author/reviewer/admin), created_at |
| `projects` | id, name, description, created_by(FK users), created_at |
| `requirements` | id, project_id(FK), code(REQ-001), title, description, status(draft/confirmed/rejected), priority, source_document_id, parent_id(自关联), created_by, created_at |
| `requirement_versions` | id, requirement_id(FK), version_no(int), title, description, change_type(create/update/confirm), changed_by, created_at |
| `documents` | id, project_id, filename, file_type(docx/xlsx/md/pdf), raw_path, status, uploaded_by, created_at |
| `document_chunks` | id, document_id(FK), chunk_index, content, embedding(vector) |
| `review_rules` | id, project_id, name, rule_type(ambiguity/conflict/duplicate/unverifiable), config(json), enabled(bool) |
| `review_tasks` | id, project_id, requirement_id, trigger_by, status(pending/running/done/failed), created_at |
| `review_findings` | id, task_id(FK), requirement_id(FK), finding_type, location, evidence, suggestion, severity(high/medium/low), status(pending/confirmed/ignored) |
| `clarification_questions` | id, requirement_id(FK), gap_type, question, suggestion, status(open/answered/closed) |
| `test_cases` | id, requirement_id(FK), title, precondition, steps, input_data, expected_result, priority |
| `traceability` | id, requirement_id(FK), test_case_id(FK) —— 需求↔用例多对多 |
| `defect_documents` | id, project_id, filename, raw_path, uploaded_by |
| `defects` | id, project_id, requirement_id, title, description, severity, status(open/fixed/closed), defect_document_id |
| `audit_logs` | id, user_id, action, target_type, target_id, detail(json), created_at |
| `notifications` | id, user_id, title, content, is_read, created_at |

---

## 3. 后端 API 清单（按模块）

| 模块 | 接口 |
|---|---|
| 认证 | `POST /api/auth/register`、`POST /api/auth/login`(返回 JWT)、`GET /api/auth/me` |
| 项目 | `GET/POST /api/projects`、`GET/PUT/DELETE /api/projects/{id}` |
| 需求 | `GET /api/projects/{pid}/requirements`、`POST /api/projects/{pid}/requirements`、`GET/PUT/DELETE /api/requirements/{id}`、`GET /api/requirements/{id}/versions`、`POST /api/requirements/{id}/confirm` |
| 文档 | `POST /api/projects/{pid}/documents`(上传)、`POST /api/documents/{id}/parse`(解析→生成草稿)、`GET /api/documents/{id}/chunks` |
| 规则 | `GET/POST /api/projects/{pid}/rules`、`PUT/DELETE /api/rules/{id}` |
| 审查 | `POST /api/reviews`(发起)、`GET /api/reviews/{id}`、`GET /api/reviews/{id}/findings`、`PUT /api/findings/{id}`(确认/忽略) |
| 澄清 | `GET/POST /api/clarifications`、`PUT /api/clarifications/{id}`(回答) |
| 测试用例 | `POST /api/requirements/{id}/testcases/generate`(LLM)、`GET/POST /api/testcases`、`PUT/DELETE /api/testcases/{id}` |
| 追踪 | `GET /api/projects/{pid}/traceability`、`GET /api/projects/{pid}/traceability/uncovered` |
| 缺陷 | `POST /api/projects/{pid}/defects/import`、`GET/POST /api/defects`、`GET /api/projects/{pid}/defects/acceptance` |
| 报告 | `GET /api/projects/{pid}/reports/export`(导出 xlsx/md) |

---

## 4. AI 审查流水线（核心，成员B 主责）

```
run_review(requirement, rules):
    1. 确定性规则（无 LLM，快）：
       rule_engine.check_ambiguity(text)     # 歧义词表 + 正则（可能/大概/尽快/等等/或）
       rule_engine.check_unverifiable(text)  # 缺量化指标（应<阈值、百分比、时间）
       rule_engine.check_duplicate(text)     # 与已有需求余弦相似度 > 0.85 判重
    2. 向量检索上下文：
       chunks = vector_store.search(text, top_k=5)   # 知识库 + 已有需求
    3. LLM 语义审查：
       prompt = render(prompts/review_prompt.txt, text=text, context=chunks, rules=rules)
       resp   = llm.chat(prompt, response_format={"type":"json_object"})
       findings += parse_json(resp)          # 校验后追加
    4. 合并去重（按 location+type）→ 落库 review_findings
```

**LLM 结构化输出 JSON Schema（写死在 Prompt 里）：**

```json
{
  "findings": [
    {
      "type": "ambiguity | conflict | duplicate | unverifiable | gap",
      "location": "第2段第3句",
      "evidence": "原文片段 + 依据规则或冲突需求编号",
      "suggestion": "改写后的建议文本",
      "severity": "high | medium | low"
    }
  ]
}
```

**Prompt 要点**：system 里声明角色、要求严格按 JSON 输出、要求每条问题必须给出「位置 + 原文依据 + 建议」三要素、禁止编造不存在的上下文。

**信息缺口（成员B）**：按需求模板完整性检测——每条需求是否缺 `前置条件 / 约束 / 验收标准 / 异常场景` 四个字段；缺哪个就生成一条 `clarification_question`，**不写回需求**。

---

## 5. 前端页面清单（成员D）

- `Login.vue`
- `layout/`（侧边栏 + 顶栏）
- `Dashboard.vue`（ECharts：问题分布、覆盖度、趋势）
- `requirement/RequirementList.vue`、`RequirementEdit.vue`、`RequirementImport.vue`、`RequirementVersions.vue`
- `rule/RuleList.vue`、`RuleEdit.vue`
- `review/ReviewLaunch.vue`、`ReviewResult.vue`（问题列表 + 确认/忽略按钮）
- `clarification/ClarificationList.vue`
- `testcase/TestCaseList.vue`
- `traceability/TraceabilityMatrix.vue`（矩阵表格，未覆盖需求红色高亮）
- `defect/DefectImport.vue`、`DefectAcceptance.vue`
- `report/ReportExport.vue`
- `admin/UserManage.vue`、`ProjectManage.vue`

Pinia stores：`auth / project / requirement / rule / review / testcase / traceability / defect`

---

## 6. 每人任务（具体到文件 / 接口 / 表 + 日期）

### 成员A（需求管理模块）

| 日期 | 任务 | 具体产出 |
|---|---|---|
| 11-04 ~ 11-06 | 需求 CRUD + 版本 | `models/requirement.py`、`models/requirement_versions.py`、`api/requirements.py`（CRUD）、编号生成 `REQ-{3位}`、`requirement_versions` 写版本 |
| 11-04 ~ 11-08 | 文档解析导入 | `services/parser_service.py`：docx(python-docx 按标题/编号拆条)、xlsx(openpyxl 每行一条)、md(按标题/列表拆条)；`api/documents.py`；解析结果生成 `requirements` 草稿(status=draft) |
| 11-06 ~ 11-12 | 确认修改流程 | `api/requirements.py` 的 `POST .../{id}/confirm`：状态机 draft→confirmed/rejected，确认时写 `requirement_versions`；`audit_logs` 记操作 |
| 11-09 ~ 11-12 | 与前端联调 | 配合成员D 调通需求管理 4 个页面 |
| 全程 | 文档 + 接口 | 可行性/开发计划/SRS/月报/总结（文档排期见第 8 节） |

### 成员B（AI 审查引擎）

| 日期 | 任务 | 具体产出 |
|---|---|---|
| 11-04 ~ 11-06 | 规则引擎 | `services/rule_engine.py`；`data/ambiguous_words.txt` 歧义词表；`check_ambiguity/check_unverifiable`（正则）；`check_duplicate`（余弦相似度 >0.85）；`models/review_rules.py` + `api/rules.py` |
| 11-05 ~ 11-08 | 向量检索 | `services/embedding.py`（DeepSeek Embedding 或 bge-small-zh）；`services/vector_store.py`（ChromaDB 封装）；`document_chunks` 切片（chunk=500 字，overlap=50） |
| 11-06 ~ 11-10 | DeepSeek 接入 | `services/llm_client.py`（openai SDK，`base_url=https://api.deepseek.com`，`BaseLLM` 抽象 + 缓存 + 成本统计） |
| 11-08 ~ 11-12 | AI 审查流水线 | `services/review_service.py`、`prompts/review_prompt.txt`、`api/reviews.py`、JSON Schema 校验、结果落 `review_findings` |
| 11-09 ~ 11-14 | 信息缺口 | `services/gap_service.py`（模板完整性：前置/约束/验收/异常 4 字段检测）、`api/clarifications.py`、`models/clarification_questions.py` |

### 成员C（后端基础 + 用例/追踪/缺陷）

| 日期 | 任务 | 具体产出 |
|---|---|---|
| 11-04 ~ 11-06 | 脚手架 + 数据库 | `backend/` 骨架、`requirements.txt`、`config.py`、`database.py`、`alembic/` 初始化、`docker-compose.yml`；建 17 张表 |
| 11-04 ~ 11-08 | 认证 / RBAC | `core/security.py`（JWT 签发/校验、bcrypt）、`api/auth.py`、`models/user.py`；权限依赖 `require_role(author/reviewer/admin)` |
| 11-06 ~ 11-10 | 测试用例生成 | `models/testcase.py`、`api/testcases.py`、`services/testcase_service.py`（LLM 生成：前置/步骤/输入/预期/优先级），关联 `requirement_id` |
| 11-08 ~ 11-12 | 追踪矩阵 | `models/traceability.py`、`api/traceability.py`；`/traceability/uncovered` 查无用例关联的需求 |
| 11-10 ~ 11-14 | 缺陷验收 + 报告 | `models/defect.py`、`api/defects.py`（导入/验收统计）、`services/report_service.py`（openpyxl 导出 xlsx/md） |

### 成员D（前端 + 测试 + 部署）

| 日期 | 任务 | 具体产出 |
|---|---|---|
| 11-04 ~ 11-06 | 前端脚手架 | Vite 建 `frontend/`；装 element-plus/pinia/vue-router/axios/echarts；`router/index.ts`、`stores/`、`api/request.ts`（axios 拦截器注入 JWT、统一错误处理） |
| 11-05 ~ 11-10 | 基础页面 | `Login.vue`、`layout/`、`Dashboard.vue`、`RequirementList/Edit/Import/Versions.vue` |
| 11-08 ~ 11-14 | 功能页面 | `RuleList/RuleEdit`、`ReviewLaunch/ReviewResult`、`ClarificationList`、`TestCaseList`、`TraceabilityMatrix`、`DefectImport/Acceptance`、`ReportExport`、`UserManage/ProjectManage` |
| 11-14 ~ 11-17 | 测试 + 部署 | `tests/`（pytest 接口测试，覆盖认证/需求/审查/追踪主流程）；`backend/Dockerfile`、`frontend/Dockerfile`、完善 `docker-compose.yml`、缺陷修复 |

---

## 7. 时间轴与依赖顺序

```
10-28~11-03  需求分析（SRS 定稿 + 审查规则清单，成员A/B）
11-04~11-08  设计 + 脚手架：成员C 建库、成员D 建前端、成员B 规则引擎
11-06~11-14  编码：后端(成员C) / 前端(成员D) / AI引擎(成员B) / 需求模块(成员A) 并行
11-09~11-17  测试：测试计划(11-09~12) → 执行(11-14~17) → 测试分析报告
11-15~11-19  收尾：用户/操作手册、部署、月报、总结、交付
```

**硬依赖（卡点）**：`SRS(11-03)` → `数据库设计(11-08)` → `后端 models` → `前端联调`；`规则引擎(11-06)` → `AI 审查流水线(11-08)` → `审查结果页面`。

---

## 8. 文档排期（简要，内容非重点）

| 文档 | 负责人 | 时间 |
|---|---|---|
| 可行性报告 / 开发计划 / SRS | 成员A | 10-28 ~ 11-03 |
| 数据要求 / 详细设计 / 数据库设计 | 成员C | 11-04 ~ 11-11 |
| 概要设计 | 成员B | 11-04 ~ 11-08 |
| 用户手册 / 操作手册 / 测试计划 / 测试分析报告 | 成员D | 11-09 ~ 11-18 |
| 模块开发卷宗 | 各模块负责人 | 11-09 ~ 11-17 |
| 月报 / 总结报告 | 成员A | 11-18 ~ 11-19 |

---

## 9. 协作规范

- 分支：`main`（稳定）+ 每人 `feature/<模块>`；合并由成员A 统一合入。
- 提交：`<type>(<scope>): <描述>`，type = feat/fix/docs/test/chore。
- 提交前本地 `pytest` 通过；`.env` 不入库，用 `.env.example` 占位。
- 每日 1 次 `git pull --rebase`，冲突当天解决。
