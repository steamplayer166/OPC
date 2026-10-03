# 「进组」· 本科生科研入门与导师匹配助手

把「我想做科研但不知道找谁、方向看不懂、不敢开口」压缩成一次浏览 + 一次对话，产出可执行的行动计划。

- **在线演示**：https://jinzu-demo.app.workbuddy.host/
- **产品形态**：Web 应用（纯前端单文件 + WorkBuddy 云服务），免登录可体验
- **参赛定位**：大学生 OPC 创新 AI Agent 赛道 · AI 学习规划助手 / AI 校园知识服务助手

---

## 一、这是什么

三大功能模块：

1. **信息检索库** — 术语解释词典，覆盖「学科术语」（光电探测、拓扑绝缘体…）和「流程黑话」（优营、预推免、套磁…）两类，词条含本意、作用、时间窗口、相关概念、示例。
2. **流程介绍** — 推免全流程时间轴，术语高亮、就地展开。
3. **进组智能助手** — 六环动线：信息采集 → 官网入口 → 导师卡片墙 → 追问收敛 → 排序推荐 → 行动交付（邮件草稿 + 时间清单 + 行动计划表）。

**核心设计**：对话只占很小一部分。槽位填满后界面切换为结构化结果页，最终交付物是可打印、可分享的报告。

---

## 二、技术栈

| 层 | 用什么 |
|---|---|
| 前端 | 原生 HTML / CSS / JavaScript，**单文件、零构建、零 npm 依赖** |
| 智能体 LLM | WorkBuddy 云服务「免密钥大模型」（`cloud.llm`），用户无需自带 API Key |
| 数据 | 术语/导师数据内置数组兜底 + 云数据库读取（见「已知事项」） |
| 发布 | WorkBuddy 云服务（`workbuddy_sites_deploy`） |

> ⚠️ 本项目**没有后端文件**，没有 `.env`、没有 `adapter.py`、没有需要配置的 API Key。智能体逻辑全部在 `index.html` 的 JavaScript 里。

---

## 三、文件结构

```
jinzu-demo/
├── index.html    # 全部代码（页面 + 样式 + 逻辑 + 云服务接入）
└── README.md     # 本文件
```

---

## 四、如何本地预览

直接用浏览器打开 `index.html` 即可（双击）。

> 说明：云服务的「免密钥大模型」和「数据库」会因浏览器 Origin 校验在本地 `file://` 下失效，页面会自动降级到内置演示数据，不影响界面与流程展示。要体验真实 LLM 生成，请访问线上链接。

---

## 五、如何发布到公网

本项目通过 WorkBuddy 云服务发布（免备案、免服务器、免自定义域名）：

1. 在 WorkBuddy 中打开本项目目录；
2. 使用 `workbuddy_sites_deploy` 工具，复用应用 `applicationId: wbapp_IGbrNU7ygzuIYCP7p7gm88`（保持域名与云服务 Origin 不变）；
3. 发布后链接不变：`https://jinzu-demo.app.workbuddy.host/`。

---

## 六、智能体代码在哪（交接重点）

全部在 `index.html` 内，按函数名搜索即可定位：

| 函数 / 变量 | 作用 | 能否改 |
|---|---|---|
| `CLOUD_CONFIG` | 云服务 endpoint + publishableKey | ❌ **不要动**（动了 LLM/数据库全部失效） |
| `LLM_SYSTEM` | system 提示词护栏（不编造、不评价导师） | ✅ 可改，但需保留「不编造/不评价」两条 |
| `llmGenerate()` | LLM 调用封装：`models.list()` 选模型 → 流式调用 → 失败返回 `null` | ✅ 可改内部，**必须保留「失败返回 null」** |
| `enhanceReport()` | 推荐理由 + 邮件草稿的 prompt | ✅ 改这里调优生成效果 |
| `scheduleTermFallback()` | 术语搜索无结果时的大模型兜底解释 | ✅ 可改 prompt |
| `renderRank()` | 排序逻辑（当前为纯前端加权求和） | ✅ 改这里加多因子排序 |
| `TERMS` / `CANDIDATES` | 术语词典 / 导师演示数据 | ✅ 直接改数据 |
| `renderTerms` / `renderWall` / `renderReportHTML` 等 | 界面渲染 | ⚠️ 改样式可，改结构需同步前后端契约 |

### 规则兜底（务必保留）

`llmGenerate()` 失败会返回 `null`，所有调用方都做了回退：

- 报告页先**同步渲染**规则模板，再异步用 LLM 增强，失败则保持模板；
- 术语兜底失败则只显示「未找到匹配词条」。

**这条兜底是产品稳定性的底线**：模型超时/报错/配额不足时，网站必须仍能正常出结果，不能白屏。任何人改智能体都不得删除这些回退逻辑。

---

## 七、数据与合规

- 数据源：OpenAlex（CC0）、Semantic Scholar（ODC-BY，界面需署名回链）、院系官网公开页面。
- 合规红线：不采集学生个人信息、不做任何对导师的评价/打分、AI 生成内容标注「仅供参考，不替代导师本人意见」、检索不到的信息不写入、禁止编造。
- 当前导师与术语为**演示数据**（姓氏 + 老师称谓），正式使用需替换为经人工核验的真实数据。

---

## 八、已知事项

1. **云数据库表尚未建立**：建表用的管理工具（`workbuddy_cloudservice_db_exec_sql` 等）在当前环境未接入，`terms` / `candidates` 两张表的建表 SQL 已备好（见下），工具恢复后执行即可，前端读取代码无需改动、会自动从云库取数。
2. **建表 SQL（待执行）**：

```sql
CREATE TABLE terms (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  term TEXT NOT NULL, type TEXT NOT NULL,
  origin TEXT, plain TEXT, role TEXT, time_window TEXT, example TEXT,
  related TEXT[] NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE candidates (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name TEXT NOT NULL, tags TEXT[] NOT NULL DEFAULT '{}',
  works INT, last_active INT,
  match_score NUMERIC, activity_score NUMERIC, contact_score NUMERIC,
  source_url TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
ALTER TABLE terms ENABLE ROW LEVEL SECURITY;
GRANT SELECT ON TABLE public.terms TO authenticated, anon;
CREATE POLICY terms_read_all ON terms FOR SELECT TO authenticated, anon USING (true);
ALTER TABLE candidates ENABLE ROW LEVEL SECURITY;
GRANT SELECT ON TABLE public.candidates TO authenticated, anon;
CREATE POLICY candidates_read_all ON candidates FOR SELECT TO authenticated, anon USING (true);
```

---

## 九、如何协作修改

推荐走 Git（而非发密码）：

1. 将本目录推到 GitHub Organization 仓库；
2. 给队友**仓库写权限**；
3. 队友 `git clone` → 改 `index.html` 里的智能体函数 → 开 PR；
4. 负责人 review → 合并 → 重新发布（链接不变）。

---

## 十、版本

- 演示站 v1.0（含云服务免密钥大模型接入）
- 配套规划文档见项目主目录 `01_技术栈与落地方案.md` 至 `06_协作权限与接口开放设计.md`
