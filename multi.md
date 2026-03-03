# AI Dev Team - Complete Technical Documentation

## Table of Contents
1. [System Overview](#system-overview)
2. [Project Structure](#project-structure)
3. [Architecture & Design Principles](#architecture--design-principles)
4. [Application Entry Point](#application-entry-point)
5. [Execution Flow](#execution-flow)
6. [Core Components](#core-components)
7. [State Management](#state-management)
8. [Agent Details](#agent-details)
9. [Utility Systems](#utility-systems)
10. [Data Flow](#data-flow)
11. [Configuration & Dependencies](#configuration--dependencies)

---

## 1. System Overview

### What is AI Dev Team?

AI Dev Team is an autonomous multi-agent system that transforms a software requirement into a complete, working application. It uses LangGraph (a state machine framework) and Google's Gemini AI to orchestrate multiple specialized AI agents that work together like a real development team.

### Current Implementation: Phase 3

The system currently implements three phases:

- **Phase 1**: PM Agent - Converts vague requirements into detailed specifications
- **Phase 2**: Architect Agent - Designs database schema, APIs, and frontend structure
- **Phase 3**: Planner Agent - Creates an ordered build plan and sets up a sandbox workspace

### Technology Stack

- **Runtime**: Node.js 18+ (ES Modules)
- **AI Framework**: LangGraph v1.2.0 (state machine orchestration)
- **LLM**: Google Gemini 2.5 Flash via @google/genai
- **State Persistence**: Redis (optional) or in-memory
- **Language**: JavaScript (ES6+)

---

## 2. Project Structure

```
ai-dev-team-phase3/
├── src/
│   ├── index.js                    # Application entry point
│   ├── agents/                     # AI agent implementations
│   │   ├── pmAgent.js             # Project Manager Agent
│   │   ├── architectAgent.js      # Architect Agent (5 steps)
│   │   ├── blueprintValidator.js  # Blueprint validation logic
│   │   └── plannerAgent.js        # Planner Agent
│   ├── nodes/                      # LangGraph node functions
│   │   ├── humanInput.js          # Terminal input handler
│   │   ├── setupSandbox.js        # Sandbox creation node
│   │   └── sandboxHealthCheck.js  # Sandbox verification node
│   ├── config/                     # Configuration files
│   │   ├── state.js               # Complete state definition
│   │   └── graph.js               # LangGraph wiring
│   └── utils/                      # Utility modules
│       ├── gemini.js              # Gemini API wrapper
│       ├── tokenTracker.js        # Token usage tracking
│       └── sandboxManager.js      # Sandbox filesystem operations
├── tests/                          # Test files
├── package.json                    # Dependencies & scripts
├── .env                           # Environment variables
└── README.md                      # User documentation
```

### File Responsibilities

| File | Purpose | LLM Calls |
|------|---------|-----------|
| `index.js` | Entry point, CLI handling, output formatting | No |
| `pmAgent.js` | Requirement clarification & spec generation | Yes (1-2) |
| `architectAgent.js` | 5-step architecture design | Yes (5) |
| `blueprintValidator.js` | Cross-validation of architecture | No |
| `plannerAgent.js` | Build plan generation | Yes (1) |
| `humanInput.js` | User Q&A via terminal | No |
| `setupSandbox.js` | Create project workspace | No |
| `sandboxHealthCheck.js` | Verify sandbox integrity | No |
| `state.js` | State schema definition | No |
| `graph.js` | LangGraph node wiring | No |
| `gemini.js` | LLM API calls & token tracking | No |
| `tokenTracker.js` | Token usage display | No |
| `sandboxManager.js` | Filesystem operations | No |

---

## 3. Architecture & Design Principles

### First Principles: State Machine Architecture

The system is built on **LangGraph**, which models workflows as directed graphs where:

1. **Nodes** = Functions that perform work (agents, validators, I/O)
2. **Edges** = Transitions between nodes (conditional or direct)
3. **State** = Shared data structure that ALL nodes read from and write to
4. **Checkpointing** = Automatic state persistence after each node

### Key Design Decisions

#### 1. No Direct Function Calls Between Nodes

```javascript
// ❌ WRONG: Direct coupling
function nodeA() {
  const result = nodeB();  // Direct call
  return result;
}

// ✅ CORRECT: State-based communication
function nodeA(state) {
  return { dataForB: "value" };  // Write to state
}

function nodeB(state) {
  const data = state.dataForB;  // Read from state
  return { result: "processed" };
}
```

**Why?** Checkpointing requires serializable state. Direct function calls can't be saved/resumed.

#### 2. Token Tracking via Deltas

```javascript
// ❌ WRONG: Mutating shared object (causes duplication)
function agent(state) {
  state.tokenUsage.calls.push(newCall);  // Mutates
  return state.tokenUsage;
}

// ✅ CORRECT: Return delta, let reducer merge
function agent(state) {
  return {
    tokenUsage: {
      newCalls: [newCall],      // Only new data
      addedInput: 100,
      addedOutput: 200,
      addedCost: 0.0001
    }
  };
}
```

**Why?** LangGraph reducers merge old + new state. Returning full objects causes exponential duplication.

#### 3. Deterministic Validation (No LLM)

The Blueprint Validator uses pure JavaScript logic instead of LLM calls because:
- Validation is deterministic (checking if table X exists)
- 100% accurate vs. LLM's probabilistic nature
- Zero tokens spent
- Instant execution

#### 4. Sandbox Isolation

All generated code lives in an isolated sandbox (`/tmp/ai-dev-team/sandbox-{timestamp}`):
- Prevents AI from modifying system files
- Enables git-based snapshots and rollbacks
- Prepares for Docker containerization in Phase 4

---

## 4. Application Entry Point

### File: `src/index.js`

This is where execution begins when you run `node src/index.js`.

### Initialization Sequence

```javascript
// Line 1-10: Import dependencies
import "dotenv/config";              // Load .env variables
import * as readline from "readline"; // Terminal I/O
import { initGemini } from "./utils/gemini.js";
import { buildGraph, createCheckpointer } from "./config/graph.js";

// Line 90-95: main() function starts
async function main() {
  printBanner();  // Display ASCII art header
  
  // Step 1: Initialize Gemini client
  initGemini(process.env.GEMINI_API_KEY);
  
  // Step 2: Create checkpointer (Redis or in-memory)
  const checkpointer = await createCheckpointer();
  
  // Step 3: Build the LangGraph
  const graph = buildGraph({ checkpointer });
  
  // Step 4: Get user requirement
  let requirement = process.argv.slice(2).join(" ");
  if (!requirement) {
    requirement = await askUser("Your idea: ");
  }
  
  // Step 5: Run the graph
  const finalState = await graph.invoke({
    userRequirement: requirement,
    tokenBudget: 2.0
  }, config);
  
  // Step 6: Display results
  printSpec(finalState.clarifiedSpec);
  printBlueprint(finalState.blueprint);
  printTokenSummary(finalState.tokenUsage);
}
```

### Command-Line Interface

```bash
# Method 1: Pass requirement as argument
node src/index.js "Build a todo app with categories"

# Method 2: Interactive prompt
node src/index.js
# (will ask: "Your idea: ")
```

### Environment Variables

Required in `.env`:
```
GEMINI_API_KEY=your_key_here
GEMINI_MODEL=gemini-2.5-flash    # Optional, defaults to this
TOKEN_BUDGET=2.0                  # Max cost in USD
REDIS_URL=redis://localhost:6379 # Optional
```

---

## 5. Execution Flow

### High-Level Flow Diagram

```
START
  ↓
[index.js main()]
  ↓
Initialize Gemini → Create Checkpointer → Build Graph
  ↓
graph.invoke({ userRequirement, tokenBudget })
  ↓
┌─────────────────────────────────────────────┐
│         LANGGRAPH EXECUTION                 │
├─────────────────────────────────────────────┤
│ START → pmAgent                             │
│           ↓                                 │
│    needs_clarification? → humanInput → pmAgent
│           ↓                                 │
│    spec_ready? → architectStep1             │
│                    ↓                        │
│                  architectStep2             │
│                    ↓                        │
│                  architectStep3             │
│                    ↓                        │
│                  architectStep4             │
│                    ↓                        │
│                  architectStep5             │
│                    ↓                        │
│                  blueprintValidator         │
│                    ↓                        │
│              valid? → plannerAgent          │
│              invalid? → back to step 2/3/4  │
│                    ↓                        │
│                  setupSandbox               │
│                    ↓                        │
│                  sandboxHealthCheck         │
│                    ↓                        │
│                   END                       │
└─────────────────────────────────────────────┘
  ↓
[index.js receives finalState]
  ↓
Display: Spec → Blueprint → Build Plan → Token Usage
  ↓
EXIT
```

### Detailed Step-by-Step Execution

#### Step 1: Application Startup (index.js:90-120)

1. **Print Banner** - Display ASCII art header
2. **Initialize Gemini** - Create AI client with API key
3. **Create Checkpointer** - Connect to Redis or use in-memory storage
4. **Build Graph** - Wire all nodes and edges together
5. **Get Requirement** - From CLI args or interactive prompt

#### Step 2: Graph Invocation (index.js:150-160)

```javascript
const finalState = await graph.invoke(
  {
    userRequirement: requirement,
    tokenBudget: 2.0
  },
  {
    configurable: {
      thread_id: `project-${Date.now()}`
    }
  }
);
```

This triggers the LangGraph execution engine, which:
- Starts at the START node
- Follows edges to the next node
- Executes each node function
- Saves state after each node (checkpointing)
- Continues until reaching END

#### Step 3: PM Agent Phase (pmAgent.js)

**First Invocation** (no conversation history):

```javascript
// pmAgent.js:45-50
if (state.pmConversation.length === 0) {
  userPrompt = `User's project requirement:\n"${state.userRequirement}"`;
}
```

The PM Agent analyzes the requirement and decides:

**Option A: Need Clarification**
```javascript
// pmAgent.js:70-85
if (response.status === "needs_clarification") {
  return {
    pmStatus: "needs_clarification",
    pmQuestions: ["Q1?", "Q2?", ...],
    pmConversation: [{ role: "pm", questions: [...] }]
  };
}
```

**Graph routing** (graph.js:50-54):
```javascript
graph.addConditionalEdges("pmAgent", (state) => {
  if (state.pmStatus === "needs_clarification") return "humanInput";
  if (state.pmStatus === "spec_ready") return "architectStep1";
});
```

**Option B: Spec Ready**
```javascript
// pmAgent.js:87-100
if (response.status === "spec_ready") {
  return {
    pmStatus: "spec_ready",
    clarifiedSpec: response.spec,
    currentPhase: "architect"
  };
}
```

#### Step 4: Human Input Phase (humanInput.js)

Only executes if PM needs clarification:

```javascript
// humanInput.js:40-70
export async function humanInputNode(state) {
  const questions = state.pmQuestions;
  
  // Display questions
  questions.forEach((q, i) => {
    console.log(`  ${i + 1}. ${q}`);
  });
  
  // Get user's answers
  const answer = await askUser("Your answers: ");
  
  // Return to state
  return {
    pmConversation: [{ role: "user", answers: answer }],
    pmStatus: "idle"  // Reset so PM processes answers
  };
}
```

**Graph routing**: Always returns to pmAgent (graph.js:56)

**Second PM Invocation** (with answers):

```javascript
// pmAgent.js:52-60
else {
  userPrompt = `Original requirement:\n"${state.userRequirement}"\n\n`;
  userPrompt += `Conversation so far:\n`;
  for (const entry of state.pmConversation) {
    // Include all Q&A history
  }
  userPrompt += `\nNow generate the FINAL spec...`;
}
```

This time, PM returns `spec_ready` and routes to Architect.


#### Step 5: Architect Phase (architectAgent.js)

The Architect builds the blueprint in 5 sequential steps:

**Step 1: Identify Entities** (architectStep1Node)

```javascript
// architectAgent.js:30-50
export async function architectStep1Node(state) {
  const result = await callGemini({
    systemPrompt: STEP1_PROMPT,  // "Identify entities and relationships"
    userPrompt: JSON.stringify(state.clarifiedSpec),
    agentName: "architectStep1"
  });
  
  return {
    blueprint: { entities: result.parsed.entities },
    tokenUsage: makeTokenDelta("architectStep1", result.tokens)
  };
}
```

**Output Example**:
```json
{
  "entities": [
    {
      "name": "User",
      "description": "Application users with authentication",
      "relationships": [
        { "target": "Todo", "type": "one-to-many", "description": "Users own todos" }
      ]
    },
    {
      "name": "Todo",
      "description": "Task items with title, description, due date"
    }
  ]
}
```

**Step 2: Design Database Schema** (architectStep2Node)

```javascript
// architectAgent.js:70-95
export async function architectStep2Node(state) {
  // Check if validator sent issues to fix
  const validationIssues = state.blueprintValidation?.issues || [];
  const fixContext = validationIssues.length > 0
    ? `\n\nPREVIOUS VALIDATION ISSUES TO FIX:\n${JSON.stringify(validationIssues)}`
    : "";
  
  const result = await callGemini({
    systemPrompt: STEP2_PROMPT,  // "Design database schema"
    userPrompt: `Entities:\n${JSON.stringify(state.blueprint.entities)}\n${fixContext}`
  });
  
  return {
    blueprint: { dbSchema: result.parsed },
    tokenUsage: makeTokenDelta("architectStep2", result.tokens)
  };
}
```

**Output Example**:
```json
{
  "databaseType": "PostgreSQL",
  "tables": [
    {
      "name": "users",
      "fields": [
        { "name": "id", "type": "UUID", "constraints": ["PRIMARY KEY"] },
        { "name": "email", "type": "VARCHAR(255)", "constraints": ["UNIQUE", "NOT NULL"] },
        { "name": "password_hash", "type": "VARCHAR(255)", "constraints": ["NOT NULL"] },
        { "name": "created_at", "type": "TIMESTAMP", "constraints": ["DEFAULT NOW()"] }
      ],
      "indexes": ["email"]
    }
  ]
}
```

**Step 3: Design API Endpoints** (architectStep3Node)

```javascript
// architectAgent.js:115-140
export async function architectStep3Node(state) {
  const result = await callGemini({
    systemPrompt: STEP3_PROMPT,  // "Design REST API endpoints"
    userPrompt: `DB Schema:\n${JSON.stringify(state.blueprint.dbSchema)}`
  });
  
  return {
    blueprint: { apiEndpoints: result.parsed.apiEndpoints },
    tokenUsage: makeTokenDelta("architectStep3", result.tokens)
  };
}
```

**Output Example**:
```json
{
  "apiEndpoints": [
    {
      "method": "POST",
      "path": "/api/auth/register",
      "description": "Create new user account",
      "requiresAuth": false,
      "requestBody": { "email": "string", "password": "string" },
      "responseBody": { "user": "object", "token": "string" },
      "relatedTable": "users"
    },
    {
      "method": "GET",
      "path": "/api/todos",
      "description": "Get all todos for authenticated user",
      "requiresAuth": true,
      "roleAccess": ["user"],
      "relatedTable": "todos"
    }
  ]
}
```

**Step 4: Design Frontend Pages** (architectStep4Node)

```javascript
// architectAgent.js:160-185
export async function architectStep4Node(state) {
  const result = await callGemini({
    systemPrompt: STEP4_PROMPT,  // "Design React pages"
    userPrompt: `API Endpoints:\n${JSON.stringify(state.blueprint.apiEndpoints)}`
  });
  
  return {
    blueprint: { frontendPages: result.parsed.frontendPages },
    tokenUsage: makeTokenDelta("architectStep4", result.tokens)
  };
}
```

**Output Example**:
```json
{
  "frontendPages": [
    {
      "name": "Dashboard",
      "route": "/dashboard",
      "requiresAuth": true,
      "components": [
        {
          "name": "TodoList",
          "description": "Displays all todos",
          "apiCalls": ["/api/todos"]
        },
        {
          "name": "AddTodoForm",
          "apiCalls": ["/api/todos"]
        }
      ]
    }
  ]
}
```

**Step 5: Folder Structure & Dependencies** (architectStep5Node)

```javascript
// architectAgent.js:205-230
export async function architectStep5Node(state) {
  const result = await callGemini({
    systemPrompt: STEP5_PROMPT,  // "Generate folder structure"
    userPrompt: `DB: ${state.blueprint.dbSchema.databaseType}\nAPIs: ${state.blueprint.apiEndpoints.length}`
  });
  
  return {
    blueprint: {
      folderStructure: result.parsed.folderStructure,
      dependencies: result.parsed.dependencies
    },
    tokenUsage: makeTokenDelta("architectStep5", result.tokens)
  };
}
```

**Output Example**:
```json
{
  "folderStructure": "backend/\n├── src/\n│   ├── models/\n│   ├── routes/\n│   └── middleware/\nfrontend/\n├── src/\n│   ├── pages/\n│   └── components/",
  "dependencies": {
    "backend": {
      "dependencies": {
        "express": "^4.18.2",
        "pg": "^8.11.0",
        "bcryptjs": "^2.4.3",
        "jsonwebtoken": "^9.0.0"
      }
    },
    "frontend": {
      "dependencies": {
        "react": "^18.2.0",
        "react-router-dom": "^6.14.0"
      }
    }
  }
}
```

#### Step 6: Blueprint Validation (blueprintValidator.js)

**Pure JavaScript validation** - No LLM calls:

```javascript
// blueprintValidator.js:50-60
export async function blueprintValidatorNode(state) {
  const { dbSchema, apiEndpoints, frontendPages, entities } = state.blueprint;
  const issues = [];
  
  // CHECK 1: Every entity has a DB table
  for (const entity of entities) {
    const hasTable = dbSchema.tables.some(t => 
      t.name.toLowerCase().includes(entity.name.toLowerCase())
    );
    if (!hasTable) {
      issues.push({
        type: "missing_table",
        severity: "error",
        fixTarget: "architectStep2",
        message: `Entity "${entity.name}" has no matching DB table`
      });
    }
  }
  
  // CHECK 2: Foreign keys reference existing tables
  for (const table of dbSchema.tables) {
    for (const fk of table.foreignKeys || []) {
      const refTable = fk.references.match(/^(\w+)\(/)[1];
      if (!dbSchema.tables.some(t => t.name === refTable)) {
        issues.push({
          type: "invalid_foreign_key",
          severity: "error",
          fixTarget: "architectStep2",
          message: `FK references non-existent table "${refTable}"`
        });
      }
    }
  }
  
  // CHECK 3: API endpoints reference existing tables
  for (const endpoint of apiEndpoints) {
    if (endpoint.relatedTable) {
      if (!dbSchema.tables.some(t => t.name === endpoint.relatedTable)) {
        issues.push({
          type: "orphan_endpoint",
          severity: "error",
          fixTarget: "architectStep3",
          message: `API "${endpoint.path}" references non-existent table`
        });
      }
    }
  }
  
  // CHECK 4: Frontend pages call existing APIs
  for (const page of frontendPages) {
    for (const comp of page.components || []) {
      for (const apiCall of comp.apiCalls || []) {
        if (!apiEndpoints.some(e => e.path === apiCall)) {
          issues.push({
            type: "missing_api",
            severity: "warning",
            fixTarget: "architectStep3",
            message: `Page "${page.name}" calls non-existent API "${apiCall}"`
          });
        }
      }
    }
  }
  
  // CHECK 5: Auth consistency
  // CHECK 6: No orphan tables
  // ... (similar logic)
  
  if (issues.length === 0) {
    return {
      blueprintValidation: { isValid: true, issues: [], validationCycles: 1 },
      currentPhase: "planner"
    };
  }
  
  // If issues found, route back to fix
  return {
    blueprintValidation: { isValid: false, issues, validationCycles: 1 }
  };
}
```

**Routing Logic** (blueprintValidator.js:280-300):

```javascript
export function blueprintValidatorRouter(state) {
  if (state.blueprintValidation?.isValid) {
    return "__end__";  // Proceed to planner
  }
  
  // Find first error and route to its fix target
  const errors = state.blueprintValidation.issues.filter(i => i.severity === "error");
  if (errors.length > 0) {
    const target = errors[0].fixTarget;  // "architectStep2", "architectStep3", etc.
    return target;
  }
  
  return "__end__";
}
```

**Validation Loop Example**:

1. Architect Step 2 creates table "user" (singular)
2. Architect Step 3 creates API referencing "users" (plural)
3. Validator detects mismatch → routes back to Step 3
4. Step 3 re-runs with validation issues in context
5. Fixes the table reference
6. Validator passes → proceeds to Planner

**Max Cycles**: 2 (prevents infinite loops)

#### Step 7: Planner Phase (plannerAgent.js)

```javascript
// plannerAgent.js:60-90
export async function plannerAgentNode(state) {
  // Build concise summary (save tokens)
  const blueprintSummary = {
    databaseType: state.blueprint.dbSchema.databaseType,
    tables: state.blueprint.dbSchema.tables.map(t => ({
      name: t.name,
      fieldCount: t.fields.length
    })),
    apiEndpoints: state.blueprint.apiEndpoints.map(e => ({
      method: e.method,
      path: e.path
    })),
    // ... etc
  };
  
  const result = await callGemini({
    systemPrompt: PLANNER_PROMPT,  // "Create ordered build plan"
    userPrompt: `Blueprint:\n${JSON.stringify(blueprintSummary)}`
  });
  
  return {
    taskQueue: result.parsed,
    currentPhaseIndex: 0,
    currentTaskIndex: 0,
    tokenUsage: makeTokenDelta("plannerAgent", result.tokens)
  };
}
```

**Output Example**:
```json
{
  "phases": [
    {
      "phaseNumber": 1,
      "phaseName": "setup",
      "tasks": [
        {
          "taskId": "setup-1",
          "title": "Database connection and User model",
          "filesToCreate": [
            "/backend/src/config/db.js",
            "/backend/src/models/User.js"
          ],
          "filesNeeded": [],
          "canParallelize": false,
          "estimatedTokens": 500
        }
      ]
    },
    {
      "phaseNumber": 2,
      "phaseName": "middleware",
      "tasks": [
        {
          "taskId": "middleware-1",
          "title": "JWT authentication middleware",
          "filesToCreate": ["/backend/src/middleware/auth.js"],
          "filesNeeded": ["/backend/src/models/User.js"],
          "canParallelize": false
        }
      ]
    }
  ],
  "totalTasks": 15,
  "estimatedTotalTokens": 8000
}
```

#### Step 8: Sandbox Setup (setupSandbox.js)

```javascript
// setupSandbox.js:10-25
export async function setupSandboxNode(state) {
  const { folderStructure, dependencies } = state.blueprint;
  
  const sandboxId = await createSandbox(folderStructure, dependencies);
  
  return {
    sandboxId,
    currentPhase: "sandbox"
  };
}
```

**What createSandbox does** (sandboxManager.js:50-150):

1. **Create base directory**: `/tmp/ai-dev-team/sandbox-{timestamp}`
2. **Create standard folders**:
   - `backend/src/models/`
   - `backend/src/routes/`
   - `backend/src/middleware/`
   - `frontend/src/pages/`
   - `frontend/src/components/`
3. **Parse folderStructure** and create additional directories
4. **Write package.json** files for backend and frontend
5. **Write .env.example** files
6. **Initialize Git**:
   ```bash
   git init
   git add -A
   git commit -m "Initial scaffold"
   git tag v0.0.0
   ```
7. **Store sandbox metadata** in memory Map

#### Step 9: Sandbox Health Check (sandboxHealthCheck.js)

```javascript
// sandboxHealthCheck.js:15-40
export async function sandboxHealthCheckNode(state) {
  const result = await healthCheck(state.sandboxId);
  
  if (result.healthy) {
    return { sandboxHealthy: true };
  }
  
  return {
    sandboxHealthy: false,
    error: `Sandbox unhealthy: ${result.failures.join("; ")}`
  };
}
```

**Health Checks** (sandboxManager.js:160-210):

1. Backend directory exists
2. Frontend directory exists
3. package.json files exist
4. Git is initialized
5. Standard directories exist (src/models, src/routes, etc.)
6. Disk space available (>100MB)

**Routing** (sandboxHealthCheck.js:45-55):
```javascript
export function sandboxHealthRouter(state) {
  if (state.sandboxHealthy) {
    return "__end__";  // Phase 3 complete
  }
  return "__end__";  // For now, just end with error
}
```

#### Step 10: Return to index.js

```javascript
// index.js:150-200
const finalState = await graph.invoke(...);

// Display results
if (finalState.clarifiedSpec) {
  printSpec(finalState.clarifiedSpec);
}

if (finalState.blueprint?.entities?.length) {
  printBlueprint(finalState.blueprint, finalState.blueprintValidation);
}

if (finalState.taskQueue?.phases?.length) {
  // Print build plan
  for (const phase of finalState.taskQueue.phases) {
    console.log(`Phase ${phase.phaseNumber}: ${phase.phaseName}`);
    phase.tasks.forEach(t => {
      console.log(`  ${t.taskId}: ${t.title}`);
    });
  }
}

if (finalState.sandboxId) {
  console.log(`Sandbox: ${finalState.sandboxId}`);
  console.log(`Healthy: ${finalState.sandboxHealthy ? "Yes" : "No"}`);
}

printTokenSummary(finalState.tokenUsage);
```

---

## 6. Core Components

### 6.1 LangGraph State Machine (graph.js)

**Purpose**: Wire all nodes together and define execution flow.

```javascript
// graph.js:20-90
export function buildGraph(options = {}) {
  const graph = new StateGraph(AgentState);
  
  // Add all nodes
  graph.addNode("pmAgent", pmAgentNode);
  graph.addNode("humanInput", humanInputNode);
  graph.addNode("architectStep1", architectStep1Node);
  // ... etc
  
  // Define edges
  graph.addEdge(START, "pmAgent");
  
  // Conditional routing
  graph.addConditionalEdges("pmAgent", (state) => {
    if (state.pmStatus === "needs_clarification") return "humanInput";
    if (state.pmStatus === "spec_ready") return "architectStep1";
    return END;
  });
  
  // Sequential architect chain
  graph.addEdge("architectStep1", "architectStep2");
  graph.addEdge("architectStep2", "architectStep3");
  graph.addEdge("architectStep3", "architectStep4");
  graph.addEdge("architectStep4", "architectStep5");
  graph.addEdge("architectStep5", "blueprintValidator");
  
  // Validator routing (can loop back)
  graph.addConditionalEdges("blueprintValidator", blueprintValidatorRouter, {
    __end__: "plannerAgent",
    architectStep2: "architectStep2",
    architectStep3: "architectStep3",
    architectStep4: "architectStep4"
  });
  
  // Planner → Sandbox
  graph.addEdge("plannerAgent", "setupSandbox");
  graph.addEdge("setupSandbox", "sandboxHealthCheck");
  
  graph.addConditionalEdges("sandboxHealthCheck", sandboxHealthRouter, {
    __end__: END,
    setupSandbox: "setupSandbox"
  });
  
  // Compile with checkpointer
  return graph.compile({ checkpointer: options.checkpointer || new MemorySaver() });
}
```

**Key Concepts**:

- **START**: Special node where execution begins
- **END**: Special node where execution terminates
- **addEdge(A, B)**: Always go from A to B
- **addConditionalEdges(A, routerFn, mapping)**: Call routerFn(state) to decide next node
- **compile()**: Converts graph definition into executable state machine


### 6.2 Checkpointing System

**Purpose**: Save state after every node so execution can resume after crashes.

```javascript
// graph.js:92-105
export async function createCheckpointer() {
  const redisUrl = process.env.REDIS_URL;
  
  if (redisUrl) {
    try {
      const { RedisSaver } = await import("@langchain/langgraph-checkpoint-redis");
      const saver = await RedisSaver.fromUrl(redisUrl);
      return saver;
    } catch (error) {
      console.warn(`Redis failed: ${error.message}. Using in-memory.`);
    }
  }
  
  return new MemorySaver();  // Fallback
}
```

**How it works**:

1. After each node completes, LangGraph serializes the state
2. Checkpointer saves it to Redis (or memory) with thread_id as key
3. If process crashes, you can resume:
   ```javascript
   const state = await graph.getState(config);
   await graph.invoke(null, config);  // Resumes from last checkpoint
   ```

**Redis vs. In-Memory**:

| Feature | Redis | In-Memory |
|---------|-------|-----------|
| Persistence | Survives restarts | Lost on exit |
| Multi-process | Shared state | Isolated |
| Setup | Requires Redis server | Zero config |
| Use case | Production | Development/testing |

---

## 7. State Management

### 7.1 State Schema (state.js)

**Complete state definition** for all 30 planned nodes (Phase 1-7):

```javascript
// state.js:20-250
export const AgentState = Annotation.Root({
  // User input
  userRequirement: Annotation({
    reducer: (_, y) => y ?? "",
    default: () => ""
  }),
  
  // PM Agent
  pmStatus: Annotation({
    reducer: (_, y) => y ?? "idle",
    default: () => "idle"
  }),
  pmQuestions: Annotation({
    reducer: (_, y) => y ?? [],
    default: () => []
  }),
  pmConversation: Annotation({
    reducer: (existing, incoming) => {
      if (!incoming) return existing;
      if (Array.isArray(incoming)) return [...existing, ...incoming];
      return [...existing, incoming];
    },
    default: () => []
  }),
  clarifiedSpec: Annotation({
    reducer: (_, y) => y ?? null,
    default: () => null
  }),
  
  // Architect Agent
  blueprint: Annotation({
    reducer: (existing, incoming) => {
      if (!incoming) return existing;
      return { ...existing, ...incoming };  // Merge
    },
    default: () => ({
      entities: [],
      dbSchema: {},
      apiEndpoints: [],
      frontendPages: [],
      folderStructure: "",
      dependencies: {}
    })
  }),
  
  // Blueprint Validator
  blueprintValidation: Annotation({
    reducer: (_, y) => y ?? { isValid: false, issues: [], validationCycles: 0 },
    default: () => ({ isValid: false, issues: [], validationCycles: 0 })
  }),
  
  // Planner Agent
  taskQueue: Annotation({
    reducer: (_, y) => y ?? { phases: [] },
    default: () => ({ phases: [] })
  }),
  
  // Sandbox
  sandboxId: Annotation({
    reducer: (_, y) => y ?? "",
    default: () => ""
  }),
  sandboxHealthy: Annotation({
    reducer: (_, y) => y ?? false,
    default: () => false
  }),
  
  // Token tracking
  tokenUsage: Annotation({
    reducer: (existing, incoming) => {
      if (!incoming) return existing;
      return {
        calls: [...(existing.calls || []), ...(incoming.newCalls || [])],
        totalInput: existing.totalInput + (incoming.addedInput || 0),
        totalOutput: existing.totalOutput + (incoming.addedOutput || 0),
        estimatedCost: existing.estimatedCost + (incoming.addedCost || 0)
      };
    },
    default: () => ({
      calls: [],
      totalInput: 0,
      totalOutput: 0,
      estimatedCost: 0.0
    })
  }),
  tokenBudget: Annotation({
    reducer: (_, y) => y ?? 2.0,
    default: () => 2.0
  }),
  
  // Control
  currentPhase: Annotation({
    reducer: (_, y) => y ?? "pm",
    default: () => "pm"
  }),
  error: Annotation({
    reducer: (_, y) => y ?? null,
    default: () => null
  })
  
  // ... (more fields for Phase 4-7)
});
```

### 7.2 Reducer Patterns

**Pattern 1: Last Write Wins** (simple values)

```javascript
userRequirement: Annotation({
  reducer: (_, y) => y ?? "",  // Ignore old, use new
  default: () => ""
})
```

**Pattern 2: Array Accumulation**

```javascript
pmConversation: Annotation({
  reducer: (existing, incoming) => {
    if (!incoming) return existing;
    if (Array.isArray(incoming)) return [...existing, ...incoming];
    return [...existing, incoming];
  },
  default: () => []
})
```

**Pattern 3: Object Merging**

```javascript
blueprint: Annotation({
  reducer: (existing, incoming) => {
    if (!incoming) return existing;
    return { ...existing, ...incoming };  // Shallow merge
  },
  default: () => ({ entities: [], dbSchema: {} })
})
```

**Pattern 4: Delta Accumulation** (token tracking)

```javascript
tokenUsage: Annotation({
  reducer: (existing, incoming) => {
    if (!incoming) return existing;
    return {
      calls: [...existing.calls, ...incoming.newCalls],
      totalInput: existing.totalInput + incoming.addedInput,
      totalOutput: existing.totalOutput + incoming.addedOutput,
      estimatedCost: existing.estimatedCost + incoming.addedCost
    };
  },
  default: () => ({ calls: [], totalInput: 0, totalOutput: 0, estimatedCost: 0 })
})
```

### 7.3 Why Define All Fields Upfront?

Even though Phase 3 only uses ~15 fields, we define all 30+ fields now because:

1. **Checkpoint Compatibility**: Changing state shape later breaks existing checkpoints
2. **Type Safety**: Agents know what fields exist
3. **Documentation**: State schema serves as system documentation
4. **Future-Proofing**: Phase 4-7 agents can use fields immediately

---

## 8. Agent Details

### 8.1 PM Agent (pmAgent.js)

**Role**: Convert vague requirements into detailed specifications.

**System Prompt** (lines 5-60):
- Asks 5-8 clarifying questions if requirement is ambiguous
- Makes reasonable assumptions for minor details
- Focuses on business logic, not tech stack (tech is fixed)
- Returns either `needs_clarification` or `spec_ready`

**Input**: `state.userRequirement`, `state.pmConversation`

**Output**:
```javascript
{
  pmStatus: "needs_clarification" | "spec_ready",
  pmQuestions: ["Q1?", "Q2?"],  // If needs clarification
  clarifiedSpec: { appName, features, pages, ... },  // If ready
  pmConversation: [{ role: "pm" | "user", ... }],
  tokenUsage: { newCalls: [...], addedInput: 100, ... }
}
```

**LLM Calls**: 1-2 (1 if requirement is clear, 2 if needs Q&A)

### 8.2 Architect Agent (architectAgent.js)

**Role**: Design complete system architecture in 5 steps.

**Step 1: Entities** (lines 30-50)
- Input: clarifiedSpec
- Output: List of entities with relationships
- LLM: Yes

**Step 2: Database Schema** (lines 70-95)
- Input: entities, clarifiedSpec, validationIssues (if looping back)
- Output: Tables with fields, types, constraints, foreign keys, indexes
- LLM: Yes

**Step 3: API Endpoints** (lines 115-140)
- Input: dbSchema, clarifiedSpec, validationIssues
- Output: REST endpoints with method, path, auth, request/response bodies
- LLM: Yes

**Step 4: Frontend Pages** (lines 160-185)
- Input: apiEndpoints, clarifiedSpec, validationIssues
- Output: React pages with routes, components, API calls
- LLM: Yes

**Step 5: Folder Structure** (lines 205-230)
- Input: dbSchema, apiEndpoints, frontendPages
- Output: Folder tree + package.json dependencies
- LLM: Yes

**Total LLM Calls**: 5 (one per step)

**Key Feature**: Each step can be re-run independently if validator finds issues.

### 8.3 Blueprint Validator (blueprintValidator.js)

**Role**: Cross-validate architecture for consistency.

**Validation Checks**:

1. **Entity-Table Mapping** (lines 50-70)
   - Every entity has a corresponding DB table
   - Handles plural/singular variations (user → users)

2. **Foreign Key Integrity** (lines 75-95)
   - Every FK references an existing table
   - Parses "table_name(field)" format

3. **API-Table Mapping** (lines 100-120)
   - Every API's relatedTable exists in schema

4. **Page-API Mapping** (lines 125-155)
   - Every API call from frontend exists in endpoints
   - Handles parameterized routes (/api/todos/:id)

5. **Auth Consistency** (lines 160-180)
   - Pages calling auth-required APIs must have requiresAuth: true

6. **Orphan Detection** (lines 185-210)
   - No tables without APIs
   - No APIs without tables

**Output**:
```javascript
{
  blueprintValidation: {
    isValid: true | false,
    issues: [
      {
        type: "missing_table" | "invalid_foreign_key" | ...,
        severity: "error" | "warning",
        fixTarget: "architectStep2" | "architectStep3" | "architectStep4",
        message: "Human-readable description"
      }
    ],
    validationCycles: 1
  }
}
```

**Routing Logic** (lines 280-300):
- If valid → return "__end__" (proceed to planner)
- If errors → return fixTarget of first error
- Max 2 cycles → force proceed with warnings

**LLM Calls**: 0 (pure JavaScript logic)

### 8.4 Planner Agent (plannerAgent.js)

**Role**: Create ordered build plan with dependency tracking.

**System Prompt** (lines 15-70):
- Mandatory phase order: setup → middleware → backend → frontend → integration
- Each task creates 1-3 files
- Track dependencies (filesNeeded vs. filesToCreate)
- Flag parallelizable tasks
- Estimate token usage per task

**Input**: blueprint (summarized to save tokens), clarifiedSpec

**Output**:
```javascript
{
  taskQueue: {
    phases: [
      {
        phaseNumber: 1,
        phaseName: "setup",
        tasks: [
          {
            taskId: "setup-1",
            title: "Database connection",
            filesToCreate: ["/backend/src/config/db.js"],
            filesNeeded: [],
            canParallelize: false,
            estimatedTokens: 500
          }
        ],
        verificationCommand: "node backend/src/config/db.js"
      }
    ],
    totalTasks: 15,
    estimatedTotalTokens: 8000
  }
}
```

**LLM Calls**: 1

---

## 9. Utility Systems

### 9.1 Gemini API Wrapper (gemini.js)

**Purpose**: Centralized LLM calls with token tracking and error handling.

**Core Function**: callGemini (lines 30-120)

```javascript
export async function callGemini({
  systemPrompt,
  userPrompt,
  agentName = "unknown",
  currentCost = 0,
  tokenBudget = 2.0,
  model = null
}) {
  // 1. Budget check
  if (currentCost >= tokenBudget) {
    throw new Error("TOKEN_BUDGET_EXCEEDED");
  }
  
  // 2. Build full prompt
  const fullPrompt = `${systemPrompt}\n\n---\n\nINPUT:\n${userPrompt}\n\n---\n\nIMPORTANT: Respond with ONLY valid JSON.`;
  
  // 3. Call Gemini API (with retries)
  for (let attempt = 1; attempt <= 3; attempt++) {
    try {
      const response = await client.models.generateContent({
        model: modelName,
        contents: fullPrompt,
        config: { responseMimeType: "application/json" }
      });
      
      const rawText = response.text;
      
      // 4. Extract token counts
      const inputTokens = response.usageMetadata?.promptTokenCount || Math.ceil(fullPrompt.length / 4);
      const outputTokens = response.usageMetadata?.candidatesTokenCount || Math.ceil(rawText.length / 4);
      
      // 5. Calculate cost (Gemini 2.5 Flash pricing)
      const cost = (inputTokens / 1_000_000) * 0.15 + (outputTokens / 1_000_000) * 0.60;
      
      // 6. Parse JSON
      let parsed = JSON.parse(rawText.trim());
      
      // 7. Return everything
      return {
        parsed,
        raw: rawText,
        tokens: { input: inputTokens, output: outputTokens, cost }
      };
      
    } catch (error) {
      if (attempt === 3) throw error;
      await new Promise(r => setTimeout(r, Math.pow(2, attempt) * 1000));  // Exponential backoff
    }
  }
}
```

**Key Features**:

1. **Budget Enforcement**: Throws error if cost exceeds limit
2. **Automatic Retries**: 3 attempts with exponential backoff
3. **JSON Parsing**: Handles markdown code blocks (```json)
4. **Token Tracking**: Uses API metadata or estimates
5. **Cost Calculation**: Gemini 2.5 Flash rates ($0.15/1M input, $0.60/1M output)

**Helper Function**: makeTokenDelta (lines 125-135)

```javascript
export function makeTokenDelta(agentName, tokens) {
  return {
    newCalls: [{
      agent: agentName,
      inputTokens: tokens.input,
      outputTokens: tokens.output,
      timestamp: Date.now()
    }],
    addedInput: tokens.input,
    addedOutput: tokens.output,
    addedCost: tokens.cost
  };
}
```

**Why Deltas?** Prevents duplication in state reducer. Agent returns only NEW data, reducer adds it to existing totals.

### 9.2 Token Tracker (tokenTracker.js)

**Purpose**: Display token usage summary at end of execution.

```javascript
// tokenTracker.js:10-40
export function printTokenSummary(tokenUsage) {
  if (!tokenUsage || tokenUsage.calls.length === 0) {
    console.log("\n📊 No LLM calls made yet.\n");
    return;
  }
  
  // Group by agent
  const byAgent = {};
  for (const call of tokenUsage.calls) {
    if (!byAgent[call.agent]) {
      byAgent[call.agent] = { calls: 0, input: 0, output: 0 };
    }
    byAgent[call.agent].calls++;
    byAgent[call.agent].input += call.inputTokens;
    byAgent[call.agent].output += call.outputTokens;
  }
  
  // Print table
  console.log("╔══════════════════════════════════════════════════════════╗");
  console.log("║               📊 TOKEN USAGE SUMMARY                    ║");
  console.log("╠══════════════════════════════════════════════════════════╣");
  
  for (const [agent, data] of Object.entries(byAgent)) {
    const cost = (data.input / 1_000_000) * 0.15 + (data.output / 1_000_000) * 0.60;
    console.log(`║  ${agent.padEnd(22)} ${data.calls} call(s)  ~$${cost.toFixed(4)} ║`);
  }
  
  console.log("╠══════════════════════════════════════════════════════════╣");
  console.log(`║  TOTAL: ${tokenUsage.calls.length} calls | ${totalTokens} tokens | ~$${totalCost.toFixed(4)}  ║`);
  console.log("╚══════════════════════════════════════════════════════════╝\n");
}
```

**Example Output**:
```
╔══════════════════════════════════════════════════════════╗
║               📊 TOKEN USAGE SUMMARY                    ║
╠══════════════════════════════════════════════════════════╣
║  pmAgent                2 call(s)  ~$0.0045 ║
║  architectStep1         1 call(s)  ~$0.0012 ║
║  architectStep2         1 call(s)  ~$0.0018 ║
║  architectStep3         1 call(s)  ~$0.0015 ║
║  architectStep4         1 call(s)  ~$0.0013 ║
║  architectStep5         1 call(s)  ~$0.0010 ║
║  plannerAgent           1 call(s)  ~$0.0020 ║
╠══════════════════════════════════════════════════════════╣
║  TOTAL: 8 calls | 12450 tokens | ~$0.0133  ║
╚══════════════════════════════════════════════════════════╝
```

### 9.3 Sandbox Manager (sandboxManager.js)

**Purpose**: Isolated workspace for AI-generated code.

**Core Functions**:

#### createSandbox (lines 50-150)

```javascript
export async function createSandbox(folderStructure, dependencies) {
  const sandboxId = `sandbox-${Date.now()}`;
  const sandboxPath = path.join(os.tmpdir(), "ai-dev-team", sandboxId);
  
  // 1. Create base directories
  fs.mkdirSync(sandboxPath, { recursive: true });
  fs.mkdirSync(path.join(sandboxPath, "backend"), { recursive: true });
  fs.mkdirSync(path.join(sandboxPath, "frontend"), { recursive: true });
  
  // 2. Create standard subdirectories
  const backendDirs = ["src", "src/models", "src/routes", "src/middleware", "src/config", "src/utils"];
  const frontendDirs = ["src", "src/pages", "src/components", "src/hooks", "src/context", "src/utils"];
  
  backendDirs.forEach(d => fs.mkdirSync(path.join(sandboxPath, "backend", d), { recursive: true }));
  frontendDirs.forEach(d => fs.mkdirSync(path.join(sandboxPath, "frontend", d), { recursive: true }));
  
  // 3. Parse folderStructure string and create additional folders
  if (typeof folderStructure === "string") {
    const lines = folderStructure.split("\n");
    for (const line of lines) {
      const match = line.match(/(?:├──|└──)\s*(.+)/);
      if (match) {
        const item = match[1].trim();
        if (!item.includes(".")) {  // Directory (no extension)
          fs.mkdirSync(path.join(sandboxPath, item), { recursive: true });
        }
      }
    }
  }
  
  // 4. Write package.json files
  if (dependencies?.backend) {
    const backendPkg = {
      name: dependencies.backend.name || "backend",
      version: "1.0.0",
      type: "module",
      dependencies: dependencies.backend.dependencies || {}
    };
    fs.writeFileSync(
      path.join(sandboxPath, "backend", "package.json"),
      JSON.stringify(backendPkg, null, 2)
    );
  }
  
  // 5. Initialize Git
  execSync("git init", { cwd: sandboxPath, stdio: "pipe" });
  execSync("git add -A", { cwd: sandboxPath, stdio: "pipe" });
  execSync('git commit -m "Initial scaffold" --allow-empty', { cwd: sandboxPath, stdio: "pipe" });
  execSync("git tag v0.0.0", { cwd: sandboxPath, stdio: "pipe" });
  
  // 6. Store metadata
  sandboxes.set(sandboxId, {
    path: sandboxPath,
    backendPath: path.join(sandboxPath, "backend"),
    frontendPath: path.join(sandboxPath, "frontend"),
    createdAt: Date.now(),
    snapshotCount: 0
  });
  
  return sandboxId;
}
```

#### healthCheck (lines 160-210)

```javascript
export async function healthCheck(sandboxId) {
  const sandbox = sandboxes.get(sandboxId);
  if (!sandbox) return { healthy: false, failures: ["Sandbox not found"] };
  
  const failures = [];
  
  // Check 1: Directories exist
  if (!fs.existsSync(sandbox.backendPath)) failures.push("Backend directory missing");
  if (!fs.existsSync(sandbox.frontendPath)) failures.push("Frontend directory missing");
  
  // Check 2: package.json exists
  if (!fs.existsSync(path.join(sandbox.backendPath, "package.json"))) {
    failures.push("Backend package.json missing");
  }
  
  // Check 3: Git initialized
  try {
    execSync("git status", { cwd: sandbox.path, stdio: "pipe" });
  } catch (e) {
    failures.push("Git not initialized");
  }
  
  // Check 4: Standard directories
  const requiredDirs = ["backend/src", "backend/src/models", "frontend/src", "frontend/src/pages"];
  for (const dir of requiredDirs) {
    if (!fs.existsSync(path.join(sandbox.path, dir))) {
      failures.push(`Missing directory: ${dir}`);
    }
  }
  
  return {
    healthy: failures.length === 0,
    failures,
    sandboxPath: sandbox.path
  };
}
```

#### Other Functions

- **writeFile(sandboxId, filePath, content)**: Write file to sandbox
- **readFile(sandboxId, filePath)**: Read file from sandbox
- **executeCommand(sandboxId, command)**: Run shell command in sandbox
- **snapshot(sandboxId, message)**: Git commit + tag (v0.1.0, v0.2.0, etc.)
- **rollback(sandboxId, tag)**: Git checkout to previous snapshot
- **getFileList(sandboxId)**: List all files (excluding node_modules, .git)
- **destroySandbox(sandboxId)**: Delete sandbox directory

**Why Git?** Enables snapshots and rollbacks for debugging in Phase 5.


---

## 10. Data Flow

### 10.1 Complete Data Flow Diagram

```
USER INPUT
    ↓
[index.js] Captures requirement
    ↓
[graph.invoke()] Starts state machine
    ↓
┌─────────────────────────────────────────────────────────────┐
│                    STATE OBJECT                             │
│  { userRequirement, pmStatus, pmQuestions, clarifiedSpec,   │
│    blueprint, taskQueue, sandboxId, tokenUsage, ... }       │
└─────────────────────────────────────────────────────────────┘
    ↓
[pmAgent] Reads: userRequirement, pmConversation
    ↓ Calls Gemini API
    ↓ Writes: pmStatus, pmQuestions, clarifiedSpec, tokenUsage
    ↓
STATE UPDATED (checkpoint saved)
    ↓
[Conditional Edge] Reads: pmStatus
    ↓
    ├─ "needs_clarification" → [humanInput]
    │       ↓ Reads: pmQuestions
    │       ↓ Terminal I/O
    │       ↓ Writes: pmConversation (user answers)
    │       ↓ Routes back to [pmAgent]
    │
    └─ "spec_ready" → [architectStep1]
            ↓ Reads: clarifiedSpec
            ↓ Calls Gemini API
            ↓ Writes: blueprint.entities, tokenUsage
            ↓
        [architectStep2]
            ↓ Reads: blueprint.entities, blueprintValidation.issues
            ↓ Calls Gemini API
            ↓ Writes: blueprint.dbSchema, tokenUsage
            ↓
        [architectStep3]
            ↓ Reads: blueprint.dbSchema, blueprintValidation.issues
            ↓ Calls Gemini API
            ↓ Writes: blueprint.apiEndpoints, tokenUsage
            ↓
        [architectStep4]
            ↓ Reads: blueprint.apiEndpoints, blueprintValidation.issues
            ↓ Calls Gemini API
            ↓ Writes: blueprint.frontendPages, tokenUsage
            ↓
        [architectStep5]
            ↓ Reads: blueprint (all parts)
            ↓ Calls Gemini API
            ↓ Writes: blueprint.folderStructure, blueprint.dependencies, tokenUsage
            ↓
        [blueprintValidator]
            ↓ Reads: blueprint (all parts)
            ↓ Pure JavaScript validation (no API)
            ↓ Writes: blueprintValidation.isValid, blueprintValidation.issues
            ↓
        [Conditional Edge] Reads: blueprintValidation.isValid
            ↓
            ├─ isValid=true → [plannerAgent]
            │       ↓ Reads: blueprint, clarifiedSpec
            │       ↓ Calls Gemini API
            │       ↓ Writes: taskQueue, tokenUsage
            │       ↓
            │   [setupSandbox]
            │       ↓ Reads: blueprint.folderStructure, blueprint.dependencies
            │       ↓ Filesystem operations (no API)
            │       ↓ Writes: sandboxId
            │       ↓
            │   [sandboxHealthCheck]
            │       ↓ Reads: sandboxId
            │       ↓ Filesystem checks (no API)
            │       ↓ Writes: sandboxHealthy
            │       ↓
            │   END
            │
            └─ isValid=false → Routes back to architectStep2/3/4
                    ↓ (validation loop, max 2 cycles)
                    ↓ Re-runs with issues in context
                    ↓ Eventually proceeds to plannerAgent
    ↓
[graph.invoke() returns finalState]
    ↓
[index.js] Reads: finalState.clarifiedSpec, finalState.blueprint, 
                  finalState.taskQueue, finalState.sandboxId, 
                  finalState.tokenUsage
    ↓
[Display Functions] Format and print results
    ↓
EXIT
```

### 10.2 State Transitions

**Phase 1: PM Agent**

```
Initial State:
{
  userRequirement: "Build a todo app",
  pmStatus: "idle",
  pmConversation: [],
  tokenUsage: { calls: [], totalInput: 0, totalOutput: 0, estimatedCost: 0 }
}

After pmAgent (needs clarification):
{
  userRequirement: "Build a todo app",
  pmStatus: "needs_clarification",
  pmQuestions: ["Should todos have categories?", "Do you need user authentication?"],
  pmConversation: [{ role: "pm", questions: [...] }],
  tokenUsage: { calls: [{ agent: "pmAgent", inputTokens: 150, outputTokens: 80 }], totalInput: 150, totalOutput: 80, estimatedCost: 0.0007 }
}

After humanInput:
{
  ...previous state,
  pmStatus: "idle",
  pmConversation: [
    { role: "pm", questions: [...] },
    { role: "user", answers: "Yes to both" }
  ]
}

After pmAgent (second call):
{
  ...previous state,
  pmStatus: "spec_ready",
  clarifiedSpec: {
    appName: "todo-app",
    features: [...],
    pages: [...],
    authRequired: true
  },
  tokenUsage: { calls: [..., { agent: "pmAgent", inputTokens: 200, outputTokens: 300 }], totalInput: 350, totalOutput: 380, estimatedCost: 0.0018 }
}
```

**Phase 2: Architect Agent**

```
After architectStep1:
{
  ...previous state,
  blueprint: {
    entities: [{ name: "User", ... }, { name: "Todo", ... }],
    dbSchema: {},
    apiEndpoints: [],
    frontendPages: [],
    folderStructure: "",
    dependencies: {}
  }
}

After architectStep2:
{
  ...previous state,
  blueprint: {
    entities: [...],
    dbSchema: { databaseType: "PostgreSQL", tables: [...] },
    apiEndpoints: [],
    frontendPages: [],
    folderStructure: "",
    dependencies: {}
  }
}

... (steps 3, 4, 5 progressively fill blueprint)

After architectStep5:
{
  ...previous state,
  blueprint: {
    entities: [...],
    dbSchema: {...},
    apiEndpoints: [...],
    frontendPages: [...],
    folderStructure: "backend/\n├── src/\n...",
    dependencies: { backend: {...}, frontend: {...} }
  }
}
```

**Phase 2.5: Validation**

```
After blueprintValidator (with issues):
{
  ...previous state,
  blueprintValidation: {
    isValid: false,
    issues: [
      {
        type: "orphan_endpoint",
        severity: "error",
        fixTarget: "architectStep3",
        message: "API /api/categories references non-existent table"
      }
    ],
    validationCycles: 1
  }
}

(Routes back to architectStep3, which re-runs with issues in context)

After blueprintValidator (valid):
{
  ...previous state,
  blueprintValidation: {
    isValid: true,
    issues: [],
    validationCycles: 2
  }
}
```

**Phase 3: Planner & Sandbox**

```
After plannerAgent:
{
  ...previous state,
  taskQueue: {
    phases: [
      { phaseNumber: 1, phaseName: "setup", tasks: [...] },
      { phaseNumber: 2, phaseName: "middleware", tasks: [...] },
      ...
    ],
    totalTasks: 15,
    estimatedTotalTokens: 8000
  },
  currentPhaseIndex: 0,
  currentTaskIndex: 0
}

After setupSandbox:
{
  ...previous state,
  sandboxId: "sandbox-1709123456789"
}

After sandboxHealthCheck:
{
  ...previous state,
  sandboxHealthy: true
}
```

### 10.3 API Request Flow

**Gemini API Call Sequence**:

```
[Agent Node]
    ↓
callGemini({
  systemPrompt: "You are the PM Agent...",
  userPrompt: "User's requirement: Build a todo app",
  agentName: "pmAgent",
  currentCost: 0,
  tokenBudget: 2.0
})
    ↓
[gemini.js] Budget check: 0 < 2.0 ✓
    ↓
[gemini.js] Build full prompt
    ↓
[Google GenAI SDK] POST to Gemini API
    ↓
    ├─ Success → Parse response
    │       ↓ Extract tokens from usageMetadata
    │       ↓ Calculate cost
    │       ↓ Parse JSON
    │       ↓ Return { parsed, raw, tokens }
    │
    └─ Error → Retry (max 3 attempts)
            ↓ Exponential backoff (2s, 4s, 8s)
            ↓ If all fail → throw error
    ↓
[Agent Node] Receives result
    ↓
makeTokenDelta(agentName, result.tokens)
    ↓
Return {
  ...agentData,
  tokenUsage: {
    newCalls: [{ agent: "pmAgent", inputTokens: 150, outputTokens: 80 }],
    addedInput: 150,
    addedOutput: 80,
    addedCost: 0.0007
  }
}
    ↓
[State Reducer] Merges delta into existing tokenUsage
    ↓
tokenUsage.calls = [...oldCalls, ...newCalls]
tokenUsage.totalInput = oldTotal + addedInput
tokenUsage.totalOutput = oldTotal + addedOutput
tokenUsage.estimatedCost = oldCost + addedCost
```

### 10.4 Filesystem Operations Flow

**Sandbox Creation**:

```
[setupSandboxNode]
    ↓
createSandbox(folderStructure, dependencies)
    ↓
[sandboxManager.js]
    ↓
1. Generate sandboxId = "sandbox-" + Date.now()
    ↓
2. Create directory: /tmp/ai-dev-team/sandbox-1709123456789/
    ↓
3. Create subdirectories:
   - backend/src/models/
   - backend/src/routes/
   - backend/src/middleware/
   - frontend/src/pages/
   - frontend/src/components/
    ↓
4. Parse folderStructure string:
   "backend/\n├── src/\n│   ├── utils/\n..."
   Extract paths, create directories
    ↓
5. Write package.json files:
   - backend/package.json (with dependencies from blueprint)
   - frontend/package.json (with dependencies from blueprint)
    ↓
6. Write .env.example files
    ↓
7. Initialize Git:
   $ git init
   $ git add -A
   $ git commit -m "Initial scaffold"
   $ git tag v0.0.0
    ↓
8. Store metadata in Map:
   sandboxes.set(sandboxId, {
     path: "/tmp/ai-dev-team/sandbox-1709123456789",
     backendPath: ".../backend",
     frontendPath: ".../frontend",
     createdAt: 1709123456789,
     snapshotCount: 0
   })
    ↓
9. Return sandboxId
    ↓
[setupSandboxNode] Returns { sandboxId }
    ↓
[State] sandboxId = "sandbox-1709123456789"
```

**Health Check**:

```
[sandboxHealthCheckNode]
    ↓
healthCheck(sandboxId)
    ↓
[sandboxManager.js]
    ↓
1. Lookup sandbox in Map
    ↓
2. Check backend/ exists → fs.existsSync()
    ↓
3. Check frontend/ exists → fs.existsSync()
    ↓
4. Check backend/package.json exists
    ↓
5. Check frontend/package.json exists
    ↓
6. Check Git initialized → execSync("git status")
    ↓
7. Check standard directories exist
    ↓
8. Check disk space (optional)
    ↓
9. Return { healthy: true/false, failures: [...] }
    ↓
[sandboxHealthCheckNode] Returns { sandboxHealthy: true/false }
```

---

## 11. Configuration & Dependencies

### 11.1 package.json

```json
{
  "name": "ai-dev-team",
  "version": "1.0.0",
  "type": "module",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "test:graph": "node tests/test-graph-skeleton.js",
    "test:pm": "node tests/test-pm-agent.js",
    "test:architect": "node tests/test-architect.js",
    "test:validator": "node tests/test-validator.js",
    "test:planner": "node tests/test-planner.js",
    "test:sandbox": "node tests/test-sandbox.js"
  },
  "dependencies": {
    "@langchain/langgraph": "^1.2.0",
    "@langchain/langgraph-checkpoint-redis": "^1.0.1",
    "@google/genai": "^1.43.0",
    "dotenv": "^16.4.7",
    "readline": "^1.3.0"
  }
}
```

**Key Dependencies**:

1. **@langchain/langgraph** (v1.2.0)
   - State machine framework
   - Provides: StateGraph, Annotation, START, END, MemorySaver
   - Used in: graph.js, state.js

2. **@langchain/langgraph-checkpoint-redis** (v1.0.1)
   - Redis-based state persistence
   - Provides: RedisSaver
   - Used in: graph.js (createCheckpointer)
   - Optional: Falls back to MemorySaver if Redis unavailable

3. **@google/genai** (v1.43.0)
   - Google Gemini API client
   - Provides: GoogleGenAI class
   - Used in: gemini.js

4. **dotenv** (v16.4.7)
   - Environment variable loader
   - Loads .env file into process.env
   - Used in: index.js (import "dotenv/config")

5. **readline** (v1.3.0)
   - Terminal I/O for user input
   - Provides: createInterface, question
   - Used in: humanInput.js, index.js

### 11.2 Environment Variables

**Required**:

```bash
GEMINI_API_KEY=your_api_key_here
```

Get from: https://aistudio.google.com/apikey

**Optional**:

```bash
# Model selection (default: gemini-2.5-flash)
GEMINI_MODEL=gemini-2.5-flash

# Token budget in USD (default: 2.0)
TOKEN_BUDGET=2.0

# Redis URL for persistent checkpointing (default: in-memory)
REDIS_URL=redis://localhost:6379

# Debug mode (default: false)
DEBUG=true
```

### 11.3 .env File Example

```bash
# Required
GEMINI_API_KEY=AIzaSyD...your_key_here

# Optional
GEMINI_MODEL=gemini-2.5-flash
TOKEN_BUDGET=5.0
REDIS_URL=redis://localhost:6379
DEBUG=false
```

### 11.4 Redis Setup (Optional)

**Using Docker**:

```bash
docker run -d -p 6379:6379 redis:latest
```

**Using Homebrew (macOS)**:

```bash
brew install redis
brew services start redis
```

**Using apt (Ubuntu)**:

```bash
sudo apt install redis-server
sudo systemctl start redis
```

**Verify Connection**:

```bash
redis-cli ping
# Should return: PONG
```

**In Application**:

```javascript
// graph.js:92-105
export async function createCheckpointer() {
  const redisUrl = process.env.REDIS_URL;
  
  if (redisUrl) {
    try {
      const { RedisSaver } = await import("@langchain/langgraph-checkpoint-redis");
      const saver = await RedisSaver.fromUrl(redisUrl);
      console.log("✅ Redis checkpointer connected");
      return saver;
    } catch (error) {
      console.warn(`⚠️ Redis failed: ${error.message}. Using in-memory.`);
    }
  }
  
  return new MemorySaver();
}
```

**Benefits of Redis**:
- State persists across restarts
- Can resume interrupted workflows
- Enables multi-process coordination (future)

**When to Use In-Memory**:
- Development/testing
- Single-run workflows
- No need for persistence


---

## 12. Error Handling & Edge Cases

### 12.1 Token Budget Enforcement

**Location**: gemini.js:40-45

```javascript
export async function callGemini({ currentCost, tokenBudget, ... }) {
  if (currentCost >= tokenBudget) {
    throw new Error(
      `TOKEN_BUDGET_EXCEEDED: $${currentCost.toFixed(4)} >= budget $${tokenBudget}`
    );
  }
  // ... proceed with API call
}
```

**Flow**:
1. Every agent passes `currentCost: state.tokenUsage.estimatedCost`
2. callGemini checks before making API call
3. If exceeded → throws error
4. Error propagates to index.js
5. index.js catches and displays user-friendly message

**User Experience**:

```bash
❌ Token budget exceeded! Increase TOKEN_BUDGET in .env
   Current: $2.0045 / Budget: $2.00
```

### 12.2 API Retry Logic

**Location**: gemini.js:60-115

```javascript
const MAX_RETRIES = 3;

for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
  try {
    const response = await client.models.generateContent(...);
    // Success → return result
    return { parsed, raw, tokens };
    
  } catch (error) {
    lastError = error;
    
    // Don't retry budget errors
    if (error.message?.includes("TOKEN_BUDGET_EXCEEDED")) throw error;
    
    // Last attempt → give up
    if (attempt === MAX_RETRIES) throw error;
    
    // Exponential backoff: 2s, 4s, 8s
    const waitMs = Math.pow(2, attempt) * 1000;
    console.warn(`[${agentName}] Attempt ${attempt} failed. Retrying in ${waitMs}ms...`);
    await new Promise(r => setTimeout(r, waitMs));
  }
}
```

**Handles**:
- Network timeouts
- Rate limiting (429 errors)
- Temporary API outages
- Transient failures

**Does NOT retry**:
- Budget exceeded errors
- Invalid API keys
- Malformed requests

### 12.3 JSON Parsing Errors

**Location**: gemini.js:85-100

```javascript
try {
  let cleanText = rawText.trim();
  
  // Remove markdown code blocks if present
  if (cleanText.startsWith("```")) {
    cleanText = cleanText.replace(/^```(?:json)?\n?/, "").replace(/\n?```$/, "");
  }
  
  parsed = JSON.parse(cleanText);
  
} catch (parseError) {
  console.error(`[${agentName}] JSON parse failed (attempt ${attempt}):`, rawText.slice(0, 200));
  
  if (attempt === MAX_RETRIES) {
    throw new Error(`JSON_PARSE_FAILED after ${MAX_RETRIES} attempts.`);
  }
  
  lastError = parseError;
  continue;  // Retry
}
```

**Why JSON Fails**:
1. LLM returns markdown: ` ```json\n{...}\n``` `
2. LLM adds explanation: `Here's the JSON: {...}`
3. Truncated response (rare)

**Solution**:
- Strip markdown code blocks
- Retry with same prompt (LLM usually fixes itself)
- After 3 attempts → fail with clear error

### 12.4 Validation Loop Protection

**Location**: blueprintValidator.js:240-260

```javascript
const MAX_VALIDATION_CYCLES = 2;

export async function blueprintValidatorNode(state) {
  const currentCycles = state.blueprintValidation?.validationCycles || 0;
  
  // ... run validation checks
  
  if (issues.length === 0) {
    return { blueprintValidation: { isValid: true, ... } };
  }
  
  // Force proceed after max cycles
  if (currentCycles >= MAX_VALIDATION_CYCLES) {
    console.log(`⚠️ Max validation cycles (${MAX_VALIDATION_CYCLES}) reached. Proceeding with warnings.`);
    return {
      blueprintValidation: {
        isValid: true,  // Force proceed
        issues: issues,
        validationCycles: currentCycles + 1
      }
    };
  }
  
  // Route back for fixes
  return {
    blueprintValidation: {
      isValid: false,
      issues: issues,
      validationCycles: currentCycles + 1
    }
  };
}
```

**Why Needed**:
- Prevents infinite loops if Architect can't fix issues
- After 2 cycles, proceeds with warnings
- User can manually fix issues in sandbox

### 12.5 Sandbox Creation Failures

**Location**: setupSandbox.js:15-30

```javascript
export async function setupSandboxNode(state) {
  try {
    const sandboxId = await createSandbox(folderStructure, dependencies);
    return { sandboxId };
    
  } catch (error) {
    console.error(`❌ Sandbox creation failed: ${error.message}`);
    return {
      sandboxId: "",
      error: `Sandbox creation failed: ${error.message}`
    };
  }
}
```

**Common Failures**:
1. **Disk full**: /tmp has no space
2. **Permission denied**: Can't write to /tmp
3. **Git not installed**: execSync("git init") fails

**Handling**:
- Error stored in state.error
- sandboxHealthCheck detects empty sandboxId
- User sees clear error message

### 12.6 Missing Environment Variables

**Location**: index.js:95-105

```javascript
try {
  initGemini(process.env.GEMINI_API_KEY);
  console.log(`✅ Gemini initialized`);
} catch (error) {
  console.error(`❌ ${error.message}`);
  console.error("   Create a .env file with GEMINI_API_KEY=your_key");
  process.exit(1);
}
```

**Location**: gemini.js:15-20

```javascript
export function initGemini(apiKey) {
  if (!apiKey) {
    throw new Error("GEMINI_API_KEY is required. Get one from https://aistudio.google.com/apikey");
  }
  aiClient = new GoogleGenAI({ apiKey });
  return aiClient;
}
```

**User Experience**:

```bash
❌ GEMINI_API_KEY is required. Get one from https://aistudio.google.com/apikey
   Create a .env file with GEMINI_API_KEY=your_key
```

---

## 13. Testing

### 13.1 Test Structure

```
tests/
├── test-graph-skeleton.js    # Mock test (no API)
├── test-pm-agent.js          # Real API test
├── test-architect.js         # Real API test
├── test-validator.js         # Mock test (no API)
├── test-planner.js           # Real API test
└── test-sandbox.js           # Mock test (no API)
```

### 13.2 Mock Tests (No API Key Required)

**test-graph-skeleton.js**:

Tests LangGraph wiring without calling Gemini:

```javascript
// Mock agent that returns hardcoded data
function mockPmAgent(state) {
  if (state.pmConversation.length === 0) {
    return {
      pmStatus: "needs_clarification",
      pmQuestions: ["Mock question?"],
      pmConversation: [{ role: "pm", questions: ["Mock question?"] }]
    };
  }
  return {
    pmStatus: "spec_ready",
    clarifiedSpec: { appName: "mock-app", features: [] }
  };
}

// Build graph with mock nodes
const graph = new StateGraph(AgentState);
graph.addNode("pmAgent", mockPmAgent);
// ... etc

// Test execution
const result = await graph.invoke({ userRequirement: "test" });
assert(result.clarifiedSpec !== null, "Should have spec");
```

**test-validator.js**:

Tests validation logic with hardcoded blueprints:

```javascript
const mockState = {
  blueprint: {
    entities: [{ name: "User" }],
    dbSchema: {
      tables: [{ name: "users", fields: [...] }]
    },
    apiEndpoints: [
      { path: "/api/users", relatedTable: "users" }
    ]
  }
};

const result = await blueprintValidatorNode(mockState);
assert(result.blueprintValidation.isValid === true, "Should be valid");
```

**test-sandbox.js**:

Tests sandbox operations:

```javascript
const sandboxId = await createSandbox("backend/\n├── src/", {
  backend: { dependencies: { express: "^4.18.2" } }
});

const health = await healthCheck(sandboxId);
assert(health.healthy === true, "Sandbox should be healthy");

writeFile(sandboxId, "backend/test.js", "console.log('test');");
const content = readFile(sandboxId, "backend/test.js");
assert(content === "console.log('test');", "File should match");

destroySandbox(sandboxId);
```

### 13.3 Real API Tests (Require GEMINI_API_KEY)

**test-pm-agent.js**:

```javascript
import "dotenv/config";
import { initGemini } from "../src/utils/gemini.js";
import { pmAgentNode } from "../src/agents/pmAgent.js";

initGemini(process.env.GEMINI_API_KEY);

// Test 1: Vague requirement → questions
const state1 = {
  userRequirement: "Build an app",
  pmConversation: [],
  tokenUsage: { calls: [], totalInput: 0, totalOutput: 0, estimatedCost: 0 },
  tokenBudget: 2.0
};

const result1 = await pmAgentNode(state1);
assert(result1.pmStatus === "needs_clarification", "Should ask questions");
assert(result1.pmQuestions.length > 0, "Should have questions");

// Test 2: Detailed requirement → spec
const state2 = {
  userRequirement: "Build a todo app with user auth, categories, due dates, and priority levels",
  pmConversation: [],
  tokenUsage: { calls: [], totalInput: 0, totalOutput: 0, estimatedCost: 0 },
  tokenBudget: 2.0
};

const result2 = await pmAgentNode(state2);
assert(result2.pmStatus === "spec_ready", "Should generate spec");
assert(result2.clarifiedSpec.appName !== undefined, "Should have app name");
assert(result2.clarifiedSpec.features.length > 0, "Should have features");
```

**test-architect.js**:

```javascript
// Test all 5 architect steps
const spec = {
  appName: "todo-app",
  features: [{ name: "Todo CRUD", ... }],
  authRequired: true
};

const state = {
  clarifiedSpec: spec,
  blueprint: { entities: [], dbSchema: {}, ... },
  tokenUsage: { calls: [], totalInput: 0, totalOutput: 0, estimatedCost: 0 },
  tokenBudget: 2.0
};

// Step 1
const result1 = await architectStep1Node(state);
assert(result1.blueprint.entities.length > 0, "Should have entities");

// Step 2
state.blueprint.entities = result1.blueprint.entities;
const result2 = await architectStep2Node(state);
assert(result2.blueprint.dbSchema.tables.length > 0, "Should have tables");

// ... etc for steps 3, 4, 5
```

### 13.4 Running Tests

```bash
# Mock tests (no API key needed)
npm run test:graph
npm run test:validator
npm run test:sandbox

# Real API tests (requires GEMINI_API_KEY in .env)
npm run test:pm
npm run test:architect
npm run test:planner

# All mock tests
npm run test:all:mock
```

---

## 14. Performance Considerations

### 14.1 Token Optimization

**Problem**: Each LLM call costs money and time.

**Solutions Implemented**:

1. **Concise System Prompts**
   - PM Agent: 60 lines vs. 200+ lines in naive approach
   - Architect: Separate prompts per step (focused context)

2. **Blueprint Summarization** (plannerAgent.js:60-80)
   ```javascript
   // Don't send full blueprint (10KB+)
   const blueprintSummary = {
     databaseType: blueprint.dbSchema.databaseType,
     tables: blueprint.dbSchema.tables.map(t => ({
       name: t.name,
       fieldCount: t.fields.length  // Not full field definitions
     })),
     // ... etc
   };
   ```

3. **Deterministic Validation**
   - Blueprint Validator uses pure JS (0 tokens)
   - Alternative: LLM validation would cost ~500 tokens per cycle

4. **Budget Enforcement**
   - Prevents runaway costs
   - Fails fast if budget exceeded

**Token Usage (Typical Todo App)**:

| Agent | Calls | Tokens | Cost |
|-------|-------|--------|------|
| PM Agent | 2 | 800 | $0.0008 |
| Architect Step 1 | 1 | 600 | $0.0006 |
| Architect Step 2 | 1 | 1200 | $0.0012 |
| Architect Step 3 | 1 | 1000 | $0.0010 |
| Architect Step 4 | 1 | 900 | $0.0009 |
| Architect Step 5 | 1 | 800 | $0.0008 |
| Planner | 1 | 1500 | $0.0015 |
| **Total** | **8** | **6800** | **$0.0068** |

### 14.2 Execution Time

**Typical Flow** (todo app):

| Phase | Duration | Bottleneck |
|-------|----------|------------|
| PM Agent (2 calls) | 8-12s | Gemini API latency |
| Architect (5 steps) | 20-30s | Gemini API latency |
| Validator | <1s | Pure JS |
| Planner | 5-8s | Gemini API latency |
| Sandbox Setup | 1-2s | Filesystem I/O |
| Health Check | <1s | Filesystem checks |
| **Total** | **35-55s** | **API latency** |

**Optimization Opportunities** (Future):

1. **Parallel Architect Steps**: Steps 2, 3, 4 could run in parallel (requires state refactor)
2. **Streaming Responses**: Display partial results as they arrive
3. **Caching**: Cache common patterns (auth middleware, CRUD routes)

### 14.3 Memory Usage

**State Size Growth**:

```
Initial:        ~1 KB
After PM:       ~5 KB (spec)
After Architect: ~50 KB (blueprint)
After Planner:  ~100 KB (task queue)
After Phase 4:  ~500 KB (file registry)
After Phase 7:  ~2 MB (full project)
```

**Checkpointing Impact**:

- **Redis**: Serializes state to Redis (network overhead)
- **In-Memory**: Keeps state in Node.js heap (faster, but lost on exit)

**Recommendation**: Use Redis for long-running projects (>30 tasks), in-memory for quick prototypes.

---

## 15. Future Phases (Phase 4-7)

### 15.1 Phase 4: Context Builder + Coder Agent

**New Nodes**:
- `selectNextTask`: Pick next task from queue
- `buildContext`: Gather relevant files for task
- `coderAgent`: Write code for task (LLM)
- `updateRegistry`: Track created files

**Flow**:
```
... Phase 3 END
    ↓
[selectNextTask] Reads: taskQueue, currentPhaseIndex, currentTaskIndex
    ↓ Picks next task
    ↓ Writes: currentTask
    ↓
[buildContext] Reads: currentTask.filesNeeded, fileRegistry
    ↓ Loads file contents
    ↓ Writes: taskContext
    ↓
[coderAgent] Reads: currentTask, taskContext, blueprint
    ↓ Calls Gemini to write code
    ↓ Writes: generatedFiles
    ↓
[updateRegistry] Reads: generatedFiles
    ↓ Writes files to sandbox
    ↓ Updates fileRegistry
    ↓
[Conditional] More tasks? → selectNextTask : reviewerAgent
```

### 15.2 Phase 5: Reviewer + Executor + Debugger

**New Nodes**:
- `reviewerAgent`: Check code quality (LLM)
- `executorAgent`: Run code in sandbox
- `debuggerAgent`: Fix errors (LLM)

**Flow**:
```
[reviewerAgent] Reads: generatedFiles
    ↓ Static analysis
    ↓ Writes: reviewResult (pass/fail/issues)
    ↓
[Conditional] Pass? → executorAgent : coderAgent (fix)
    ↓
[executorAgent] Reads: currentTask.verificationCommand
    ↓ Runs in sandbox
    ↓ Writes: executionResult (success/error)
    ↓
[Conditional] Success? → selectNextTask : debuggerAgent
    ↓
[debuggerAgent] Reads: executionResult.errors, generatedFiles
    ↓ Analyzes error
    ↓ Writes: debugState, fixSuggestions
    ↓ Routes back to coderAgent
```

### 15.3 Phase 6: Feedback Loop + Deploy Agent

**New Nodes**:
- `userFeedbackNode`: Collect user input on generated app
- `feedbackAnalyzer`: Determine scope drift
- `deployAgent`: Generate deployment instructions

**Flow**:
```
... All tasks complete
    ↓
[userFeedbackNode] Prompt: "Try the app. Any changes?"
    ↓ Writes: userFeedback
    ↓
[feedbackAnalyzer] Reads: userFeedback, clarifiedSpec
    ↓ Calculate scope drift
    ↓ Writes: scopeDrift (0.0-1.0)
    ↓
[Conditional] scopeDrift < 0.3? → implement : reject
    ↓
[deployAgent] Reads: blueprint, sandboxId
    ↓ Generate Dockerfile, docker-compose.yml
    ↓ Generate deployment instructions
    ↓ Writes: deploymentConfig
    ↓
END
```

### 15.4 Phase 7: React Dashboard

**New Component**: Web UI instead of CLI

- Real-time progress display
- Interactive Q&A (WebSocket)
- Code preview
- Token usage graphs
- Sandbox file browser

---

## 16. Troubleshooting Guide

### 16.1 Common Issues

**Issue**: "GEMINI_API_KEY is required"

**Solution**:
```bash
# Create .env file
echo "GEMINI_API_KEY=your_key_here" > .env

# Get key from: https://aistudio.google.com/apikey
```

---

**Issue**: "TOKEN_BUDGET_EXCEEDED"

**Solution**:
```bash
# Increase budget in .env
TOKEN_BUDGET=5.0

# Or reduce scope of requirement
```

---

**Issue**: "Redis connection failed"

**Solution**:
```bash
# Option 1: Start Redis
docker run -d -p 6379:6379 redis:latest

# Option 2: Remove REDIS_URL from .env (use in-memory)
```

---

**Issue**: "Sandbox creation failed: ENOSPC"

**Solution**:
```bash
# Free up disk space
df -h /tmp

# Or change sandbox location (edit sandboxManager.js:50)
const sandboxPath = path.join("/your/custom/path", sandboxId);
```

---

**Issue**: "JSON_PARSE_FAILED after 3 attempts"

**Cause**: LLM returned invalid JSON

**Solution**:
1. Check if requirement is too complex (simplify)
2. Try different model: `GEMINI_MODEL=gemini-2.0-flash-exp`
3. Check Gemini API status: https://status.cloud.google.com/

---

**Issue**: Validation loop (keeps routing back to architect)

**Cause**: Architect can't fix validation issues

**Solution**:
- System automatically proceeds after 2 cycles
- Check blueprintValidation.issues in output
- Manually fix issues in sandbox after completion

---

### 16.2 Debug Mode

Enable detailed logging:

```bash
# In .env
DEBUG=true
```

```javascript
// In index.js:210
if (process.env.DEBUG) {
  console.error(error.stack);
}
```

---

## 17. Architectural Patterns

### 17.1 State Machine Pattern

**Why**: Workflows with conditional branching, loops, and human-in-the-loop.

**Alternative**: Sequential function calls
```javascript
// ❌ Hard to checkpoint, resume, or branch
const spec = await pmAgent(requirement);
const blueprint = await architectAgent(spec);
const plan = await plannerAgent(blueprint);
```

**LangGraph Advantage**:
- Automatic checkpointing
- Conditional routing
- Loop detection
- State inspection

### 17.2 Agent Pattern

**Why**: Specialized AI agents for different tasks (PM, Architect, Coder).

**Alternative**: Single mega-prompt
```javascript
// ❌ Unfocused, expensive, error-prone
const result = await callGemini({
  prompt: "You are a full-stack developer. Given this requirement, design the database, APIs, frontend, write all code, test it, and deploy it."
});
```

**Multi-Agent Advantage**:
- Focused prompts (better quality)
- Smaller token usage per call
- Easier to debug (isolate which agent failed)
- Parallelizable (future)

### 17.3 Validator Pattern

**Why**: Catch errors early before expensive code generation.

**Cost Analysis**:
- Validator (pure JS): 0 tokens, <1s
- Fixing broken code later: 500+ tokens, 10+ seconds, multiple iterations

**ROI**: Validator saves ~$0.005 per validation cycle.

### 17.4 Sandbox Pattern

**Why**: Isolate AI-generated code from system.

**Risks Without Sandbox**:
- AI writes `rm -rf /`
- AI modifies system files
- AI creates infinite loops

**Sandbox Benefits**:
- Safe execution
- Git-based rollback
- Easy cleanup
- Prepares for Docker (Phase 4)

---

## 18. Conclusion

This system demonstrates a production-ready multi-agent architecture for autonomous software development. Key innovations:

1. **State Machine Orchestration**: LangGraph provides robust workflow management
2. **Specialized Agents**: Each agent has a focused role and optimized prompts
3. **Deterministic Validation**: Pure JS validation saves tokens and improves reliability
4. **Sandbox Isolation**: Safe execution environment with git-based versioning
5. **Token Tracking**: Transparent cost monitoring and budget enforcement

**Current Capabilities** (Phase 3):
- Requirement → Specification (with Q&A)
- Specification → Architecture Blueprint (5-step design)
- Blueprint → Build Plan (ordered tasks with dependencies)
- Sandbox → Ready workspace with git

**Next Steps** (Phase 4-7):
- Code generation (Coder Agent)
- Testing & debugging (Reviewer, Executor, Debugger)
- User feedback loop
- Deployment automation
- Web dashboard

**Total Lines of Code**: ~2,500 (excluding tests)
**Total LLM Calls**: 7-9 per project
**Average Cost**: $0.005-0.015 per project
**Average Time**: 35-55 seconds

---

**Document Version**: 1.0  
**Last Updated**: Phase 3 Implementation  
**Author**: AI Dev Team Project

