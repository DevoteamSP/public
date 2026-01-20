# Scaling Snowflake Intelligence: A Practical Guide to Atomic Instructions

**Managing Hundreds of Semantic Views and AI Agents with Reusable Rules**

---

## Abstract

When managing Snowflake Intelligence at scale—dozens of semantic views and agents across multiple business domains—instruction management quickly becomes overwhelming. Instructions get duplicated across agents, business logic changes require hunting through hundreds of YAML files, and inconsistencies creep in.

This whitepaper presents **Atomic Instructions**, a practical approach to managing Snowflake Intelligence configurations at scale by decomposing complex instructions into small, reusable rules. Based on real-world implementation, we show how atomic instructions enable teams to scale from 10 to 100+ semantic views and agents while maintaining consistency, traceability, and quality.

**Target Audience:** Snowflake developers, data architects, and teams managing Snowflake Intelligence deployments.

---

## Table of Contents

1. [The Problem: Instruction Management at Scale](#1-the-problem-instruction-management-at-scale)
2. [The Solution: Atomic Instructions](#2-the-solution-atomic-instructions)
3. [How Atomic Instructions Work](#3-how-atomic-instructions-work)
4. [Repository Organization](#4-repository-organization)
5. [Versioning and Deployment](#5-versioning-and-deployment)
6. [Best Practices](#6-best-practices)
7. [Conclusion](#7-conclusion)

---

## 1. The Problem: Instruction Management at Scale

### 1.1 The Starting Point

You start with a few Snowflake Intelligence agents. Each agent has its own instruction block:

```yaml
# Agent: Sales Analyst
instructions:
  - When comparing time periods, ensure both periods have the same number of days
  - Default to current calendar year if no period is specified
  - Never include PII in responses
  - Format responses with summary, data, and context
```

This works fine for 5-10 agents.

### 1.2 The Scaling Problem

Fast forward 6 months. You now have:
- **40+ semantic views** across Sales, HR, Finance, Operations
- **25+ agents** serving different teams and use cases
- **Multiple environments** (dev, staging, production)

Suddenly you face these challenges:

**Problem 1: Duplication**
The "compare equal time periods" instruction is copy-pasted across 30+ agents. When your CFO decides to change from calendar year to fiscal year, you need to update 30+ files.

**Problem 2: Inconsistency**
Agent A says "compare equal periods," Agent B says "align time windows," Agent C forgot the rule entirely. Users get different results depending which agent they use.

**Problem 3: No Traceability**
Which agents use which business rules? When you update the PII filtering logic, which agents are affected? You don't know without reading every file.

**Problem 4: Testing Nightmare**
How do you test that all agents handle time comparisons correctly? Manual testing doesn't scale to 40+ views.

**Problem 5: Team Coordination**
Multiple teams edit instruction blocks simultaneously. Merge conflicts. Duplicated efforts. Knowledge silos.

### 1.3 Real-World Impact

- **Maintenance Burden:** Data teams spend 40% of their time fixing instruction inconsistencies
- **Quality Issues:** Different agents give different answers to the same question
- **Slow Deployment:** Fear of breaking things means changes take weeks instead of days
- **Technical Debt:** Instructions become monolithic, undocumented, and unmaintainable

---

## 2. The Solution: Atomic Instructions

### 2.1 The Core Concept

Instead of writing monolithic instruction blocks, decompose them into **atomic, reusable rules**:

```
Monolithic Instruction Block (150+ lines)
              ↓
   Atomic Rules (10-15 rules × 10 lines each)
              ↓
    Rules organized by category
              ↓
  Agents reference rule IDs, not full text
```

### 2.2 What Is an Atomic Instruction?

An **atomic instruction** (or "rule") is:
- **Single-purpose:** One rule = one business concept
- **Reusable:** Used by multiple agents and semantic views
- **Versioned:** Changes are tracked over time
- **Testable:** Can be validated independently
- **Discoverable:** Tagged and categorized for easy finding

### 2.3 Example: Before and After

**Before (Monolithic):**
```yaml
# Agent: Sales Analyst
instructions: |
  When comparing time periods, ensure both periods have the same number of days.
  If today is March 15, 2024 and user asks for YoY comparison, compare
  Jan 1 - Mar 15, 2024 vs Jan 1 - Mar 15, 2023.

  If user doesn't specify a time period, default to current calendar year
  (January 1 to December 31, UTC timezone).

  Never include PII in responses. Aggregate or anonymize data.

  Format responses with: 1) Summary, 2) Data table, 3) Context, 4) Next steps.
```

**After (Atomic):**
```yaml
# Agent: Sales Analyst
rule_id:
  - aligned_period_comparison
  - default_period
  - pii_filtering
  - response_format
```

The rules are defined once, centrally:

```yaml
# rules/temporal/aligned_period_comparison.yaml
id: aligned_period_comparison
category: temporal
version: "1.2"
tags: [comparison, ytd, growth]
description: >
  When performing any time-based comparison (e.g., Growth question,
  Year vs. Year-1, Quarter vs. Quarter), always compare equivalent
  periods. If the current period is incomplete, automatically restrict
  both periods to the same number of elapsed days.
examples:
  - input: "Compare sales YoY"
    context: "Today is March 15, 2024"
    behavior: "Compare Jan 1 - Mar 15, 2024 vs Jan 1 - Mar 15, 2023"
```

### 2.4 Benefits

| Benefit | Impact |
|---------|--------|
| **Reusability** | Write once, use across 40+ agents |
| **Consistency** | Same rule = same behavior everywhere |
| **Maintainability** | Update one rule, all agents inherit change |
| **Traceability** | Know exactly which agents use which rules |
| **Testability** | Validate rules individually |
| **Scalability** | Add new agents without copying instructions |

---

## 3. How Atomic Instructions Work

### 3.1 Rule Structure

Every rule follows this simple structure:

```yaml
id: unique_rule_identifier
version: "1.0"
category: "category_name"
tags:
  - purpose_tag
  - type_tag
depends_on:  # Optional: prerequisite rules
  - other_rule_id
description: >
  The actual instruction that the LLM must follow.
  Written in clear, imperative language.
examples:
  - input: "User question or scenario"
    context: "Relevant context"
    behavior: "Expected agent behavior"
```

**Example: Fiscal Year Rule**

```yaml
# rules/temporal/fiscal_year_handling.yaml
id: fiscal_year_handling
version: "1.0"
tags:
  - temporal
  - fiscal_year
depends_on:
  - default_period
description: >
  The company fiscal year runs from April 1 to March 31.
  When users refer to "fiscal year" or "FY", interpret dates accordingly.
  FY2024 = April 1, 2023 to March 31, 2024.
examples:
  - input: "Show me FY2024 revenue"
    behavior: "Query revenue from April 1, 2023 to March 31, 2024"
```

### 3.2 Agent Structure

Agents reference rules by ID:

```yaml
# agents/sales_analyst.yaml
id: sales_analyst
name: Sales Analyst
description: >
  Analyzes sales performance, revenue trends, and customer metrics.

semantic_view: sales_metrics

metadata:
  owner: data-team
  created: "2024-01-15"
  status: active

rule_ids:
  - unauthorized_questions
  - aligned_period_comparison
  - default_period
  - fiscal_year_handling
  - pii_filtering
  - response_format

sample_questions:
  - "What are total sales by region this year?"
  - "Compare Q1 sales vs last year"
  - "Show me top 10 products by revenue"

# Optional: Notes for maintainers
notes: >
  Write here notes for the mainteners
```

### 3.3 Semantic View Structure

Semantic views also reference rules, organized by instruction type:

```yaml
# semantic_views/sales_metrics.yaml
id: sales_metrics
name: Sales Metrics
description: >
  Core sales semantic view providing revenue, orders, and customer metrics.

database: ANALYTICS
schema: SEMANTIC
view_name: V_SALES_METRICS

metadata:
  owner: data-team
  status: active

# Rules for Cortex Analyst's instruction sections
orchestration:
  - aligned_period_comparison
  - default_period

system:
  - pii_filtering

response:
  - response_format

sample_questions:
  - "What is total revenue by region?"
  - "Show order trends over time"
```

### 3.4 Assembly Process

An **assembler** resolves rules and generates complete instruction blocks:

```
1. Read all rules from rules/ directory
2. Read agent/semantic view configuration
3. Resolve rule_ids → full rule definitions
4. Handle dependencies (topological sort)
5. Generate complete YAML or Markdown
6. Deploy to Snowflake
```

**Generated Output:**

```yaml
# Generated for sales_analyst agent
planning_instructions:
  - id: unauthorized_questions
    description: >
      Please first use the tool "ACCEPT_RULE" to check if the question is accepted or refused...

  - id: aligned_period_comparison
    description: >
      When performing any time-based comparison, always compare
      equivalent periods...

  - id: default_period
    description: >
      If the user does not specify a time period, default to using
      the current calendar year...
```

---

## 4. Repository Organization

### 4.1 Directory Structure

Organize your configuration repository like this:

```
atomic/
├── rules/                          # Rules organized by category
│   ├── orchestration/             # Tool routing, question validation
│   │   ├── unauthorized_questions.yaml
│   │   └── tool_routing.yaml
│   ├── temporal/                   # Time periods, comparisons
│   │   ├── aligned_period_comparison.yaml
│   │   ├── default_period.yaml
│   │   └── fiscal_year_handling.yaml
│   ├── security/                   # PII, access control
│   │   ├── pii_filtering.yaml
│   │   └── access_control.yaml
│   └── formatting/                 # Output format, language
│       ├── response_format.yaml
│       └── multilingual_response.yaml
│
├── agents/                         # Agent configurations
│   ├── sales_analyst.yaml
│   ├── hr_assistant.yaml
│   ├── finance_reporter.yaml
│   └── supplier_analytics.yaml
│
├── semantic_views/                 # Semantic view configurations
│   ├── sales_metrics/
│   │   ├── semantic.yaml          # Rule references
│   │   └── details.yaml           # Snowflake semantic spec
│   ├── hr_metrics/
│   ├── finance_metrics/
│   └── supplier_benchmark/
│
├── scripts/
│   └── assembler.py                # Assembler logic
│
├── tests/                          # Validation tests
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
└── README.md
```

### 4.2 Rule Categories

Organize rules into logical categories:

| Category | Purpose | Example Rules |
|----------|---------|---------------|
| **orchestration** | Tool routing, validation, workflow | `unauthorized_questions`, `tool_routing` |
| **temporal** | Time periods, comparisons, calendars | `aligned_period_comparison`, `fiscal_year_handling` |
| **security** | PII filtering, access control | `pii_filtering`, `access_control` |
| **formatting** | Output format, language | `response_format`, `multilingual_response` |
| **domain** | Business-specific logic | `sales_commission_rules`, `hr_policy_compliance` |

### 4.3 Why This Structure Works

**Benefits:**
- **Clear ownership:** Each category can have a dedicated team
- **Easy discovery:** Find rules by browsing category folders
- **Parallel development:** Teams work on different categories without conflicts
- **Logical grouping:** Related rules are co-located

---

## 5. Versioning and Deployment

### 5.1 Rule Versioning

Version every rule to track changes:

```yaml
id: aligned_period_comparison
version: "1.2"   # Increment on changes
```

**Versioning Strategy:**
- **Major version (1.0 → 2.0):** Breaking changes to rule logic
- **Minor version (1.1 → 1.2):** Clarifications, examples added
- **Track changes:** Document what changed and why

**Example Version History:**

```yaml
# Version 1.0 (2024-01-15)
# - Initial release

# Version 1.1 (2024-02-20)
# - Added examples for fiscal year scenarios

# Version 1.2 (2024-03-10)
# - Clarified behavior for partial quarters
```

### 5.2 Deployment Pipeline

Use CI/CD to automate validation and deployment:

```
Developer → Feature Branch → Validation → PR Review → Merge → Deploy
```

**Validation Steps:**
1. **YAML syntax check:** Ensure files are valid YAML
2. **Schema validation:** Check required fields exist
3. **Dependency check:** Verify all rule_ids reference existing rules
4. **Circular dependency detection:** Prevent infinite loops
5. **Test execution:** Run unit and integration tests

### 5.3 CI/CD Example

```yaml
# .github/workflows/validate.yml
name: Validate Configurations

on:
  pull_request:
    paths:
      - 'rules/**'
      - 'agents/**'
      - 'semantic_views/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install pyyaml

      - name: Validate configurations
        run: |
          python scripts/validate.py

      - name: Check for orphaned rules
        run: |
          python scripts/check_usage.py
```

### 5.4 Deployment Workflow

1. **Development:** Create/modify rules in feature branch
2. **Local testing:** Run assembler locally to preview changes
3. **Pull request:** Automated validation runs
4. **Code review:** Data team reviews rule changes
5. **Merge:** Changes merged to main branch
6. **Deploy:** Assembler generates updated agent configs
7. **Snowflake update:** Deploy to Snowflake via SQL or API

### 5.5 Environment Management

Manage separate configurations for each environment:

```
├── config/
│   ├── dev.yaml
│   ├── staging.yaml
│   └── production.yaml
```

**Example:**

```yaml
# config/production.yaml
environment: production

snowflake:
  account: "company.us-east-1"
  database: "ANALYTICS_PROD"
  schema: "SEMANTIC_MODELS"

deployment:
  agents:
    - sales_analyst
    - hr_assistant
    - finance_reporter

  semantic_views:
    - sales_metrics
    - hr_metrics
    - finance_metrics
```

---

## 6. Best Practices

### 6.1 Writing Good Rules

**Principle 1: One Rule = One Responsibility**

Bad:
```yaml
description: >
  Handle time periods and PII filtering and response formatting.
```

Good:
```yaml
# Three separate rules
- aligned_period_comparison  # Just time periods
- pii_filtering              # Just PII
- response_format            # Just formatting
```

**Principle 2: Use Imperative Language**

Bad:
```yaml
description: >
  You might want to try comparing equal time periods when possible.
```

Good:
```yaml
description: >
  Always compare equivalent time periods. If the current period is
  incomplete, restrict both periods to the same number of elapsed days.
```

**Principle 3: Include Examples**

Every rule should have concrete examples:

```yaml
examples:
  - input: "Compare Q1 2024 vs Q1 2023"
    context: "Today is February 15, 2024"
    behavior: "Compare Jan 1 - Feb 15, 2024 vs Jan 1 - Feb 15, 2023"

  - input: "Compare Q1 2024 vs Q1 2023"
    context: "Today is April 10, 2024 (Q1 complete)"
    behavior: "Compare full Q1 2024 (Jan-Mar) vs full Q1 2023 (Jan-Mar)"
```

**Principle 4: Define Edge Cases**

Address boundary conditions explicitly:

```yaml
description: >
  When user asks for "this year":
  - If today is Jan 1 - Dec 31: Use current calendar year
  - If user specifies "fiscal year": Use fiscal year (Apr 1 - Mar 31)
  - If data only exists for partial year: Use available date range
```

### 6.2 Agent Configuration Best Practices

**Start with Core Rules**

Every agent should include these foundational rules:
- `default_period` - Handle unspecified time ranges
- `pii_filtering` - Protect sensitive data
- `response_format` - Consistent output structure

**Add Domain-Specific Rules**

Then layer on business logic:
- Sales agents: `sales_commission_rules`, `revenue_recognition`
- HR agents: `hr_policy_compliance`, `employee_confidentiality`
- Finance agents: `fiscal_year_handling`, `accrual_basis_accounting`

**Use Meaningful Sample Questions**

Include real user questions, not just examples:

```yaml
sample_questions:
  - "What are total sales by region this year?"        # Good
  - "Show me revenue"                                  # Too vague
  - "Compare Q1 sales vs last year"                    # Good
  - "Test query"                                       # Not useful
```

### 6.3 Quality Checklist

Before deploying an agent or semantic view:

**Structure:**
- [ ] Has unique `id` in snake_case
- [ ] Has descriptive `name` and `description`
- [ ] Has complete `metadata` (owner, version, status, dates)

**References:**
- [ ] All `rule_ids` exist in rules directory
- [ ] `semantic_view` reference is valid (for agents)
- [ ] No circular dependencies

**Testing:**
- [ ] Has meaningful `sample_questions`
- [ ] Sample questions cover main use cases
- [ ] Sample questions include edge cases

**Documentation:**
- [ ] Purpose and scope clearly documented
- [ ] Owner and maintenance team identified
- [ ] Related agents/views documented

**Clarity**
- [ ] Rules are unambiguous
- [ ] Rules are written in a compact format
- [ ] Used rules in a Semantic view or Agent are not contradictory

### 6.4 Common Pitfalls

**Pitfall 1: Rules Too Broad**

Don't create "mega-rules" that do everything:
```yaml
# BAD: One rule does too much
id: handle_everything
description: >
  Handle time periods, filter PII, format output, route tools...
```

**Pitfall 2: Missing Dependencies**

Declare dependencies explicitly:
```yaml
# GOOD: Explicit dependency
id: fiscal_year_handling
depends_on:
  - default_period  # Fiscal year builds on default period logic
```

**Pitfall 3: Orphaned Rules**

Regularly audit for unused rules:
```bash
# Find rules not referenced by any agent/view
python scripts/check_usage.py --find-orphans
```

**Pitfall 4: Vague Language**

Be specific, not ambiguous:
```yaml
# BAD
description: "Try to use recent data when possible"

# GOOD
description: "Default to the most recent 12 months of available data"
```

### 6.5 Testing Strategy

**Unit Tests: Individual Rules**

Test each rule works correctly:

```python
def test_aligned_period_comparison():
    rule = load_rule("aligned_period_comparison")

    # Test with incomplete period
    result = apply_rule(
        rule,
        question="Compare sales YoY",
        current_date="2024-03-15"
    )

    assert "Jan 1 - Mar 15, 2024" in result
    assert "Jan 1 - Mar 15, 2023" in result
```

**Integration Tests: Agent Configurations**

Test that agents assemble correctly:

```python
def test_sales_analyst_assembly():
    agent = load_agent("sales_analyst")
    assembled = assemble_instructions(agent)

    assert "aligned_period_comparison" in assembled.rules
    assert len(assembled.rules) == 6
    assert no_circular_dependencies(assembled)
```

**End-to-End Tests: Snowflake Queries**

Test actual agent behavior in Snowflake:

```python
def test_agent_e2e():
    response = query_agent(
        agent="sales_analyst",
        question="Compare Q1 2024 sales vs Q1 2023"
    )

    assert response.sql_generated
    assert "2024-01-01" in response.sql
    assert "2023-01-01" in response.sql
```

---

## 7. Conclusion

**1. Atomic Instructions Enable Scale**

By decomposing monolithic instruction blocks into reusable rules, you can scale from 10 to 100+ agents without exponential complexity.

**2. Reusability Drives Consistency**

Write a rule once, use it across dozens of agents. Update once, all agents inherit the change. This ensures consistent behavior across your entire Snowflake Intelligence ecosystem.

**3. Versioning and Testing Are Essential**

Track rule versions, validate configurations automatically, and test at multiple levels (unit, integration, e2e). Treat your semantic layer as production code.

**4. Organization Matters**

Organize rules by category, agents by domain, and configurations by environment. Clear structure enables team autonomy and parallel development.

**5. Start Simple, Scale Gradually**

Begin with 10-15 core rules covering common scenarios. Add domain-specific rules as needed. Don't over-engineer upfront.

---

## Appendix: Rule Template

Use this template when creating new rules:

```yaml
id: rule_name_in_snake_case
category: "category_name"
version: "1.0"
tags:
  - purpose_tag
  - type_tag
depends_on: []  # List any prerequisite rules

description: >
  Clear, imperative instruction that the LLM must follow.
  Use "Always", "Never", "If... then..." language.
  Be specific about edge cases and boundary conditions.

examples:
  - input: "Example user question"
    context: "Relevant context (date, state, etc.)"
    behavior: "Expected agent behavior"

  - input: "Another example"
    context: "Different context"
    behavior: "Expected behavior"

# Optional: Notes for maintainers
notes: >
  Additional context about why this rule exists,
  related business requirements, or implementation details.
```

---

**Writer**: Laurent Letourmy for Devoteam Snowflake Partner

**Document Version:** 1.0.0

**Publication Date:** January 2026

**Based on:** Atomic Instruction Architecture

**License:** MIT

---

*This whitepaper is based on real-world implementation of atomic instructions for Snowflake Intelligence at scale. Your specific requirements may vary, but the core principles apply universally.*
