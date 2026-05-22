---
name: dreamschool-subject-kb
description: 当 agent 需要使用 DreamSchool 学科知识库时使用；支持浏览真实学科资料目录、检索 OCR 教辅页面、读取知识点/考点/题目/答案解析，并可基于知识库生成讲义、教案、练习题、试卷、课堂例题、课后作业等 Markdown 教学材料。
---

# DreamSchool 学科知识库

使用这个 skill 访问 DreamSchool 学科知识库，并在需要时基于知识库生成教学材料。知识库是一个只读资料库，包含高中学科教辅、教材、复习讲义、专题训练、试卷等 OCR Markdown 页面，以及 OSS 托管的页面图片。

## 核心工作流

这个 skill 只有一个基础能力：通过学科知识库检索和读取资料。所有上层任务，包括生成讲义、教案、练习题和试卷，都必须先完成资料检索与读页。

### 1. 先完成知识库检索

无论用户是要查资料、回答知识点问题，还是生成教学材料，都先读取 [references/kb-api.md](references/kb-api.md)，并按需读取 [references/kb-structure.md](references/kb-structure.md) 理解真实资料分布。

基础检索流程：

1. 不确定学科时调用 `/health`。
2. 广泛找资料时调用 `/overview` 和 `/list` 两层两层浏览目录。
3. 查具体知识点、考点、题型、章节名时调用 `/search/catalog`。
4. 用 `/read-pages` 或 `/material-pack` 读取命中页面和相邻页。
5. 保留 `page_id`、`relative_path`、`relative_pdf`、`page_no` 等来源信息。

如果用户只是要资料查找或知识问答，基于读取到的页面直接回答，并引用来源。

### 2. 再按目标执行上层任务

如果用户要生成讲义、教案、试卷、课堂例题、练习题、课后作业等 Markdown 教学内容，在完成基础检索后继续读取 [references/teaching-material-generation.md](references/teaching-material-generation.md)。

教学材料生成不是独立于知识库的模式，而是“检索资料 -> 组织材料 -> 质量检查 -> 输出 Markdown”的上层流程：

1. 若用户没有给出清晰小节，读取 [references/section-planning.md](references/section-planning.md)，先规划小节。
2. 对每个小节设计检索词，回到 `/search/catalog` 找资料。
3. 用 `/read-pages` 或 `/material-pack` 读取正文、例题、答案、解析和相邻页。
4. 严格基于已读取的知识库资料整理内容。知识库缺失时，按生成规范处理，不要悄悄编造。
5. 交付前做质量检查：结构完整、例题闭环、公式可读、图片可访问、无 API Key、无内部路径泄露。

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
- 生成教学材料时，先检索资料再写作；不能只凭模型记忆生成。
