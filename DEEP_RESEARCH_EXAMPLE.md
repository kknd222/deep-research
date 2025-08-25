# “深度研究”端到端处理流程实例演示

本文档通过一个具体的案例，一步步展示单次研究任务中，数据是如何被抓取、加工、交付给大语言模型（LLM），以及 LLM 是如何进行处理和产出的。

---

## 案例设定

- **用户研究主题**: "电动汽车电池技术的未来发展趋势"
- **LLM 生成的研究子任务 (之一)**:
  - **搜索查询 (Query)**: "固态电池技术最新进展与挑战" (Latest progress and challenges of solid-state battery technology)
  - **研究目标 (Research Goal)**: "理解固态电池技术的最新技术突破、面临的主要挑战以及预计的商业化时间表。" (Understand the latest technological breakthroughs, main challenges, and expected commercialization timeline of solid-state battery technology.)

---

## 阶段一：数据收集 (模拟)

系统执行了上述搜索查询，并从一个技术网站上获取了以下内容。

**抓取的文章内容样例 (Sample Crawled Content)**:

```text
<article>
  <h2>固态电池：开启新纪元</h2>
  <p>
    固态电池（Solid-State Batteries）被广泛认为是下一代电池技术的颠覆者。与使用液体电解质的传统锂离子电池不同，固态电池使用固体电解质，从根本上提升了能量密度和安全性。
  </p>
  <p>
    近期，QuantumScape 公司宣布其最新原型在测试中取得了重大突破。该原型在超过1000次充放电循环后，依然能保持90%以上的容量，这在以前被认为是不可能实现的。此外，其能量密度据称达到了 450 Wh/kg，远超当前锂离子电池的 250-300 Wh/kg 水平。这意味着未来的电动汽车单次充电续航里程有望轻松突破1000公里。
  </p>
  <p>
    然而，商业化之路依旧充满挑战。主要障碍在于制造成本和材料稳定性。固体电解质与电极之间的界面接触问题（固-固界面）是导致内阻增大和性能衰减的关键技术难题。此外，规模化生产所需的精密工艺和高成本材料，使得其价格短期内难以与成熟的锂离子电池竞争。业界普遍预计，固态电池的大规模商业化应用至少还需要5到7年时间。
  </p>
</article>
```

---

## 阶段二：数据交付给大模型 (核心步骤)

系统 **不会** 对上面的文章内容进行切片或复杂的清洗。它会将这段 **原始文本** 直接嵌入到一个精心设计的 **提示词 (Prompt)** 中，然后发送给大模型。

**交付给大模型的完整提示词 (Actual Prompt Sent to LLM)**:

```text
Here is a search query and a research goal from the user.
Your task is to carefully read the provided context from a search result and extract the key information that directly answers the research goal.
Please synthesize the information into a concise and comprehensive summary.

[QUERY]
固态电池技术最新进展与挑战

[RESEARCH GOAL]
理解固态电池技术的最新技术突破、面临的主要挑战以及预计的商业化时间表。

[CONTEXT]
<content index="1" url="http://example.tech/solid-state-batteries-explained">
<article>
  <h2>固态电池：开启新纪元</h2>
  <p>
    固态电池（Solid-State Batteries）被广泛认为是下一代电池技术的颠覆者。与使用液体电解质的传统锂离子电池不同，固态电池使用固体电解质，从根本上提升了能量密度和安全性。
  </p>
  <p>
    近期，QuantumScape 公司宣布其最新原型在测试中取得了重大突破。该原型在超过1000次充放电循环后，依然能保持90%以上的容量，这在以前被认为是不可能实现的。此外，其能量密度据称达到了 450 Wh/kg，远超当前锂离子电池的 250-300 Wh/kg 水平。这意味着未来的电动汽车单次充电续航里程有望轻松突破1000公里。
  </p>
  <p>
    然而，商业化之路依旧充满挑战。主要障碍在于制造成本和材料稳定性。固体电解质与电极之间的界面接触问题（固-固界面）是导致内阻增大和性能衰减的关键技术难题。此外，规模化生产所需的精密工艺和高成本材料，使得其价格短期内难以与成熟的锂离子电池竞争。业界普遍预计，固态电池的大规模商业化应用至少还需要5到7年时间。
  </p>
</article>
</content>

[END OF CONTEXT]

Please provide your summary below.
```

**关键点解读**:
- **没有信息损失**: 原始文章被原封不动地放在 `<content>` 标签内。
- **上下文清晰**: LLM 同时接收到了原始查询 (`[QUERY]`) 和本次任务的具体目标 (`[RESEARCH GOAL]`)。这使得它的总结极具针对性。
- **指令明确**: 提示词的开头部分清晰地指示了 LLM 需要做什么：“仔细阅读...提取关键信息...回答研究目标...整合成简洁的摘要”。

---

## 阶段三：大模型的处理与产出

大模型在接收到上述完整的提示词后，会进行阅读、理解和总结，最终产出以下内容。这个产出在系统里被称为一个 **“学习点 (Learning)”**。

**大模型产出的“学习点”样例 (Sample LLM Output - "Learning")**:

```json
{
  "summary": "固态电池技术在性能上取得了显著突破，特别是能量密度和循环寿命。例如，QuantumScape 的原型能量密度达到 450 Wh/kg，远超现有技术，并能在1000次循环后保持90%以上的容量。然而，其商业化面临两大核心挑战：一是高昂的制造成本，二是技术上的固-固界面难题导致性能衰减。基于这些挑战，业界普遍预测固态电池的大规模应用还需要5到7年时间。",
  "source_url": "http://example.tech/solid-state-batteries-explained"
}
```

---

## 总结与后续

1.  **循环往复**: 上述的 **“收集 -> 交付 -> 产出”** 过程会为研究计划中的 **每一个** 搜索查询都执行一遍，从而产生多个独立的“学习点”。

2.  **最终报告**: 当所有子任务都完成后，系统会将 **所有** 产出的“学习点”（比如上面那个 JSON 中的 `summary` 内容）集合起来，连同最初的研究计划和引用来源，一起交给大模型，并使用 `writeFinalReportPrompt` 指示它撰写最终的、完整的深度研究报告。

通过这个案例，我们可以清晰地看到，该系统的核心竞争力在于其 **强大的 LLM 调度能力** 和 **精准的提示词工程**，而非传统的、繁琐的数据处理流水线。
