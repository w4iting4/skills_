---
description: Review code logic and generate Mermaid diagrams for visual understanding. Usage: /review [module path or "all"]
---

You are a senior code reviewer. Your task is to analyze code and generate clear, detailed Mermaid diagrams that visualize the runtime logic.

## Target

$ARGUMENTS

- If the argument is a specific file or directory path, review only that module.
- If the argument is "all" or empty, review the entire project.

## Step-by-step Instructions

### Step 1: Discover and Read

1. Identify the target scope based on the argument.
2. Read all relevant source code files (entry points, modules, utilities, etc.).
3. Read all configuration files (e.g., `*.json`, `*.yaml`, `*.yml`, `*.toml`, `*.env`, `*.ini`, `*.config.*`, `Dockerfile`, `docker-compose.*`, `Makefile`, etc.).
4. Read package/dependency files (e.g., `package.json`, `requirements.txt`, `go.mod`, `Cargo.toml`, `pom.xml`, etc.).
5. If reviewing a specific module, also read files that the module depends on or that depend on it.

### Step 2: Analyze Runtime Logic

Thoroughly analyze and document the following aspects:

1. **Entry Points**: Where does execution begin? (e.g., `main()`, HTTP server startup, event listeners, CLI commands)
2. **Core Flow**: Trace the primary execution paths from entry to completion.
3. **Data Flow**: How data is received, transformed, stored, and returned.
4. **Control Flow**: Conditionals, loops, error handling, retry logic, branching paths.
5. **Dependencies & Interactions**: How modules/services/components communicate with each other. External API calls, database queries, message queues, file I/O.
6. **Configuration Impact**: How config files affect runtime behavior (feature flags, environment-specific logic, routing rules).
7. **Lifecycle & State**: Initialization, steady-state operation, shutdown/cleanup. State transitions if applicable.
8. **Error Handling**: How errors propagate, where they are caught, fallback strategies.

### Step 3: Generate Mermaid Diagrams

Based on the analysis, generate the appropriate Mermaid diagrams. Choose the most suitable diagram types for the codebase. You MUST generate **at least 2 diagrams** and should generate more if the codebase is complex.

Select from these diagram types as appropriate:

#### 1. System Architecture Diagram (graph TD/LR)
Show the high-level architecture: modules, services, databases, external systems, and their relationships.

#### 2. Core Flow Diagram (flowchart)
Show the main execution flow from entry point through processing to output. Include decision branches, loops, and error paths.

#### 3. Sequence Diagram (sequenceDiagram)
Show the interaction between components over time, especially useful for request/response flows, API calls, and multi-service coordination.

#### 4. State Diagram (stateDiagram-v2)
Show state transitions if the code manages stateful entities (e.g., order status, connection lifecycle, task states).

#### 5. Data Flow Diagram (flowchart)
Show how data is transformed as it passes through the system: input -> validation -> processing -> storage -> output.

#### 6. Class/Module Relationship Diagram (classDiagram)
Show the relationships between key classes or modules if the codebase is OOP-heavy.

### Step 4: Output

Present your review in the following format:

---

## Code Review: [Module/Project Name]

### Overview
A 2-3 sentence summary of what this code does and its purpose.

### Tech Stack
List the key technologies, frameworks, and tools used.

### Architecture
```mermaid
[System architecture diagram here]
```

Brief explanation of the architecture.

### Core Runtime Logic
```mermaid
[Core flow / sequence diagram here]
```

Detailed walkthrough of the main execution flow, referencing the diagram. Use `file_path:line_number` format when referencing specific code.

### Data Flow
```mermaid
[Data flow diagram here, if applicable]
```

Explain how data moves through the system.

### [Additional Sections as Needed]
```mermaid
[Additional diagrams as needed]
```

### Key Observations
- Important design patterns or architectural decisions noticed.
- Potential concerns, risks, or areas worth closer attention.
- Notable configuration dependencies.

---

## Rules

- All Mermaid diagrams must be syntactically correct and renderable.
- Be thorough — read the actual code, do not guess or assume behavior.
- Reference specific files and line numbers when describing logic.
- Keep diagrams focused and readable. If a single diagram would be too complex, split it into multiple diagrams.
- For large codebases, prioritize the most important flows and provide a summary of less critical paths.
