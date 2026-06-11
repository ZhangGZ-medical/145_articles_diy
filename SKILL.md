---
name: 145_articles_diy
description: |
  文献元数据批量采集与结构化填表技能。根据论文名称，自动从Crossref API、期刊分区查询平台、
  PubMed/Semantic Scholar等多源获取完整书目信息，按11列标准化表格输出。
  触发词：文献调研、论文信息采集、文献元数据、article metadata、填文献表、
  论文作者单位期刊查询、sci论文信息汇总、文献检索填表。
agent_created: true
---

# 145_articles_diy — 文献元数据批量采集与结构化填表

根据用户提供的论文名称列表，逐篇检索并填写标准化11列文献信息表格。交付格式：Markdown 表格 + xlsx（11列 x N篇文献）。

---

## 输出表格结构（固定11列）

| # | 列名 | 说明 |
|---|------|------|
| 1 | 序号 | 按用户提供顺序编号（1, 2, 3...） |
| 2 | 论文名称 | 完整英文/中文标题 |
| 3 | 作者姓名 | 全部作者，以逗号分隔 |
| 4 | 单位 | 全部作者所属机构（合并去重） |
| 5 | 所属会议/期刊（期刊号） | 期刊全称（ISSN） |
| 6 | 刊物级别 | SCIE/SSCI/ESCI收录、JCR分区、中科院分区、影响因子 |
| 7 | 投稿日期 | Received日期（精确到日） |
| 8 | 收录或发表日期 | Accepted日期 + Published online日期 + 印刷出版日期 |
| 9 | 论文内容与项目关系 | 按固定模板撰写（见下文） |
| 10 | 页码 | 起止页码（如219–227） |
| — | DOI（辅助列） | 唯一标识符，用于后续查证 |
| — | PMID（辅助列） | PubMed ID |

---

## 数据采集策略（多源并行）

### 源1：Crossref API（主力）— 获取核心元数据

```python
import requests, json

def crossref_query(title):
    url = f"https://api.crossref.org/works?query.title={requests.utils.quote(title)}&rows=1"
    r = requests.get(url, timeout=15)
    data = r.json()
    item = data["message"]["items"][0]
    
    # 作者
    authors = item.get("author", [])
    author_names = [f"{a.get('given','')} {a.get('family','')}" for a in authors]
    affiliations = []
    for a in authors:
        for aff in a.get("affiliation", []):
            affiliations.append(aff.get("name", ""))
    
    # 期刊
    container = item.get("container-title", [""])[0] if item.get("container-title") else ""
    issn = item.get("ISSN", [""])[0] if item.get("ISSN") else ""
    
    # 日期
    created = item.get("created", {}).get("date-parts", [[None]])[0]  # 投稿日期
    published_online = item.get("published-online", {}).get("date-parts", [[None]])[0]
    published_print = item.get("published-print", {}).get("date-parts", [[None]])[0]
    
    # 页码 & 卷期
    page = item.get("page", "")
    volume = item.get("volume", "")
    issue = item.get("issue", "")
    
    # DOI & PMID
    doi = item.get("DOI", "")
    
    return {
        "title": item.get("title", [""])[0],
        "authors": "; ".join(author_names),
        "affiliations": "; ".join(set(affiliations)),
        "journal": container,
        "issn": issn,
        "volume": volume,
        "issue": issue,
        "page": page,
        "received": format_date(created),    # Crossref的created通常为投稿日
        "published_online": format_date(published_online),
        "published_print": format_date(published_print),
        "doi": doi
    }
```

**关键字段映射**：
- Crossref `created` → 投稿日期（Received）
- Crossref `published-online` → 在线发表日期
- Crossref `published-print` → 印刷出版日期
- **Accepted日期在Crossref中不直接提供**，需从PubMed或其他源补充

### 源2：期刊分区查询 — 获取刊物级别

期刊级别信息从以下平台获取，按优先级降序：

| 优先级 | 数据源 | URL模板 | 适用 |
|--------|--------|---------|------|
| 1 | iikx.com（LetPub中文镜像） | `https://www.iikx.com/sci/medcine/` + 期刊编号 | 中国用户首选，中科院分区+JCR+SCIE |
| 2 | Bioxbio | `https://www.bioxbio.com/journal/INT-J-NEUROSCI` | IF+JCR分区 |
| 3 | Academic Accelerator | `https://academic-accelerator.com/Impact-of-Journal/` + ISSN | IF趋势 |

**查询方式**：用 `WebFetch` 抓取期刊分区页面，提取中科院分区、JCR Q等级、SCIE收录状态、最新影响因子。

**输出格式**：`SCIE收录，JCR Q4（神经科学），中科院大类医学4区/小类神经科学4区，IF=1.5（2024），非Top/非综述期刊`

### 源3：PubMed / Europe PMC — 补充PMID和摘要

```python
# 通过DOI查找PMID
def get_pmid_by_doi(doi):
    url = f"https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term={doi}[doi]&retmode=json"
    r = requests.get(url, timeout=10)
    data = r.json()
    id_list = data.get("esearchresult", {}).get("idlist", [])
    return id_list[0] if id_list else None

# 通过PMID获取摘要
def get_abstract_by_pmid(pmid):
    url = f"https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=pubmed&id={pmid}&retmode=xml"
    r = requests.get(url, timeout=10)
    # 解析XML提取<Abstract>节点
```

### 源4：Semantic Scholar API（备用）

当Crossref或PubMed不可用时，使用Semantic Scholar API作为fallback：

```python
def semantic_scholar_query(title):
    url = f"https://api.semanticscholar.org/graph/v1/paper/search?query={title}&limit=1&fields=title,authors,year,journal,externalIds,publicationVenue"
    # 注意：有rate limit，建议在调用间隔加1s延迟
```

### 源5：WebSearch（兜底）

当所有结构化API失败时，用`WebSearch`直接搜索论文标题，从搜索结果页手动提取元数据。

---

## 论文内容与项目关系 — 固定模板

**格式要求**（必须严格遵循三段式）：

```
论文提出了......，研究了......，有助于......。
```

**三段定义**：

| 段 | 内容 | 示例 |
|----|------|------|
| 提出了 | 论文的核心科学假说/理论框架 | "提出了Wnt信号通路介导的NSC移植修复机制假说" |
| 研究了 | 论文的具体实验/分析方法与关键发现 | "研究了NSC侧脑室移植对MCAO小鼠缺血性脑损伤的修复效果及分子机制（脑水肿减轻、梗死体积缩小、神经功能改善，均与Wnt通路激活相关）" |
| 有助于 | 对本项目的具体贡献 | "有助于为hNPC01注射液临床研究提供Wnt通路作为疗效生物标志物的理论依据，支持NSC来源细胞产品对缺血性脑卒中后神经修复的可行性" |

**写作原则**：
1. 使用中文撰写，专业术语保留英文缩写
2. "有助于"段必须与用户当前项目直接挂钩
3. 如有具体定量数据（p值、显著差异），在"研究了"段中注明
4. 反复迭代：用户可能要求调整措辞，每次修改后展示完整版本确认

---

## 完整工作流（SOP）

### Phase 1：准备

1. 明确项目全称（供"论文内容与项目关系"字段使用）
2. 确认文献列表（论文名称、顺序）
3. 如用户未提供项目名称，先澄清再执行

### Phase 2：逐篇采集

对每篇论文：

```
1. Crossref API → 获取作者、单位、期刊、ISSN、日期、页码、DOI
2. 期刊分区查询 → iikx.com或bioxbio → JCR分区、中科院分区、IF
3. PubMed/Europe PMC → 补充PMID、Accepted日期、摘要
4. 写"论文内容与项目关系" → 三段式模板
5. 汇总为单行表格记录
```

### Phase 3：汇总输出

- 内存中维护累积表格数据
- 每完成一篇展示给用户确认
- 全部完成后生成 `.xlsx` 文件

### Phase 4：交付

- 展示Markdown汇总表
- 生成xlsx文件并 `deliver_attachments`

---

## 常用API端点速查

| API | 端点 | 用途 |
|-----|------|------|
| Crossref | `https://api.crossref.org/works?query.title={title}&rows=1` | 核心元数据 |
| Crossref | `https://api.crossref.org/works/{doi}` | 通过DOI精确查询 |
| PubMed E-search | `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term={doi}[doi]&retmode=json` | DOI→PMID |
| PubMed E-fetch | `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=pubmed&id={pmid}&retmode=xml` | PMID→摘要/完整记录 |
| Semantic Scholar | `https://api.semanticscholar.org/graph/v1/paper/search?query={title}&limit=1&fields=...` | 备用查询（有429限制） |
| Europe PMC | `https://www.ebi.ac.uk/europepmc/webservices/rest/search?query=DOI:{doi}&resultType=core&format=json` | 备用PMID/摘要 |

---

## 踩坑经验

1. **Crossref `created` ≠ Received** — Crossref的created可能是数据库记录创建日期而非实际投稿日。需要交叉验证PubMed的`history`字段中的`received`日期。
2. **Accepted日期缺失** — 多数API不直接返回Accepted日期，需从期刊官网HTML页面或PDF中提取。
3. **期刊级别信息来源** — LetPub中文镜像（iikx.com）比英文LetPub提供更完整的中科院分区数据。
4. **Semantic Scholar rate limit** — 免费API有严格的429限制，每次调用间隔≥1秒，或申请API key。
5. **PubMed E-utils缺乏HTTPS支持** — 部分环境下`eutils.ncbi.nlm.nih.gov`可能无法通过HTTPS访问，需回退到Europe PMC REST API。
6. **作者顺序** — 必须按论文原文顺序排列，Crossref返回的作者列表通常按贡献排序，不要打乱。
7. **"论文内容与项目关系"迭代** — 用户常常要求调整措辞（如去掉IIa期、改动项目全称），每篇论文的这个字段需要单独维护和更新。

---

## 技能依赖

| 类型 | 技能 | 用途 |
|------|------|------|
| 输出 | `xlsx` 或 `minimax-xlsx` | 最终生成Excel表格 |
| 可选 | `medical-research-toolkit` | 生物医学论文PubMed深度检索 |
| 可选 | `multi-search-engine` | 中英文论文多引擎覆盖 |
| 可选 | `structured-research-reporting` | 若需DOCX交付 |

---

## 路径约定

| 路径 | 用途 |
|------|------|
| `~/.workbuddy/skills/145_articles_diy/` | 技能目录 |
| `{WORKSPACE}/results/` | 产出xlsx存储 |

---

## 文件结构

```
145_articles_diy/
├── SKILL.md       # 本说明文件（工作流+API+模板）
└── README.md      # 用户简介
```
