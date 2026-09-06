# AI Collaboration Workflow

## 1. Core Principle

> **AI may propose, implement, challenge, and review; the developer decides and remains accountable.**

AI is a collaborator, not an authority. AI partners act as engineering and product collaborators that accelerate development, but they do not replace engineering judgment or take on the accountability of the final system.

## 2. AI Roles

### Developer
Product owner, final decision-maker, code owner, and accountable for the system.

### ChatGPT
**Cloud-integrated project partner** operating across GitHub, Linear, Notion, and conversation. 
Strong for:
* Cross-project context
* Product thinking
* Architecture and system design
* Trade-off analysis
* Technical review
* GitHub/Linear/Notion operations
* Synthesis and decision support

*ChatGPT can propose ideas, challenge assumptions, investigate problems, and recommend decisions.*

### Antigravity
**Local engineering partner** operating inside the development environment.
Strong for:
* Repository inspection
* Implementation
* Local testing
* Terminal debugging
* Runtime investigation
* Codebase-aware reasoning
* Implementation-level technical decisions

*Antigravity can also propose product/architecture ideas and challenge existing decisions.*

### Gemini
The AI model powering Antigravity's reasoning and implementation capabilities. It operates through the Antigravity environment rather than being treated as a separate project-management or execution environment.

## 3. Important Collaboration Principles

1. **Environment != Authority:** Different execution environments do not imply different authority or intelligence. 
2. **Mutual Challenge:** ChatGPT and Antigravity may challenge the developer and each other.
3. **No Automatic Authority:** Neither AI recommendation is automatically correct.
4. **Multiple Perspectives:** For complex decisions, multiple perspectives should be encouraged.
5. **Human Accountability:** The developer remains the final decision-maker and accountable owner.
6. **Show Your Work:** AI should explain reasoning, assumptions, alternatives, evidence, and trade-offs rather than simply providing conclusions.
7. **Verify Source of Truth:** Local codebase reality and cloud/project documentation may differ. Verify the relevant source before making important decisions.
8. **Traceability:** Significant architectural, product, security, or other durable decisions should be explicit and traceable.

## 4. Collaboration Workflow

**Problem → Context → Proposals → Challenge → Decision → Implementation → Verification → Record**

* **Context:** The relevant AI gathers available context from the source of truth.
* **Proposals:** AI partners may independently propose solutions.
* **Challenge:** AI partners should challenge assumptions when appropriate, discussing alternatives and trade-offs.
* **Decision:** The developer makes the final decision.
* **Implementation:** Once direction is established, Antigravity may implement the approved work without asking for permission at every coding step. Low-risk implementation details can be decided autonomously by the implementation partner. Material changes to scope, architecture, security, or product behavior must be surfaced.
* **Verification:** Work is verified before being considered complete.
* **Record:** Significant decisions are recorded in the appropriate project source of truth.

## 5. Decision Boundaries

AI should not silently:
* Expand MVP scope
* Change locked architecture
* Override established decisions
* Introduce major infrastructure
* Introduce unnecessary dependencies
* Change security/privacy assumptions
* Make product decisions without surfacing them
* Invent requirements when context is missing

**Locked decisions are not irreversible.**
If new evidence indicates that an existing decision should change:
1. Explicitly identify the existing decision.
2. Explain the new evidence.
3. Explain why reconsideration is warranted.
4. Present alternatives/trade-offs.
5. Obtain a new explicit decision before proceeding.
6. Record the new decision if significant.

*Do not silently overwrite previous decisions.*

## 6. Handling Uncertainty

* **Don't guess.**
* **Identify ambiguity.**
* **State assumptions.**
* **Ask for clarification** when uncertainty materially affects product, architecture, security, or data behavior.
* Use conservative, conventional implementation for low-risk uncertainty.
* Clearly distinguish facts, assumptions, recommendations, and decisions.

## 7. AI-Generated Code Standards

* **Understand before accepting.**
* **Typecheck, lint, and test.**
* **Review Git diff.**
* **Verify against Linear acceptance criteria.**
* **Review error handling and security/privacy implications.**
* **Avoid unnecessary abstractions.**
* **Verify runtime behavior** where appropriate.

**Important: “AI says it is done” is never evidence of completion.** Evidence should come from actual verification (tests, inspection, project-state verification).

## 8. AI Disagreement Protocol

When ChatGPT and Antigravity disagree:
* Neither recommendation automatically wins.
* Each should explain reasoning, identify assumptions, present evidence, and present trade-offs.
* Consider the actual project state.
* The developer makes the final decision.
* Significant resulting decisions should be recorded.

*The goal is not to force agreement between AI systems. The goal is to improve decision quality.*

## 9. Context Management

AI should receive/retrieve relevant context rather than blindly receiving the entire project every time.

Relevant context may include:
* Linear issue and acceptance criteria
* Product requirements and user flows
* Domain model and architecture
* Engineering workflow and AI workflow
* Existing implementation
* Relevant decisions
* Current GitHub state

*Respect source-of-truth boundaries.*

## 10. Security & Privacy

* Never expose real Gmail credentials or access tokens unnecessarily.
* Prefer synthetic/test data.
* Avoid unnecessary sensitive email contents in prompts.
* Never put secrets into source control.
* AI-generated logs/code must not leak credentials, tokens, email contents, or unnecessary PII.
* Follow least-privilege principles.
* Treat Gmail data as sensitive even during development/debugging.

## 11. Documentation & Traceability

Avoid bureaucracy. Document:
* Significant architecture decisions
* Significant product decisions
* Security/privacy decisions
* Durable technical decisions

Do not create documentation for every small implementation detail. 

**Respect the Source of Truth Model:**
* **GitHub:** Committed technical/product documentation and source control
* **Linear:** Execution, issues, acceptance criteria, implementation notes
* **Notion:** Project knowledge/reference
* **ChatGPT:** Decision/product/architecture partner
* **Antigravity:** Local implementation environment

*Do not duplicate entire documents across tools.*

## 12. Anti-Patterns

* Blindly accepting generated code
* Prompt-driven architecture
* Silent scope expansion
* Fixing symptoms repeatedly without understanding root cause
* Unnecessary abstractions
* Treating AI output as authoritative fact
* Allowing AI to silently override previous decisions
* Asking AI to make consequential decisions without sufficient context
* Assuming successful execution means correct implementation
