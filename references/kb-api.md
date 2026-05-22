# 学科知识库 API

公网 API：

```text
https://dreamschool.cloud/api/kb
```

除 `/health` 外，所有接口都需要 API Key。优先从环境变量读取：

```bash
export KB_API_KEY="<your-key>"
```

请求头：

```http
X-API-Key: <KB_API_KEY>
```

或：

```http
Authorization: Bearer <KB_API_KEY>
```

不要在输出、日志、仓库或示例中写入真实 key。

## 推荐调用流程

1. `/health` 确认可用学科。
2. `/overview?subject=<学科>&depth=2` 获取学科下前两级目录。
3. `/list?id=<目录ID>&depth=2` 继续向下浏览。
4. `/search/catalog` 按知识点、考点、题型、章节名、标题检索。
5. `/read-pages` 读取命中页和相邻页。
6. `/material-pack` 合并多个命中页作为生成上下文。

不要调用 `kb_search_text`。当前没有全文检索接口。

## GET /health

公开接口。返回服务状态、学科列表、catalog 行数和根目录 Markdown 数量。

```bash
curl -s "https://dreamschool.cloud/api/kb/health"
```

典型返回：

```json
{
  "ok": true,
  "subjects": ["高中化学", "高中数学", "高中物理", "高中生物"],
  "catalog_rows": 55099,
  "root_md_count": 20
}
```

## GET /overview

查看某个学科的目录概览。

```http
GET /api/kb/overview?subject=<subject>&depth=2&limit=80
```

参数：

- `subject`：精确学科名，例如 `高中物理`。不传时返回全部学科。
- `depth`：返回目录深度，默认 `2`，最大 `4`。
- `limit`：每个目录返回的子目录数量，服务端有上限。

示例：

```bash
curl -s \
  -H "X-API-Key: $KB_API_KEY" \
  --get "https://dreamschool.cloud/api/kb/overview" \
  --data-urlencode "subject=高中物理" \
  --data-urlencode "depth=2"
```

重要字段：

- `root.id`：目录 ID，可传给 `/list`。
- `root.children`：子目录列表。
- `child_dir_count`：子目录数量。
- `direct_md_count`：当前目录直接包含的 Markdown 数量。
- `md_count`：当前目录下递归包含的 Markdown 总量。
- `readable`：该目录树下是否有可读 Markdown。
- `truncated`：是否因 `limit` 截断。

## GET /list

从一个目录继续向下浏览。

```http
GET /api/kb/list?id=<kb-directory-id>&depth=2&limit=100
```

推荐每次 `depth=2`，让 agent 逐步缩小资料范围。

示例：

```bash
curl -s \
  -H "X-API-Key: $KB_API_KEY" \
  --get "https://dreamschool.cloud/api/kb/list" \
  --data-urlencode "id=kb://高中物理/一轮复习讲义" \
  --data-urlencode "depth=2"
```

返回结构和 `/overview` 类似。

## GET /search/catalog

检索 `_meta/md_title_catalog.csv` 中的标题和资料路径。适合知识点、考点、章节名、题型名、讲义标题、试卷标题。

```http
GET /api/kb/search/catalog?q=<query>&subject=<subject>&stage=<stage>&limit=20
```

参数：

- `q`：必填，例如 `圆周运动`、`三角函数`、`遗传规律`、`电离平衡`。
- `subject`：可选，精确学科名，例如 `高中物理`。
- `stage`：可选，路径过滤词，例如 `一轮`、`二轮`、`同步`、`真题`。
- `limit`：返回数量，默认 `20`，最大 `100`。

示例：

```bash
curl -s \
  -H "X-API-Key: $KB_API_KEY" \
  --get "https://dreamschool.cloud/api/kb/search/catalog" \
  --data-urlencode "subject=高中物理" \
  --data-urlencode "q=圆周运动" \
  --data-urlencode "limit=5"
```

典型结果项：

```json
{
  "score": 100,
  "subject": "高中物理",
  "title": "16.圆周运动",
  "page_id": "kb://高中物理/.../page_001/page_001.md",
  "relative_path": "高中物理/.../page_001/page_001.md",
  "relative_pdf": "高中物理/.../16.圆周运动.pdf",
  "page_no": 1
}
```

检索不理想时：

- 缩短关键词，例如 `圆周运动临界问题` 改成 `圆周运动`。
- 换教材常用词，例如 `机械能守恒`、`功能关系`。
- 去掉 `stage` 限制。
- 先用 `/overview` 或 `/list` 找到更合适的资料目录。

## POST /read-pages

根据 `page_id` 读取 Markdown 页面。题干、答案或解析可能跨页，默认应读取相邻页。

单页请求：

```json
{
  "page_id": "kb://高中物理/.../page_001/page_001.md",
  "before": 1,
  "after": 1,
  "max_chars": 30000
}
```

多页请求：

```json
{
  "page_ids": [
    "kb://高中物理/.../page_001/page_001.md",
    "kb://高中物理/.../page_002/page_002.md"
  ],
  "before": 0,
  "after": 1
}
```

字段：

- `page_id`：单个页面 ID。
- `page_ids`：多个页面 ID。
- `before`：向前读取相邻页数量，默认 `1`，最大 `10`。
- `after`：向后读取相邻页数量，默认 `1`，最大 `10`。
- `max_chars`：最多返回字符数，服务端有上限。

示例：

```bash
curl -s \
  -H "X-API-Key: $KB_API_KEY" \
  -H "Content-Type: application/json" \
  -X POST "https://dreamschool.cloud/api/kb/read-pages" \
  -d '{
    "page_id": "kb://高中物理/.../page_001/page_001.md",
    "before": 1,
    "after": 1
  }'
```

返回字段：

- `pages[].page_id`：页面来源 ID。
- `pages[].relative_path`：知识库内相对路径。
- `pages[].text`：OCR Markdown 文本，图片已改写为 OSS URL。
- `pages[].chars`：当前页字符数。
- `total_chars`：总字符数。
- `truncated`：是否被截断。

## POST /material-pack

一次读取多个 catalog 命中页。适合生成讲义、教案、练习时聚合多个资料来源。

```json
{
  "items": [
    {
      "page_id": "kb://高中物理/.../page_001/page_001.md",
      "before": 0,
      "after": 1
    },
    {
      "page_id": "kb://高中物理/.../page_006/page_006.md",
      "before": 1,
      "after": 0
    }
  ],
  "max_chars": 30000
}
```

返回中包含 `sources` 和 `pages`。生成教学材料时，应内部保存 `sources`，但正式正文默认不展示内部路径。

## Python 示例

```python
import json
import os
import urllib.parse
import urllib.request

BASE_URL = "https://dreamschool.cloud/api/kb"
API_KEY = os.environ["KB_API_KEY"]

def request(path, params=None, method="GET", payload=None):
    url = BASE_URL + path
    if params:
        url += "?" + urllib.parse.urlencode(params)
    data = None
    headers = {"X-API-Key": API_KEY}
    if payload is not None:
        data = json.dumps(payload, ensure_ascii=False).encode("utf-8")
        headers["Content-Type"] = "application/json"
    req = urllib.request.Request(url, data=data, headers=headers, method=method)
    with urllib.request.urlopen(req, timeout=40) as resp:
        return json.loads(resp.read().decode("utf-8"))

results = request(
    "/search/catalog",
    {"subject": "高中物理", "q": "圆周运动", "limit": 3},
)

bundle = request(
    "/material-pack",
    method="POST",
    payload={
        "items": [
            {"page_id": item["page_id"], "before": 1, "after": 1}
            for item in results["items"]
        ],
        "max_chars": 30000,
    },
)
```

## 图片

API 会把 Markdown 中的 `_images_hashed/...` 改写成：

```text
https://aidreamschool.oss-cn-shenzhen.aliyuncs.com/_images_hashed/<image-file>
```

题目依赖图像、表格、实验装置、几何图或坐标图时，必须检查或提示查看图片。
