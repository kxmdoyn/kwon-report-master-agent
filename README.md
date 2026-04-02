# 📄 Report Master MCP Agent

![Award](https://img.shields.io/badge/Koscom%20AI%20Agent%20Challenge-Grand%20Prize-gold)

🏆 Part of the **Grand Prize–winning project** at the **Koscom AI Agent Challenge 2025**

This repository contains the **report-master MCP server**, a regulatory reporting automation agent developed as part of the **K-WON Risk Management AI Agent** platform.

This module focuses on **LLM-based compliance reporting orchestration**, including structured reasoning workflows, Slack-based approval pipelines, and MCP tool integration.

---

## 🚀 Overview

`report-master` is an MCP-based reporting automation agent designed to support compliance workflows for a KRW stablecoin issuer.

This agent replaces manual regulatory reporting workflows with an automated, LLM-assisted compliance pipeline designed for stablecoin issuance environments.

It automates:

* real-time risk snapshot monitoring
* regulatory compliance report generation
* collateral and liquidity status summarization
* Slack-based human approval workflow
* natural language report interaction via Claude MCP client

---

## ⚡ End-to-End Reporting Flow

```
Real-time monitoring (every 15 min)
→ risk snapshot storage
→ scheduled monthly reasoning pipeline (LangGraph, 12 steps)
→ report generation (DOCX)
→ Slack approval request (Human-in-the-loop interrupt)
→ revision loop if rejected  ← max_revisions 초과 시 강제 종료
→ final report distribution
```

---

## 🏗 Architecture

```text
Claude MCP Client
        ↓
 report-master MCP Server  (gateway.py — Flask, port 5900)
        ↓
 FastAPI Backend            (mcp_server.py — Uvicorn, port 8000)
        ↓
 ┌──────────────────────────────────────────┐
 │ app_mcp/                                 │
 │  ├── graph/      LangGraph workflow      │
 │  ├── core/       scheduler, risk rules   │
 │  ├── tools/      Slack, reserves, banks  │
 │  └── api/        REST endpoints          │
 └──────────────────────────────────────────┘
        ↓
 Slack Approval / Report Artifact / SQLite DB
```

---

## 🧩 Design Decisions

This agent separates:

- **MCP gateway** (Flask) — lightweight tool routing layer for Claude MCP client
- **reasoning backend** (FastAPI) — LangGraph pipeline execution and DB management
- **scheduler orchestration** (APScheduler) — decoupled from request cycle, runs independently
- **human approval workflow** (Slack interrupt) — async loop isolated from main pipeline state

to support modular deployment and independent scaling of each layer.

---

## 📁 Project Structure

```
report-master/
├── gateway.py                      # MCP HTTP Gateway (Flask, port 5900)
├── mcp_server.py                   # FastAPI backend + APScheduler startup
├── report_routes.py                # Report query REST routes
├── report_generator_routes.py      # Report generation REST routes
├── claude_mcp_kwon_reports.py      # Claude MCP tool definitions
├── claude_mcp_provider.py          # Claude API provider
├── artifacts/                      # Generated report files (.docx, .txt)
└── app_mcp/
    ├── graph/
    │   ├── mcp_flow.py             # LangGraph 12-step reasoning pipeline
    │   └── mcp_flow_interrupt.py   # Human Review interrupt handler
    ├── core/
    │   ├── scheduler.py            # APScheduler job registration
    │   ├── risk_rules.py           # Risk grading thresholds (A/B/C/D)
    │   ├── config.py
    │   └── db.py
    ├── tools/
    │   ├── slack.py                # Slack slash command & signature verify
    │   ├── slack_alerts.py         # Real-time alert dispatcher
    │   ├── reserves.py
    │   └── banks.py
    ├── api/
    │   ├── mcp.py                  # MCP run endpoint
    │   ├── human_review.py         # Human review decision endpoint
    │   └── realtime.py             # Real-time snapshot endpoint
    └── models/
        ├── human_review_task.py
        └── realtime_risk_snapshot.py
```

---

## 🔁 Workflow Design

### 1️⃣ Real-time Risk Detection Agent

APScheduler가 **15분마다** 자동으로 `realtime_monitoring_job()`을 실행합니다.

```python
# core/scheduler.py
scheduler.add_job(
    realtime_monitoring_job,
    "interval",
    minutes=15,
)
```

* 온체인/오프체인 스냅샷 수집 및 리스크 룰 평가
* 위험 구간 진입 시 Slack Webhook으로 즉시 경보 발송
* 모든 스냅샷을 DB에 자동 기록

---

### 2️⃣ Monthly Compliance Report Agent

**매월 1일 00:05**에 `run_monthly_mcp_job()` cron이 자동으로 월간 보고서를 생성합니다.

```python
# core/scheduler.py
scheduler.add_job(
    run_monthly_mcp_job,
    "cron",
    day=1, hour=0, minute=5,
)
```

내부적으로 LangGraph 기반 **12단계 reasoning pipeline**을 실행합니다:

```
Load Data
→ Validate Quality
→ Assess Collateral
→ Evaluate Liquidity
→ Verify Proof-of-Reserve
→ Cross-Validate Indicators
→ Summarize Risk Status
→ Generate Report (DOCX)
→ [INTERRUPT] Slack Human Review 요청
→ Wait for Approval  ← human_decision: "pending" | "approve" | "revise"
→ (반려 시 재생성 루프, max_revisions 초과 시 강제 종료)
→ Finalize & Notify
```

**Human-in-the-loop 상세 흐름:**

```
AI가 보고서 초안 생성
    ↓
DB에 HumanReviewTask 생성
    ↓
Slack에 승인 요청 카드 발송 (승인 / 반려 / 재생성 버튼)
    ↓
담당자 클릭
    ↓
승인 → 보고서 확정 및 배포
반려 → revision_count 증가 후 재생성 루프
```

---

### 3️⃣ Interactive MCP Tool Agent

Claude가 자연어 질문을 해석해 적절한 MCP Tool을 자동으로 선택하고 실행합니다.

Example queries:

* "Generate this month's compliance report"
* "Show current collateral ratio"
* "Summarize recent compliance alerts"

---

## 📐 Risk Grading Logic

직접 설계한 도메인 리스크 룰셋입니다. 담보율, 페그 이탈, 유동성 비율 각각에 대해 A~D 등급 기준을 정의하고, 종합 등급으로 집계합니다.

```python
# core/risk_rules.py

# 담보 비율 (준비금 / 발행량)
COLLATERAL_THRESHOLDS = {
    "A": 1.15,   # >= 115%  안전 마진 확보
    "B": 1.10,   # >= 110%  양호
    "C": 1.03,   # >= 103%  최소 안전 마진
                 #  < 103%  D — 즉시 조치 필요
}

# 페그 이탈 (절댓값, 작을수록 안전)
PEG_DEVIATION_THRESHOLDS = {
    "A": 0.002,  # <= 0.2%  정상
    "B": 0.005,  # <= 0.5%  주의
    "C": 0.010,  # <= 1.0%  경고
                 #  > 1.0%  D — 위기
}
```

실시간 알림 레벨은 별도 `RiskLevel` Enum (`OK` / `WARN` / `CRIT`)으로 운영 알림 기준을 분리했습니다.

---

## 🧰 MCP Tools

| Tool | Description |
|---|---|
| `run_monthly_report(period)` | Generates compliance report |
| `get_latest_report()` | Returns latest report summary |
| `get_report(period)` | Retrieves specific report metadata |
| `get_collateral_status()` | Shows collateral safety level |
| `get_risk_summary()` | Returns integrated risk overview |
| `get_compliance_alerts()` | Lists recent compliance alerts |
| `get_human_review_tasks()` | Lists pending human review tasks |

---

## ⚙ Tech Stack

* Python
* FastAPI + Flask
* APScheduler (interval & cron jobs)
* LangGraph (multi-step reasoning + interrupt)
* MCP (Model Context Protocol)
* Claude API
* Slack Webhook + Slash Command

---

## ▶ Getting Started

### 1️⃣ Clone repository

```bash
git clone https://github.com/kxmdoyn/kwon-report-master-agent.git
cd kwon-report-master-agent
```

### 2️⃣ Setup environment variables

```bash
cp .env.example .env
```

### 3️⃣ Install dependencies

```bash
pip install -r requirements.txt
```

### 4️⃣ Run server

```bash
python gateway.py
```

---

## 👩‍💻 My Contribution

As part of **Team DANCOM**, I designed and implemented the **report-master MCP server**.

My responsibilities included:

* designing the regulatory reporting workflow
* structuring multi-step LangGraph reasoning pipeline with Human-in-the-loop interrupt
* defining risk grading thresholds (collateral, peg deviation, liquidity)
* implementing APScheduler-based automated job scheduling (15min interval + monthly cron)
* implementing Slack-based approval system with signature verification
* connecting Claude MCP tools with reporting routes
* organizing automated report generation architecture

---

## 🏆 Competition Context

This module was developed as part of the larger:

**K-WON Risk Management AI Agent Platform**

which integrates:

* KRW reserve validation agent (`krw-full-reserve`)
* bank monitoring agent (`bank_monitering`)
* compliance reporting agent (`report-master`)
* transaction audit agent (`tx_audit`)

The full system won the **Grand Prize** at the **Koscom AI Agent Challenge 2025**.

→ [Full project repository](https://github.com/dancom-MCP-AI-Agent/final_koscom_ai_agent_public)

---

## 🔐 Security Notice

Sensitive runtime configuration is excluded.

Please use `.env.example` for environment setup.

---

# 🇰🇷 프로젝트 설명 (Korean Summary)

본 저장소는 **Koscom AI Agent Challenge 2025 대상 수상 프로젝트**인

**K-WON 원화 스테이블코인 리스크 관리 AI Agent 시스템**

중 제가 담당한 **report-master MCP 서버**를 정리한 포트폴리오용 레포입니다.

이 모듈은 다음 기능을 수행합니다:

* 규제 준수 보고서 자동 생성
* 담보율 및 유동성 상태 분석 (A/B/C/D 등급 리스크 룰셋 직접 설계)
* APScheduler 기반 자동화 (15분 실시간 감지 + 매월 1일 cron 보고서 생성)
* LangGraph 12단계 reasoning pipeline + Human-in-the-loop 반려/재생성 루프
* Slack 기반 담당자 승인 workflow (서명 검증 포함)
* 자연어 기반 보고서 조회 기능 제공

본 레포는 전체 팀 프로젝트 중 **보고서 자동화 Agent 설계 및 구현 부분**을 중심으로 정리한 개인 기여 포트폴리오입니다.
