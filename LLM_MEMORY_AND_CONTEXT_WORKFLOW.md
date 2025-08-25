# LLM 的“记忆”机制：深度研究中多步总结的工作流详解

您好！针对您提出的“大模型如何记住多次输入”以及“上下文超出限制怎么办”这两个核心问题，本文件将为您详细拆解其工作原理。

## 核心理念：应用程序是 LLM 的“外部记忆体”

您的直觉完全正确：**大语言模型（LLM）本身是无状态、无记忆的**。每一次对它的 API 调用都是一次独立的、全新的事件。

那么“记忆”是如何实现的呢？答案是：**充当“指挥官”的应用程序（我们的后端服务）负责扮演 LLM 的记忆体**。它通过精巧的流程设计来解决上下文长度限制问题。

整个流程可以分为两个主要阶段：
1.  **阶段 A：原子化的总结（独立生成记忆碎片）**：系统让 LLM 逐一处理搜集到的资料，每次只专注于一份内容，生成一份独立的摘要（我们称之为“学习点”或“记忆碎片”）。
2.  **阶段 B：综合性的合成（组装所有记忆碎片）**：在最后阶段，应用程序会将 **所有** 之前生成的“记忆碎片”一次性地、全部地提供给 LLM，让它基于这些完整的上下文来撰写最终报告。如果碎片太多，就会进入下面的“阶段 C”。

---

## 详细工作流演示

**研究主题**: "电动汽车电池技术的未来发展趋势"

### 阶段 A：原子化的总结（生成记忆碎片）

(此阶段与之前版本相同，展示了如何独立总结“固态电池”和“钠离子电池”的资料，并产出 `学习点 A` 和 `学习点 B`。)

#### 任务 1：总结“固态电池”资料 -> 产出 `学习点 A`
#### 任务 2：总结“钠离子电池”资料 -> 产出 `学习点 B`

... (假设我们继续执行了多个任务，最终得到了10个学习点: `A, B, C, D, E, F, G, H, I, J`) ...

**应用程序的操作**：后端服务将所有10个学习点保存在一个列表中。
- `learnings_list = [ learning_A, learning_B, ..., learning_J ]`

---

### 阶段 B：综合性的合成（正常流程）

如果 `learnings_list` 中所有学习点的文本总长度 **没有超过** LLM 的上下文窗口限制，那么流程很简单：

1.  **应用程序的操作**：应用程序取出 **所有** 存储的“学习点”，并将它们全部填充到 `writeFinalReportPrompt` 提示词模板中。
2.  **发送给 LLM 的最终提示词**:
    ```text
    Here is the collected knowledge:
    <learning> [学习点 A 的内容] </learning>
    <learning> [学习点 B 的内容] </learning>
    ...
    <learning> [学习点 J 的内容] </learning>
    ...
    Please write a final, comprehensive research report...
    ```
3.  **LLM 最终产出**: 一份完整的报告。

---

## 新增：阶段 C - 处理超长上下文：当“记忆”溢出时

这是对您最新问题的解答。如果 `learnings_list` 中所有学习点的文本总长度 **超过了** LLM 的上下文窗口限制，应用程序会启动 **“递归总结” (Recursive Summarization)** 策略，也常被称为 **“Map-Reduce”** 策略。

#### 工作原理：总结“总结”

1.  **Map 阶段**: 这就是我们已经完成的“阶段 A”，将每一份原始文档（Document）映射（Map）成一份独立的、简短的总结（Summary/Learning）。
2.  **Reduce 阶段**: 如果总结太多，应用程序会对其进行“降维打击”（Reduce）。它会把这些总结分批，让 LLM **总结这些总结**，生成更高级、更浓缩的“中间摘要”。如果中间摘要合起来还是太长，这个过程可以再重复一遍，直到最终的上下文能被 LLM 一次性处理。

#### 详细步骤演示：

假设我们的10个学习点（A-J）总长度超限。

1.  **应用程序的操作：分组**
    应用程序检测到超限，决定将10个学习点分成两组，每组5个。
    - `batch_1 = [A, B, C, D, E]`
    - `batch_2 = [F, G, H, I, J]`
    > **注意**: `src/utils/text.ts` 中的 `splitText` 工具函数在这种场景下会非常有用，它可以帮助智能地进行文本批次划分。

2.  **生成中间摘要（Reduce 操作）**
    应用程序会发起 **新的 LLM 调用**，让它总结第一批学习点。

    **发送给 LLM 的提示词 (Prompt for Intermediate Summary 1)**:
    ```text
    [GOAL]
    You are a research analyst. The following are several summarized learnings from different sources. Please synthesize them into a single, more concise summary that captures the key themes and findings from this batch.

    [CONTEXT]
    <learning> [学习点 A 的内容] </learning>
    <learning> [学习点 B 的内容] </learning>
    <learning> [学习点 C 的内容] </learning>
    <learning> [学习点 D 的内容] </learning>
    <learning> [学习点 E 的内容] </learning>
    ```

    **LLM 的产出 (中间摘要 1)**:
    > "本批次资料主要探讨了下一代电池技术。固态电池在能量密度和安全性上展现了巨大潜力但成本高昂，而钠离子电池则凭借成本优势在经济型市场表现突出..."

    应用程序对 `batch_2` 执行完全相同的操作，得到 **`中间摘要 2`**。

3.  **最终合成（Final Synthesis）**
    现在，应用程序不再需要处理10个零散的学习点，而是只需要处理2个高度浓缩的中间摘要。

    **发送给 LLM 的最终提示词 (Final Prompt with Intermediate Summaries)**:
    ```text
    [RESEARCH PLAN]
    研究电动汽车电池技术的未来发展趋势...

    Here is the collected knowledge:
    <learning>
    [中间摘要 1 的内容]
    </learning>
    <learning>
    [中间摘要 2 的内容]
    </learning>

    [SOURCES]
    ... (所有原始来源的列表) ...

    Please write a final, comprehensive research report based on the plan and the provided learnings.
    ```
4.  **LLM 的最终产出**: LLM 基于这两个高度概括的中间摘要，生成最终的研究报告。

---

### 图示化流程

```
[文档 A] -> [学习点 A] \
[文档 B] -> [学习点 B]  |
[文档 C] -> [学习点 C]  | -> [中间摘要 1] \
[文档 D] -> [学习点 D]  |                 |
[文档 E] -> [学习点 E] /                  | -> [最终报告]
                                          |
[文档 F] -> [学习点 F] \                  |
[文档 G] -> [学习点 G]  |                 |
[文档 H] -> [学习点 H]  | -> [中间摘要 2] /
[文档 I] -> [学习点 I]  |
[文档 J] -> [学习点 J] /
```

## 总结

- **LLM 自身无记忆**：每次调用都是独立的。
- **应用程序是记忆的载体**：它负责保存和传递上下文。
- **正常流程**：先独立总结，再统一合成。
- **超限流程 (Map-Reduce)**：当“记忆”太多时，引入“总结摘要”的中间步骤，对信息进行层层压缩，确保最终交付给 LLM 的上下文既全面，又在 Token 限制之内。这是处理海量信息的关键工程技巧。
