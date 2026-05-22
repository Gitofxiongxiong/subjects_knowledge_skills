---
name: dreamschool-subject-kb
description: 当 agent 需要使用 DreamSchool 学科知识库时使用；支持浏览真实学科资料目录、检索 OCR 教辅页面、读取知识点/考点/题目/答案解析，并可基于知识库生成讲义、教案、练习题、试卷、课堂例题、课后作业等 Markdown 教学材料。
---

# DreamSchool 学科知识库

使用这个 skill 访问 DreamSchool 学科知识库，并在需要时基于知识库生成教学材料。知识库是一个只读资料库，包含高中学科教辅、教材、复习讲义、专题训练、试卷等 OCR Markdown 页面，以及 OSS 托管的页面图片。

## 核心工作流

这个 skill 的核心能力是使用学科知识库，但不同任务的入口顺序不同。

资料问答类任务直接进入知识库检索；教学材料生成类任务必须先理解讲义、教案、练习或试卷的内容组织逻辑，再带着结构化需求去知识库找材料。不要在生成任务里一开始就盲目检索。

### 1. 判断任务类型

如果用户要查资料、回答知识点问题、定位考点、读取题目或解析，进入“资料检索流程”。

如果用户要生成讲义、教案、试卷、课堂例题、练习题、课后作业等 Markdown 教学内容，进入“教学材料生成流程”。

### 2. 资料检索流程

用于查找资料、回答问题、定位知识点和读取原始 OCR 页面。

1. 读取 [references/kb-api.md](references/kb-api.md)。
2. 按需读取 [references/kb-structure.md](references/kb-structure.md) 理解真实资料分布。
3. 不确定学科时调用 `/health`。
4. 广泛找资料时调用 `/overview` 和 `/list` 两层两层浏览目录。
5. 查具体知识点、考点、题型、章节名时调用 `/search/catalog`。
6. 用 `/read-pages` 或 `/material-pack` 读取命中页面和相邻页。
7. 回答时引用 `page_id`、`relative_path`、`relative_pdf`、`page_no` 等来源信息。

### 3. 教学材料生成流程

教学材料生成建立在知识库之上，但正确顺序是先确定材料组织逻辑，再检索资料。

1. 先读取 [references/teaching-material-generation.md](references/teaching-material-generation.md)，明确目标材料的结构、质量要求、例题闭环要求和交付格式。
2. 如果用户没有给出清晰小节，再读取 [references/section-planning.md](references/section-planning.md)，先规划小节。
3. 根据材料结构和小节规划，列出每一节需要的概念、公式、例题、练习、答案解析、图片或实验材料。
4. 再读取 [references/kb-api.md](references/kb-api.md)，必要时读取 [references/kb-structure.md](references/kb-structure.md)。
5. 针对每个小节的材料需求设计检索词，调用 `/search/catalog` 找资料。
6. 用 `/read-pages` 或 `/material-pack` 读取正文、例题、答案、解析和相邻页。
7. 严格基于已读取的知识库资料组织 Markdown。知识库缺失时，按生成规范处理，不要悄悄编造。
8. 交付前做质量检查：结构完整、例题闭环、公式可读、图片可访问、无 API Key、无内部路径泄露。

推荐顺序可以记为：

```text
生成规范 -> 小节规划 -> 材料需求 -> 知识库检索 -> 读页 -> 组织成稿 -> 质量检查
```

正式讲义和教案正文默认不暴露内部路径；但 agent 内部应保留 source manifest。用户要求溯源时，再单独列出来源。

## API 快速信息

公网 API：

```text
https://dreamschool.cloud/api/kb
```

除 `/health` 外，请求需要 API Key：

```http
X-API-Key: <KB_API_KEY>
```

或：

```http
Authorization: Bearer <KB_API_KEY>
```

不要在回答、日志、代码仓库或示例里暴露真实 key。脚本中从环境变量 `KB_API_KEY` 读取。

## 重要约束

- 不要调用 `kb_search_text`。当前没有全文检索接口。
- 不要直接读取 `_meta` 内部文件；检索标题目录使用 `/search/catalog`。
- 不要假设可以访问服务器文件系统；公开 skill 只通过 `/api/kb` 使用知识库。
- OCR 结果可能有错字、公式识别问题或排版噪声。引用和讲解时要谨慎。
- 页面中的 `_images_hashed/...` 图片路径会通过 API 改写成 OSS URL。题目依赖图像时，要提醒查看图片。
- 生成教学材料时，先理解内容组织逻辑，再检索资料，最后写作；不能只凭模型记忆生成。
