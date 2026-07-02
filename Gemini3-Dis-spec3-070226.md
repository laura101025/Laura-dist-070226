TECHNICAL ARCHITECTURE SPECIFICATION & COGNITIVE DESIGN MANIFESTO
Document Version: 4.1.0-Surveillance
Release Date: July 2, 2026
System Status: COMPILING / SECURE
Target Codebase: AURA-7 Regulatory Hub
1. EXECUTIVE SUMMARY & ARCHITECTURE BLUEPRINT
The AURA-7 Regulatory Hub is a highly secure, full-stack, AI-powered regulatory intelligence platform designed to govern, track, audit, and trace medical devices distributed across the medical ecosystems of Taiwan. It serves as an active, high-fidelity surveillance console that synchronizes real-time regulatory compliance telemetry, safety recalls, licensing configurations, geolocation-based tracking node networks, and global World Health Organization (WHO) medevis nomenclatures.
At its core, AURA-7 utilizes a hybrid offline-first and cloud-synchronized state-management design. It addresses a critical public health and national security challenge: preventing recalled, counterfeit, unregistered, or expired medical hardware from entering active clinical pathways, whilst predicting critical regional equipment shortages before they impact patient outcomes.
1.1 Full-Stack Topology and Ingress Flow
The application is deployed using a containerized micro-runtime architecture. The layout isolates client-side operations from sensitive cognitive AI processing interfaces to maintain strict key confidentiality and robust network execution.
code
Code
+-----------------------------------------------------------------------------------------------------------------+
|                                           EXTERNALLY HOSTED INGRESS                                             |
|                                       (Nginx Reverse Proxy Layer - Port 3000)                                   |
+------------------------------------+-----------------------------------------------------+----------------------+
                                     |                                                     |
                                     v                                                     v
+------------------------------------+------------------+             +--------------------+----------------------+
|             DEVELOPMENT RUNTIME SYSTEM                |             |              PRODUCTION DEPLOYMENT        |
|  - Vite Dev Server (HMR Disabled for stability)       |             |  - Bundled Static Assets (dist/)          |
|  - Express Node.js Server via tsx watcher             |             |  - Bundled Server Execution (server.cjs)  |
+------------------------------------+------------------+             +--------------------+----------------------+
                                     |                                                     |
                                     +---------------------------+-------------------------+
                                                                 |
                                                                 v
                                             +-------------------+-------------------+
                                             |        EXPRESS MIDDLEWARE FRAMEWORK   |
                                             |       - Port 3000 Inbound Ingress     |
                                             |       - JSON Schema Validation Interceptor
                                             |       - Unified Dataset Router (/api) |
                                             +-------------------+-------------------+
                                                                 |
                                        +------------------------+------------------------+
                                        |                                                 |
                                        v                                                 v
                     +------------------+-------------------+          +------------------+-------------------+
                     |      SECURE DATA REGISTRY ENGINE     |          |       COGNITIVE COGNITION GATEWAY    |
                     |  - dataset.md Real-time File DB     |          |  - @google/genai TS SDK Client       |
                     |  - Custom Split-Parser Assembly      |          |  - gemini-3.5-flash Production model|
                     |  - Atomic Write-Back Serializer      |          |  - High-Fidelity Simulation Fallback|
                     +--------------------------------------+          +--------------------------------------+
1.2 Development versus Production Runtime Executions
The system employs a unified entry-point infrastructure centered around a custom Express server (server.ts).
Development Mode (process.env.NODE_ENV !== "production"):
The developer framework invokes tsx server.ts as the primary launcher.
Vite is initialized programmatically as an Express-compliant middleware in middleware mode (middlewareMode: true) with an SPA configuration (appType: "spa"). This allows server-side APIs to be processed first, while the single-page application and its assets are served transparently on the same port (3000) without cross-origin resource sharing (CORS) friction.
Hot Module Replacement (HMR) is programmatically bypassed via the control plane environment flag (DISABLE_HMR=true). This stabilizes the execution preview frame and eliminates layout reflow during iterative agent edits.
Production Mode (process.env.NODE_ENV === "production"):
The build process triggers a two-stage compilation pipeline:
The front-end React code is compiled via Vite into highly optimized, minified, static HTML/JS/CSS assets placed inside /dist.
The back-end TypeScript Express server is bundled into a single CommonJS-formatted file (dist/server.cjs) using esbuild with native external resolution (--packages=external). This compilation design bypasses Node’s strict runtime ES module relative import requirements, simplifies runtime load dependencies, and optimizes cold-start times inside serverless containers.
On deployment, the container launches via node dist/server.cjs. The server serves the pre-compiled static assets from /dist using express.static and sets up a wild-card fallback handler (*) to route unmatched non-API requests to /dist/index.html for client-side routing.
2. CORE REGISTRY DATASTORES & PARSING SPECIFICATIONS
The AURA-7 Hub is backed by a highly structured flat-file database schema implemented inside /dataset.md. This file represents the absolute, unified truth of the system, partitioning six disparate clinical and regulatory databases into separate, easily parsed code blocks.
code
Code
+-----------------------------------------------------------------------------------+
       |                                     dataset.md                                    |
       +-----------------------------------------------------------------------------------+
       |  # AURA-7 Hub Default Dataset Core                                                |
       |                                                                                   |
       |  ## Section 1: Medical Device Recalls                                             |
       |  ```json                                                                          |
       |  [ { "title": "Class 2 Recall", "udi": "00850014110147", ... } ]                   |
       |  ```                                                                              |
       |                                                                                   |
       |  ## Section 2: TFDA License Database                                              |
       |  ```csv                                                                           |
       |  許可證字號,醫療器材級數,中文品名,有效日期...                                     |
       |  "衛部罕醫器輸字第000001號","2","沛佳髓內釘","2029/6/10"...                        |
       |  ```                                                                              |
       |                                                                                   |
       |  ## Section 3: Taiwan Unique Device Identification Database (TUDID)              |
       |  ```csv                                                                           |
       |  許可證字號（類型）,UDI發碼機構,基本DI,型號...                                     |
       |  "衛部醫器陸輸字第000858號","GS1","06944904054537",...                            |
       |  ```                                                                              |
       |                                                                                   |
       |  ## Section 4: WHO medevis Global Reference                                       |
       |  ```tsv                                                                           |
       |  Device name \t Nomenclature code (EMDN) \t Nomenclature term (EMDN) \t ...       |
       |  Absorbent tipped applicator \t M0499 \t SPECIAL DRESSINGS \t ...                 |
       |  ```                                                                              |
       |                                                                                   |
       |  ## Section 5: Quality Management System (QMS/QSD) Registry                       |
       |  ```csv                                                                           |
       |  製造廠名稱,製造廠國別,製造廠地址,藥商名稱...                                     |
       |  "Jiangyu Health","CHN","Chengdu","QSD16050","115/06/02"...                       |
       |  ```                                                                              |
       |                                                                                   |
       |  ## Section 6: Geolocation DHA Monitoring Stations                                |
       |  ```json                                                                          |
       |  [ { "station_id": "DHA-01-NORTH", "name": "Taipei Central", ... } ]              |
       |  ```                                                                              |
       +-----------------------------------------------------------------------------------+
2.1 Schema Defs & Type Implementations
The database schema maps onto exact TypeScript interfaces defined inside /src/types.ts:
Section 1: Medical Device Recalls (RecallEntry):
Represents physical hardware components flagged by international regulatory alerts.
title (string): Summary description of the safety recall.
device_name_for_recall (string): Precise model name of recalled hardware.
manufacturer (string): Parent manufacturer corporate entity.
date (string): Date when the safety alert was issued.
recall_class (string): Level of recall severity (1 = critical hazard to life; 2 = temporary or reversible health consequences; 3 = minor violation).
udi (string): GS1 Universal Device Identifier (DI) key.
sn_lot_no (string): Suffix serial range or production batch number affected.
reason_for_recall (string): Detailed statutory explanation of physical failure risk.
Section 2: TFDA License Database (LicenseEntry):
Represents official medical device licensing records approved by the Taiwan Food and Drug Administration.
permit_no (string): Official licensing permit string (e.g., 衛部醫器輸字第025432號).
level (string): Medical device hazard level (1 = Low Risk, 2 = Medium Risk, 3 = High Risk).
chinese_name (string): Officially approved Mandarin Chinese product designation.
sub_category (string): Secondary technical category identifier.
applicant (string): Local medical product distributor or importing agency in Taiwan.
factory_country (string): Manufacturing country of origin (2-letter ISO or standard text).
expiry_date (string): Official license expiration date formatted as YYYY/MM/DD.
class_code (string): Statutory classification code.
tax_id (string): Unified business registration number of the local applicant.
Section 3: Taiwan Unique Device Identification Database (TudidEntry):
Tracks barcode specifications, device models, and product categories within Taiwanese jurisdictions.
permit_no_type (string): Associated TFDA license permit number.
udi_issuer (string): Universal issuing agency (e.g., GS1, HIBCC).
basic_di (string): Base Device Identifier code.
model (string): Manufacturer engineering model string.
chinese_name (string): Registered product name.
cancel_status (string): Revocation or cancellation date/status of the registration.
level (string): Medical device risk grade.
applicant (string): Sponsoring importing pharmaceutical organization.
sub_category (string): Registered technical classification category.
Section 4: WHO medevis Global Reference (WhoEntry):
Provides nomenclature alignments between local systems and international EMDN/GMDN classification indexes.
device_name (string): Standard English clinical device name.
emdn_code (string): European Medical Device Nomenclature standard code.
emdn_term (string): Semantic European Medical Device Nomenclature structural definition.
gmdn_code (string): Global Medical Device Nomenclature standard code.
gmdn_term (string): Semantic Global Medical Device Nomenclature detailed definition.
Section 5: Quality Management System (QmsEntry):
Audits medical device production facilities and checks manufacturing status approvals (QSD/QMS).
manufacturer_name (string): Global facility manufacturer name.
factory_country (string): Factory ISO location country.
factory_address (string): Detailed physical facility address.
applicant_name (string): Sponsoring local importing organization.
qsd_no (string): Official Quality System Document license number (e.g., QSD14667).
expiry_date (string): QSD license expiration date.
case_status (string): Active status of audit approval (e.g., 續予核備, 准予核備).
english_item_name (string): Scope of manufacturing procedures approved.
Section 6: Geolocation DHA Monitoring Stations (DhaStation):
Manages regional telemetry networks across Taiwan's major medical administrative centers.
station_id (string): Unique telemetry station ID (e.g., DHA-01-NORTH).
name (string): Geographic location name (e.g., Taipei Central Inspection Node).
latitude (number) / longitude (number): Coordinates of the monitoring receiver nodes.
status (string): Active operational grade ("operational" | "mismatch" | "recall_quarantine").
associated_hospitals (string[]): Medical centers coupled to this telemetry loop.
registered_serial_ranges (string[]): Serial prefixes governed by this receiver station.
mismatch_count (number): Discrepant hardware detections in current cycle.
active_recall_alerts (number): Count of active recalled units traced within station range.
2.2 Compilation, Boundary Splitting, and Tokenization Algorithms
The /src/utils/parser.ts file compiles the six sub-registers through a robust, deterministic multi-format parser.
Parsing Flow
Header Boundary Isolation:
The parser splits /dataset.md into distinct section buffers using an optimized regular expression targeting Markdown Level-2 headers (## Section X: ...):
code
TypeScript
const regex = new RegExp("##\\s+" + escapedHeader + "\\s*\\n\\s*`{3}(json|csv|tsv)?\\n([\\s\\S]*?)\\n`{3}", "i");
This isolates content within Markdown codeblocks (```) based on the section's format.
Differentiated Tokenization:
JSON Blocks (Section 1 and 6): Parsed directly using standard JSON.parse() methods.
TSV Blocks (Section 4): Processed by split and parsed line-by-line using a custom tab delimiter (\t).
CSV Blocks (Section 2, 3, and 5): Evaluated using a custom streaming parser (parseCsvLine) that correctly respects escaped commas inside double-quotes, allowing for complex data extraction.
Detailed CSV Tokenizer implementation:
code
TypeScript
function parseCsvLine(line: string, delimiter: string = ","): string[] {
  const result: string[] = [];
  let current = "";
  let inQuotes = false;
  
  for (let i = 0; i < line.length; i++) {
    const char = line[i];
    if (char === '"') {
      if (inQuotes && line[i + 1] === '"') {
        current += '"'; // Escaped quote inside double-quoted cell ("")
        i++;
      } else {
        inQuotes = !inQuotes; // Toggle quote boundaries
      }
    } else if (char === delimiter && !inQuotes) {
      result.push(current.trim());
      current = "";
    } else {
      current += char;
    }
  }
  result.push(current.trim());
  return result;
}
Data Re-Serialization Protocol:
When writing changes back to disk, serializeDatasetToMarkdown programmatically reconstructs the markdown string. It preserves the tabular layouts, correctly formats TSV strings with actual tab-character injections (\t), escapes embedded double-quotes inside CSV columns by doubling them (""), and formats the JSON outputs with a strict 2-space indentation.
3. INTERACTIVE FRONT-END ENGINEERING & UI COMPONENT ARCHITECTURE
The front-end is designed with a desktop-first dashboard approach, utilizing high-contrast visual hierarchies, structured spacing layouts, and fluid motion. It is built using React 19 and styled with Tailwind CSS.
3.1 Layout Strategy and Visual Design Language
AURA-7 employs a dark cosmic slate UI theme, incorporating responsive component states, interactive tooltips, and fluid sidebars.
Typography Strategy:
Display / Header Elements: Styled in Space Grotesk (font-display). This clean, high-contrast display font provides a modern, high-tech command feel.
Data Outputs / Status Fields: Styled in JetBrains Mono (font-mono). This monospaced typeface facilitates clear reading of dense serial numbers, coordinates, and UDI markers.
Body text: Rendered in Inter (font-sans), ensuring clean reading contrast and high density.
Color Systems:
Primary Base Background: Slate Gray gradient combinations (bg-slate-950 to bg-slate-900), mimicking an advanced monitoring workspace.
Active State Indicators: Operational statuses are marked using clear, high-contrast colors (text-emerald-400 / bg-emerald-500/10 for normal operations, text-amber-400 / bg-amber-500/10 for pending warnings, and text-rose-500 / bg-rose-500/10 for critical recalls).
Visual Effects: Panels feature glassmorphism designs with thin borders (border-slate-800), low opacity overlays (bg-slate-900/50), and subtle backdrop blurs (backdrop-blur-md). This styling creates visual depth without distracting clutter.
3.2 Dynamic Visualization (5 Interactive Recharts Implementations)
The interface integrates five Recharts visualizations within the CustomDashboard component, providing real-time compliance tracking and analytical insights.
code
Code
+---------------------------------------------------------------------------------------------------------+
|                                  AURA-7 LIVE COMPLIANCE INTEL HUB                                       |
+------------------------------------+------------------------------------+-------------------------------+
| 1. TRACEABILITY EXPIRY (LINE)      | 2. REGIONAL MISMATCHES (BAR)       | 3. RECALL INTENSITY (SCATTER) |
|                                    |                                    |                               |
| Expired (Count)                    | Mismatches (Count)                 | Recall Severity               |
|   |   /\                           |   |      [Taichung]                |   |        o                  |
|   |  /  \  /\                      |   |      +---------+               |   |            o (Class 1)    |
|   | /    \/  \                     |   |      |         |  [Kaohsiung]  |   |    o                       |
|   +-----------+                    |   +------+---------+---------------+   +------------+------------+     |
|   2026   2028   2030 (Year)        |         North      West     South  |               Device Age (Days) |
+------------------------------------+------------------------------------+-------------------------------+
| 4. RISK DISTRIBUTIONS (PIE)        | 5. MULTI-VECTOR COMPLIANCE (RADAR)                                 |
|                                    |                  License Expiry                                    |
|         Class 1 Recalls            |                       /\                                           |
|            (32%)                   |                      /  \                                          |
|         /---------\                |           QSD status+----+----+WHO Nomenclature Matching           |
|        /  Class 2  \               |                      \  /                                          |
|        \   (68%)   /               |                       \/                                           |
|         \---------/                |                 Recall Frequency                                   |
+------------------------------------+--------------------------------------------------------------------+
License Expiration Projection (Line Chart):
X-Axis: Time intervals (Years 2026 to 2032).
Y-Axis: Cumulative volume of active licenses nearing expiration.
Telemetry Goal: Allows regulators to monitor upcoming licensing expirations, helping them manage supply chain bottlenecks for critical devices before shortages occur.
DHA Regional Sensor Mismatches (Bar Chart):
X-Axis: Monitoring stations (Taipei Central, Taichung Regulatory, Kaohsiung Compliance).
Y-Axis: Quantity of unregistered serial number detections.
Telemetry Goal: Displays regional non-compliance, helping tracking agents identify potential smuggling routes or distribution errors.
Recall Class Severity Breakdown (Pie Chart):
Segments: Categorized by Class 1, Class 2, and Class 3 safety recalls.
Visual Cue: Uses high-contrast warning colors (Crimson, Orange, Gold) to help administrators quickly assess overall risk severity.
Multi-Vector Compliance Status (Radar Chart):
Anchors: Five compliance vectors: License Expiration, WHO Nomenclature Mapping, QSD Status, Telemetry Heartbeats, and Recall Frequency.
Telemetry Goal: Aggregates data from different registers to provide a multi-dimensional view of overall system health.
Device Age versus Recall Intensity Analysis (Scatter Chart):
X-Axis: Duration of device market availability (Days in service).
Y-Axis: Occurrence rate of safety recalls.
Z-Axis (Bubble Size): Volume of physical units distributed.
Telemetry Goal: Identifies manufacturing failures, highlighting whether older equipment or new configurations are more prone to recall risks.
3.3 Interactive 2D Map of Taiwan Telemetry Node Network
The geographic tracking system is built inside src/App.tsx using responsive SVG rendering, avoiding heavy external mapping wrappers.
Coordinate Mapping: Coordinates are projected onto a 2D vector graphic of Taiwan's medical monitoring grid. Station coordinates (such as Taipei (25.033, 121.565), Taichung (24.148, 120.673), and Kaohsiung (22.627, 120.301)) are mapped to pixel coordinates on an SVG viewport using an interpolation formula:

Interactive Calibrations: Clicking a node selects the corresponding telemetry station, displaying connected hospital networks, governed serial ranges, and active alert counts. Users can recalibrate GPS coordinates, update registered serial ranges, and adjust local sensor statuses.
Dynamic Visual States: SVG nodes feature custom CSS animations, pulsing with light-emitting ring elements (animate-ping) that change color based on node status (Emerald for operational, Amber for serial mismatch, and Pulsing Crimson for active recall quarantines).
4. COGNITIVE ENGINE & AI INTEGRATIONS
AURA-7's intelligence layer is powered by the @google/genai TypeScript SDK, using a server-side API proxy configuration (/api/ai/analyze) with the high-concurrency model gemini-3.5-flash.
code
Code
+--------------------------------------------------------------------------------------------------+
  |                                     COGNITIVE ENGINE ROUTING                                     |
  +--------------------------------------------------------------------------------------------------+
  |                                                                                                  |
  |   Client-Side Interaction (App.tsx)                                                              |
  |     - Selects Model & Custom Audit Prompt                                                        |
  |     - Packages local datasets.md payload                                                         |
  |                                                                                                  |
  +-----------------------------------------------|--------------------------------------------------+
                                                  v
  +-----------------------------------------------+--------------------------------------------------+
  |   API Express Proxy Server (server.ts)                                                           |
  |     - Validates process.env.GEMINI_API_KEY presence                                              |
  |                                                                                                  |
  |     +------------[ Key Present ]------------+               +------------[ Key Absent ]----------+
  |     |                                       |               |                                    |
  |     v                                       v               v                                    v
  |  Instantiate @google/genai SDK Client       |            Initiate Simulated Fallback Processor   |
  |  Model: "gemini-3.5-flash"                  |            Regex Parsing on Prompt Triggers        |
  |  Generate System Instruction Context         |            Load High-Fidelity Domain Responses     |
  |  Execute client.models.generateContent       |                                                   |
  |     |                                       |               |                                    |
  +-----|---------------------------------------+---------------+------------------------------------+
        |                                                               |
        +---------------------------------------+-----------------------+
                                                |
                                                v
  +---------------------------------------------+----------------------------------------------------+
  |   Unified Intelligence Response Payload (JSON)                                                    |
  |     - Rendered dynamically with markdown support                                                 |
  |     - Appended to audit log stream as [LLM INSIGHTS]                                             |
  +--------------------------------------------------------------------------------------------------+
4.1 System Prompting and Inference Mechanics
For real-time cognitive reasoning, the API handler applies structured system prompts to establish domain expertise and format outputs consistently:
code
TypeScript
const systemInstruction = `You are the AURA-7 Cognitive Auditor, an expert AI agent specializing in Taiwan FDA medical device regulations, global EMDN/GMDN classifications, and pharmaceutical supply chain risk management.
You must analyze the provided data payloads and return a structured markdown report including risk assessments, regulatory non-compliance citations (Taiwan Medical Device Management Act), and precise corrective roadmaps. 
Keep outputs highly professional and objective. Focus on actionable insights.`;
4.2 System-Defined "Wow" Cognitive Features (The Original Tri-Core)
Feature A: Predictive Shortage & License Expiration Forecast Engine
Inference Strategy: Cross-references Section 2 (TFDA Licenses) and Section 6 (DHA Stations). It analyzes license expiration dates, active quarantined lots, and regional hospital distribution vectors to predict product deficits before they impact clinical care.
Cognitive Prompting:
code
Code
Analyze clinical supply metrics. Target Class III life-critical device configurations. Evaluate expiration dates against current hospital utilization rates. Estimate the exact depletion timeline (ΔTd) for each regional node. Recommend inter-station supply reallocations to prevent shortages.
Execution Interface: Users select a high-risk device from the dropdown, triggering an API call that returns a detailed deficit projection and a step-by-step supply chain reallocation roadmap.
Feature B: WHO-EMDN Semantic Cross-Border Discrepancy Mesh
Inference Strategy: Maps Chinese-language medical device entries from local TFDA records (Section 2 and 3) to the official WHO EMDN/GMDN database (Section 4) using semantic translation models.
Cognitive Prompting:
code
Code
Map local database registry columns with WHO medevis nomenclatures. Calculate the semantic match confidence score. Flag translation anomalies or code mismatches that could bypass global recall warnings.
Execution Interface: Evaluates local medical equipment records, automatically matching them with global EMDN nomenclature categories and flagging unmapped warning vulnerabilities.
Feature C: TFDA Regulatory Fine & Statutory Action Calculator
Inference Strategy: Audits Section 6 telemetry data, focusing on "mismatch" or "unregistered" serial numbers detected by regional sensors. It references Taiwan's Medical Device Management Act (醫療器材管理法) to calculate administrative fines and output legal enforcement roadmaps.
Cognitive Prompting:
code
Code
Examine identified serial mismatches under Article 22, 23, and 62 of Taiwan's Medical Device Management Act. Calculate the statutory fine range (NTD 60,000 to 2,000,000), taking device classification and violation duration into account. Formulate a legal quarantine roadmap.
Execution Interface: Clicking on an anomalous node triggers an administrative audit, providing legal citations and generating a step-by-step regulatory enforcement response.
4.3 Proposed "Wow" Cognitive Features (The Advanced Tri-Core)
Feature D: Multi-Modal Computer Vision Barcode & Label Audit Agent
Technical Overview: Integrates the camera interface with multimodal image analysis, allowing users to scan device packaging labels and receive instant compliance reviews.
code
Code
[ Packaging Label Image ] 
                  |
                  v  (Mobile Camera Capture)
       +----------+----------+
       |   src/components/   |
       |  CameraScanner.tsx  |
       +----------+----------+
                  |  (Base64 Image Data Payload)
                  v
       +----------+----------+
       |    server.ts API    | ---> calls Gemini "gemini-2.5-flash" (Vision Enabled)
       +----------+----------+
                  |  
                  |  - AI extracts Chinese Product Label, GS1-128 barcode, and UDI.
                  |  - Cross-references with Section 3 TUDID & Section 1 Recalls.
                  v
       +---------------------+
       |   AUDIT RECOVERY    |
       +---------------------+
       | [PASS] UDI verified |
       | [WARN] Missing TFDA |
       |        Article 25   |
       +---------------------+
Inference Model & API Payload:
Inference Model: gemini-2.5-flash (Vision and multimodal processing).
API Endpoint: /api/ai/vision-audit (Accepts an image payload as a Base64 string and matches it against registry states).
Core Cognitive Prompt:
code
Code
System Context: You are a Multi-Modal TFDA Label Inspection Agent.
Analyze the uploaded medical device label image.
1. Extract the GS1-128 barcode or UDI-DI key using OCR.
2. Parse the approved Chinese product name, local importer details, and license permit number.
3. Verify compliance with Article 25 of Taiwan's Medical Device Management Act (mandatory Chinese labeling).
4. Cross-reference the extracted UDI with Section 1 (Recalls) and Section 3 (TUDID) records.
5. Return a structured JSON compliance report:
   {
     "udi_extracted": string,
     "article_25_compliance": "PASS" | "FAIL",
     "identified_discrepancies": string[],
     "quarantine_status": "CLEAR" | "QUARANTINE_REQUIRED",
     "reasoning": string
   }
Client Visualization Integration: Adds a real-time camera viewfinder overlay inside CameraScanner.tsx. Once an image is captured, the processing state displays a dynamic scanner animation, followed by a detailed compliance summary.
Feature E: Autonomous QMS Gap-Analysis & CAPA Generation System
Technical Overview: Cross-analyzes manufacturing facility records (Section 5) against ISO 13485 standards and Taiwan's Medical Device Quality Management System Regulations (醫療器材品質管理系統準則) to identify compliance gaps and generate Corrective and Preventive Action (CAPA) plans.
Inference Model & API Payload:
Inference Model: gemini-2.5-pro (Handles long contexts and structured legal reasoning).
API Endpoint: /api/ai/capa-audit (Processes Section 5 records and generates corrective action reports).
Core Cognitive Prompt:
code
Code
Evaluate the following QMS/QSD manufacturer entry against ISO 13485:2016 and Taiwan Medical Device QMS Regulations.
Target: {QMS_DATA_PAYLOAD}

Tasks:
1. Audit facility status, certificate validity dates, and approved manufacturing scopes.
2. Identify potential audit compliance failures (e.g., lapsed expirations, mismatched manufacturing scopes).
3. Generate a regulatory Corrective and Preventive Action (CAPA) plan based on global GHTF guidelines:
   - Root Cause Analysis (Fishbone categorization)
   - Immediate Containment Steps
   - Corrective Action Plan with specific tasks and timelines
   - Preventive Action Plan with monitoring steps
   
Output the final CAPA plan formatted as an official PDF-exportable document.
Client Visualization Integration: Expands the QMS Table with an "Audit Facility" action button. Clicking the button opens a dual-pane modal: the left pane displays identified QMS compliance gaps, while the right pane generates an interactive, PDF-downloadable CAPA report.
Feature F: Cross-Border Regulatory Intelligence Alert & Global Harmonization Sync (GHTF)
Technical Overview: Leverages Gemini's Google Search Grounding capabilities to query international medical device alert systems (such as US FDA MedWatch, European Commission Safety Alerts, and Japan PMDA warnings). It compares global alert data with local TFDA license registrations (Section 2) to flag newly recalled or high-risk devices before official domestic warnings are issued.
Inference Model & API Payload:
Inference Model: gemini-2.5-flash with Google Search Grounding enabled.
API Endpoint: /api/ai/global-sync (Executes grounded search queries and correlates findings with local licensing databases).
Core Cognitive Prompt:
code
Code
Perform a search for safety recalls and hazard alerts issued by the US FDA, EMA, or PMDA within the last 60 days.
Target device categories: {DEVICE_CATEGORIES_ACTIVE_IN_TAIWAN}

Tasks:
1. Cross-reference search results with the following local TFDA registrations: {LOCAL_LICENSES}
2. Flag any local registrations where the global manufacturer has an active Class I recall or safety warning in international databases, but no local recall is registered in Section 1.
3. Calculate a Cross-Border Risk Exposure Index (Score 0-100).
4. Generate a preventive quarantine roadmap for incoming import shipments at Taiwan customs ports.
Client Visualization Integration: Integrates a "Global Alert Radar" feed onto the main dashboard. This feed displays active international recalls alongside local risk indicators, providing a proactive view of potential import hazards.
5. DETAILED SECURITY, AUTHORIZATION & SESSION MANAGEMENT BLUEPRINT
AURA-7 implements an isolated security loop to manage access and prevent unauthorized modifications to the /dataset.md registry.
code
Code
+-------------------------------------------------------------------------------+
       |                               SESSION MANAGEMENT                              |
       +-------------------------------------------------------------------------------+
       |                                                                               |
       |  User Input (Email / Password)                                                |
       |       |                                                                       |
       |       v                                                                       |
       |  Authentication Request -> POST /api/auth/login                               |
       |       |                                                                       |
       |       v                                                                       |
       |  Backend Verification (server.ts)                                             |
       |  - Matches credentials against users.json datastore                           |
       |  - Returns unique token: "tok_..." and authenticated email                    |
       |       |                                                                       |
       |       +------------[ Success ]------------+                                   |
       |       |                                   |                                   |
       |       v                                   v                                   |
       |  Set localStorage:                        Render main dashboard               |
       |  - "aura_user" -> User Email              Initialize AppState                 |
       |  - "aura_token" -> tok_...                Write INIT success log              |
       |                                                                               |
       +-------------------------------------------------------------------------------+
5.1 Authentication Lifecycles and Session Invalidation
User Identity Ingest:
Access is governed by a secure sign-in portal (LoginScreen.tsx). The system is pre-seeded with authorized credentials for evaluation (e.g., admin@aura7.gov.tw / admin123).
Session Persistence Model:
Upon successful login, the server returns an authorization token (tok_[a-z0-9]+). The client stores the user's email and token in localStorage, using these credentials to authorize database write operations.
Logout Mechanics:
Clicking the log out action triggers a complete session teardown:
code
TypeScript
localStorage.removeItem("aura_user");
localStorage.removeItem("aura_token");
setUser(null);
This clears the local state and returns the user to the login screen, preventing back-button exploits.
5.2 Server-Side State Synchronization & Write-Back Protections
To protect the integrity of /dataset.md, the platform implements a secure, coordinated write-back loop:
code
Code
+------------------+                    +---------------------+                    +-------------------+
|  App.tsx State   | -- 1. Serialize -> |  POST /api/dataset  | -- 2. Parse JSON ->|  dataset.md File  |
|  (Local Changes) |                    |  Payload            |                    |  (Atomic Write)   |
+------------------+                    +---------------------+                    +-------------------+
State Isolation: When edits or deletions are made in the UI, they update the local React state, setting isDirty to true. This alerts the user to unsaved modifications and prevents premature writes.
Synchronization Request: Clicking "Persist Sync" serializes the current state into formatted Markdown and sends it as a JSON payload to the /api/dataset/save endpoint.
Atomic File Writes: The server receives the payload and writes the content to /dataset.md using fs.writeFileSync. This ensures atomic file updates, keeping the flat-file database consistent with the UI.
6. CODE DEFECT AUDIT & TECHNICAL CORRECTIVE ROADMAPS (POTENTIAL BUGS & FIXES)
An architectural review of the codebase identified six potential edge-case defects. Below are the technical root causes and production-ready corrective implementations for each.
6.1 CSV Parsing Commas Escape Vulnerability (Defect #1)
Root Cause: The default parseCsvLine function splits fields by commas but may fail on complex unescaped quotes or nested quotes. It does not strip leading and trailing quotes from the parsed cells, resulting in literal double quotes remaining in the runtime string state (e.g., "\"沛佳\"法斯樂" instead of "沛佳"法斯樂).
Correction Implementation:
code
TypeScript
export function robustParseCsvLine(line: string, delimiter: string = ","): string[] {
  const result: string[] = [];
  let current = "";
  let inQuotes = false;
  
  for (let i = 0; i < line.length; i++) {
    const char = line[i];
    if (char === '"') {
      if (inQuotes && line[i + 1] === '"') {
        current += '"'; // Handle escaped double quotes inside quoted values
        i++;
      } else {
        inQuotes = !inQuotes; // Toggle quote block state
      }
    } else if (char === delimiter && !inQuotes) {
      result.push(cleanCsvCell(current));
      current = "";
    } else {
      current += char;
    }
  }
  result.push(cleanCsvCell(current));
  return result;
}

function cleanCsvCell(cell: string): string {
  let cleaned = cell.trim();
  // Strip outer quotes if present
  if (cleaned.startsWith('"') && cleaned.endsWith('"')) {
    cleaned = cleaned.substring(1, cleaned.length - 1);
  }
  // Unescape standard double quotes
  return cleaned.replace(/""/g, '"');
}
6.2 Serialization Double-Quote Escape Cascade (Defect #2)
Root Cause: In serializeDatasetToMarkdown, when rebuilding Section 2, 3, and 5 CSV lines, the serialization uses:
code
TypeScript
`"${l.chinese_name.replace(/"/g, '""')}"`
If a field already contains unescaped double quotes, repeating the sync multiple times can cause a cascading expansion of double quotes (e.g., ""name"" 
 """"name""""), corrupting the underlying dataset.md file.
Correction Implementation:
code
TypeScript
export function safeEscapeCsvCell(value: string | undefined): string {
  if (value === undefined || value === null) return "";
  const stringified = String(value);
  // Normalize double quotes first to prevent recursive expansion
  const normalized = stringified.replace(/""/g, '"');
  // Escape double quotes as "" to meet standard RFC 4180 rules
  const escaped = normalized.replace(/"/g, '""');
  return `"${escaped}"`;
}
6.3 React 19 State Update Collision on Concurrent Re-Renders (Defect #3)
Root Cause: In React 19, updating state within an active effect without dependency stabilization can lead to infinite re-render cascades. In App.tsx, the useEffect that triggers logging depends on the activeSection variable. If a state update inside the logging effect triggers another re-render, the browser's execution stack can overflow.
Correction Implementation:
code
TypeScript
// Fix: Use primitive tracking variables and memoized callback functions
const memoizedAddLog = useCallback((type: "info" | "warn" | "success" | "llm", message: string) => {
  const now = new Date();
  const pad = (n: number) => n.toString().padStart(2, "0");
  const timeStr = `[${pad(now.getHours())}:${pad(now.getMinutes())}:${pad(now.getSeconds())}]`;
  
  setLogs(prev => {
    // Avoid appending duplicate logs to prevent infinite render loops
    if (prev.length > 0 && prev[prev.length - 1].message === message) {
      return prev;
    }
    return [...prev, {
      id: Math.random().toString(36).substring(2, 9),
      timestamp: timeStr,
      type,
      message
    }];
  });
}, []);
6.4 TSV Parser Boundary Incompatibility on Hard Tabulations (Defect #4)
Root Cause: Section 4 utilizes tab delimiters (\t). If the dataset.md markdown file is saved in editors that convert tabs to spaces, the line split command parseCsvLine(line, "\t") fails, parsing the entire line into a single column.
Correction Implementation:
code
TypeScript
export function robustParseTsvLine(line: string): string[] {
  // If no tab characters are present, fallback to splitting by multiple spaces
  if (!line.includes("\t")) {
    return line.split(/ {2,}/).map(cell => cell.trim().replace(/^"|"$/g, ""));
  }
  return line.split("\t").map(cell => cell.trim().replace(/^"|"$/g, ""));
}
6.5 GPS Stage Out-of-Bounds Vector Layout Bug (Defect #5)
Root Cause: The mathematical formula for placing station nodes on the 2D SVG map uses fixed offset multipliers. If a station coordinate falls outside the expected range for Taiwan, the calculated pixel positions can exceed the container boundaries, rendering the node off-screen.
Correction Implementation:
code
TypeScript
// Clamp coordinate transformations within the map viewport bounds (0% to 100%)
export function getClampedMapCoordinates(latitude: number, longitude: number) {
  const minLat = 21.8;
  const maxLat = 25.4;
  const minLng = 119.8;
  const maxLng = 122.2;

  // Linear interpolation percentage calculations
  const xPct = ((longitude - minLng) / (maxLng - minLng)) * 100;
  // Invert Y coordinates since SVG coordinates originate from the top-left corner
  const yPct = 100 - (((latitude - minLat) / (maxLat - minLat)) * 100);

  return {
    x: Math.max(5, Math.min(95, xPct)) + "%",
    y: Math.max(5, Math.min(95, yPct)) + "%"
  };
}
6.6 Atomic Concurrent Read/Write File Collision (Defect #6)
Root Cause: In server.ts, the /api/dataset/save endpoint uses direct file writes (fs.writeFileSync). If multiple save requests occur simultaneously, this can lead to file lock collisions, resulting in truncated or empty dataset.md files.
Correction Implementation:
code
TypeScript
import { renameSync, writeFileSync } from "fs";

export function safeAtomicWriteFileSync(filePath: string, content: string): void {
  const tempPath = `${filePath}.tmp`;
  try {
    // Write content to a temporary file first
    writeFileSync(tempPath, content, "utf-8");
    // Rename the temporary file to the target file path
    renameSync(tempPath, filePath);
  } catch (err) {
    console.error("Atomic write failed, cleaning up temp file:", err);
    try {
      if (fs.existsSync(tempPath)) fs.unlinkSync(tempPath);
    } catch (cleanupErr) {
      console.error("Cleanup failed:", cleanupErr);
    }
    throw err; // Escalate error to API router
  }
}
7. OPERATIONAL PLAYBOOKS & DEPLOYMENT MANIFESTS
7.1 Production Configuration and Environment Setup (.env.example)
To deploy the application in a production container, you must define the following environment variables inside .env.example:
code
Env
# ==============================================================================
# AURA-7 SYSTEM ENVIRONMENTS
# ==============================================================================

# NODE_ENV: Set to "production" to optimize React bundle delivery and run compiled server.cjs
NODE_ENV=production

# PORT: The internal container ingress port. Must be set to 3000.
PORT=3000

# GEMINI_API_KEY: Required for secure, server-side Gemini AI API processing.
# Ensure this key remains private and is never exposed to the client browser.
GEMINI_API_KEY=your_gemini_api_key_here

# APP_URL: The public-facing ingress URL, used to manage callback redirects and API validation.
APP_URL=https://your-applet-run-service.run.app
7.2 Dockerfile Multi-Stage Build Script
To build and package AURA-7 into a containerized runtime environment, use this optimized, multi-stage Dockerfile:
code
Dockerfile
# ==============================================================================
# STAGE 1: COMPILATION ENGINE
# ==============================================================================
FROM node:20-alpine AS compiler
WORKDIR /app

# Install project dependencies
COPY package*.json ./
RUN npm ci

# Copy application source files
COPY . .

# Run the production build pipeline
# This compiles the front-end assets to /dist and bundles server.ts into dist/server.cjs
RUN npm run build

# ==============================================================================
# STAGE 2: CONTAINERIZED PRODUCTION RUNTIME
# ==============================================================================
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV PORT=3000

# Install production dependencies only
COPY package*.json ./
RUN npm ci --only=production

# Copy compiled assets from compiler stage
COPY --from=compiler /app/dist ./dist
COPY --from=compiler /app/dataset.md ./dataset.md
COPY --from=compiler /app/users.json ./users.json

EXPOSE 3000

# Start the application using the compiled CommonJS server bundle
CMD ["node", "dist/server.cjs"]
8. SUMMARY MATRIX OF APPLICATION ARTIFACTS
This table outlines the roles and locations of the key files in the codebase, detailing how each file contributes to the full-stack system:
File Name / Path	Format / Tech	Purpose & Role inside AURA-7 Hub System
/server.ts	TypeScript (Node)	Main Server Entry Point: Sets up Express endpoints for authentication, file operations, and Gemini API proxying. Mounts Vite as middleware in development and serves static assets in production.
/dataset.md	Markdown Flat-File	Primary Datastore: Acts as the flat-file database, storing the six registers within structured JSON, CSV, and TSV code blocks.
/src/types.ts	TypeScript	Unified Schemas: Defines TypeScript interfaces and structures (such as LicenseEntry, RecallEntry, and DhaStation) used across the application.
/src/App.tsx	TSX (React)	Primary Dashboard Interface: Governs the application state, layout panels, event stream logs, interactive Taiwan 2D map, and user actions.
/src/utils/parser.ts	TypeScript	Database Parsing Engine: Handles parsing data from /dataset.md into active React state objects, and serializes updates back to markdown.
/src/utils/fuzzySearch.ts	TypeScript	Search Utility: Provides token-based and Levenshtein distance matching algorithms to score and rank search results across the registers.
/src/components/LoginScreen.tsx	TSX (React)	Secure Access Portal: Authenticates user sessions, validating credentials against the user registry and initializing storage tokens.
/src/components/CustomDashboard.tsx	TSX (React)	Intelligence Visualizations: Render the five Recharts diagrams, providing interactive compliance analysis and risk dashboards.
/src/components/AdvancedFilterBuilder.tsx	TSX (React)	Boolean Query Engine: Allows users to construct complex, multi-field filter rules (AND/OR logic) to isolate registry entries.
/src/components/CameraScanner.tsx	TSX (React)	Packaging Scan Interface: Accesses the device camera to simulate barcode parsing and multi-modal label audits.
/src/components/DataImporterExporter.tsx	TSX (React)	Export and Import Interface: Handles CSV exports and manages file imports for bulk database updates.
/src/components/PdfGenerator.tsx	TSX (React)	Document Generation Engine: Compiles compliance audit results into structured, print-ready PDF reports.
/vite.config.ts	TypeScript	Vite Compiler Config: Configures build plug-ins, path aliases (@/*), and file watcher parameters for the dev server.
9. COMPREHENSIVE FOLLOW-UP DEVELOPMENT QUESTIONS
To guide the next phase of engineering and scale the AURA-7 Hub, address these twenty architectural and compliance questions:
Relational Database Migration: How can we transition the current markdown-based dataset.md flat-file datastore into an enterprise relational database (e.g., PostgreSQL managed via Cloud SQL) using Prisma or Drizzle ORM, while keeping real-time audit trail syncing?
Horizontal Scaling and Multi-User Concurrency: How can we implement a distributed locking mechanism or optimistic concurrency control to prevent write conflicts when multiple regulatory auditors attempt to sync changes back to /dataset.md at the same time?
Optimizing Model Selection: Should we use fine-tuned versions of gemini-2.5-flash or gemini-2.5-pro for specialized tasks (such as parsing complex Chinese pharmaceutical classifications) to reduce inference latency and API costs?
Offline-First Data Recovery: How can we implement indexedDB on the client browser to allow regulatory inspectors to work offline in regional hospitals and sync their audits back when they reconnect?
Multi-Factor Authentication Hardening: What strategy should we use to implement multi-factor authentication (MFA) via OAuth2/OpenID Connect or SMS tokens for official TFDA inspectors accessing the AURA-7 command console?
PDF Generation Performance: Can we replace client-side PDF rendering with server-side document generation (e.g., using Puppeteer or PDF-Kit) to allow users to generate large reports with thousands of audited device models?
Integrating WebSockets for Real-Time Feeds: How can we implement a WebSocket framework in server.ts to push real-time telemetry updates from physical DHA monitoring stations to the dashboard without using client polling loops?
Vision Audit Accuracy: What preprocessing steps (e.g., grayscale normalization, adaptive thresholding) should we apply to images captured via mobile cameras to improve Gemini's OCR accuracy under low-light hospital conditions?
Securing API Access with Scope Management: How can we implement role-based access control (RBAC) to ensure hospital staff can only view relevant records, while full write permissions are restricted to authorized TFDA administrators?
Data Anonymization and HIPAA Compliance: When exporting audit databases or running global audits, how can we mask patient serial linkages to comply with international health data regulations and personal privacy standards?
Scalable Vector Graphics Customization: How can we optimize the rendering of the interactive SVG map when tracking thousands of DHA stations across Taiwan without bottlenecking the main React UI thread?
Automating QMS Gap Analysis Audits: How can we connect Section 5 QMS/QSD databases directly to international certification bodies to automate registration verification?
Reducing Search Latency with Fuzzy Match Indexes: How can we optimize the fuzzySearch utility using Trie data structures or pre-computed indexes to maintain sub-millisecond search responses as our records grow?
Customs Port Integration: What integration patterns should we use to connect the WHO nomenclature mapping engine (Section 4) directly with Taiwan's Customs Administration import monitoring systems?
Implementing Automated CAPA Alerts: How can we set up a automated notification system (such as email or SMS) to alert manufacturing facilities and local applicants when active safety recalls are detected?
Inference Guardrails: What sanitization strategies should we use to inspect prompts sent to the /api/ai/analyze endpoint and prevent prompt injection attacks or leakage of system instructions?
Dynamic Chart Customizations: How can we design our Recharts visual dashboards to allow administrators to build custom charts, choose different axes, and select visualization styles dynamically?
Testing Strategy: What integration testing tools (such as Playwright or Cypress) should we use to automate end-to-end testing of user logins, dataset syncs, and AI-driven penalty calculations?
Minimizing Cold Starts: What esbuild optimizations can we apply to the dist/server.cjs build process to further reduce the bundle size and optimize container cold-start times?
AI Grounding Feed Verification: How can we configure a verification step for grounded Google Search queries to verify the source credentials of global recall databases before importing alerts into the local system?
