# CONTRACT (LCA Database)

> 📋 **CONTRACT (LCA Database) v0.1.0**
> Effective: 2026-03-09
> Status: **Draft** — Initial scope definition

---

## 1. Purpose

This contract governs the development, deployment, and maintenance of the **LCA Database** feature of [livingsystems.earth](http://livingsystems.earth/). It defines scope boundaries, build methodology, data governance, and the evolution path from consumer MVP to enterprise platform.

This contract operates under and is subordinate to [CONTRACT (livingsystems.earth)](https://www.notion.so/35560bdc09cf4e78bdf4ea9466ea9f91). Build execution is governed by [CONTRACT (Open-Claw Deployment)](https://www.notion.so/545a69a1ea2a41de9a5fb5edaaaa1ea5).

---

## 2. Feature Definition

The LCA Database is a **public-facing Life Cycle Assessment data tool** hosted on [livingsystems.earth](http://livingsystems.earth/) that enables users to search, browse, and compare environmental impact data for materials, products, and processes.

- **v1 scope:** Consumer MVP — free access to public LCA datasets

- **Evolution target:** B2B enterprise platform (LCA Pro) — licensed datasets, compliance tools, API access

---

## 3. Scope Boundaries

### In Scope (v1)

- Public-facing search and browse interface for LCA data

- Data ingestion from public U.S. federal datasets (USLCI, USDA LCA Digital Commons)

- Impact assessment using TRACI methodology

- Basic comparison tools for consumer users

- Freemium model (free tier only at launch)

### Out of Scope (v1)

- Licensed/proprietary datasets (ecoinvent, GaBi, EU PEF)

- B2B features (compliance workflows, EPD generation, API access)

- Custom/proprietary data upload

- Multi-tenant enterprise workspaces

- Revenue collection (deferred until freemium expansion)

### Deferred (Documented Evolution Targets)

- LCA Pro enterprise platform (see [LCA Pro Technical Reference](https://www.notion.so/99a5eeb05e904512b013536bc3f7923b))

- Licensed dataset integration (gated by B2B revenue)

- EU regulatory compliance module (PEF, DPP, Green Claims Directive)

---

## 4. Build Methodology

### 4.1 Builder: Magi

This project is built by the Magi system under [CONTRACT (Open-Claw Deployment)](https://www.notion.so/545a69a1ea2a41de9a5fb5edaaaa1ea5).

### 4.2 Data Pipeline

An **economical, efficient model** crawls and tabulates the public LCA datasets. This is volume execution work — repetitive, structured data extraction across large datasets — ideally suited to low-cost inference.

Pipeline stages:

1. **Crawl** — Access public APIs and bulk download endpoints

1. **Parse** — Extract process records, flows, and characterization factors

1. **Deduplicate** — Cross-reference across overlapping sources

1. **Normalize** — Map to common internal schema

1. **Store** — Load into backend database

1. **Validate** — Human review at checkpoints

### 4.3 Infrastructure Constraints

- **Modular and portable** — no hard lock-in to any platform

- **Evolution plan required** — documented migration path if starting on a limited platform

- **API-first** — data layer swappable independently of frontend

- Infrastructure platform selection is a **separate decision** requiring a dedicated session

---

## 5. Data Governance

### 5.1 Data Sources

Authoritative data source reference: [LCA Data Sources Reference](https://www.notion.so/c18b5d62281440c7831cf13b4169b9f4)

**v1 Core:**

- USLCI (NREL) — primary inventory database

- USDA LCA Digital Commons — agricultural/food system data

- TRACI (EPA) — impact assessment methodology

**v1.x Expansion Candidates:**

- Federal LCA Commons (multi-agency aggregator)

- ELCD, Agribalyse, BONSAI (free international sources)

### 5.2 Data Quality

- Source provenance tracked for every record

- Data quality flags where source data is dated or incomplete

- Human validation checkpoints in the Magi pipeline

- Users informed of data source, vintage, and quality indicators in the UI

### 5.3 Usage Terms

- All v1 datasets are publicly funded and free to use

- Legal review required before launch to confirm commercial freemium use is compliant with each source's terms

- Licensed datasets (Phase 2+) require separate licensing agreements

---

## 6. Revenue Model

### 6.1 v1: Freemium Loss Leader

- Free access to all public dataset search and browse features

- Purpose: increase visibility of [livingsystems.earth](http://livingsystems.earth/), demonstrate analytical capability

- No revenue collection in v1

### 6.2 Evolution: Economics-Driven Expansion

- Premium features introduced as economics make sense

- B2B pricing introduced when licensed dataset costs are justified by subscription revenue

- Tier structure documented in [LCA Pro Technical Reference](https://www.notion.so/99a5eeb05e904512b013536bc3f7923b)

---

## 7. Architecture Decisions

Canonical scope decisions: [LCA Scope & Architecture Decisions](https://www.notion.so/24b9ac775f234725a04e1f2787dc57f8)

Key constraints carried into all implementation:

1. **Consumer MVP first** — B2B is Phase 2+

1. **Datasets follow scope** — only what the MVP needs

1. **Modular/portable infrastructure** — with evolution plan

1. **Magi builds it** — economical model for volume data work

1. **Freemium loss leader** — free tier for visibility, expand when economics justify

---

## 8. Success Criteria (v1)

- [ ] Public LCA data searchable and browsable on [livingsystems.earth](http://livingsystems.earth/)

- [ ] USLCI, USDA, and TRACI data ingested and normalized

- [ ] Basic comparison functionality available

- [ ] Data provenance and quality indicators visible to users

- [ ] Infrastructure documented with evolution plan

- [ ] Legal review of public dataset usage terms completed

---

## 9. Version History

| **Version** | **Date** | **Change** |
| --- | --- | --- |
| v0.1.0 | 2026-03-09 | Initial draft — scope definition, build methodology, data governance, evolution path |
