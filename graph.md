


```
flowchart TD

%%======== LAYER 1: ROLES ========%%
subgraph Roles
  Client[Client]
  POBA[PO / BA]
  Dev[Developer]
  QA[QA Engineer]
  DevOps[DevOps]
  Mgmt[Management]
end

%%======== LAYER 2: SDLC PROCESSES ========%%
subgraph Processes
  Req[Requirements & Discovery]
  DesignCode[Design & Coding]
  CodeReview[Code Review]
  QAProc[QA / Testing]
  Docs[Documentation & Knowledge]
  CICDProc[CI/CD & Incidents]
  ReportOnboard[Reporting & Onboarding]
end

%% Role → Process
Client --> POBA
POBA --> Req
Dev --> DesignCode
Dev --> CodeReview
QA --> QAProc
DevOps --> CICDProc
Mgmt --> ReportOnboard

%%======== LAYER 3: AI TOOLS / CAPABILITIES ========%%
subgraph AITools[AI Tools & Capabilities]
  ReqAI[Requirements AI\n(Claude via ADO MCP)]
  StoriesRefined[Refined User Stories\n(ADO)]
  AmbigFlags[Ambiguity Flags\n(ADO)]

  ContDev[Continue.dev\n(IDE Assistant)]
  Copilot[Copilot\n(optional)]

  PRReviewAI[PR Review AI\n(CodeRabbit / Claude)]
  PRComments[PR Comments & Summaries\n(ADO)]

  TestGen[Test Gen AI\n(Claude)]
  ADOTests[ADO Test Plans]
  CICDTests[CI/CD Regression Tests]

  ObsVault[Obsidian Vault\n(Project Notes, ADRs)]
  N8NIdx[n8n Indexer]
  QdrantDB[Qdrant Vector DB\n+ multilingual-e5]
  RAGAPI[RAG / Knowledge Graph API]

  IncidentAI[Incident AI\n(Claude + n8n)]
  Runbooks[Runbooks\n(Obsidian)]
  AnnotPipes[Annotated Pipelines\n(ADO)]

  ReportAI[Reporting AI\n(Claude + Metabase)]
  Dashboards[Metabase Dashboards]
  StatusReports[Client Status Reports]
  OnboardAI[Onboarding AI\n(Knowledge Packs)]
  ContextPacks[New Joiner Context Packs]
end

%% Requirements flow
Req --> ReqAI
ReqAI --> StoriesRefined
ReqAI --> AmbigFlags

%% Design & Coding flow (Developer + Continue.dev)
DesignCode --> ContDev
DesignCode --> Copilot

%% Code Review flow
CodeReview --> PRReviewAI
PRReviewAI --> PRComments

%% QA / Testing flow
QAProc --> TestGen
TestGen --> ADOTests
TestGen --> CICDTests

%% Documentation & Knowledge flow
Docs --> ObsVault
ObsVault --> N8NIdx
N8NIdx --> QdrantDB
QdrantDB --> RAGAPI

%% CI/CD & Incidents flow
CICDProc --> IncidentAI
IncidentAI --> Runbooks
IncidentAI --> AnnotPipes

%% Reporting & Onboarding flow
ReportOnboard --> ReportAI
ReportAI --> Dashboards
ReportAI --> StatusReports
ReportOnboard --> OnboardAI
OnboardAI --> ContextPacks

%%======== LAYER 4: INFRASTRUCTURE / DATA ========%%
subgraph Infra[Infrastructure & Data]
  ADO[Azure DevOps (ADO)]
  LiteLLM[LiteLLM Gateway]
  Dokploy[Dokploy]
  Obsidian[Obsidian (Git-backed)]
  N8N[n8n Automations]
  Qdrant[Qdrant + multilingual-e5]
  CICDPipes[CI/CD Pipelines]
end

%% ADO connections
ADO <--> StoriesRefined
ADO <--> AmbigFlags
ADO <--> PRComments
ADO <--> ADOTests
ADO <--> AnnotPipes
ADO <--> Dashboards
ADO <--> StatusReports

%% LiteLLM as common gateway
ContDev --> LiteLLM
ReqAI --> LiteLLM
PRReviewAI --> LiteLLM
TestGen --> LiteLLM
IncidentAI --> LiteLLM
ReportAI --> LiteLLM

LiteLLM --> Dokploy

%% Knowledge graph infra
ObsVault --> Obsidian
Obsidian --> N8N
N8N --> Qdrant
Qdrant --> RAGAPI

%% CI/CD infra
CICDPipes --> CICDTests
CICDPipes --> IncidentAI
CICDPipes --> ReportAI

%%======== HIGHLIGHT: DEVELOPER FLOW COMMENT ========%%
%% Developer -> Design & Coding -> Continue.dev -> LiteLLM -> Models/Knowledge Graph
%% Developer -> Code Review -> PR Review AI -> ADO PR Comments -> back to Developer
```


