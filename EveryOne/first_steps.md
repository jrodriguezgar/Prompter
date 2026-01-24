# 🚀 Mastering LLM Interactions: Core Principles

Shift your mindset from "using a tool" to "collaborating with another kind of intelligence."

---

## ⚡ Quick Start

1. **Define a role** → "Act as a Senior Python Developer"
2. **Be specific** → State exactly what you need, not vague goals
3. **Provide context** → Background, constraints, examples
4. **Specify output** → Format, length, structure
5. **Iterate** → Refine through dialogue, not one-shot prompts

---

## 💡 The Mindset

| Principle | Description |
|-----------|-------------|
| Never Underestimate | Assume the LLM is more capable than you think |
| Let It Lead | Give autonomy to propose solutions you haven't considered |
| Make It a Conversation | Iterative dialogue yields better results than single prompts |
| Be Communicative | Explain your reasoning and "why" behind tasks |
| Don't Be Shy | Ask anything, do whatever — there are no dumb questions or "wrong" experiments with an LLM |

---

## 🛠 Prompt Engineering

| Technique | Action |
|-----------|--------|
| Eliminate Ambiguity | Use precise language, no vague terms |
| Provide Full Context | Quality output ∝ quality of background info |
| Compress Information | Dense, high-signal content; remove fluff |
| Trigger Deep Thinking | Use "think step-by-step", keywords, Chain-of-Thought |

### Before/After Example

❌ **Vague:** "Write a function to process data"

✅ **Specific:** "Write a Python function that reads a CSV file, filters rows where 'status' equals 'active', and returns a list of dictionaries. Include type hints and docstring."

### Effective Prompt Patterns

Follow one of these proven structural patterns:

- **Role - Task - Example**: Define who the LLM should act as, what it needs to do, and provide a concrete example
- **Instruction - Context - Data**: State the action, provide background information, then supply the data to work with
- **Plan - Knowledge Domain - Tool**: Outline the strategy, specify the area of expertise needed, and indicate any tools or frameworks to use

### Anti-Patterns to Avoid

| Anti-Pattern | Why to Avoid |
|--------------|--------------|
| Over-Constraint | Too many restrictions limit creative and optimal solutions |
| Under-Specification | Insufficient detail leads to generic, unhelpful outputs |
| Error Case Overload | Don't focus on what could go wrong; state what you want to achieve |

### Controlling Model Behavior

| Best Practice | Application |
|---------------|-------------|
| **Minimize Noise & Entropy** | Keep prompts focused; remove irrelevant information that dilutes intent |
| **Bridge Unrelated Domains** | Provide explicit guidance when connecting disparate concepts to prevent unexpected outputs |
| **Enforce Thoroughness** | Specify execution style (step-by-step, iterate through all items, validate each step) to overcome lazy evaluation patterns |

**Thoroughness Examples:**

❌ **Lazy:** "Review the code files and fix any issues"

✅ **Explicit:** "Review each file in the `src/` directory one by one. For each file: (1) Check for syntax errors, (2) Identify security vulnerabilities, (3) Suggest performance improvements. Complete all three checks for every file before moving to the next."

❌ **Vague:** "Analyze this dataset and give insights"

✅ **Detailed:** "Process all rows in the dataset. For each row: validate data types, check for null values, compute statistical summaries. Then iterate through each column and provide distribution analysis. Finally, cross-reference all findings in a summary table."

### Output Format Preferences

**Prefer YAML over JSON** for tabular data retrieval:
- More human-readable
- Less verbose syntax
- Easier to parse and maintain
- Better for configuration and structured data

---

## 🧹 Context Management

### Active Management

| Strategy | Application |
|----------|-------------|
| **Window Awareness** | Know your LLM's limits (~128k tokens); monitor usage |
| **Progressive Loading** | Core info first → details later; layer information |
| **Structured Input** | Use markdown, tables, bullets for dense data |
| **Prioritization** | Critical info at start/end (better retention zones) |
| **Anchor Points** | Create references: "as in section A..." for coherence |

### Maintenance & Optimization

| Technique | When/How |
|-----------|----------|
| **Periodic Summaries** | Every 10-15 exchanges → request executive summary |
| **Context Injection** | Repeat key info when switching subtasks |
| **Task Chunking** | Divide long conversations into thematic sessions |
| **Reference Outputs** | "Using the function from earlier..." maintains continuity |

### Degradation Signals & Recovery

**Warning Signs:** Generic responses • Forgets prior decisions • Contradicts earlier info • Repetitive patterns

**Solution:** Compile summary → Start fresh session with clean context → Use workspace as external memory

### Preservation Tactics

- Export key decisions to files
- Maintain updated "state document"
- Leverage workspace as persistent memory layer

---

## 🎯 Problem-Solving Techniques

| Technique | How to Apply |
|-----------|--------------|
| First Principles | Discard assumptions → Rebuild from fundamentals |
| Planning | Strategy first → Define roadmap before execution |
| Task Decomposition | Break into sub-tasks → Solve parts → Integrate |
| Constraint Relaxation | Remove limits → See solution → Reintroduce constraints |
| Simplification | Core problem first → Add complexity gradually |
| Exemplification | Use concrete examples, sample I/O, analogies |
| Reverse Engineering | Start from outcome → Work backwards |
| Iteration | Draft → Improve → Polish |

---

## ❌ Common Mistakes

| Mistake | Fix |
|---------|-----|
| "Make it better" | Specify what "better" means (faster, shorter, clearer) |
| Assuming context | Always provide background, don't assume LLM knows your project |
| Multiple tasks in one prompt | One clear task per prompt |
| No output format | Always specify structure, length, style |
| Giving up after first try | Iterate and refine through dialogue |
