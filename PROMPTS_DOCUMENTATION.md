# Lux AI Prompts Documentation

This document provides a comprehensive catalog of all AI prompts used in the Lux multi-agent system. Each prompt is documented with its purpose, configuration, and exact implementation.

## Table of Contents

1. [Core Agent System Prompts](#core-agent-system-prompts)
   - [Market Researcher Agent](#market-researcher-agent)
   - [Risk Manager Agent](#risk-manager-agent)
2. [Task Processing Prompts](#task-processing-prompts)
3. [Chat and Interaction Prompts](#chat-and-interaction-prompts)
4. [Content Creation Prompts](#content-creation-prompts)
5. [Reflection and Learning Prompts](#reflection-and-learning-prompts)
6. [Documentation and Example Prompts](#documentation-and-example-prompts)

---

## Core Agent System Prompts

These prompts define the fundamental behavior and personality of specialized agents in the Lux system. They establish the agent's role, responsibilities, and output format.

### Market Researcher Agent

**Location:** `lib/lux/agents/market_researcher.ex:76-96`

**Purpose:** Defines a cryptocurrency market analysis agent that proposes trading opportunities based on market conditions. This agent analyzes market data and generates structured trade proposals with complete order specifications.

**LLM Configuration:**
- **Model:** `gpt-4o-mini`
- **Temperature:** `0.7` (balanced creativity for market analysis)
- **Response Format:** Structured JSON with strict schema enforcement
- **Schema:** Trade proposal with fields: coin, is_buy, sz, limit_px, order_type, reduce_only, rationale

**System Prompt:**

```
You are a Market Research Agent specialized in analyzing cryptocurrency markets
and proposing trades. Your recommendations must be in valid JSON format and include
ALL of the following fields:

{
  "coin": "string (e.g., 'ETH', 'BTC')",
  "is_buy": boolean,
  "sz": number,
  "limit_px": number,
  "order_type": {
    "limit": {
      "tif": "Gtc"
    }
  },
  "reduce_only": boolean,
  "rationale": "string explaining the trade"
}

Always include every field with appropriate values based on your analysis.
```

**Dynamic User Prompt Template:**

When proposing a trade, the agent receives market conditions and is prompted with:

```
Based on these market conditions, propose a trade:
{market_conditions_json}

Respond with a complete trade proposal including ALL required fields.
```

**Key Characteristics:**
- Enforces strict JSON schema for trade proposals
- Requires comprehensive field completion (coin, direction, size, price, order type, risk parameters, rationale)
- Temperature set to 0.7 for analytical yet adaptive market recommendations
- Uses GPT-4o-mini for cost-efficient market analysis

---

### Risk Manager Agent

**Location:** `lib/lux/agents/risk_manager.ex:38-50`

**Purpose:** Evaluates trade proposals for risk compliance and makes execution decisions. This agent acts as a gatekeeper, reviewing proposed trades against risk management criteria before allowing execution.

**LLM Configuration:**
- **Model:** `gpt-4o-mini`
- **Temperature:** `0.3` (lower temperature for conservative, consistent risk decisions)
- **Response Format:** Structured JSON with strict schema enforcement
- **Schema:** Risk evaluation with fields: execute_trade (boolean), reasoning (string)
- **Strict Mode:** Enabled (additionalProperties: false) for precise output control

**System Prompt:**

```
You are a Risk Management Agent responsible for evaluating trade proposals
and executing trades that meet risk criteria. You will:

1. Review trade proposals and their rationale
2. Use the Risk Management Beam to evaluate trades
3. Execute approved trades
4. Provide feedback on rejected trades

Respond with a structured evaluation including whether to execute the trade
and your reasoning.
```

**Key Characteristics:**
- Lower temperature (0.3) ensures conservative, consistent risk assessment
- Binary decision output (execute_trade: boolean) with mandatory reasoning
- Designed to work with Risk Management Beam tool for quantitative evaluation
- Strict JSON schema prevents unexpected response variations
- Acts as critical control point in multi-agent trading pipeline

---

## Task Processing Prompts

These prompts are used by the company signal handler to analyze and execute tasks. They enable dynamic task decomposition, tool selection, and execution planning.

### Task Analysis Prompt

**Location:** `lib/lux/agent/companies/signal_handler/default_implementation.ex:214-227`

**Purpose:** Analyzes incoming tasks to determine feasibility, required tools, and execution strategy. This prompt helps the LLM evaluate whether a task can be completed with available resources and what approach should be taken.

**LLM Configuration:**
- **Model:** Inherits from agent/context configuration
- **Temperature:** Inherits from agent/context configuration
- **Response Format:** JSON with AnalysisFeasibilitySchema
- **Schema:** Object with `feasibility.possible` (boolean) and `feasibility.reason` (string)

**Used In:** Task assignment handling within company workflows

**Prompt:**

```
Analyze this task and determine if it is possible to complete with the available tools:
Title: {task_title}
Description: {task_description}

Consider:
1. Is this task possible to complete with the available tools?
2. What type of task is this?
3. What tools might be needed?
4. What should the output look like?
5. Are there any constraints to consider?
6. What artifacts should be produced?
7. Are the requirements clear and complete?
```

**Key Characteristics:**
- Comprehensive task evaluation framework (7 consideration points)
- Feasibility assessment before execution attempts
- Tool availability check against task requirements
- Output format and artifact planning
- Requirement completeness validation
- JSON response enforces structured decision-making
- Prevents wasted resources on impossible tasks

---

### Task Execution Prompt

**Location:** `lib/lux/agent/companies/signal_handler/default_implementation.ex:284-295`

**Purpose:** Guides the LLM to execute a previously analyzed task using available tools. This prompt receives the task analysis results and directs the agent to select appropriate tools, determine parameters, and combine results.

**LLM Configuration:**
- **Model:** Inherits from agent/context configuration
- **Temperature:** Inherits from agent/context configuration
- **Response Format:** Depends on task requirements
- **Tools:** Full access to agent's beams, lenses, and prisms

**Used In:** Task execution phase after successful analysis

**Prompt Template:**

```
Given this task analysis:
{formatted_analysis}

Execute the task using the available tools.
Determine:
1. Which tools to use
2. What parameters to provide
3. How to combine the results

If at any point you determine the task cannot be completed, explain why.
```

**Analysis Format Included:**

The `{formatted_analysis}` contains:
```
Task Type: {task_type}
Required Capabilities: {required_capabilities}
Expected Outputs: {expected_outputs}
Success Criteria: {success_criteria}
```

**Key Characteristics:**
- Two-phase approach: analysis then execution
- Tool selection guided by analysis results
- Parameter determination based on task requirements
- Result combination strategy
- Graceful failure handling with explanations
- Leverages OpenAI's native tool calling for tool execution
- Context-aware execution based on prior analysis

---

## Chat and Interaction Prompts

These prompts handle conversational interactions between agents and enable dynamic response generation based on agent configuration and context.

### Agent Chat Response Prompt

**Location:** `lib/lux/prisms/handle_chat.ex:65-74`

**Purpose:** Generates contextual chat responses based on an agent's identity, goal, and incoming messages. This prompt enables agents to maintain consistent personality and purpose while engaging in conversations.

**LLM Configuration:**
- **Model:** Inherits from agent's llm_config
- **Temperature:** Inherits from agent's llm_config
- **Response Format:** Plain text response
- **Tools:** Currently no tools passed (empty list), but designed to support agent's beams and prisms

**Used In:** HandleChat prism for processing chat signals between agents

**Prompt Template:**

```
You are {agent.name}, an AI agent with the following goal:
{agent.goal}

You received the following chat message:
{message}

Generate an appropriate response based on your goal and capabilities.
Keep the response concise and relevant to the conversation.
```

**Key Characteristics:**
- Dynamic agent identity injection (uses agent.name and agent.goal)
- Goal-driven response generation
- Conciseness emphasis for efficient communication
- Relevance constraint to maintain conversation focus
- Designed for inter-agent communication
- Supports threaded conversations via context metadata

---

## Content Creation Prompts

These prompts are used by content writer agents to create, structure, and refine written content. They demonstrate a multi-stage content production pipeline.

### Content Outline Creation Prompt

**Location:** `test/support/agents/writer.ex:52-78`

**Purpose:** Generates detailed content outlines based on research input. This prompt creates structured frameworks for blog posts with hierarchical organization (sections, subsections, key points).

**LLM Configuration:**
- **Model:** Inherits from agent configuration
- **Temperature:** `0.7` (creative yet structured)
- **Response Format:** JSON with outline structure
- **Schema:** Nested structure with title, sections, points, and subsections

**Used In:** Writer agent's `create_outline` task handler

**Prompt Template:**

```
Create a detailed outline based on this research:
{research_data}

Include:
1. Introduction section
2. Main sections with key points
3. Supporting subsections
4. Conclusion section

Respond with a JSON object containing:
{
  "title": "proposed title",
  "sections": [
    {
      "title": "section title",
      "points": ["list", "of", "points"],
      "subsections": [
        {
          "title": "subsection title",
          "points": ["list", "of", "points"]
        }
      ]
    }
  ]
}
```

**Key Characteristics:**
- Four-part structure requirement (intro, main, subsections, conclusion)
- Hierarchical organization with nested subsections
- Point-by-point breakdown for detailed planning
- JSON output enforces structural consistency
- Research-driven content planning
- Designed for blog post workflows

---

### Blog Post Draft Writing Prompt

**Location:** `test/support/agents/writer.ex:86-108`

**Purpose:** Converts structured outlines into complete blog post drafts with metadata. This prompt focuses on content quality, flow, and engagement while maintaining alignment with the outline.

**LLM Configuration:**
- **Model:** Inherits from agent configuration
- **Temperature:** `0.7` (balanced creativity)
- **Response Format:** JSON with content and metadata
- **Schema:** Title, full content, and metadata (word_count, reading_time, target_audience)

**Used In:** Writer agent's `write_draft` task handler

**Prompt Template:**

```
Write a blog post draft following this outline:
{outline}

Ensure:
1. Engaging introduction
2. Clear flow between sections
3. Supporting evidence for claims
4. Strong conclusion
5. Appropriate tone and style

Respond with a JSON object containing:
{
  "title": "final title",
  "content": "full blog post content",
  "metadata": {
    "word_count": number,
    "reading_time": "estimated reading time",
    "target_audience": "intended audience"
  }
}
```

**Key Characteristics:**
- Five-point quality framework (engagement, flow, evidence, conclusion, tone)
- Outline-driven content generation
- Rich metadata generation (word count, reading time, audience)
- Comprehensive output including title and full content
- Designed for publication-ready drafts
- Supports content planning and analytics

---

### Content Editing and Refinement Prompt

**Location:** `test/support/agents/writer.ex:116-138`

**Purpose:** Reviews and improves existing content with detailed change tracking and quality metrics. This prompt acts as an editorial reviewer, providing both improved content and analytical feedback.

**LLM Configuration:**
- **Model:** Inherits from agent configuration
- **Temperature:** `0.7` (balanced for editorial judgment)
- **Response Format:** JSON with edited content, changes, and metrics
- **Schema:** Edited content, change list, and improvement metrics (clarity, engagement, technical accuracy)

**Used In:** Writer agent's `edit_content` task handler

**Prompt Template:**

```
Edit and improve this content:
{content}

Focus on:
1. Clarity and conciseness
2. Grammar and style
3. Flow and transitions
4. Technical accuracy
5. Engagement factor

Respond with a JSON object containing:
{
  "edited_content": "improved content",
  "changes_made": ["list", "of", "changes"],
  "improvement_metrics": {
    "clarity": "score 1-10",
    "engagement": "score 1-10",
    "technical_accuracy": "score 1-10"
  }
}
```

**Key Characteristics:**
- Five-dimension editorial focus (clarity, grammar, flow, accuracy, engagement)
- Change tracking for transparency and review
- Quantitative quality metrics (1-10 scoring)
- Both content and meta-analysis output
- Supports iterative content refinement
- Provides measurable improvement indicators

---

## Reflection and Learning Prompts

These prompts enable agents to perform self-reflection, learn from experience, and adapt their behavior over time.

### Agent Self-Reflection Prompt

**Location:** `lib/lux/reflection.ex:116-149`

**Purpose:** Enables agents to analyze their own performance, identify patterns, and plan future actions. This prompt implements a metacognitive loop where agents evaluate their history, learned patterns, and metrics to decide on next steps.

**LLM Configuration:**
- **Model:** `gpt-4` (default)
- **Temperature:** `0.7`
- **Max Tokens:** `1000`
- **Response Format:** JSON with reflection analysis and action recommendations

**Used In:** Reflection module's `reflect/3` function for periodic agent self-evaluation

**Prompt Template:**

```
You are {agent.name}'s reflection process.
Your goal is to help achieve: {agent.goal}

Current context:
{current_context}

Recent history:
{formatted_history}

Learned patterns:
{formatted_patterns}

Performance metrics:
{metrics}

Based on this information, what actions should the agent take?
Respond in the following JSON format:
{
  "reflection": {
    "thoughts": "your reasoning process",
    "patterns_identified": [],
    "improvement_suggestions": []
  },
  "actions": [
    {
      "type": "prism|beam|lens",
      "name": "action_name",
      "params": {},
      "expected_outcome": "description"
    }
  ]
}
```

**Key Characteristics:**
- Metacognitive design (agent reflecting on itself)
- Multi-input analysis (context, history, patterns, metrics)
- Pattern identification for learning
- Action planning with expected outcomes
- JSON structure enforces systematic thinking
- History retention (last 5 reflections shown, 100 total stored)
- Pattern accumulation (top 50 patterns maintained)
- Performance tracking integration
- Supports continuous improvement loop
- Tool selection based on reflection (prisms, beams, lenses)

**History Format:**
```
{timestamp}: {thoughts_from_previous_reflection}
```

**Metrics Included:**
- `successful_actions`: Count of successful operations
- `failed_actions`: Count of failed operations
- `avg_response_time`: Average execution time
- `learning_rate`: Adaptive learning coefficient (decreases over time)
- `total_reflections`: Number of reflection cycles
- `total_actions`: Cumulative actions taken

---

