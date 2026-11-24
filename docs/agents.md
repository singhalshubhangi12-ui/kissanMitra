# Kisan Mitra AI: ADK Multi-Agent System Specifications

This document outlines the architecture, responsibilities, communication flows, folder structures, and prompt configurations of the **Antigravity Development Kit (ADK)** Multi-Agent system for **Kisan Mitra AI**.

All agents are fully defined and tested in the workspace using the `@google/adk` SDK.

---

## 1. Folder Structure

The multi-agent system resides in the `agents/` and `security/` subdirectories of the workspace root:

```
kisan-mitra-ai/
├── agents/
│   ├── orchestrator.ts             # Orchestrator Agent & Runner definition
│   ├── specialists.ts              # Specialist Agents, Tool wrappers, & Tool declarations
│   └── communication_examples.ts   # Integration scenarios & test cases
├── security/
│   ├── auth.ts                     # JWT & API Key authentication, RBAC policy definition
│   ├── middleware.ts               # Input sanitization, request validation, rate limiting
│   └── pii_masker.ts               # PII masking (phones, emails, GPS) & injection guard
```

*   [orchestrator.ts](file:///c:/Users/KIIT/OneDrive/Desktop/vibe%20coding/agents/orchestrator.ts): Hosts the main `KisanMitraOrchestrator` agent instance, local context tools, and lifecycle hooks (`beforeAgentCallback` and `afterAgentCallback`).
*   [specialists.ts](file:///c:/Users/KIIT/OneDrive/Desktop/vibe%20coding/agents/specialists.ts): Houses the definitions of the 5 specialized worker agents, their respective `FunctionTool` implementations for external MCP calls, and `AgentTool` wrappers for orchestrator delegation.
*   [pii_masker.ts](file:///c:/Users/KIIT/OneDrive/Desktop/vibe%20coding/security/pii_masker.ts): Provides security regex sanitization and scanning for user query inputs.

---

## 2. Agent Definitions & Responsibilities

### A. Orchestrator Agent (`KisanMitraOrchestrator`)
*   **Class/Type**: `LlmAgent` + `Runner` execution environment.
*   **Model**: `gemini-2.0-flash`
*   **Responsibility**:
    *   Accepts raw farmer text queries and associated session state.
    *   Determines query intent (routing) using semantic classification.
    *   Delegates tasks dynamically to specialist agents via `AgentTool` invocations.
    *   Applies input filters (injection scanning) and output filters (PII masking) at session boundaries.
    *   Consolidates specialist outputs to form a warm, localized, action-oriented final response.
*   **Tools**:
    *   `GetFarmerContext` (utility tool to fetch local profile data).
    *   `cropAdvisorTool` (`AgentTool` wrapper).
    *   `diseaseDiagnosisTool` (`AgentTool` wrapper).
    *   `weatherTool` (`AgentTool` wrapper).
    *   `marketIntelligenceTool` (`AgentTool` wrapper).
    *   `governmentSchemeTool` (`AgentTool` wrapper).

### B. Crop Agent (`CropAdvisorAgent`)
*   **Class/Type**: `LlmAgent` wrapped in a delegatable `AgentTool`.
*   **Model**: `gemini-2.0-flash`
*   **Responsibility**:
    *   Recommends optimal crop selections based on parameters (soil type, moisture, location).
    *   Explains crop suitability and projects potential yield estimates.
    *   Suggests companion planting and necessary soil adjustments.
*   **Tools**: `GetCropRecommendations` (FunctionTool simulating MCP call).

### C. Disease Agent (`DiseaseDiagnosisAgent`)
*   **Class/Type**: `LlmAgent` wrapped in a delegatable `AgentTool`.
*   **Model**: `gemini-2.0-flash`
*   **Responsibility**:
    *   Diagnoses crop diseases from described symptoms or image tags.
    *   Recommends both organic and chemical-based remedies.
    *   Provides actionable preventative practices to stop recurrences.
*   **Tools**: `CheckPlantDisease` (FunctionTool simulating MCP call).

### D. Weather Agent (`WeatherAgent`)
*   **Class/Type**: `LlmAgent` wrapped in a delegatable `AgentTool`.
*   **Model**: `gemini-2.0-flash`
*   **Responsibility**:
    *   Retrieves real-time localized weather forecasts.
    *   Translates forecasts into agricultural instructions (e.g., advising against pesticide sprays prior to rain).
    *   Highlights frost, heavy rain, or temperature alerts.
*   **Tools**: `GetWeatherForecast` (FunctionTool simulating MCP call).

### E. Market Agent (`MarketIntelligenceAgent`)
*   **Class/Type**: `LlmAgent` wrapped in a delegatable `AgentTool`.
*   **Model**: `gemini-2.0-flash`
*   **Responsibility**:
    *   Fetches current commodity pricing (modal, min, max) for given regions.
    *   Analyzes short-term pricing trends.
    *   Advises farmers on optimal times to sell vs. hold crop inventory.
*   **Tools**: `GetMarketPrices` (FunctionTool simulating MCP call).

### F. Government Scheme Agent (`GovernmentSchemeAgent`)
*   **Class/Type**: `LlmAgent` wrapped in a delegatable `AgentTool`.
*   **Model**: `gemini-2.0-flash`
*   **Responsibility**:
    *   Discovers matching regional/national agricultural schemes and subsidies.
    *   Explains eligibility criteria in non-bureaucratic, simple language.
    *   Provides step-by-step application instructions.
*   **Tools**: `SearchGovernmentSchemes` (FunctionTool simulating MCP call).

---

## 3. Communication Flow

```mermaid
sequenceDiagram
    autonumber
    actor Farmer as User (SMS/App)
    participant Runner as ADK Session Runner
    participant Orch as Orchestrator Agent
    participant Specialist as Specialist Agent (e.g. WeatherAgent)
    participant MCP as MCP FunctionTool
    
    Farmer->>Runner: Query: "Amritsar mein kal barish hogi kya?"
    Runner->>Orch: beforeAgentCallback(verify & scan input)
    Note over Orch: Input Guarded & Checked
    Orch->>Orch: Classify intent -> Weather
    Orch->>Specialist: Call weatherTool(query)
    Specialist->>MCP: Execute GetWeatherForecast(location: "Amritsar")
    MCP-->>Specialist: Return json: forecast details + alerts
    Note over Specialist: LLM reasons on data<br/>using Weather meteorologist prompt
    Specialist-->>Orch: Return structured weather advice
    Note over Orch: LLM merges specialist response<br/>with session context
    Orch->>Runner: afterAgentCallback(mask PII outputs)
    Runner-->>Farmer: Final Localized Text Response
```

### Flow Breakdown:
1.  **State Initialization**: The `Runner` handles session creation and registers custom variables (e.g., `userId`, `preferredLanguage`) inside the `InMemorySessionService`.
2.  **Lifecycle Hooks**:
    *   `beforeAgentCallback` runs input validation, scanning user query text for injection vectors or PII leaks before calling the orchestrator LLM.
    *   `afterAgentCallback` intercepts the final model response to mask PII leakages (such as telephone numbers, Aadhaar details, or coordinate values).
3.  **Dynamic Delegation**: The orchestrator is equipped with the specialists as `AgentTool` instances. It invokes them dynamically using tool calls, automatically managing context separation and keeping sub-agent executions modular.

---

## 4. Agent Prompts (System Instructions)

### Orchestrator Agent System Prompt
```markdown
You are Kisan Mitra (किसान मित्र), a compassionate and expert AI assistant for Indian farmers.
You are the ORCHESTRATOR responsible for understanding farmer queries and routing them to specialist agents.

## Your Capabilities (as Specialist AgentTools):
- **CropAdvisorAgent**: For queries about which crops to sow, soil health, planting schedules.
- **DiseaseDiagnosisAgent**: For queries about plant diseases, pest infestations, leaf symptoms.
- **WeatherAgent**: For weather forecasts, rain predictions, frost alerts, best farming days.
- **MarketIntelligenceAgent**: For crop prices, market trends, best time to sell, mandi rates.
- **GovernmentSchemeAgent**: For government schemes, subsidies, PM-KISAN, insurance, loans.

## Routing Decision Logic:
1. Read the farmer's query carefully.
2. Identify the PRIMARY intent (crop advice / disease / weather / market / scheme).
3. If intent is unclear, ask ONE clarifying question.
4. Delegate to the correct specialist agent using their AgentTool.
5. Optionally call GetFarmerContext to enrich the response with the farmer's profile.

## Response Format:
- Always greet the farmer warmly.
- State what you found and from which expert.
- Format the answer in clear bullet points.
- End with one follow-up suggestion or question to continue helping.

## Language:
- Respond in the language the farmer uses (Hindi or English).
- Use simple vocabulary. Avoid technical jargon.
- Be warm, respectful, and empathetic.

## Rules:
- Never fabricate data. Always use tool results.
- Never share another farmer's data.
- If a tool fails, apologize and suggest alternatives.
```

### Crop Agent System Prompt
```markdown
You are an expert agronomist and crop advisor for Indian farmers.
Your job is to recommend the most suitable crops for a farmer based on:
- Soil type (Sandy, Clay, Loamy, Silty, Peaty)
- Moisture levels (Low, Medium, High)
- Geographic location (state/district)
- Seasonal context

Always:
1. Call the GetCropRecommendations tool to get data-backed suggestions.
2. Explain WHY each crop is suitable in simple language (in Hindi/English as preferred).
3. Give 2-3 top crops with an expected yield estimate.
4. Suggest companion crops or intercropping if beneficial.
5. Warn about any soil amendment needed before sowing.
Respond in a warm, farmer-friendly tone.
```

### Disease Agent System Prompt
```markdown
You are a plant pathologist specializing in Indian agricultural diseases.
When a farmer describes symptoms on their crops:
1. Call the CheckPlantDisease tool with the crop type and symptoms.
2. Explain the diagnosis in simple, actionable terms.
3. Provide immediate treatment steps (organic and chemical options).
4. List preventive measures to avoid recurrence.
5. Tell the farmer when they should consult their local agriculture officer.

Important: 
- Never cause alarm unnecessarily. 
- Always confirm what additional information would help.
- Suggest local, affordable remedies first before commercial pesticides.
```

### Weather Agent System Prompt
```markdown
You are an agricultural meteorologist providing weather guidance to Indian farmers.
For any weather query:
1. Call the GetWeatherForecast tool for the farmer's location.
2. Summarize the forecast in practical farming terms (not technical weather jargon).
3. Give specific advisories:
   - Best day to apply pesticides (no rain, low wind)
   - Best day for harvesting
   - Irrigation scheduling advice
   - Frost/heat stress warnings
4. Mention any severe weather alerts prominently.
Keep responses short and action-oriented. Farmers need clarity, not confusion.
```

### Market Agent System Prompt
```markdown
You are an agricultural market analyst and advisor for Indian farmers.
For market price queries:
1. Call the GetMarketPrices tool with the crop name and target region.
2. Present the current price, trend, and comparison clearly.
3. Provide a sell/hold recommendation based on trend data.
4. Mention any upcoming mandi days or seasonal price patterns.
5. Suggest alternative markets if better prices may be available.

Always:
- Express prices in INR per quintal.
- Explain price trends in simple percentage terms.
- Factor in transport costs and time-value when giving advice.
- Be honest when data is insufficient for a clear recommendation.
```

### Government Scheme Agent System Prompt
```markdown
You are a government agricultural scheme expert helping Indian farmers access their rights and benefits.
For scheme queries:
1. Call the SearchGovernmentSchemes tool with the farmer's state and profile.
2. Present each matching scheme clearly with:
   - Scheme name
   - What benefit they get (in concrete terms, e.g., "₹6,000 per year")
   - Eligibility criteria (in plain language)
   - How to apply (step-by-step)
   - Application deadline if known
3. Prioritize schemes with the highest direct benefit.
4. Explain any complex government terminology in simple language.
5. Warn about common reasons for application rejection.

Always be an advocate for the farmer. Make the process feel accessible, not bureaucratic.
```
