---
title: "AI Agent 实践心得：Agent、RAG、MCP 全解析"
date: 2024-05-18
classes: wide
layout: splash
categories:
  - AI
tags:
  - Agent
  - MCP
  - 工作流
  - RAG
---

<style>
h1, h2, h3, h4 {
    /* 关键修改：通过 fit-content 保持块级特性 */
    width: fit-content;
    max-width: 100%; /* 防止溢出容器 */

    background-image: linear-gradient(
        135deg, 
        rgb(110, 185, 252) 5%, 
        rgb(248, 136, 210) 45%, 
        rgb(253, 149, 125) 85%
    );
    background-repeat: no-repeat;
    background-size: 100% 100%; 
    -webkit-background-clip: text;
    background-clip: text;
    -webkit-text-fill-color: transparent;
}

.page__content h2 {
    border-bottom: 0px;
}
</style>

## AI Agent 实践心得：Agent、RAG、MCP 全解析

### 前言

本次分享将结合我们团队的 AI 助手案例，讲解 AI Agent 的核心概念、主流的实现方案。以及当下备受关注的 MCP 是什么，如何借助 MCP 构建高效、可扩展的企业级 AI 工作流。

---

### AI 助手演示

**《demo》**

---

### AI Agent

#### 开发 AI Agent 的挑战

- **意图识别**  
  如何让 AI 准确理解用户需求，并调用合适的 Agent 工具解决问题。

- **Prompt 优化**  
  AI 对 Prompt 的理解与设计意图不一致，导致输出结果偏差，调优难度大。

- **RAG 知识库质量**  
  传统 RAG、GraphRAG、LightRAG、DeepSearcher 等多种技术方案选择困难，知识召回效果影响整体表现。

这些挑战不仅影响 AI 应用的准确性和实用性，还对实际落地产生制约。

---

### 目前主流 AI Agent 方案

#### 1. 传统单节点工作流

{% include figure popup=true image_path="/assets/images/2025-05-18/image1.png" %}

**优点：**

- 单节点 Agent 工作流，适用于简单的业务处理

**缺点：**

- 准确性依赖知识库或工具的返回结果
- 无法处理复杂、多步骤或需要推理的问题

---

#### 2. Workflow Agent（顺序工作流）

{% include figure popup=true image_path="/assets/images/2025-05-18/image2.png" %}

**特征：**

- 多个 Agent 串联
- 支持条件判断
- 全局状态管理

**优点：**

- 通过多个 Agent 的协作，处理复杂的任务
- 通过预设流程，能够对结果进行精细化控制

**缺点：**

- 需要对每个 Agent 的输入和输出进行精细化控制

---

#### 通过 Reflection 节点优化工作流

以下是一种常用的反思节点设计方法，用于评估和优化 Agent 的决策过程。

```text
你是一个评估从FAQ文档中检索到的信息与用户问题相关性的评估者。
如果文档包含中的FAQ的答案能直接回复用户的问题的话，请评估其为相关。

注意：不要自己回答问题。只需评估文档与问题的相关性即可。

返回一个 JSON 对象，其中包含两个键：
- score：值为"yes"或"no"，表示文档是否包含至少一些与问题相关的信息。
- reasoning：需要解释做出此决定的理由。
```

---

#### Workflow Agent 编排演示

展示了如何通过多 Agent 节点构建复杂的工作流。

{% include figure popup=true image_path="/assets/images/2025-05-18/image4.png" %}

---

#### 3. Agents（自主工作流）

由大模型自主决策和执行任务，无需外部流程，仅依靠自身能力完成全流程操作。

> https://www.anthropic.com/engineering/building-effective-agents

{% include figure popup=true image_path="/assets/images/2025-05-18/image3.png" %}

**优点：**

- **灵活、自主**  
  Agents 能够根据任务目标自主规划和执行步骤，适合处理开放性、变化多的场景。
- **扩展性强**  
  随着大模型能力提升，Agents 可以胜任更多复杂任务，具备持续进化的潜力。

**缺点：**

- **难以控制，容易跑偏**
- **业务适配难度高**  
  对于需要强业务规则或合规要求的场景，单纯依赖 Agents 难以满足企业级需求。

---

#### Agents 编排演示

这里引入了 Supervisor（主管-工人 架构），是一种更具组织性、分工明确的控制方式。

- 主管：负责接受用户指令，并根据任务将工作分配给多个工人。
- 工人：执行具体子任务的智能体，完成各自分配的工作并将结果反馈给主管

{% include figure popup=true image_path="/assets/images/2025-05-18/image6.png" %}

---

#### 总结

- **Workflow Agent**：适用于复杂任务，支持多步骤和条件判断，易于控制，但需要精细化设计。
- **Agents**：灵活性高，扩展性强，但难以控制和管理。

> Workflow 是“按剧本执行”，而 Agents 则是“即兴发挥”。

> 在日常业务场景中，Workflow Agent 依然是最常用的实现方式。

---

### RAG 知识库

#### 传统 RAG 遇到的瓶颈

- 召回内容缺乏上下文关联，文档数量多时相关性降低
- 更适用于 FAQ 或一问一答等简单场景

> 以“权限申请问题”为例，系统往往召回了全部 10 篇相关文档，导致 AI 难以准确判断哪些内容真正与当前上下文相关，影响答案的精准性。

**解决思路：**

- 结构化知识库
- 借鉴 Web Search，将切片检索变成文档检索

---

#### 结构化知识库

> 便于对结构化知识库的理解，我们可以参考 AI 百科的设计思路。

- 提取业务文档的结构化信息，如：标题、摘要、正文、外部链接等进行存储
- 通过检索摘要信息、下钻全文内容等方式，提升召回的准确性和相关性

{% include figure popup=true image_path="/assets/images/2025-05-18/image7.png" %}

---

### MCP 工具介绍

#### 什么是 MCP？

**多模态通讯协议（Multi-Modal Communication Protocol）**

**特点：**

- 支持通过远程通信协议触发 AI 的工具调用（tool_call）
- 使 AI 能以类似调用 API 的方式，灵活访问各类后端服务或系统工具
- 实现自然语言指令到结构化参数的自动映射，并返回标准化结果

---

#### 强大的社区生态

- 通过社区里大家提供的各种 MCP，AI 可以轻松接入各类工具和服务

**《演示》**

---

#### MCP 开发误区

**❌ 只要把已有服务端接口包成 MCP 工具，AI 就能用了**

两个挑战：

- 自然语言 ≠ 输入参数
  原接口可能需要 status=1，但用户说的是“查一下未完成的任务”， 我们需要将“未完成”转换为接口所理解的 status=1，这通常依赖 prompt、映射表、甚至分类模型等手段。
- JSON 返回 ≠ AI 可读
  原接口返回的是复杂的嵌套 JSON，我们需要将其转换为对 AI 更友好的可读格式，例如表格、列表、摘要等 Markdown 表达。

---

#### MCP Server 云函数实践

- MCP Server 作为云函数，支持灵活扩展
- 内置 SDK，适配多种业务场景，提升 AI 工作流能力

{% include figure popup=true image_path="/assets/images/2025-05-18/image8.png" %}
{% include figure popup=true image_path="/assets/images/2025-05-18/image9.png" %}

---

### 总结

- **Workflow Agent**：适用于复杂任务，支持多步骤和条件判断，易于控制，但需要精细化设计。
- **Agents**：灵活性高，扩展性强，但难以控制和管理。
- **MCP**：通过远程通信协议触发 AI 的工具调用，支持多模态交互，提升工作流能力。
- **RAG**：通过结构化的方式管理知识库内容，再结合深度搜索，可提升召回的准确性和相关性，减少传统 RAG 系统上下文信息不全的问题。
