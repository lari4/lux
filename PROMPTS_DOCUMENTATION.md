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

