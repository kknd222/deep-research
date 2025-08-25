# “深度研究”功能代码分析报告

本文档详细分析了项目中“深度研究”功能的实现方式，涵盖了从数据抓取、数据处理，到与大语言模型（LLM）交互的完整流程。

## 核心设计理念

该项目的核心并非依赖于复杂的数据预处理流水线（如对文本进行精细切片），而是采用了一种先进的 **多步骤代理工作流（Multi-step Agentic Workflow）**。

在这个架构中，大语言模型（LLM）扮演着“指挥官”或“大脑”的角色，负责规划研究步骤、生成查询、理解原始数据、并最终综合信息生成报告。所谓的“数据工程”主要体现在 **高度精巧的提示词工程（Prompt Engineering）** 上。

---

## 详细工作流程

整个深度研究过程可以分解为以下几个关键阶段：

### 阶段一：规划与查询生成

当用户输入一个研究主题后，系统首先会让 LLM 进行规划。

1.  **生成研究计划**: 系统调用 LLM，根据用户主题生成一份结构化的研究计划。
2.  **生成搜索查询**: 基于这份计划，系统会再次调用 LLM，生成一组用于搜索引擎的查询指令（SERP Queries）。

这个过程由一个总控服务来定义，该服务向 LLM 暴露了一系列“工具（Tools）”。

- **关键文件**: `src/app/api/mcp/server.ts`
- **代码片段**: 在这个文件中，定义了 `write-research-plan` 和 `generate-SERP-query` 等工具，LLM 可以调用它们来完成规划任务。

```typescript
// src/app/api/mcp/server.ts

// ... (部分代码)
server.tool(
    "write-research-plan",
    writeResearchPlanDescription,
    // ... schema definition
    async ({ query, language }, { signal }) => {
      // ...
      const deepResearch = initDeepResearchServer({ language });
      const result = await deepResearch.writeReportPlan(query);
      // ...
    }
);

server.tool(
    "generate-SERP-query",
    generateSERPQueryDescription,
    // ... schema definition
    async ({ plan, language }, { signal }) => {
        // ...
        const deepResearch = initDeepResearchServer({ language });
        const result = await deepResearch.generateSERPQuery(plan);
        // ...
    }
);
```

- **关键文件**: `src/utils/deep-research/prompts.ts`
- **说明**: 此文件中的 `writeReportPlanPrompt` 和 `generateSerpQueriesPrompt` 函数负责生成与 LLM 沟通所需的、精确的指令（Prompt）。

---

### 阶段二：数据收集（搜索与抓取）

系统使用生成的查询从互联网上收集信息。

1.  **调用搜索引擎**: 系统会调用一个或多个第三方搜索服务，如 `Tavily`、`Firecrawl`、`Exa` 等。
2.  **直接获取内容**: 一个关键的设计是，这些现代的搜索服务很多都能在返回搜索结果的同时，**直接返回网页的主要内容（Markdown 或纯文本格式）**。这极大地简化了流程，避免了传统的“先获取链接列表，再逐一爬取”的模式。
3.  **备用抓取器**: 系统也包含一个简单的本地抓取器 (`/api/crawler`)，但从代码上看，优先使用能直接返回内容的搜索服务。

- **关键文件**: `src/utils/deep-research/search.ts`
- **代码片段**: `createSearchProvider` 函数根据配置选择不同的搜索引擎，并向其 API 发送请求。

```typescript
// src/utils/deep-research/search.ts

export async function createSearchProvider({ provider, ... }: SearchProviderOptions) {
  // ...
  if (provider === "tavily") {
    // ...
    const { results = [] } = await response.json();
    return {
      sources: (results as TavilySearchResult[])
        .filter((item) => item.content && item.url)
        .map((result) => {
          return {
            title: result.title,
            // 注意：这里直接使用了 API 返回的 content 或 rawContent
            content: result.rawContent || result.content,
            url: result.url,
          };
        }) as Source[],
      // ...
    };
  } else if (provider === "firecrawl") {
    // ...
    // 同样，直接从 API 结果中获取 markdown 内容
    return {
      sources: (data as FirecrawlDocument[])
        .filter((item) => item.description && item.url)
        .map((result) => ({
          content: result.markdown || result.description,
          url: result.url,
          title: result.title,
        })) as Source[],
      // ...
    };
  }
  // ...
}
```

---

### 阶段三：数据交付与总结（核心交互）

这是用户最关心的问题：**抓取的数据是如何交付给大模型的？**

答案是：**数据基本是“原样”交付的，没有进行切片等复杂的工程化处理。**

1.  **构建提示词**: 系统会将从搜索引擎获取到的 **原始文本内容**，直接嵌入到一个精心设计的提示词模板中。
2.  **提供上下文**: 这个模板除了包含网页内容，还包含了本次查询的目标（`researchGoal`），以及对 LLM 的明确指令（例如，“请总结以下内容并提取关键信息”）。
3.  **LLM 进行总结**: LLM 读取这个包含了原始数据的提示词，并根据指令进行阅读、理解和总结。总结后的产出被称为一个“学习点（learning）”。

- **关键文件**: `src/utils/deep-research/prompts.ts`
- **代码片段**: `processSearchResultPrompt` 函数是这个过程的核心。它将原始数据包裹在 `<content>` 标签内，并将其作为上下文交给 LLM。

```typescript
// src/utils/deep-research/prompts.ts

export function processSearchResultPrompt(
  query: string,
  researchGoal: string,
  results: Source[],
  enableReferences: boolean
) {
  // 将每个搜索结果的内容和 URL 构造成一个 XML 风格的标签
  const context = results.map(
    (result, idx) =>
      `<content index="${idx + 1}" url="${result.url}">\n${
        result.content
      }\n</content>`
  );

  // 将 context（包含所有原始数据）插入到主提示词模板中
  return (
    searchResultPrompt + (enableReferences ? `\n\n${citationRulesPrompt}` : "")
  )
    .replace("{query}", query)
    .replace("{researchGoal}", researchGoal)
    .replace("{context}", context.join("\n"));
}
```
- **相关文件**: `src/utils/text.ts`
- **说明**: 虽然这个文件里定义了文本切片函数 `splitText`，但在上述核心流程中并 **没有** 被调用。这再次印证了系统的设计哲学：依赖 LLM 的长文本理解能力，而不是预先进行文本切片。

---

### 阶段四：迭代与精炼

在第一轮搜索和总结后，系统可以进入一个迭代循环。

1.  **评估“学习点”**: LLM 会审视已经获得的所有“学习点”。
2.  **生成新查询**: 如果发现研究计划尚未完成或者有新的问题出现，LLM 可以调用 `reviewSerpQueriesPrompt` 来生成新的、更具针对性的搜索查询，从而进行更深入的研究。

---

### 阶段五：生成最终报告

当所有的研究任务完成后，系统会进入最后一步。

1.  **整合所有信息**: 系统将所有的“学习点（learnings）”、引用来源（`sources`）、以及收集到的图片（`images`）全部整合起来。
2.  **调用最终提示词**: 这些整合后的信息会被放入 `writeFinalReportPrompt` 提示词模板中。
3.  **LLM 撰写报告**: LLM 接收到这个最终的、包含所有研究成果的提示词，并根据指令，撰写出一份逻辑连贯、内容丰富的深度研究报告。

- **关键文件**: `src/utils/deep-research/prompts.ts`
- **代码片段**: `writeFinalReportPrompt` 函数负责构建这个最终的、内容最丰富的提示词。

```typescript
// src/utils/deep-research/prompts.ts

export function writeFinalReportPrompt(
  plan: string,
  learning: string[],
  source: Source[],
  images: ImageSource[],
  // ...
) {
  // 将所有 learning、source、image 整合进一个大的 prompt
  const learnings = learning.map(
    (detail) => `<learning>\n${detail}\n</learning>`
  );
  const sources = source.map(
    (item, idx) =>
      `<source index="${idx + 1}" url="${item.url}">\n${item.title}\n</source>`
  );
  // ...
  return (
    finalReportPrompt
    // ...
  )
    .replace("{plan}", plan)
    .replace("{learnings}", learnings.join("\n"))
    .replace("{sources}", sources.join("\n"))
    // ...
}
```

## 总结

该项目的“深度研究”功能是一个高度自动化和智能化的系统。它的实现精髓在于：

- **以 LLM 为中心**: 将 LLM 作为流程的调度者和推理引擎。
- **代理与工具**: 通过定义一系列清晰的“工具”，让 LLM 可以像调用函数一样与外部世界（如搜索引擎）交互。
- **提示词即数据工程**: 将复杂的原始数据（网页内容）直接作为上下文，通过精巧的提示词设计来引导 LLM 完成理解、总结和创作等任务，从而绕过了传统的数据清洗和预处理流程。
