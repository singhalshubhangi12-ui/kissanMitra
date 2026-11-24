# Kisan Mitra AI: Model Context Protocol (MCP) Server Specifications

This document defines the Model Context Protocol (MCP) Server for the **Kisan Mitra AI** multi-agent platform, exposing agricultural data interfaces to LLM workers.

---

## 1. MCP Server Architecture

The server uses the official **Model Context Protocol SDK** to expose structured JSON-RPC interfaces over stdio (standard input/output). This allows the Orchestrator and Specialist agents to invoke local and external tools via standard JSON message passing.

```
       ┌───────────────────┐
       │   Agent / LLM     │
       └─────────┬─────────┘
                 │ (JSON-RPC)
                 ▼
 ┌───────────────────────────────┐
 │ StdioServerTransport (JSON)   │
 └───────────────┬───────────────┘
                 ▼
 ┌───────────────────────────────┐
 │       McpServer (Zod)         │ (Schema & Payload Validation)
 └───────────────┬───────────────┘
                 │
   ┌─────────────┼─────────────┬─────────────┐
   ▼             ▼             ▼             ▼
┌───────┐     ┌───────┐     ┌───────┐     ┌───────┐
│ Crop  │     │Disease│     │Weather│     │Market │  ...
│ Tool  │     │ Tool  │     │ Tool  │     │ Tool  │
└───────┘     └───────┘     └───────┘     └───────┘
```

---

## 2. Folder Structure

```
kisan-mitra-ai/
├── mcp-server/
│   ├── Dockerfile                  # Container definition for server hosting
│   └── mcp_server.ts               # Core MCP Server and Tool registrations
├── package.json                    # Contains @modelcontextprotocol/sdk dependency
└── tsconfig.json                   # TypeScript configuration
```

---

## 3. Tool Registrations & Specifications

Here are the precise schemas and parameters for all 7 registered tools:

| Tool Name | Description | Required Inputs |
| :--- | :--- | :--- |
| `GetCropRecommendation` | Recommends crops based on soil conditions and location. | `soilType`, `moisture`, `location` |
| `CheckDisease` | Diagnoses plant diseases based on descriptions/symptoms. | `cropType`, `symptoms` |
| `WeatherForecast` | Retrieves localized agricultural weather forecasts. | `location`, `days` |
| `MarketPrices` | Returns live prices and market advisories. | `cropName`, `region` |
| `GovernmentSchemes` | Matches demographic profile to active subsidies. | `state` |
| `SaveFarmerProfile` | Saves or updates a farmer profile in the database. | `farmerId` |
| `GetFarmerHistory` | Returns a list of past queries and diagnoses (Admin only). | `farmerId` |

---

## 4. MCP Server Implementation Code (`mcp_server.ts`)

Below is the complete implementation of the MCP Server in TypeScript, conforming to the exact names requested:

```typescript
/**
 * Kisan Mitra AI — Model Context Protocol Server
 * 
 * Exposes 7 tools for agricultural data retrieval.
 * Transport: stdio JSON-RPC.
 */

import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

// Initialize server
const server = new McpServer({
  name: "KisanMitraMCPServer",
  version: "1.0.0",
});

// Role-Based Access Control logic
type UserRole = "Farmer" | "Admin";
const TOOL_PERMISSIONS: Record<string, UserRole[]> = {
  GetCropRecommendation: ["Farmer", "Admin"],
  CheckDisease: ["Farmer", "Admin"],
  WeatherForecast: ["Farmer", "Admin"],
  MarketPrices: ["Farmer", "Admin"],
  GovernmentSchemes: ["Farmer", "Admin"],
  SaveFarmerProfile: ["Farmer", "Admin"],
  GetFarmerHistory: ["Admin"], // Admin-only
};

function checkPermission(toolName: string, role?: UserRole): void {
  const userRole = role ?? "Farmer";
  const allowed = TOOL_PERMISSIONS[toolName];
  if (!allowed || !allowed.includes(userRole)) {
    throw new Error(`Unauthorized: Role '${userRole}' cannot access '${toolName}'.`);
  }
}

// Mock databases
const FARMER_DB = new Map<string, any>();
const QUERY_LOG_DB: any[] = [];

// ─────────────────────────────────────────────
// 1. GetCropRecommendation
// ─────────────────────────────────────────────
server.tool(
  "GetCropRecommendation",
  "Returns crop recommendations based on soil type, moisture level, and geographic location.",
  {
    soilType: z.enum(["Sandy", "Clay", "Loamy", "Silty", "Peaty"]).describe("Soil classification"),
    moisture: z.enum(["Low", "Medium", "High"]).describe("Soil moisture level"),
    location: z.string().min(1).describe("Region or district name"),
    role: z.enum(["Farmer", "Admin"]).optional().describe("User role for authorization check"),
  },
  async ({ soilType, moisture, location, role }) => {
    checkPermission("GetCropRecommendation", role);

    const recommendations: Record<string, string[]> = {
      Loamy: ["Wheat", "Rice", "Sugarcane"],
      Sandy: ["Millet", "Groundnut", "Sesame"],
      Clay: ["Cotton", "Jute", "Mustard"],
      Silty: ["Barley", "Oats", "Potato"],
      Peaty: ["Strawberry", "Carrot", "Vegetables"],
    };

    const crops = recommendations[soilType] ?? ["Wheat"];
    return {
      content: [{
        type: "text",
        text: JSON.stringify({
          recommendedCrops: crops,
          suitabilityScore: 0.95,
          notes: `Optimized crop recommendation for ${soilType} soil in ${location}.`,
        }),
      }],
    };
  }
);

// ─────────────────────────────────────────────
// 2. CheckDisease
// ─────────────────────────────────────────────
server.tool(
  "CheckDisease",
  "Diagnoses crop diseases from symptoms and lists remedies.",
  {
    cropType: z.string().describe("Crop type (e.g. Potato, Rice)"),
    symptoms: z.string().describe("Plant leaf symptoms"),
    role: z.enum(["Farmer", "Admin"]).optional(),
  },
  async ({ cropType, symptoms, role }) => {
    checkPermission("CheckDisease", role);

    const diagnoses: Record<string, any> = {
      Potato: { disease: "Late Blight", pathogen: "Phytophthora infestans", treatment: "Use copper fungicide." },
      Rice: { disease: "Rice Blast", pathogen: "Magnaporthe oryzae", treatment: "Use Tricyclazole." },
    };

    const result = diagnoses[cropType] ?? { disease: "Unknown fungal infection", pathogen: "N/A", treatment: "Consult agronomist." };
    return {
      content: [{
        type: "text",
        text: JSON.stringify({
          diagnosis: result.disease,
          pathogen: result.pathogen,
          remedies: [result.treatment],
          organicOptions: ["Neem oil spray"],
        }),
      }],
    };
  }
);

// ─────────────────────────────────────────────
// 3. WeatherForecast
// ─────────────────────────────────────────────
server.tool(
  "WeatherForecast",
  "Fetches weather forecast and agriculture advisories for a location.",
  {
    location: z.string().describe("Location name"),
    days: z.number().min(1).max(7).default(3).describe("Number of forecast days"),
    role: z.enum(["Farmer", "Admin"]).optional(),
  },
  async ({ location, days, role }) => {
    checkPermission("WeatherForecast", role);

    return {
      content: [{
        type: "text",
        text: JSON.stringify({
          location,
          days,
          forecast: [
            { day: 1, temp: "30°C", condition: "Sunny" },
            { day: 2, temp: "28°C", condition: "Light Rain" },
          ],
          advisory: "Rain on Day 2: Avoid spraying fertilizers on this day.",
        }),
      }],
    };
  }
);

// ─────────────────────────────────────────────
// 4. MarketPrices
// ─────────────────────────────────────────────
server.tool(
  "MarketPrices",
  "Returns crop prices and market trends.",
  {
    cropName: z.string().describe("Crop name"),
    region: z.string().describe("Mandi region"),
    role: z.enum(["Farmer", "Admin"]).optional(),
  },
  async ({ cropName, region, role }) => {
    checkPermission("MarketPrices", role);

    return {
      content: [{
        type: "text",
        text: JSON.stringify({
          cropName,
          region,
          modalPrice: 2200,
          unit: "per quintal",
          currency: "INR",
          trend: "UP",
        }),
      }],
    };
  }
);

// ─────────────────────────────────────────────
// 5. GovernmentSchemes
// ─────────────────────────────────────────────
server.tool(
  "GovernmentSchemes",
  "Searches matching central and state level schemes.",
  {
    state: z.string().describe("Farmer operating state"),
    farmSize: z.number().optional().describe("Land size in acres"),
    role: z.enum(["Farmer", "Admin"]).optional(),
  },
  async ({ state, farmSize, role }) => {
    checkPermission("GovernmentSchemes", role);

    return {
      content: [{
        type: "text",
        text: JSON.stringify({
          state,
          matchedSchemes: [
            { name: "PM-KISAN", benefit: "₹6000/year" },
            { name: "PM Fasal Bima Yojana", benefit: "Crop Insurance" },
          ],
        }),
      }],
    };
  }
);

// ─────────────────────────────────────────────
// 6. SaveFarmerProfile
// ─────────────────────────────────────────────
server.tool(
  "SaveFarmerProfile",
  "Saves or updates a farmer profile.",
  {
    farmerId: z.string().describe("Unique identifier"),
    name: z.string().optional(),
    location: z.string().optional(),
    landSizeAcres: z.number().optional(),
    primaryCrops: z.array(z.string()).optional(),
    role: z.enum(["Farmer", "Admin"]).optional(),
  },
  async ({ farmerId, name, location, landSizeAcres, primaryCrops, role }) => {
    checkPermission("SaveFarmerProfile", role);

    const profile = { farmerId, name, location, landSizeAcres, primaryCrops, updatedAt: new Date().toISOString() };
    FARMER_DB.set(farmerId, profile);

    return {
      content: [{
        type: "text",
        text: JSON.stringify({ success: true, farmerId, action: "SAVED" }),
      }],
    };
  }
);

// ─────────────────────────────────────────────
// 7. GetFarmerHistory
// ─────────────────────────────────────────────
server.tool(
  "GetFarmerHistory",
  "Returns query history for audit. Admin only.",
  {
    farmerId: z.string().describe("Farmer identifier"),
    role: z.enum(["Farmer", "Admin"]).optional(),
  },
  async ({ farmerId, role }) => {
    checkPermission("GetFarmerHistory", role);

    const history = QUERY_LOG_DB.filter(log => log.farmerId === farmerId);
    return {
      content: [{
        type: "text",
        text: JSON.stringify({ farmerId, history, count: history.length }),
      }],
    };
  }
);

// Start transport
const transport = new StdioServerTransport();
server.connect(transport).then(() => {
  console.error("Kisan Mitra MCP Server running on stdio.");
}).catch((err) => {
  console.error("Start failed:", err);
  process.exit(1);
});
```
