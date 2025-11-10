---
title: Solution Release & Deployment Framework
owner: Visionwaves Engineering
status: Active
version: 1.0
date: 2025-11-09
classification: Internal/Customer-Shareable
---

<div style="page-break-after: always;"></div>

# Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Purpose & Scope](#2-purpose--scope)
3. [Key Concepts & Terminology](#3-key-concepts--terminology)
4. [Architecture Overview](#4-architecture-overview)
5. [Repository Structure & Git Strategy](#5-repository-structure--git-strategy)
6. [Data Model & Database Schema](#6-data-model--database-schema)
7. [Release Workflow](#7-release-workflow)
8. [Roles & Responsibilities](#8-roles--responsibilities)
9. [Standard Operating Procedures (SOP)](#9-standard-operating-procedures-sop)
10. [CI/CD Pipeline Integration](#10-cicd-pipeline-integration)
11. [Configuration Release Unit (CRU) Management](#11-configuration-release-unit-cru-management)
12. [Deployment Promotion Strategy](#12-deployment-promotion-strategy)
13. [Monitoring & Reporting](#13-monitoring--reporting)
14. [Naming Conventions & Standards](#14-naming-conventions--standards)
15. [Appendices](#15-appendices)
16. [BOM (Bill of Materials) Management](#16-bom-bill-of-materials-management)


## 1. Executive Summary

This document establishes the **Solution Release Management Framework** for Visionwaves products and customer deployments. It provides a structured approach to:

- **Bundle multiple microservices** into cohesive solution releases
- **Manage customer-specific customizations** alongside product releases
- **Deploy unified configurations** through Configuration Release Units (CRU)
- **Track and audit** all deployments across environments
- **Enable safe promotion** from SIT → PreProd → Production

### Key Benefits

| Benefit | Description |
|---------|-------------|
| ✅ **Single Source of Truth** | Database-driven tracking of what's deployed where |
| ✅ **Atomic Deployments** | Deploy multiple services as one cohesive unit |
| ✅ **Clear Separation** | Distinct product vs. customer-specific code |
| ✅ **Automated Tracking** | Deployment history and audit trails |
| ✅ **Safe Promotion** | Controlled progression through environments |
| ✅ **Simplified Rollback** | Quick recovery to previous stable releases |

<div style="page-break-after: always;"></div>

---

## 2. Purpose & Scope

### 2.1 Purpose

This framework defines how **product owners**, **release managers**, and **DevOps engineers** collaborate to:

1. Create solution releases containing multiple product microservices
2. Package customer-specific customizations
3. Deploy unified configurations via CRU tags
4. Promote releases across environments safely
5. Maintain deployment history and audit trails

### 2.2 Scope

**In Scope:**
- ✅ Solution release creation and management
- ✅ Component (microservice) versioning and tracking
- ✅ Configuration management via CRU
- ✅ Deployment automation and promotion
- ✅ Environment-specific deployment tracking
- ✅ Rollback and recovery procedures

**Out of Scope:**
- ❌ Individual developer workflows
- ❌ Code review processes (covered in separate documents)
- ❌ Testing strategies (covered in QA documentation)
- ❌ Infrastructure provisioning
- ❌ Disaster recovery procedures

### 2.3 Target Audience

| Role | How to Use This Document |
|------|--------------------------|
| **Release Managers** | Sections 7, 9, 11 - Creating and managing releases |
| **Product Owners** | Sections 7, 9 - Adding components to releases |
| **DevOps Engineers** | Sections 10, 12, 15 - Deployment and promotion |
| **Developers** | Sections 3, 5, 11 - Understanding CRU and versioning |
| **Management** | Sections 1, 4, 8 - Overview and responsibilities |

<div style="page-break-after: always;"></div>

---

## 3. Key Concepts & Terminology

### 3.1 Core Definitions

| Term | Definition | Example |
|------|------------|---------|
| **Product** | A standalone microservice or module maintained by Visionwaves | `product-lcm`, `product-identity`, `product-scheduler` |
| **Customer Product Repo** | A forked/customized version of a product for a specific customer | `customer-acme-lcm` |
| **Solution** | A logical bundle of multiple products deployed together | `HumainOS-Core`, `HumainOS-Extended` |
| **Solution Release** | A versioned snapshot of a solution for a customer | `SL-ACME-006` |
| **Component** | A single microservice within a solution | `lcm`, `identity`, `scheduler` |
| **Component Release** | Metadata defining how a component is built and deployed | Repo URL, branch/tag, Docker image, Helm chart |
| **CRU (Configuration Release Unit)** | A single tag grouping all UI/workflow/grid configurations | `CUST-ACME-R07` |
| **Deployments Repo** | Git repository with environment branches pointing to solution releases | `deployments/cust/ACME/sit` |
| **Product Version** | Base Visionwaves product version | `1.12.3` |
| **Customer Version** | Customer-specific version including customizations | `2.3.0` |

### 3.2 Version Hierarchy

```
Solution Release: SL-ACME-006
├── Product Version: 1.12.3 (base Visionwaves release)
├── Customer Version: 2.3.0 (includes customizations)
├── CRU Tag: CUST-ACME-R07 (configuration bundle)
└── Components:
    ├── lcm: 2.3.0-CUST-ACME-R07
    ├── identity: 2.3.0-CUST-ACME-R07
    └── scheduler: 2.3.0-CUST-ACME-R07
```

### 3.3 Release Status Lifecycle

```
DRAFT ──────► READY ──────► LOCKED ──────► ARCHIVED
  │             │              │               │
  │             │              │               │
  ▼             ▼              ▼               ▼
Being         Ready for      Deployed to     Deprecated
prepared      deployment     production      & retired
```

**Status Definitions:**

| Status | Meaning | Can Modify? | Typical Duration |
|--------|---------|-------------|------------------|
| `DRAFT` | Release is being prepared | ✅ Yes | 1-3 days |
| `READY` | Complete and ready for deployment | ⚠️ Limited | Until deployed |
| `LOCKED` | Deployed to production | ❌ No | Active release |
| `ARCHIVED` | Deprecated and no longer in use | ❌ No | Historical |

<div style="page-break-after: always;"></div>

---

## 4. Architecture Overview

### 4.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    SOLUTION RELEASE                         │
│                     SL-ACME-006                             │
│                                                             │
│  Product Version: 1.12.3                                    │
│  Customer Version: 2.3.0                                    │
│  CRU Tag: CUST-ACME-R07                                     │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Component 1  │   │  Component 2  │   │  Component 3  │
│     LCM       │   │   Identity    │   │  Scheduler    │
├───────────────┤   ├───────────────┤   ├───────────────┤
│ Repo: cust-   │   │ Repo: product-│   │ Repo: product-│
│ acme-lcm      │   │ identity      │   │ scheduler     │
│ Tag: lcm-     │   │ Tag: v1.9.1   │   │ Tag: v1.7.2   │
│ acme-2.3.0    │   │               │   │               │
│ Image: lcm:   │   │ Image:        │   │ Image:        │
│ 2.3.0-CUST-   │   │ identity:     │   │ scheduler:    │
│ ACME-R07      │   │ 2.3.0-CUST-   │   │ 2.3.0-CUST-   │
│               │   │ ACME-R07      │   │ ACME-R07      │
└───────────────┘   └───────────────┘   └───────────────┘
```

### 4.2 Deployment Flow

```
┌──────────────┐
│ Release      │  1. Creates Solution Release (DRAFT)
│ Manager      │  2. Assigns CRU Tag
└──────┬───────┘  3. Sets status to READY
       │
       ▼
┌──────────────┐
│ Product      │  4. Add Component Release entries
│ Owners       │  5. Specify repos, tags, images, charts
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ DevOps       │  6. Update deployments repo (PR)
│ Engineer     │  7. Merge to environment branch
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ CI/CD        │  8. Read release.yaml
│ Pipeline     │  9. Deploy all components
└──────┬───────┘ 10. Apply CRU configurations
       │         11. Run smoke tests
       ▼
┌──────────────┐
│ Environment  │ 12. Record deployment in env_deploy
│ (SIT/Preprod │ 13. Update status (SUCCESS/FAILED)
│ /Prod)       │
└──────────────┘
```

### 4.3 System Components

| Component | Purpose | Technology |
|-----------|---------|------------|
| **Solution Release DB** | Stores release metadata | PostgreSQL/MySQL |
| **Deployments Repo** | Git-based deployment pointers | Git |
| **Product Repos** | Source code for Visionwaves products | Git |
| **Customer Repos** | Customer-specific customizations | Git |
| **Docker Registry** | Container image storage | Docker Registry / Harbor |
| **Helm Charts** | Kubernetes deployment templates | Helm 3 |
| **CI/CD Pipeline** | Automated deployment orchestration | Jenkins / GitLab CI |
| **Kubernetes Clusters** | Runtime environments (SIT/PreProd/Prod) | Kubernetes |

<div style="page-break-after: always;"></div>

---

## 5. Repository Structure & Git Strategy

### 5.1 Standard Folder Structure

All repositories (Backend and UI) **MUST** follow this uniform folder structure across **ALL branches**:
- Product main branch
- Product version branches (1.X.X, 2.X.X)
- Customer deployment branches
- Customer customized code branches

This uniformity enables:
- ✅ Automated CI/CD pipeline configuration
- ✅ Consistent developer commit patterns
- ✅ Automated CRU-based deployment
- ✅ Environment-specific configuration tracking

---

### 5.2 Backend Repository Structure

```
Backend/
├── src/                           # Source code (Java/Spring Boot)
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   └── test/
│
├── configurations/                # Configuration root (CRU-based)
│   │
│   ├── helm/                      # Helm charts for Kubernetes deployment
│   │   └── <service-name>/        # e.g., lcm-service, order-service
│   │       ├── main/              # Base chart (ALWAYS present)
│   │       │   ├── Chart.yaml
│   │       │   ├── values.yaml
│   │       │   └── templates/
│   │       └── <CRU-TAG>/         # Customer-specific overrides (OPTIONAL)
│   │           └── values.yaml    # e.g., CUST-ACME-R07/values.yaml
│   │
│   ├── release-note/              # Solution release notes (NOT CRU-based)
│   │   └── <SOLUTION-TAG>/        # e.g., SL-ACME-006/
│   │       └── release-notes.md
│   │
│   ├── cdc-script/                # Change Data Capture scripts
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── cdc-config.json
│   │
│   ├── sql/                       # Database migration scripts
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       ├── V001__initial.sql
│   │       └── V002__add_columns.sql
│   │
│   ├── api-doc/                   # API documentation (Swagger/OpenAPI)
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── swagger.yaml
│   │
│   ├── workflow/                  # Workflow definitions
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── order-workflow.json
│   │
│   ├── initial-objects/           # Initial data objects
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── seed-data.json
│   │
│   ├── documentation/             # Technical documentation
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── architecture.md
│   │
│   ├── permission-groups/         # RBAC permission groups
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── permissions.json
│   │
│   ├── profile-templates/         # User profile templates
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── templates.json
│   │
│   ├── bi-dashboards/             # Business Intelligence dashboards
│   │   ├── widget/                # Dashboard widgets
│   │   │   └── <CRU-TAG>/         # e.g., CUST-ACME-R07/
│   │   │       └── sales-widget.json
│   │   ├── dashboards/            # Dashboard definitions
│   │   │   └── <CRU-TAG>/         # e.g., CUST-ACME-R07/
│   │   │       └── executive-dashboard.json
│   │   └── datasource/            # Data source configurations
│   │       └── <CRU-TAG>/         # e.g., CUST-ACME-R07/
│   │           └── mysql-datasource.json
│   │
│   ├── aoa-templates/             # Area of Application templates
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── aoa-template.json
│   │
│   ├── modules/                   # Module configurations
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── module-config.json
│   │
│   ├── multi-lingual/             # Internationalization (i18n)
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       ├── en_US.json
│   │       ├── fr_FR.json
│   │       └── es_ES.json
│   │
│   ├── base-configuration/        # Base configuration properties
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── base-config.json
│   │
│   ├── business-configuration/    # Business configuration properties
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── business-config.json
│   │
│   ├── data-api/                  # Data API configurations
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── data-api.json
│   │
│   ├── jobs/                      # Batch jobs and ETL
│   │   ├── nifi-jobs/             # Apache NiFi jobs
│   │   │   └── <CRU-TAG>/         # e.g., CUST-ACME-R07/
│   │   │       └── data-ingestion.xml
│   │   └── spark-jobs/            # Apache Spark jobs
│   │       └── <CRU-TAG>/         # e.g., CUST-ACME-R07/
│   │           └── etl-job.py
│   │
│   ├── agents/                    # Agent configurations
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── monitoring-agent.json
│   │
│   ├── notifications/             # Notification templates
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── email-templates.json
│   │
│   ├── naming-identity/           # Naming and identity configurations
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── naming-rules.json
│   │
│   ├── unit-test-report/          # Unit test reports (NO CRU - branch level)
│   │   └── <SOLUTION-TAG>/        # e.g., SL-ACME-006/
│   │       └── junit-report.xml
│   │
│   ├── qa-test-cases/             # QA test cases (NO CRU - branch level)
│   │   └── <SOLUTION-TAG>/        # e.g., SL-ACME-006/
│   │       └── test-cases.xlsx
│   │
│   ├── security-reports/          # Security scan reports (NO CRU - branch level)
│   │   ├── sonar/                 # SonarQube reports
│   │   │   └── <SOLUTION-TAG>/    # e.g., SL-ACME-006/
│   │   ├── snyk/                  # Snyk vulnerability reports
│   │   │   └── <SOLUTION-TAG>/
│   │   ├── trivy/                 # Trivy container scanning
│   │   │   └── <SOLUTION-TAG>/
│   │   ├── owasp/                 # OWASP dependency check
│   │   │   └── <SOLUTION-TAG>/
│   │   ├── fossa/                 # FOSSA license compliance
│   │   │   └── <SOLUTION-TAG>/
│   │   └── burp/                  # Burp Suite security testing
│   │       └── <SOLUTION-TAG>/
│   │
│   └── testcases/                 # Integration test cases (NO CRU - branch level)
│       └── <SOLUTION-TAG>/        # e.g., SL-ACME-006/
│           └── integration-tests.json
│
├── pom.xml                        # Maven build configuration
├── Dockerfile                     # Docker image definition
└── README.md                      # Repository documentation
```

---

### 5.3 UI Repository Structure

```
UI/
├── src/                           # Source code (React/Angular/Vue)
│   ├── components/
│   ├── pages/
│   ├── services/
│   └── assets/
│
├── configurations/                # Configuration root (CRU-based)
│   │
│   ├── helm/                      # Helm charts for Kubernetes deployment
│   │   └── <service-name>/        # e.g., lcm-ui, order-ui
│   │       ├── main/              # Base chart (ALWAYS present)
│   │       │   ├── Chart.yaml
│   │       │   ├── values.yaml
│   │       │   └── templates/
│   │       └── <CRU-TAG>/         # Customer-specific overrides (OPTIONAL)
│   │           └── values.yaml    # e.g., CUST-ACME-R07/values.yaml
│   │
│   ├── grids/                     # Grid configurations
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── order-grid.json
│   │
│   ├── pages/                     # Page configurations
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── dashboard-page.json
│   │
│   ├── forms/                     # Form configurations
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── order-form.json
│   │
│   ├── documentation/             # UI documentation
│   │   └── <CRU-TAG>/             # e.g., CUST-ACME-R07/
│   │       └── ui-guide.md
│   │
│   ├── security-reports/          # Security scan reports (NO CRU - branch level)
│   │   ├── sonar/                 # SonarQube reports
│   │   │   └── <SOLUTION-TAG>/    # e.g., SL-ACME-006/
│   │   ├── snyk/                  # Snyk vulnerability reports
│   │   │   └── <SOLUTION-TAG>/
│   │   ├── trivy/                 # Trivy container scanning
│   │   │   └── <SOLUTION-TAG>/
│   │   ├── owasp/                 # OWASP dependency check
│   │   │   └── <SOLUTION-TAG>/
│   │   └── fossa/                 # FOSSA license compliance
│   │       └── <SOLUTION-TAG>/
│   │
│   └── testcases/                 # UI test cases (NO CRU - branch level)
│       └── <SOLUTION-TAG>/        # e.g., SL-ACME-006/
│           └── e2e-tests.json
│
├── package.json                   # NPM dependencies
├── Dockerfile                     # Docker image definition
└── README.md                      # Repository documentation
```

---

### 5.4 CRU-Based Configuration Management

#### 5.4.1 CRU Folder Naming Convention

**Format:** `<CRU-TAG>` from the database

**Examples:**
- `CUST-ACME-R07` (Customer: ACME, Release: 07)
- `CUST-BETA-R03` (Customer: BETA, Release: 03)
- `CUST-GAMMA-R12` (Customer: GAMMA, Release: 12)

#### 5.4.2 Pipeline Automation Logic

The CI/CD pipeline uses the following logic to process configurations:

```bash
#!/bin/bash
# CI/CD Pipeline Configuration Processor

CRU_TAG="CUST-ACME-R07"  # From solution_release.cru_tag

# Process each configuration folder
for config_folder in configurations/*; do
    folder_name=$(basename "$config_folder")
    
    # Skip non-CRU folders (reports, testcases)
    if [[ "$folder_name" =~ ^(unit-test-report|qa-test-cases|security-reports|testcases|release-note)$ ]]; then
        echo "⏭️  Skipping $folder_name (not CRU-based)"
        continue
    fi
    
    # Check if CRU folder exists
    if [ -d "$config_folder/$CRU_TAG" ]; then
        echo "✅ Processing $folder_name/$CRU_TAG"
        deploy_configuration "$config_folder/$CRU_TAG"
    else
        echo "⏭️  Skipping $folder_name (no CRU folder: $CRU_TAG)"
    fi
done
```

#### 5.4.3 Special Case: Helm Charts

Helm charts have a **two-tier structure**:

```bash
#!/bin/bash
# Helm Chart Deployment Logic

CRU_TAG="CUST-ACME-R07"

for service in configurations/helm/*; do
    service_name=$(basename "$service")
    
    # Step 1: Deploy base chart (ALWAYS)
    if [ -d "$service/main" ]; then
        echo "📦 Deploying base chart: $service_name/main"
        helm upgrade --install "$service_name" "$service/main" \
            --namespace production
    else
        echo "❌ ERROR: Missing base chart: $service/main"
        exit 1
    fi
    
    # Step 2: Apply CRU-specific overrides (OPTIONAL)
    if [ -d "$service/$CRU_TAG" ]; then
        echo "🔧 Applying CRU overrides: $service_name/$CRU_TAG"
        helm upgrade "$service_name" "$service/main" \
            --namespace production \
            --values "$service/$CRU_TAG/values.yaml"
    else
        echo "ℹ️  No CRU overrides for: $service_name"
    fi
done
```

**Example Helm Structure:**

```
configurations/helm/lcm-service/
├── main/                          # Base chart (product-level)
│   ├── Chart.yaml
│   ├── values.yaml                # Default values
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
│
└── CUST-ACME-R07/                 # Customer-specific overrides
    └── values.yaml                # Override values only
```

**CUST-ACME-R07/values.yaml** (overrides only):
```yaml
# Override specific values for ACME customer
replicaCount: 5                    # Override from main/values.yaml

image:
  tag: "2.3.0-acme"                # Customer-specific image tag

env:
  - name: CUSTOMER_NAME
    value: "ACME"
  - name: CUSTOM_FEATURE_ENABLED
    value: "true"

resources:
  limits:
    memory: "4Gi"                  # ACME needs more memory
```

---

### 5.5 Branch Strategy and Folder Structure

#### 5.5.1 Product Main Branch

**Branch:** `main` or `master`

**Purpose:** Base product development

**Folder Structure:** Full structure (as defined in 5.2 and 5.3)

**CRU Folders:** May contain sample/template CRU folders for testing

**Example:**
```
Backend/configurations/
├── helm/lcm-service/main/         ✅ Always present
├── workflow/CUST-SAMPLE-R01/      ✅ Sample for testing
└── grids/                         ✅ Empty or sample
```

---

#### 5.5.2 Product Version Branches

**Branch:** `1.X.X`, `2.X.X` (e.g., `1.12.3`, `2.3.0`)

**Purpose:** Stable product releases

**Folder Structure:** Full structure (identical to main)

**CRU Folders:** May contain version-specific configurations

**Example:**
```
Branch: 2.3.0

Backend/configurations/
├── helm/lcm-service/main/         ✅ Version 2.3.0 base chart
├── sql/                           ✅ Empty (no customer yet)
└── workflow/                      ✅ Empty (no customer yet)
```

---

#### 5.5.3 Customer Deployment Branches

**Branch:** `deploy/<customer>/<env>` (e.g., `deploy/ACME/production`)

**Purpose:** Customer-specific deployment configurations

**Folder Structure:** Full structure (identical to main)

**CRU Folders:** Contains actual customer CRU configurations

**Example:**
```
Branch: deploy/ACME/production

Backend/configurations/
├── helm/lcm-service/
│   ├── main/                      ✅ Base chart
│   └── CUST-ACME-R07/             ✅ ACME production overrides
├── sql/CUST-ACME-R07/             ✅ ACME SQL migrations
├── workflow/CUST-ACME-R07/        ✅ ACME workflows
├── grids/CUST-ACME-R07/           ✅ ACME grid configs
└── release-note/SL-ACME-006/      ✅ Solution release notes
```

**Tracking:** The `env_deploy` table tracks which CRU is deployed:

```sql
SELECT environment, cru_tag, deployed_at 
FROM env_deploy 
WHERE solution_release_id = (
    SELECT id FROM solution_release WHERE release_tag = 'SL-ACME-006'
);
```

Result:
```
| environment | cru_tag         | deployed_at         |
|-------------|-----------------|---------------------|
| production  | CUST-ACME-R07   | 2024-11-10 14:30:00 |
```

---

#### 5.5.4 Customer Customized Code Branches

**Branch:** `custom/<customer>` (e.g., `custom/ACME`)

**Purpose:** Customer-specific code customizations

**Folder Structure:** Full structure (identical to main)

**CRU Folders:** Contains customer-specific code AND configurations

**Example:**
```
Branch: custom/ACME

Backend/
├── src/main/java/com/acme/custom/  ✅ ACME custom code
├── configurations/
│   ├── helm/lcm-service/
│   │   ├── main/                   ✅ Base chart
│   │   └── CUST-ACME-R07/          ✅ ACME overrides
│   ├── sql/CUST-ACME-R07/          ✅ ACME SQL
│   ├── workflow/CUST-ACME-R07/     ✅ ACME workflows
│   └── api-doc/CUST-ACME-R07/      ✅ ACME API docs
```

---

### 5.6 Configuration Folder Categories

#### 5.6.1 CRU-Based Folders (Automated Deployment)

These folders **REQUIRE** CRU subfolders for pipeline automation:

| Folder | Purpose | CRU Required | Example Path |
|--------|---------|--------------|--------------|
| `helm/<service>/<CRU>` | Helm value overrides | ✅ Yes | `helm/lcm-service/CUST-ACME-R07/values.yaml` |
| `cdc-script/<CRU>` | CDC configurations | ✅ Yes | `cdc-script/CUST-ACME-R07/cdc.json` |
| `sql/<CRU>` | Database migrations | ✅ Yes | `sql/CUST-ACME-R07/V001__init.sql` |
| `api-doc/<CRU>` | API documentation | ✅ Yes | `api-doc/CUST-ACME-R07/swagger.yaml` |
| `workflow/<CRU>` | Workflow definitions | ✅ Yes | `workflow/CUST-ACME-R07/order.json` |
| `initial-objects/<CRU>` | Seed data | ✅ Yes | `initial-objects/CUST-ACME-R07/data.json` |
| `permission-groups/<CRU>` | RBAC permissions | ✅ Yes | `permission-groups/CUST-ACME-R07/perms.json` |
| `profile-templates/<CRU>` | User profiles | ✅ Yes | `profile-templates/CUST-ACME-R07/template.json` |
| `bi-dashboards/widget/<CRU>` | BI widgets | ✅ Yes | `bi-dashboards/widget/CUST-ACME-R07/widget.json` |
| `bi-dashboards/dashboards/<CRU>` | BI dashboards | ✅ Yes | `bi-dashboards/dashboards/CUST-ACME-R07/dash.json` |
| `bi-dashboards/datasource/<CRU>` | BI datasources | ✅ Yes | `bi-dashboards/datasource/CUST-ACME-R07/ds.json` |
| `aoa-templates/<CRU>` | AOA templates | ✅ Yes | `aoa-templates/CUST-ACME-R07/aoa.json` |
| `modules/<CRU>` | Module configs | ✅ Yes | `modules/CUST-ACME-R07/module.json` |
| `multi-lingual/<CRU>` | i18n translations | ✅ Yes | `multi-lingual/CUST-ACME-R07/en_US.json` |
| `base-configuration/<CRU>` | Base configs | ✅ Yes | `base-configuration/CUST-ACME-R07/config.json` |
| `business-configuration/<CRU>` | Business configs | ✅ Yes | `business-configuration/CUST-ACME-R07/config.json` |
| `data-api/<CRU>` | Data API configs | ✅ Yes | `data-api/CUST-ACME-R07/api.json` |
| `jobs/nifi-jobs/<CRU>` | NiFi jobs | ✅ Yes | `jobs/nifi-jobs/CUST-ACME-R07/job.xml` |
| `jobs/spark-jobs/<CRU>` | Spark jobs | ✅ Yes | `jobs/spark-jobs/CUST-ACME-R07/job.py` |
| `agents/<CRU>` | Agent configs | ✅ Yes | `agents/CUST-ACME-R07/agent.json` |
| `notifications/<CRU>` | Notification templates | ✅ Yes | `notifications/CUST-ACME-R07/email.json` |
| `naming-identity/<CRU>` | Naming rules | ✅ Yes | `naming-identity/CUST-ACME-R07/rules.json` |
| `grids/<CRU>` (UI) | Grid configurations | ✅ Yes | `grids/CUST-ACME-R07/order-grid.json` |
| `pages/<CRU>` (UI) | Page configurations | ✅ Yes | `pages/CUST-ACME-R07/dashboard.json` |
| `forms/<CRU>` (UI) | Form configurations | ✅ Yes | `forms/CUST-ACME-R07/order-form.json` |

**Pipeline Behavior:** If CRU folder **NOT** present → **SKIP** deployment

---

#### 5.6.2 Solution-Based Folders (Branch-Level)

These folders use **Solution Release Tag** instead of CRU:

| Folder | Purpose | Tag Format | Example Path |
|--------|---------|------------|--------------|
| `release-note/<SOLUTION>` | Release notes | Solution tag | `release-note/SL-ACME-006/notes.md` |
| `unit-test-report/<SOLUTION>` | Unit test reports | Solution tag | `unit-test-report/SL-ACME-006/junit.xml` |
| `qa-test-cases/<SOLUTION>` | QA test cases | Solution tag | `qa-test-cases/SL-ACME-006/tests.xlsx` |
| `security-reports/*/<SOLUTION>` | Security reports | Solution tag | `security-reports/sonar/SL-ACME-006/report.json` |
| `testcases/<SOLUTION>` | Integration tests | Solution tag | `testcases/SL-ACME-006/integration.json` |

**Pipeline Behavior:** These are **NOT** deployed, only stored for tracking

---

### 5.7 Developer Commit Guidelines

#### 5.7.1 Adding New Configuration

**Scenario:** Developer needs to add a new workflow for ACME customer (CRU: CUST-ACME-R07)

**Steps:**

1. **Create CRU folder** (if not exists):
   ```bash
   mkdir -p Backend/configurations/workflow/CUST-ACME-R07
   ```

2. **Add configuration file**:
   ```bash
   vi Backend/configurations/workflow/CUST-ACME-R07/order-approval.json
   ```

3. **Commit with proper message**:
   ```bash
   git add Backend/configurations/workflow/CUST-ACME-R07/
   git commit -m "feat(workflow): Add order approval workflow for ACME [CUST-ACME-R07]"
   git push origin deploy/ACME/production
   ```

4. **Pipeline automatically detects and deploys**:
   ```
   ✅ Processing workflow/CUST-ACME-R07
   ✅ Deploying order-approval.json to production
   ```

---

#### 5.7.2 Updating Helm Chart for Customer

**Scenario:** ACME needs more memory for lcm-service

**Steps:**

1. **Create/Update CRU override**:
   ```bash
   mkdir -p Backend/configurations/helm/lcm-service/CUST-ACME-R07
   vi Backend/configurations/helm/lcm-service/CUST-ACME-R07/values.yaml
   ```

2. **Add override values**:
   ```yaml
   resources:
     limits:
       memory: "4Gi"
       cpu: "2000m"
   ```

3. **Commit**:
   ```bash
   git add Backend/configurations/helm/lcm-service/CUST-ACME-R07/
   git commit -m "chore(helm): Increase memory for ACME lcm-service [CUST-ACME-R07]"
   git push origin deploy/ACME/production
   ```

4. **Pipeline applies override**:
   ```bash
   📦 Deploying base chart: lcm-service/main
   🔧 Applying CRU overrides: lcm-service/CUST-ACME-R07
   ✅ Helm upgrade successful
   ```

---

#### 5.7.3 Adding Release Notes

**Scenario:** Document changes for solution release SL-ACME-006

**Steps:**

1. **Create solution folder**:
   ```bash
   mkdir -p Backend/configurations/release-note/SL-ACME-006
   vi Backend/configurations/release-note/SL-ACME-006/release-notes.md
   ```

2. **Write release notes**:
   ```markdown
   # Release Notes - SL-ACME-006
   
   **Date:** 2024-11-10
   **CRU:** CUST-ACME-R07
   **Product Version:** 7.0.12
   
   ## New Features
   - Order approval workflow
   - Enhanced dashboard
   
   ## Bug Fixes
   - Fixed memory leak in lcm-service
   ```

3. **Commit**:
   ```bash
   git add Backend/configurations/release-note/SL-ACME-006/
   git commit -m "docs: Add release notes for SL-ACME-006"
   git push origin deploy/ACME/production
   ```

---

### 5.8 Folder Structure Validation

#### 5.8.1 Pre-Commit Hook

Add this to `.git/hooks/pre-commit`:

```bash
#!/bin/bash
# Validate folder structure before commit

VALID_CONFIG_FOLDERS=(
    "helm" "release-note" "cdc-script" "sql" "api-doc" "workflow"
    "initial-objects" "documentation" "permission-groups" "profile-templates"
    "bi-dashboards" "aoa-templates" "modules" "multi-lingual"
    "base-configuration" "business-configuration" "data-api" "jobs"
    "agents" "notifications" "naming-identity" "unit-test-report"
    "qa-test-cases" "security-reports" "testcases" "grids" "pages" "forms"
)

# Check if configurations folder exists
if [ -d "configurations" ]; then
    for folder in configurations/*; do
        folder_name=$(basename "$folder")
        
        # Check if folder is valid
        if [[ ! " ${VALID_CONFIG_FOLDERS[@]} " =~ " ${folder_name} " ]]; then
            echo "❌ ERROR: Invalid configuration folder: $folder_name"
            echo "Valid folders: ${VALID_CONFIG_FOLDERS[@]}"
            exit 1
        fi
    done
fi

echo "✅ Folder structure validation passed"
exit 0
```

Make it executable:
```bash
chmod +x .git/hooks/pre-commit
```

---

### 5.9 Folder Structure Checklist

#### For Developers

- [ ] All configuration changes are in the correct CRU folder
- [ ] CRU folder name matches database CRU tag (e.g., CUST-ACME-R07)
- [ ] Helm charts have `main/` folder with base chart
- [ ] Helm overrides are in `<service>/<CRU>/values.yaml`
- [ ] Release notes are in `release-note/<SOLUTION-TAG>/`
- [ ] Test reports are in appropriate `<SOLUTION-TAG>/` folders
- [ ] Commit message includes CRU tag: `[CUST-ACME-R07]`
- [ ] No configuration files in root of configuration folders

#### For DevOps

- [ ] Pipeline configured to read CRU tag from `solution_release.cru_tag`
- [ ] Pipeline skips folders without CRU subfolders
- [ ] Helm deployment applies base chart first, then CRU overrides
- [ ] Pipeline validates folder structure before deployment
- [ ] Deployment logs show which CRU folders were processed
- [ ] Failed deployments rollback to previous CRU version

---

**End of Section 5: Repository Structure & Git Strategy**

## 6. Data Model & Database Schema

### 6.1 Entity Relationship Diagram

```
┌─────────────────────────┐
│   solution_release      │
│─────────────────────────│
│ id (PK)                 │
│ solution_key            │
│ release_tag (UNIQUE)    │◄─────┐
│ customer                │      │
│ cru_tag                 │      │
│ product_version         │      │
│ customer_version        │      │
│ status                  │      │
│ created_at              │      │
│ created_by              │      │
└─────────────────────────┘      │
            │                    │
            │ 1:N                │
            ▼                    │
┌─────────────────────────┐      │
│   component_release     │      │
│─────────────────────────│      │
│ id (PK)                 │      │
│ solution_release_id (FK)│──────┘
│ component_key           │
│ is_customized           │
│ code_repo_url           │
│ code_ref_type           │
│ code_ref_value          │
│ code_commit_sha         │
│ image_uri               │
│ deploy_repo_url         │
│ deploy_ref_type         │
│ deploy_ref_value        │
│ helm_chart_path         │
│ helm_values_path        │
│ helm_release_name       │
└─────────────────────────┘
            ▲
            │
            │ N:1
            │
┌─────────────────────────┐
│      env_deploy         │
│─────────────────────────│
│ id (PK)                 │
│ solution_release_id (FK)│
│ customer                │
│ environment             │
│ status                  │
│ deploy_repo_branch      │
│ release_yaml_commit     │
│ started_at              │
│ finished_at             │
│ deployed_by             │
│ notes                   │
└─────────────────────────┘
```


### 6.2 Table Definitions

#### Table 1: `solution_release`

**Purpose:** Parent record representing one complete solution release for a customer.

```sql
CREATE TABLE solution_release (
  id BIGSERIAL PRIMARY KEY,
  solution_key VARCHAR(100) NOT NULL,
  release_tag VARCHAR(50) UNIQUE NOT NULL,
  customer VARCHAR(100) NOT NULL,
  cru_tag VARCHAR(50) NOT NULL,
  product_version VARCHAR(20) NOT NULL,
  customer_version VARCHAR(20) NOT NULL,
  status VARCHAR(20) NOT NULL CHECK (status IN ('DRAFT', 'READY', 'LOCKED', 'ARCHIVED')),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(100),
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_by VARCHAR(100),
  notes TEXT,
  
  INDEX idx_customer (customer),
  INDEX idx_release_tag (release_tag),
  INDEX idx_status (status),
  INDEX idx_cru_tag (cru_tag)
);
```

**Field Descriptions:**

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `id` | BIGSERIAL | Primary key | `1` |
| `solution_key` | VARCHAR(100) | Logical solution name | `HumainOS-Core` |
| `release_tag` | VARCHAR(50) | Unique release identifier | `SL-ACME-006` |
| `customer` | VARCHAR(100) | Customer/tenant name | `ACME` |
| `cru_tag` | VARCHAR(50) | Configuration Release Unit tag | `CUST-ACME-R07` |
| `product_version` | VARCHAR(20) | Base Visionwaves version | `1.12.3` |
| `customer_version` | VARCHAR(20) | Customer-specific version | `2.3.0` |
| `status` | VARCHAR(20) | Release status | `DRAFT`, `READY`, `LOCKED`, `ARCHIVED` |
| `created_at` | TIMESTAMP | Creation timestamp | `2025-11-09 10:00:00` |
| `created_by` | VARCHAR(100) | User who created the release | `rm@visionwaves.com` |
| `notes` | TEXT | Release notes and comments | `Initial release for Q4 2025` |

#### Table 2: `component_release`

**Purpose:** Defines each microservice component within a solution release.

```sql
CREATE TABLE component_release (
  id BIGSERIAL PRIMARY KEY,
  solution_release_id BIGINT NOT NULL REFERENCES solution_release(id) ON DELETE CASCADE,
  component_key VARCHAR(100) NOT NULL,
  is_customized BOOLEAN NOT NULL DEFAULT FALSE,
  
  -- Source Code Information
  code_repo_url VARCHAR(500) NOT NULL,
  code_ref_type VARCHAR(20) NOT NULL CHECK (code_ref_type IN ('branch', 'tag', 'commit')),
  code_ref_value VARCHAR(200) NOT NULL,
  code_commit_sha VARCHAR(64),
  
  -- Docker Image Information
  image_uri VARCHAR(500) NOT NULL,
  image_digest VARCHAR(100),
  
  -- Deployment Information
  deploy_repo_url VARCHAR(500) NOT NULL,
  deploy_ref_type VARCHAR(20) NOT NULL CHECK (deploy_ref_type IN ('branch', 'tag')),
  deploy_ref_value VARCHAR(200) NOT NULL,
  helm_chart_path VARCHAR(500) NOT NULL,
  helm_values_path VARCHAR(500) NOT NULL,
  helm_release_name VARCHAR(100),
  
  -- Metadata
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(100),
  
  UNIQUE(solution_release_id, component_key),
  INDEX idx_component_key (component_key),
  INDEX idx_solution_release (solution_release_id),
  INDEX idx_is_customized (is_customized)
);
```

**Field Descriptions:**

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `component_key` | VARCHAR(100) | Unique component identifier | `lcm`, `identity`, `scheduler` |
| `is_customized` | BOOLEAN | Whether this is a customer-specific version | `true` for `customer-acme-lcm` |
| `code_repo_url` | VARCHAR(500) | Git repository URL | `https://github.com/visionwaves/customer-acme-lcm` |
| `code_ref_type` | VARCHAR(20) | Type of git reference | `branch`, `tag`, `commit` |
| `code_ref_value` | VARCHAR(200) | Git reference value | `lcm-acme-2.3.0` or `v1.9.1` |
| `code_commit_sha` | VARCHAR(64) | Full commit SHA for traceability | `a1b2c3d4e5f6...` |
| `image_uri` | VARCHAR(500) | Docker image URI | `registry.visionwaves.com/lcm:2.3.0-CUST-ACME-R07` |
| `image_digest` | VARCHAR(100) | Docker image SHA256 digest | `sha256:abc123...` |
| `helm_chart_path` | VARCHAR(500) | Path to Helm chart in deploy repo | `helm/charts/lcm` |
| `helm_values_path` | VARCHAR(500) | Path to values file | `helm/values/ACME/sit/lcm.values.yaml` |
| `helm_release_name` | VARCHAR(100) | Helm release name in K8s | `acme-lcm` |

#### Table 3: `env_deploy`

**Purpose:** Tracks deployment history and status for each environment.

```sql
CREATE TABLE env_deploy (
  id BIGSERIAL PRIMARY KEY,
  solution_release_id BIGINT NOT NULL REFERENCES solution_release(id),
  customer VARCHAR(100) NOT NULL,
  environment VARCHAR(20) NOT NULL CHECK (environment IN ('sit', 'preprod', 'prod')),
  status VARCHAR(20) NOT NULL CHECK (status IN ('IN_PROGRESS', 'SUCCESS', 'FAILED', 'ROLLED_BACK')),
  
  -- Deployment Metadata
  deploy_repo_branch VARCHAR(200) NOT NULL,
  release_yaml_commit VARCHAR(64),
  pipeline_job_id VARCHAR(100),
  pipeline_url VARCHAR(500),
  
  -- Timing
  started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  finished_at TIMESTAMP,
  duration_seconds INTEGER,
  
  -- User & Notes
  deployed_by VARCHAR(100),
  notes TEXT,
  error_message TEXT,
  
  INDEX idx_customer_env (customer, environment),
  INDEX idx_solution_release (solution_release_id),
  INDEX idx_status (status),
  INDEX idx_started_at (started_at DESC)
);
```

**Field Descriptions:**

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `environment` | VARCHAR(20) | Target environment | `sit`, `preprod`, `prod` |
| `status` | VARCHAR(20) | Deployment status | `IN_PROGRESS`, `SUCCESS`, `FAILED`, `ROLLED_BACK` |
| `deploy_repo_branch` | VARCHAR(200) | Deployments repo branch | `cust/ACME/sit` |
| `release_yaml_commit` | VARCHAR(64) | Commit SHA of release.yaml | `x1y2z3...` |
| `pipeline_job_id` | VARCHAR(100) | Jenkins/CI job ID | `deploy-acme-sit-#142` |
| `pipeline_url` | VARCHAR(500) | Link to CI/CD job | `https://jenkins.visionwaves.com/job/142` |
| `deployed_by` | VARCHAR(100) | User who triggered deployment | `devops@visionwaves.com` |
| `duration_seconds` | INTEGER | Deployment duration | `420` (7 minutes) |

### 6.3 Sample Data

#### Example: `solution_release` Record

```sql
INSERT INTO solution_release (
  solution_key, release_tag, customer, cru_tag, 
  product_version, customer_version, status, created_by, notes
) VALUES (
  'HumainOS-Core', 
  'SL-ACME-006', 
  'ACME', 
  'CUST-ACME-R07',
  '1.12.3', 
  '2.3.0', 
  'READY', 
  'release.manager@visionwaves.com',
  'Q4 2025 release with new workflow engine and enhanced reporting'
);
```

#### Example: `component_release` Records

```sql
-- LCM Component (Customized)
INSERT INTO component_release (
  solution_release_id, component_key, is_customized,
  code_repo_url, code_ref_type, code_ref_value, code_commit_sha,
  image_uri, image_digest,
  deploy_repo_url, deploy_ref_type, deploy_ref_value,
  helm_chart_path, helm_values_path, helm_release_name,
  created_by
) VALUES (
  1, 'lcm', true,
  'https://github.com/visionwaves/customer-acme-lcm', 'tag', 'lcm-acme-2.3.0', 'a1b2c3d4e5f6...',
  'registry.visionwaves.com/lcm:2.3.0-CUST-ACME-R07', 'sha256:abc123...',
  'https://github.com/visionwaves/deployments', 'branch', 'cust/ACME/sit',
  'helm/charts/lcm', 'helm/values/ACME/sit/lcm.values.yaml', 'acme-lcm',
  'lcm.owner@visionwaves.com'
);

-- Identity Component (Product - Not Customized)
INSERT INTO component_release (
  solution_release_id, component_key, is_customized,
  code_repo_url, code_ref_type, code_ref_value, code_commit_sha,
  image_uri, image_digest,
  deploy_repo_url, deploy_ref_type, deploy_ref_value,
  helm_chart_path, helm_values_path, helm_release_name,
  created_by
) VALUES (
  1, 'identity', false,
  'https://github.com/visionwaves/product-identity', 'tag', 'v1.9.1', 'e5f6g7h8i9j0...',
  'registry.visionwaves.com/identity:2.3.0-CUST-ACME-R07', 'sha256:def456...',
  'https://github.com/visionwaves/deployments', 'branch', 'cust/ACME/sit',
  'helm/charts/identity', 'helm/values/ACME/sit/identity.values.yaml', 'acme-identity',
  'identity.owner@visionwaves.com'
);

-- Scheduler Component (Product - Not Customized)
INSERT INTO component_release (
  solution_release_id, component_key, is_customized,
  code_repo_url, code_ref_type, code_ref_value, code_commit_sha,
  image_uri, image_digest,
  deploy_repo_url, deploy_ref_type, deploy_ref_value,
  helm_chart_path, helm_values_path, helm_release_name,
  created_by
) VALUES (
  1, 'scheduler', false,
  'https://github.com/visionwaves/product-scheduler', 'tag', 'v1.7.2', 'k1l2m3n4o5p6...',
  'registry.visionwaves.com/scheduler:2.3.0-CUST-ACME-R07', 'sha256:ghi789...',
  'https://github.com/visionwaves/deployments', 'branch', 'cust/ACME/sit',
  'helm/charts/scheduler', 'helm/values/ACME/sit/scheduler.values.yaml', 'acme-scheduler',
  'scheduler.owner@visionwaves.com'
);
```

#### Example: `env_deploy` Record

```sql
INSERT INTO env_deploy (
  solution_release_id, customer, environment, status,
  deploy_repo_branch, release_yaml_commit, pipeline_job_id, pipeline_url,
  deployed_by, duration_seconds, notes
) VALUES (
  1, 'ACME', 'sit', 'SUCCESS',
  'cust/ACME/sit', 'x1y2z3a4b5c6...', 'deploy-acme-sit-#142',
  'https://jenkins.visionwaves.com/job/deploy-acme-sit/142',
  'devops@visionwaves.com', 420,
  'Initial SIT deployment of SL-ACME-006. All components deployed successfully. Smoke tests passed.'
);
```

### 6.4 Useful SQL Queries

#### Query 1: Get All Components in a Solution Release

```sql
SELECT 
  sr.release_tag,
  sr.customer,
  sr.cru_tag,
  cr.component_key,
  cr.is_customized,
  cr.code_ref_value AS code_version,
  cr.image_uri
FROM solution_release sr
JOIN component_release cr ON sr.id = cr.solution_release_id
WHERE sr.release_tag = 'SL-ACME-006'
ORDER BY cr.component_key;
```

#### Query 2: Get Latest Deployment Per Environment

```sql
SELECT DISTINCT ON (customer, environment)
  customer,
  environment,
  sr.release_tag,
  ed.status,
  ed.started_at,
  ed.deployed_by
FROM env_deploy ed
JOIN solution_release sr ON ed.solution_release_id = sr.id
WHERE customer = 'ACME'
ORDER BY customer, environment, started_at DESC;
```

#### Query 3: Get Deployment History for a Customer

```sql
SELECT 
  sr.release_tag,
  ed.environment,
  ed.status,
  ed.started_at,
  ed.finished_at,
  ed.duration_seconds,
  ed.deployed_by
FROM env_deploy ed
JOIN solution_release sr ON ed.solution_release_id = sr.id
WHERE ed.customer = 'ACME'
ORDER BY ed.started_at DESC
LIMIT 20;
```

#### Query 4: Find All Customized Components Across Customers

```sql
SELECT 
  sr.customer,
  sr.release_tag,
  cr.component_key,
  cr.code_repo_url,
  cr.code_ref_value
FROM component_release cr
JOIN solution_release sr ON cr.solution_release_id = sr.id
WHERE cr.is_customized = true
ORDER BY sr.customer, cr.component_key;
```

<div style="page-break-after: always;"></div>

---

## 7. Release Workflow

### 7.1 Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    RELEASE CREATION PHASE                       │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │ 1. Release Manager Creates            │
        │    Solution Release (DRAFT)           │
        │    - Assigns release tag              │
        │    - Assigns CRU tag                  │
        │    - Sets product/customer versions   │
        └───────────────┬───────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────┐
        │ 2. Product Owners Add Components      │
        │    - Specify repos, branches/tags     │
        │    - Provide Docker image URIs        │
        │    - Define Helm chart paths          │
        │    - Set values file locations        │
        └───────────────┬───────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────┐
        │ 3. Release Manager Reviews            │
        │    - Validates all components         │
        │    - Checks CRU configuration         │
        │    - Sets status to READY             │
        └───────────────┬───────────────────────┘
                        │
┌─────────────────────────────────────────────────────────────────┐
│                    DEPLOYMENT PHASE                             │
└─────────────────────────────────────────────────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────┐
        │ 4. DevOps Updates Deployments Repo    │
        │    - Edit release.yaml in env branch  │
        │    - Update solutionRelease pointer   │
        │    - Create Pull Request              │
        └───────────────┬───────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────┐
        │ 5. PR Review & Approval               │
        │    - Peer review changes              │
        │    - Verify release tag               │
        │    - Merge to environment branch      │
        └───────────────┬───────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────┐
        │ 6. CI/CD Pipeline Triggered           │
        │    - Read release.yaml                │
        │    - Query solution_release table     │
        │    - Load component_release entries   │
        │    - Create env_deploy record         │
        └───────────────┬───────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────┐
        │ 7. Deploy Each Component              │
        │    - Checkout deploy repo             │
        │    - Run Helm install/upgrade         │
        │    - Apply CRU configurations         │
        │    - Verify pod health                │
        └───────────────┬───────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────┐
        │ 8. Post-Deployment                    │
        │    - Run smoke tests                  │
        │    - Update env_deploy status         │
        │    - Send notifications               │
        │    - Generate deployment report       │
        └───────────────────────────────────────┘
```

### 7.2 State Transitions

**Solution Release Status:**

```
DRAFT ──────────────► READY ──────────────► LOCKED ──────────────► ARCHIVED
  │                     │                      │                       │
  │ RM creates          │ RM approves          │ Deployed to prod      │ Deprecated
  │ Components added    │ All components set   │ No modifications      │ No longer used
  │ Can be modified     │ Ready for deploy     │ Read-only             │ Historical
  └─────────────────────┴──────────────────────┴───────────────────────┘
```

**Deployment Status:**

```
IN_PROGRESS ──────────► SUCCESS
     │                     │
     │                     └──► (Deployment complete, all tests passed)
     │
     ├──────────────────► FAILED
     │                     │
     │                     └──► (Deployment error, rollback may be needed)
     │
     └──────────────────► ROLLED_BACK
                           │
                           └──► (Rollback executed successfully)
```


### 7.3 Detailed Step-by-Step Process

#### Step 1: Create Solution Release (Release Manager)

**Action:** Create a new solution release record in `DRAFT` status.

**Prerequisites:**
- [ ] Product version finalized
- [ ] Customer version determined
- [ ] CRU tag created or identified
- [ ] Release notes prepared

**SQL:**
```sql
INSERT INTO solution_release (
  solution_key, release_tag, customer, cru_tag,
  product_version, customer_version, status, created_by, notes
) VALUES (
  'HumainOS-Core', 'SL-ACME-006', 'ACME', 'CUST-ACME-R07',
  '1.12.3', '2.3.0', 'DRAFT', 'rm@visionwaves.com',
  'Q4 2025 release with new workflow engine'
);
```

**Checklist:**
- [ ] Assign unique release tag following naming convention (`SL-<CUSTOMER>-<NNN>`)
- [ ] Create new CRU tag or reuse existing (`CUST-<TENANT>-R<NN>`)
- [ ] Set product version (base Visionwaves version)
- [ ] Set customer version (includes customizations)
- [ ] Add comprehensive release notes
- [ ] Notify product owners to add components

#### Step 2: Add Components (Product Owners)

**Action:** Each product owner adds their component to the release.

**Prerequisites:**
- [ ] Code is tagged/branched in source repo
- [ ] Docker image is built and pushed to registry
- [ ] Helm chart exists in deployments repo
- [ ] Values file is created for target environment
- [ ] Component tested independently

**SQL Example (LCM Product Owner):**
```sql
INSERT INTO component_release (
  solution_release_id, component_key, is_customized,
  code_repo_url, code_ref_type, code_ref_value, code_commit_sha,
  image_uri, image_digest,
  deploy_repo_url, deploy_ref_type, deploy_ref_value,
  helm_chart_path, helm_values_path, helm_release_name,
  created_by
) VALUES (
  (SELECT id FROM solution_release WHERE release_tag = 'SL-ACME-006'),
  'lcm', true,
  'https://github.com/visionwaves/customer-acme-lcm', 'tag', 'lcm-acme-2.3.0', 'a1b2c3d4...',
  'registry.visionwaves.com/lcm:2.3.0-CUST-ACME-R07', 'sha256:abc123...',
  'https://github.com/visionwaves/deployments', 'branch', 'cust/ACME/sit',
  'helm/charts/lcm', 'helm/values/ACME/sit/lcm.values.yaml', 'acme-lcm',
  'lcm.owner@visionwaves.com'
);
```

**Checklist:**
- [ ] Component code is tagged with correct version
- [ ] Docker image includes CRU tag in version
- [ ] Image digest (SHA256) is recorded
- [ ] Helm chart path is correct
- [ ] Values file path is environment-specific
- [ ] Helm release name follows convention
- [ ] Component dependencies are documented

#### Step 3: Review and Approve (Release Manager)

**Action:** Validate all components and mark release as READY.

**Prerequisites:**
- [ ] All components added by product owners
- [ ] All Docker images available in registry
- [ ] All Helm charts and values files exist
- [ ] CRU configurations are complete
- [ ] Integration testing completed

**SQL:**
```sql
UPDATE solution_release
SET status = 'READY',
    updated_at = CURRENT_TIMESTAMP,
    updated_by = 'rm@visionwaves.com'
WHERE release_tag = 'SL-ACME-006';
```

**Validation Checklist:**
- [ ] All expected components are present
- [ ] No duplicate component keys
- [ ] All image URIs are accessible
- [ ] All Helm charts exist in deployments repo
- [ ] CRU tag is consistent across all components
- [ ] Release notes are complete
- [ ] Stakeholders notified

#### Step 4: Update Deployments Repo (DevOps)

**Action:** Update release.yaml in target environment branch.

**Prerequisites:**
- [ ] Solution release status is READY
- [ ] Target environment is available
- [ ] Previous deployment is stable (if any)

**Steps:**
1. Clone deployments repo
2. Checkout target environment branch
3. Edit release.yaml
4. Create Pull Request
5. Get approval
6. Merge to trigger deployment

**Example:**
```bash
# Clone and checkout
git clone https://github.com/visionwaves/deployments.git
cd deployments
git checkout cust/ACME/sit

# Edit release.yaml
cat > cust/ACME/sit/release.yaml << 'YAML'
solutionRelease: "SL-ACME-006"
environment: "sit"
YAML

# Commit and push
git add cust/ACME/sit/release.yaml
git commit -m "Deploy SL-ACME-006 to ACME SIT environment"
git push origin cust/ACME/sit

# Create PR (via GitHub/GitLab UI or CLI)
```

**Checklist:**
- [ ] Correct environment branch selected
- [ ] Release tag is valid and exists in database
- [ ] PR description includes release notes
- [ ] PR is reviewed by peer
- [ ] Merge triggers CI/CD pipeline

#### Step 5: CI/CD Deployment (Automated)

**Action:** Pipeline deploys all components and applies CRU configurations.

**Pipeline Steps:**
1. Read release.yaml from merged branch
2. Query database for solution release details
3. Create env_deploy record with status IN_PROGRESS
4. For each component:
   - Checkout deployment repo
   - Run Helm install/upgrade
   - Verify pod health
5. Apply CRU configurations
6. Run smoke tests
7. Update env_deploy status to SUCCESS or FAILED

**See Section 10 for detailed CI/CD pipeline implementation.**

#### Step 6: Post-Deployment Verification

**Action:** Verify deployment success and update records.

**Verification Steps:**
- [ ] All pods are running and healthy
- [ ] All services are accessible
- [ ] Smoke tests passed
- [ ] CRU configurations applied correctly
- [ ] No errors in application logs
- [ ] Monitoring dashboards show normal metrics

**Notification:**
- Send deployment success/failure notification
- Update deployment tracking dashboard
- Generate deployment report
- Notify stakeholders

<div style="page-break-after: always;"></div>

---

## 8. Roles & Responsibilities

### 8.1 Role Matrix

| Role | Responsibilities | Access Level | Key Activities |
|------|------------------|--------------|----------------|
| **Release Manager** | Owns the release process end-to-end | Full access to solution_release table | Create releases, assign CRU, approve releases, coordinate stakeholders |
| **Product Owner** | Owns specific product components | Write access to component_release for their products | Add component metadata, ensure code is tagged, verify images |
| **DevOps Engineer** | Executes deployments and manages infrastructure | Full access to deployments repo and CI/CD | Update release.yaml, trigger deployments, monitor pipelines, handle rollbacks |
| **Developer** | Develops features and fixes | Read access to release data | Tag code with CRU, build images, update configurations |
| **QA Engineer** | Tests releases before deployment | Read access to release data | Verify component versions, run integration tests, validate CRU configs |
| **Customer Success** | Communicates with customers | Read access to deployment history | Inform customers of releases, coordinate deployment windows |

### 8.2 Detailed Responsibilities

#### Release Manager

**Primary Responsibilities:**
1. Create solution release records in database
2. Assign CRU tags for configuration management
3. Coordinate with product owners to add components
4. Review and validate complete releases
5. Change release status from DRAFT → READY
6. Lock releases after production deployment
7. Archive deprecated releases
8. Maintain release documentation

**Tools & Access:**
- Database access (solution_release, component_release tables)
- Release management dashboard
- Communication tools (Slack, Email)

**Success Metrics:**
- On-time release delivery
- Zero missing components in releases
- Clear and complete release notes

#### Product Owner

**Primary Responsibilities:**
1. Add component_release entries for their products
2. Ensure code is properly tagged in git
3. Verify Docker images are built and pushed
4. Confirm Helm charts and values files exist
5. Document component dependencies
6. Coordinate with developers on component readiness
7. Participate in release reviews

**Tools & Access:**
- Database access (component_release table - insert only)
- Git repositories (product and customer repos)
- Docker registry (read access)
- Deployments repo (read access)

**Success Metrics:**
- Components added on time
- Zero deployment failures due to missing artifacts
- Accurate component metadata

#### DevOps Engineer

**Primary Responsibilities:**
1. Update release.yaml in deployments repo
2. Create and review Pull Requests for deployments
3. Monitor CI/CD pipeline execution
4. Troubleshoot deployment failures
5. Execute rollbacks when necessary
6. Maintain deployment infrastructure
7. Update deployment documentation
8. Manage environment-specific configurations

**Tools & Access:**
- Full access to deployments repo
- CI/CD pipeline (Jenkins/GitLab CI)
- Kubernetes clusters (all environments)
- Monitoring and logging tools
- Database access (env_deploy table)

**Success Metrics:**
- Deployment success rate > 95%
- Mean time to deploy < 30 minutes
- Mean time to recovery < 15 minutes

#### Developer

**Primary Responsibilities:**
1. Develop features and bug fixes
2. Tag code with appropriate CRU tags
3. Ensure configurations are CRU-tagged
4. Build and test Docker images locally
5. Update Helm values files as needed
6. Participate in code reviews
7. Fix deployment issues related to their code

**Tools & Access:**
- Git repositories (read/write for assigned repos)
- Docker registry (push access)
- Development environment
- Read access to release data

**Success Metrics:**
- Code quality and test coverage
- Timely delivery of features
- Zero critical bugs in production

<div style="page-break-after: always;"></div>

---

## 9. Standard Operating Procedures (SOP)

### 9.1 SOP Summary Table

| Step | Who | Action | Duration | Dependencies |
|------|-----|--------|----------|--------------|
| 1 | Release Manager | Create solution release (DRAFT) | 15 min | Product/customer versions finalized |
| 2 | Release Manager | Assign CRU tag | 5 min | CRU naming convention |
| 3 | Product Owners | Add component entries | 30 min | Code tagged, images built |
| 4 | Release Manager | Review and set to READY | 30 min | All components added |
| 5 | DevOps | Update release.yaml (PR) | 15 min | Release is READY |
| 6 | DevOps | Merge PR to env branch | 5 min | PR approved |
| 7 | CI/CD | Deploy components | 20-30 min | Automated |
| 8 | CI/CD | Apply CRU configurations | 5-10 min | Automated |
| 9 | CI/CD | Run smoke tests | 10 min | Automated |
| 10 | DevOps | Verify and close | 15 min | Deployment SUCCESS |

**Total Time:** ~2-3 hours (from creation to deployment)

### 9.2 SOP: Create New Solution Release

**Objective:** Create a new solution release for a customer.

**Frequency:** As needed (typically 1-2 times per month per customer)

**Prerequisites:**
- Product version is finalized and tested
- Customer-specific customizations are complete
- CRU tag is created or identified

**Procedure:**

1. **Determine Release Details**
   - Identify solution key (e.g., `HumainOS-Core`)
   - Determine customer (e.g., `ACME`)
   - Assign release tag (e.g., `SL-ACME-006`)
   - Identify or create CRU tag (e.g., `CUST-ACME-R07`)
   - Set product version (e.g., `1.12.3`)
   - Set customer version (e.g., `2.3.0`)

2. **Create Database Record**
   ```sql
   INSERT INTO solution_release (
     solution_key, release_tag, customer, cru_tag,
     product_version, customer_version, status, created_by, notes
   ) VALUES (
     'HumainOS-Core', 'SL-ACME-006', 'ACME', 'CUST-ACME-R07',
     '1.12.3', '2.3.0', 'DRAFT', 'rm@visionwaves.com',
     'Q4 2025 release with new features...'
   );
   ```

3. **Notify Product Owners**
   - Send email/Slack message to all product owners
   - Include release tag and CRU tag
   - Set deadline for component additions
   - Provide link to release documentation

4. **Track Progress**
   - Monitor component additions
   - Follow up with product owners
   - Verify all components are added

5. **Review and Approve**
   - Validate all components
   - Check for completeness
   - Update status to READY

**Expected Outcome:** Solution release created and ready for deployment.

**Rollback:** Delete release record if needed (only if status is DRAFT).

### 9.3 SOP: Add Component to Release

**Objective:** Add a microservice component to an existing solution release.

**Frequency:** Once per component per release

**Prerequisites:**
- Solution release exists in DRAFT status
- Code is tagged in git repository
- Docker image is built and pushed
- Helm chart and values files exist

**Procedure:**

1. **Gather Component Information**
   - Component key (e.g., `lcm`)
   - Is it customized? (true/false)
   - Code repo URL
   - Code ref type and value (tag/branch)
   - Commit SHA
   - Docker image URI and digest
   - Helm chart path
   - Helm values file path
   - Helm release name

2. **Verify Artifacts**
   ```bash
   # Verify git tag exists
   git ls-remote --tags <repo-url> | grep <tag-name>
   
   # Verify Docker image exists
   docker pull <image-uri>
   
   # Verify Helm chart exists
   helm show chart <chart-path>
   ```

3. **Insert Component Record**
   ```sql
   INSERT INTO component_release (
     solution_release_id, component_key, is_customized,
     code_repo_url, code_ref_type, code_ref_value, code_commit_sha,
     image_uri, image_digest,
     deploy_repo_url, deploy_ref_type, deploy_ref_value,
     helm_chart_path, helm_values_path, helm_release_name,
     created_by
   ) VALUES (
     (SELECT id FROM solution_release WHERE release_tag = 'SL-ACME-006'),
     'lcm', true,
     'https://github.com/visionwaves/customer-acme-lcm', 'tag', 'lcm-acme-2.3.0', 'a1b2c3d4...',
     'registry.visionwaves.com/lcm:2.3.0-CUST-ACME-R07', 'sha256:abc123...',
     'https://github.com/visionwaves/deployments', 'branch', 'cust/ACME/sit',
     'helm/charts/lcm', 'helm/values/ACME/sit/lcm.values.yaml', 'acme-lcm',
     'lcm.owner@visionwaves.com'
   );
   ```

4. **Verify Insertion**
   ```sql
   SELECT * FROM component_release 
   WHERE solution_release_id = (SELECT id FROM solution_release WHERE release_tag = 'SL-ACME-006')
   AND component_key = 'lcm';
   ```

5. **Notify Release Manager**
   - Confirm component added
   - Provide any special notes or dependencies

**Expected Outcome:** Component successfully added to release.

**Rollback:** Delete component record if incorrect.

### 9.4 SOP: Deploy Solution Release

**Objective:** Deploy a solution release to a target environment.

**Frequency:** As needed (SIT: daily, PreProd: weekly, Prod: monthly)

**Prerequisites:**
- Solution release status is READY
- Target environment is available
- Deployment window is scheduled
- Stakeholders are notified

**Procedure:**

1. **Prepare Deployment**
   ```bash
   # Clone deployments repo
   git clone https://github.com/visionwaves/deployments.git
   cd deployments
   
   # Checkout target environment branch
   git checkout cust/ACME/sit
   git pull origin cust/ACME/sit
   ```

2. **Update Release Pointer**
   ```bash
   # Edit release.yaml
   cat > cust/ACME/sit/release.yaml << 'YAML'
   solutionRelease: "SL-ACME-006"
   environment: "sit"
   YAML
   
   # Commit changes
   git add cust/ACME/sit/release.yaml
   git commit -m "Deploy SL-ACME-006 to ACME SIT
   
   Release Notes:
   - New workflow engine
   - Enhanced reporting
   - Bug fixes
   
   Components:
   - lcm: 2.3.0-CUST-ACME-R07
   - identity: 2.3.0-CUST-ACME-R07
   - scheduler: 2.3.0-CUST-ACME-R07
   "
   ```

3. **Create Pull Request**
   ```bash
   # Push to remote
   git push origin cust/ACME/sit
   
   # Create PR (via GitHub/GitLab UI or CLI)
   # Title: "Deploy SL-ACME-006 to ACME SIT"
   # Description: Include release notes and component list
   ```

4. **Review and Merge**
   - Get peer review
   - Verify release tag is correct
   - Merge PR to trigger CI/CD pipeline

5. **Monitor Deployment**
   - Watch CI/CD pipeline logs
   - Monitor Kubernetes pod status
   - Check application logs
   - Verify smoke tests pass

6. **Verify Deployment**
   ```bash
   # Check pod status
   kubectl get pods -n acme-sit
   
   # Check deployment status in database
   SELECT * FROM env_deploy 
   WHERE customer = 'ACME' AND environment = 'sit'
   ORDER BY started_at DESC LIMIT 1;
   ```

7. **Post-Deployment**
   - Run additional tests if needed
   - Notify stakeholders of success
   - Update deployment tracking dashboard
   - Document any issues or notes

**Expected Outcome:** Solution release successfully deployed to target environment.

**Rollback:** See Section 15 for rollback procedures.


<div style="page-break-after: always;"></div>

---

## 10. CI/CD Pipeline Integration

### 10.1 Pipeline Overview

The CI/CD pipeline is triggered automatically when a Pull Request is merged to an environment branch in the deployments repository.

**Trigger:** Merge to `cust/<CUSTOMER>/<ENVIRONMENT>` branch

**Pipeline Stages:**
1. **Read Configuration** - Parse release.yaml
2. **Validate Release** - Query database and verify release exists
3. **Initialize Deployment** - Create env_deploy record
4. **Deploy Components** - Deploy each microservice via Helm
5. **Apply CRU Configurations** - Apply configuration data
6. **Run Tests** - Execute smoke tests
7. **Finalize** - Update deployment status and notify

### 10.2 Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GIT WEBHOOK TRIGGER                      │
│         (Merge to cust/ACME/sit branch)                     │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  STAGE 1: Read Configuration                                │
│  - Parse release.yaml                                       │
│  - Extract solutionRelease tag                              │
│  - Extract environment                                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  STAGE 2: Validate Release                                  │
│  - Query solution_release table                             │
│  - Verify status is READY or LOCKED                         │
│  - Load all component_release entries                       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  STAGE 3: Initialize Deployment                             │
│  - Create env_deploy record (status: IN_PROGRESS)           │
│  - Record pipeline job ID and URL                           │
│  - Set started_at timestamp                                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  STAGE 4: Deploy Components (Loop)                          │
│  For each component_release:                                │
│    - Checkout deploy_repo_url@deploy_ref_value              │
│    - Load Helm chart from helm_chart_path                   │
│    - Load values from helm_values_path                      │
│    - Run: helm upgrade --install <release-name> \           │
│           --set image.repository=<image-uri> \              │
│           --values <values-file>                            │
│    - Wait for pods to be ready                              │
│    - Verify health checks                                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  STAGE 5: Apply CRU Configurations                          │
│  - Query configuration data by CRU tag                      │
│  - Apply UI configurations                                  │
│  - Apply workflow definitions                               │
│  - Apply grid configurations                                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  STAGE 6: Run Smoke Tests                                   │
│  - Health check endpoints                                   │
│  - Basic functionality tests                                │
│  - Configuration validation                                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  STAGE 7: Finalize Deployment                               │
│  - Update env_deploy.status (SUCCESS/FAILED)                │
│  - Set finished_at timestamp                                │
│  - Calculate duration_seconds                               │
│  - Send notifications (Slack, Email)                        │
│  - Generate deployment report                               │
└─────────────────────────────────────────────────────────────┘
```

### 10.3 Jenkins Pipeline Example (Jenkinsfile)

```groovy
pipeline {
    agent any
    
    environment {
        DEPLOYMENTS_REPO = 'https://github.com/visionwaves/deployments.git'
        DB_HOST = credentials('db-host')
        DB_USER = credentials('db-user')
        DB_PASS = credentials('db-password')
        KUBECONFIG = credentials('kubeconfig-sit')
    }
    
    stages {
        stage('Read Configuration') {
            steps {
                script {
                    // Parse release.yaml
                    def releaseConfig = readYaml file: "${env.BRANCH_NAME}/release.yaml"
                    env.SOLUTION_RELEASE = releaseConfig.solutionRelease
                    env.ENVIRONMENT = releaseConfig.environment
                    env.CUSTOMER = env.BRANCH_NAME.split('/')[1]
                    
                    echo "Deploying ${env.SOLUTION_RELEASE} to ${env.CUSTOMER} ${env.ENVIRONMENT}"
                }
            }
        }
        
        stage('Validate Release') {
            steps {
                script {
                    // Query database for release details
                    def query = """
                        SELECT sr.id, sr.status, sr.cru_tag, sr.customer
                        FROM solution_release sr
                        WHERE sr.release_tag = '${env.SOLUTION_RELEASE}'
                    """
                    
                    def release = sh(
                        script: "psql -h ${DB_HOST} -U ${DB_USER} -d releases -t -c \"${query}\"",
                        returnStdout: true
                    ).trim()
                    
                    if (!release) {
                        error("Release ${env.SOLUTION_RELEASE} not found in database")
                    }
                    
                    def (releaseId, status, cruTag, customer) = release.split('\\|')
                    
                    if (status != 'READY' && status != 'LOCKED') {
                        error("Release status is ${status}, must be READY or LOCKED")
                    }
                    
                    env.RELEASE_ID = releaseId.trim()
                    env.CRU_TAG = cruTag.trim()
                    
                    echo "Release validated: ID=${env.RELEASE_ID}, CRU=${env.CRU_TAG}"
                }
            }
        }
        
        stage('Initialize Deployment') {
            steps {
                script {
                    // Create env_deploy record
                    def insertQuery = """
                        INSERT INTO env_deploy (
                            solution_release_id, customer, environment, status,
                            deploy_repo_branch, release_yaml_commit, pipeline_job_id, pipeline_url,
                            deployed_by
                        ) VALUES (
                            ${env.RELEASE_ID}, '${env.CUSTOMER}', '${env.ENVIRONMENT}', 'IN_PROGRESS',
                            '${env.BRANCH_NAME}', '${env.GIT_COMMIT}', '${env.BUILD_TAG}', '${env.BUILD_URL}',
                            '${env.BUILD_USER}'
                        ) RETURNING id
                    """
                    
                    env.DEPLOY_ID = sh(
                        script: "psql -h ${DB_HOST} -U ${DB_USER} -d releases -t -c \"${insertQuery}\"",
                        returnStdout: true
                    ).trim()
                    
                    echo "Deployment initialized: ID=${env.DEPLOY_ID}"
                }
            }
        }
        
        stage('Deploy Components') {
            steps {
                script {
                    // Get all components for this release
                    def componentsQuery = """
                        SELECT component_key, image_uri, helm_chart_path, helm_values_path, helm_release_name
                        FROM component_release
                        WHERE solution_release_id = ${env.RELEASE_ID}
                        ORDER BY component_key
                    """
                    
                    def components = sh(
                        script: "psql -h ${DB_HOST} -U ${DB_USER} -d releases -t -c \"${componentsQuery}\"",
                        returnStdout: true
                    ).trim().split('\n')
                    
                    for (component in components) {
                        def (key, imageUri, chartPath, valuesPath, releaseName) = component.split('\\|')
                        
                        echo "Deploying component: ${key.trim()}"
                        
                        // Deploy with Helm
                        sh """
                            helm upgrade --install ${releaseName.trim()} \
                                ${chartPath.trim()} \
                                --set image.repository=${imageUri.trim().split(':')[0]} \
                                --set image.tag=${imageUri.trim().split(':')[1]} \
                                --values ${valuesPath.trim()} \
                                --namespace ${env.CUSTOMER.toLowerCase()}-${env.ENVIRONMENT} \
                                --create-namespace \
                                --wait \
                                --timeout 10m
                        """
                        
                        echo "Component ${key.trim()} deployed successfully"
                    }
                }
            }
        }
        
        stage('Apply CRU Configurations') {
            steps {
                script {
                    echo "Applying CRU configurations: ${env.CRU_TAG}"
                    
                    // Call configuration service API to apply CRU
                    sh """
                        curl -X POST http://config-service/api/cru/apply \
                            -H "Content-Type: application/json" \
                            -d '{"cruTag": "${env.CRU_TAG}", "environment": "${env.ENVIRONMENT}"}'
                    """
                    
                    echo "CRU configurations applied successfully"
                }
            }
        }
        
        stage('Run Smoke Tests') {
            steps {
                script {
                    echo "Running smoke tests..."
                    
                    // Run smoke tests
                    sh """
                        ./scripts/smoke-tests.sh \
                            --customer ${env.CUSTOMER} \
                            --environment ${env.ENVIRONMENT} \
                            --release ${env.SOLUTION_RELEASE}
                    """
                    
                    echo "Smoke tests passed"
                }
            }
        }
    }
    
    post {
        success {
            script {
                // Update deployment status to SUCCESS
                def updateQuery = """
                    UPDATE env_deploy
                    SET status = 'SUCCESS',
                        finished_at = CURRENT_TIMESTAMP,
                        duration_seconds = EXTRACT(EPOCH FROM (CURRENT_TIMESTAMP - started_at))
                    WHERE id = ${env.DEPLOY_ID}
                """
                
                sh "psql -h ${DB_HOST} -U ${DB_USER} -d releases -c \"${updateQuery}\""
                
                // Send success notification
                slackSend(
                    color: 'good',
                    message: "✅ Deployment SUCCESS: ${env.SOLUTION_RELEASE} to ${env.CUSTOMER} ${env.ENVIRONMENT}\nPipeline: ${env.BUILD_URL}"
                )
                
                echo "Deployment completed successfully"
            }
        }
        
        failure {
            script {
                // Update deployment status to FAILED
                def updateQuery = """
                    UPDATE env_deploy
                    SET status = 'FAILED',
                        finished_at = CURRENT_TIMESTAMP,
                        duration_seconds = EXTRACT(EPOCH FROM (CURRENT_TIMESTAMP - started_at)),
                        error_message = 'Pipeline failed at stage: ${env.STAGE_NAME}'
                    WHERE id = ${env.DEPLOY_ID}
                """
                
                sh "psql -h ${DB_HOST} -U ${DB_USER} -d releases -c \"${updateQuery}\""
                
                // Send failure notification
                slackSend(
                    color: 'danger',
                    message: "❌ Deployment FAILED: ${env.SOLUTION_RELEASE} to ${env.CUSTOMER} ${env.ENVIRONMENT}\nStage: ${env.STAGE_NAME}\nPipeline: ${env.BUILD_URL}"
                )
                
                echo "Deployment failed"
            }
        }
    }
}
```

### 10.4 GitLab CI Example (.gitlab-ci.yml)

```yaml
variables:
  DB_HOST: $DB_HOST
  DB_USER: $DB_USER
  DB_PASS: $DB_PASSWORD

stages:
  - validate
  - deploy
  - test
  - finalize

read_configuration:
  stage: validate
  script:
    - export SOLUTION_RELEASE=$(yq eval '.solutionRelease' $CI_COMMIT_REF_NAME/release.yaml)
    - export ENVIRONMENT=$(yq eval '.environment' $CI_COMMIT_REF_NAME/release.yaml)
    - export CUSTOMER=$(echo $CI_COMMIT_REF_NAME | cut -d'/' -f2)
    - echo "SOLUTION_RELEASE=$SOLUTION_RELEASE" >> deploy.env
    - echo "ENVIRONMENT=$ENVIRONMENT" >> deploy.env
    - echo "CUSTOMER=$CUSTOMER" >> deploy.env
  artifacts:
    reports:
      dotenv: deploy.env

validate_release:
  stage: validate
  dependencies:
    - read_configuration
  script:
    - |
      RELEASE_DATA=$(psql -h $DB_HOST -U $DB_USER -d releases -t -c "
        SELECT id, status, cru_tag FROM solution_release WHERE release_tag = '$SOLUTION_RELEASE'
      ")
      
      if [ -z "$RELEASE_DATA" ]; then
        echo "Release $SOLUTION_RELEASE not found"
        exit 1
      fi
      
      RELEASE_ID=$(echo $RELEASE_DATA | cut -d'|' -f1 | xargs)
      STATUS=$(echo $RELEASE_DATA | cut -d'|' -f2 | xargs)
      CRU_TAG=$(echo $RELEASE_DATA | cut -d'|' -f3 | xargs)
      
      if [ "$STATUS" != "READY" ] && [ "$STATUS" != "LOCKED" ]; then
        echo "Release status is $STATUS, must be READY or LOCKED"
        exit 1
      fi
      
      echo "RELEASE_ID=$RELEASE_ID" >> deploy.env
      echo "CRU_TAG=$CRU_TAG" >> deploy.env
  artifacts:
    reports:
      dotenv: deploy.env

initialize_deployment:
  stage: deploy
  dependencies:
    - validate_release
  script:
    - |
      DEPLOY_ID=$(psql -h $DB_HOST -U $DB_USER -d releases -t -c "
        INSERT INTO env_deploy (
          solution_release_id, customer, environment, status,
          deploy_repo_branch, release_yaml_commit, pipeline_job_id, pipeline_url,
          deployed_by
        ) VALUES (
          $RELEASE_ID, '$CUSTOMER', '$ENVIRONMENT', 'IN_PROGRESS',
          '$CI_COMMIT_REF_NAME', '$CI_COMMIT_SHA', '$CI_PIPELINE_ID', '$CI_PIPELINE_URL',
          '$GITLAB_USER_EMAIL'
        ) RETURNING id
      " | xargs)
      
      echo "DEPLOY_ID=$DEPLOY_ID" >> deploy.env
  artifacts:
    reports:
      dotenv: deploy.env

deploy_components:
  stage: deploy
  dependencies:
    - initialize_deployment
  script:
    - |
      psql -h $DB_HOST -U $DB_USER -d releases -t -c "
        SELECT component_key, image_uri, helm_chart_path, helm_values_path, helm_release_name
        FROM component_release
        WHERE solution_release_id = $RELEASE_ID
        ORDER BY component_key
      " | while IFS='|' read -r key imageUri chartPath valuesPath releaseName; do
        key=$(echo $key | xargs)
        imageUri=$(echo $imageUri | xargs)
        chartPath=$(echo $chartPath | xargs)
        valuesPath=$(echo $valuesPath | xargs)
        releaseName=$(echo $releaseName | xargs)
        
        echo "Deploying component: $key"
        
        helm upgrade --install $releaseName $chartPath \
          --set image.repository=$(echo $imageUri | cut -d':' -f1) \
          --set image.tag=$(echo $imageUri | cut -d':' -f2) \
          --values $valuesPath \
          --namespace ${CUSTOMER,,}-$ENVIRONMENT \
          --create-namespace \
          --wait \
          --timeout 10m
        
        echo "Component $key deployed successfully"
      done

apply_cru:
  stage: deploy
  dependencies:
    - deploy_components
  script:
    - |
      curl -X POST http://config-service/api/cru/apply \
        -H "Content-Type: application/json" \
        -d "{\"cruTag\": \"$CRU_TAG\", \"environment\": \"$ENVIRONMENT\"}"

smoke_tests:
  stage: test
  dependencies:
    - apply_cru
  script:
    - ./scripts/smoke-tests.sh --customer $CUSTOMER --environment $ENVIRONMENT --release $SOLUTION_RELEASE

finalize_success:
  stage: finalize
  dependencies:
    - smoke_tests
  when: on_success
  script:
    - |
      psql -h $DB_HOST -U $DB_USER -d releases -c "
        UPDATE env_deploy
        SET status = 'SUCCESS',
            finished_at = CURRENT_TIMESTAMP,
            duration_seconds = EXTRACT(EPOCH FROM (CURRENT_TIMESTAMP - started_at))
        WHERE id = $DEPLOY_ID
      "
    - echo "✅ Deployment SUCCESS"

finalize_failure:
  stage: finalize
  dependencies:
    - smoke_tests
  when: on_failure
  script:
    - |
      psql -h $DB_HOST -U $DB_USER -d releases -c "
        UPDATE env_deploy
        SET status = 'FAILED',
            finished_at = CURRENT_TIMESTAMP,
            duration_seconds = EXTRACT(EPOCH FROM (CURRENT_TIMESTAMP - started_at)),
            error_message = 'Pipeline failed'
        WHERE id = $DEPLOY_ID
      "
    - echo "❌ Deployment FAILED"
```

### 10.5 Pipeline Best Practices

| Best Practice | Description | Benefit |
|---------------|-------------|---------|
| **Idempotent Deployments** | Use `helm upgrade --install` to make deployments repeatable | Can safely retry failed deployments |
| **Atomic Operations** | Deploy all components or none (rollback on failure) | Prevents partial deployments |
| **Health Checks** | Wait for pods to be ready before proceeding | Ensures components are actually running |
| **Timeout Limits** | Set reasonable timeouts for each stage | Prevents hanging pipelines |
| **Detailed Logging** | Log every step with timestamps | Easier troubleshooting |
| **Database Tracking** | Record deployment status in database | Audit trail and monitoring |
| **Notifications** | Send alerts on success/failure | Immediate awareness of issues |
| **Rollback Capability** | Keep previous release info for quick rollback | Fast recovery from failures |

<div style="page-break-after: always;"></div>

---

## 11. Configuration Release Unit (CRU) Management

### 11.1 CRU Concept

A **Configuration Release Unit (CRU)** is a single tag that groups all UI, workflow, and grid configurations for a solution release.

**Key Principles:**
- ✅ **ONE CRU per solution release** - All components share the same CRU tag
- ✅ **Immutable** - Once created, CRU configurations should not change
- ✅ **Versioned** - CRU tags follow a sequential numbering scheme
- ✅ **Customer-specific** - Each customer has their own CRU sequence

### 11.2 CRU Naming Convention

**Format:** `CUST-<TENANT>-R<NN>`

**Examples:**
- `CUST-ACME-R07` - ACME customer, release 7
- `CUST-BETA-R12` - BETA customer, release 12
- `CUST-GAMMA-R03` - GAMMA customer, release 3

**Components:**
- `CUST` - Prefix indicating customer-specific configuration
- `<TENANT>` - Customer/tenant identifier (uppercase)
- `R` - Release indicator
- `<NN>` - Sequential release number (zero-padded to 2 digits)

### 11.3 CRU Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│  1. CREATE CRU                                              │
│     - Assign new CRU tag                                    │
│     - Initialize configuration records                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  2. POPULATE CRU                                            │
│     - Add UI configurations                                 │
│     - Add workflow definitions                              │
│     - Add grid configurations                               │
│     - Tag all with CRU                                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  3. VALIDATE CRU                                            │
│     - Verify all configurations tagged                      │
│     - Test configurations in SIT                            │
│     - Validate completeness                                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  4. DEPLOY CRU                                              │
│     - Apply configurations to environment                   │
│     - Verify application                                    │
│     - Test functionality                                    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  5. LOCK CRU                                                │
│     - Mark as deployed to production                        │
│     - Prevent modifications                                 │
│     - Archive for historical reference                      │
└─────────────────────────────────────────────────────────────┘
```

### 11.4 CRU Configuration Storage

All configuration data should be tagged with the CRU tag for traceability.

**Example Configuration Table:**

```sql
CREATE TABLE ui_configuration (
  id BIGSERIAL PRIMARY KEY,
  cru_tag VARCHAR(50) NOT NULL,
  customer VARCHAR(100) NOT NULL,
  config_type VARCHAR(50) NOT NULL,  -- 'form', 'grid', 'workflow', etc.
  config_key VARCHAR(200) NOT NULL,
  config_data JSONB NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(100),
  
  INDEX idx_cru_tag (cru_tag),
  INDEX idx_customer (customer),
  INDEX idx_config_type (config_type)
);
```

**Example Configuration Records:**

```sql
-- Form configuration
INSERT INTO ui_configuration (cru_tag, customer, config_type, config_key, config_data, created_by)
VALUES (
  'CUST-ACME-R07', 'ACME', 'form', 'employee_form',
  '{"fields": [{"name": "firstName", "type": "text", "required": true}, ...]}',
  'config.admin@visionwaves.com'
);

-- Grid configuration
INSERT INTO ui_configuration (cru_tag, customer, config_type, config_key, config_data, created_by)
VALUES (
  'CUST-ACME-R07', 'ACME', 'grid', 'employee_grid',
  '{"columns": [{"field": "firstName", "header": "First Name", "width": 150}, ...]}',
  'config.admin@visionwaves.com'
);

-- Workflow configuration
INSERT INTO ui_configuration (cru_tag, customer, config_type, config_key, config_data, created_by)
VALUES (
  'CUST-ACME-R07', 'ACME', 'workflow', 'approval_workflow',
  '{"steps": [{"name": "submit", "action": "submit_for_approval"}, ...]}',
  'config.admin@visionwaves.com'
);
```

### 11.5 CRU Deployment Process

**Step 1: Query Configurations by CRU Tag**

```sql
SELECT config_type, config_key, config_data
FROM ui_configuration
WHERE cru_tag = 'CUST-ACME-R07'
ORDER BY config_type, config_key;
```

**Step 2: Apply Configurations via API**

```bash
#!/bin/bash

CRU_TAG="CUST-ACME-R07"
ENVIRONMENT="sit"
CONFIG_SERVICE_URL="http://config-service/api"

# Get all configurations for CRU
CONFIGS=$(psql -h $DB_HOST -U $DB_USER -d releases -t -c "
  SELECT config_type, config_key, config_data
  FROM ui_configuration
  WHERE cru_tag = '$CRU_TAG'
")

# Apply each configuration
echo "$CONFIGS" | while IFS='|' read -r type key data; do
  type=$(echo $type | xargs)
  key=$(echo $key | xargs)
  data=$(echo $data | xargs)
  
  echo "Applying $type configuration: $key"
  
  curl -X POST "$CONFIG_SERVICE_URL/config/apply" \
    -H "Content-Type: application/json" \
    -d "{
      \"cruTag\": \"$CRU_TAG\",
      \"environment\": \"$ENVIRONMENT\",
      \"configType\": \"$type\",
      \"configKey\": \"$key\",
      \"configData\": $data
    }"
done

echo "All CRU configurations applied successfully"
```

### 11.6 CRU Best Practices

| Best Practice | Description | Rationale |
|---------------|-------------|-----------|
| **One CRU per Release** | Each solution release has exactly one CRU | Simplifies tracking and deployment |
| **Sequential Numbering** | CRU numbers increment sequentially | Easy to identify latest release |
| **Immutable CRUs** | Never modify existing CRU configurations | Ensures reproducibility |
| **Tag Everything** | All configs must be tagged with CRU | Complete traceability |
| **Test Before Deploy** | Validate CRU in SIT before production | Catch configuration errors early |
| **Document Changes** | Maintain changelog for each CRU | Understand what changed |
| **Version Control** | Store CRU configs in git if possible | Additional backup and history |

### 11.7 CRU Validation Checklist

Before deploying a CRU, verify:

- [ ] All UI forms are configured and tagged
- [ ] All grids are configured and tagged
- [ ] All workflows are configured and tagged
- [ ] All menu items are configured and tagged
- [ ] All permissions are configured and tagged
- [ ] Configurations tested in SIT environment
- [ ] No orphaned configurations (missing CRU tag)
- [ ] CRU tag matches solution release CRU tag
- [ ] Configuration data is valid JSON/format
- [ ] No duplicate configuration keys


<div style="page-break-after: always;"></div>

---

## 12. Deployment Promotion Strategy

### 12.1 Environment Progression

Deployments follow a strict progression through environments:

```
SIT ──────► PreProd ──────► Production
 │            │                │
 │            │                │
 ▼            ▼                ▼
Daily      Weekly          Monthly
Testing    Staging         Live
```

**Environment Characteristics:**

| Environment | Purpose | Deployment Frequency | Approval Required | Rollback Window |
|-------------|---------|---------------------|-------------------|-----------------|
| **SIT** (System Integration Testing) | Development testing, integration validation | Daily / On-demand | No | Immediate |
| **PreProd** (Pre-Production) | Staging, UAT, performance testing | Weekly | Yes (Release Manager) | 24 hours |
| **Production** | Live customer environment | Monthly / As needed | Yes (Customer + RM) | 48 hours |

### 12.2 Promotion Process

#### SIT → PreProd Promotion

**Prerequisites:**
- [ ] Solution release deployed successfully to SIT
- [ ] All smoke tests passed in SIT
- [ ] Integration tests passed
- [ ] No critical bugs identified
- [ ] Minimum 3 days of stable operation in SIT

**Process:**

1. **Verify SIT Deployment**
   ```sql
   SELECT * FROM env_deploy
   WHERE customer = 'ACME' AND environment = 'sit'
   AND solution_release_id = (SELECT id FROM solution_release WHERE release_tag = 'SL-ACME-006')
   AND status = 'SUCCESS'
   ORDER BY started_at DESC LIMIT 1;
   ```

2. **Update PreProd release.yaml**
   ```bash
   git checkout cust/ACME/preprod
   cat > cust/ACME/preprod/release.yaml << 'YAML'
   solutionRelease: "SL-ACME-006"
   environment: "preprod"
   YAML
   git add cust/ACME/preprod/release.yaml
   git commit -m "Promote SL-ACME-006 to ACME PreProd"
   git push origin cust/ACME/preprod
   ```

3. **Create Pull Request**
   - Title: "Promote SL-ACME-006 to ACME PreProd"
   - Description: Include SIT test results and approval
   - Reviewers: Release Manager, DevOps Lead

4. **Approval and Merge**
   - Release Manager reviews and approves
   - Merge triggers CI/CD pipeline
   - Monitor deployment

5. **Post-Deployment Validation**
   - Run UAT test suite
   - Performance testing
   - Security scanning
   - Stakeholder sign-off

#### PreProd → Production Promotion

**Prerequisites:**
- [ ] Solution release deployed successfully to PreProd
- [ ] All UAT tests passed
- [ ] Performance benchmarks met
- [ ] Security scan passed
- [ ] Customer approval obtained
- [ ] Deployment window scheduled
- [ ] Rollback plan prepared
- [ ] Minimum 1 week of stable operation in PreProd

**Process:**

1. **Pre-Production Checklist**
   - [ ] Backup production database
   - [ ] Verify rollback procedure
   - [ ] Notify all stakeholders
   - [ ] Schedule deployment window
   - [ ] Prepare communication plan
   - [ ] Ensure on-call support available

2. **Update Production release.yaml**
   ```bash
   git checkout cust/ACME/prod
   cat > cust/ACME/prod/release.yaml << 'YAML'
   solutionRelease: "SL-ACME-006"
   environment: "prod"
   YAML
   git add cust/ACME/prod/release.yaml
   git commit -m "Promote SL-ACME-006 to ACME Production
   
   Approvals:
   - Release Manager: John Doe
   - Customer: Jane Smith (ACME)
   - DevOps Lead: Bob Johnson
   
   Deployment Window: 2025-11-15 02:00-04:00 UTC
   "
   git push origin cust/ACME/prod
   ```

3. **Create Pull Request**
   - Title: "Promote SL-ACME-006 to ACME Production"
   - Description: Include all approvals and test results
   - Reviewers: Release Manager, DevOps Lead, Customer Representative

4. **Final Approval and Merge**
   - All reviewers approve
   - Merge during scheduled deployment window
   - Monitor deployment closely

5. **Post-Deployment Validation**
   - Verify all services are running
   - Run production smoke tests
   - Monitor application metrics
   - Check error logs
   - Verify customer access
   - Send deployment success notification

6. **Lock Release**
   ```sql
   UPDATE solution_release
   SET status = 'LOCKED',
       updated_at = CURRENT_TIMESTAMP,
       updated_by = 'rm@visionwaves.com'
   WHERE release_tag = 'SL-ACME-006';
   ```

### 12.3 Promotion Approval Matrix

| Environment | Approvers Required | Approval Type |
|-------------|-------------------|---------------|
| **SIT** | None | Automatic |
| **PreProd** | Release Manager | Manual (PR approval) |
| **Production** | Release Manager + Customer Representative + DevOps Lead | Manual (PR approval + sign-off) |

### 12.4 Deployment Windows

**SIT:**
- Available: 24/7
- No restrictions

**PreProd:**
- Preferred: Tuesday-Thursday, 10:00-16:00 UTC
- Avoid: Fridays, weekends, holidays

**Production:**
- Preferred: Tuesday-Wednesday, 02:00-04:00 UTC (low traffic)
- Blackout periods: Month-end, quarter-end, major holidays
- Advance notice: Minimum 1 week

<div style="page-break-after: always;"></div>

---

## 13. Monitoring & Reporting

### 13.1 Deployment Dashboards

#### Dashboard 1: Deployment Status Overview

**Metrics:**
- Current release per environment per customer
- Deployment success rate (last 30 days)
- Average deployment duration
- Failed deployments (last 7 days)
- Pending deployments

**SQL Query:**
```sql
-- Current release per environment
SELECT DISTINCT ON (customer, environment)
  customer,
  environment,
  sr.release_tag,
  sr.cru_tag,
  ed.status,
  ed.started_at,
  ed.deployed_by
FROM env_deploy ed
JOIN solution_release sr ON ed.solution_release_id = sr.id
ORDER BY customer, environment, started_at DESC;
```

#### Dashboard 2: Release History

**Metrics:**
- All releases for a customer
- Components per release
- Deployment timeline
- Release status

**SQL Query:**
```sql
-- Release history for customer
SELECT 
  sr.release_tag,
  sr.cru_tag,
  sr.product_version,
  sr.customer_version,
  sr.status,
  sr.created_at,
  COUNT(cr.id) as component_count,
  COUNT(ed.id) as deployment_count
FROM solution_release sr
LEFT JOIN component_release cr ON sr.id = cr.solution_release_id
LEFT JOIN env_deploy ed ON sr.id = ed.solution_release_id
WHERE sr.customer = 'ACME'
GROUP BY sr.id
ORDER BY sr.created_at DESC;
```

#### Dashboard 3: Component Tracking

**Metrics:**
- Component versions across environments
- Customized vs. product components
- Component deployment frequency

**SQL Query:**
```sql
-- Component versions across environments
SELECT 
  cr.component_key,
  cr.is_customized,
  cr.code_ref_value as version,
  sr.customer,
  ed.environment,
  ed.started_at as deployed_at
FROM component_release cr
JOIN solution_release sr ON cr.solution_release_id = sr.id
JOIN env_deploy ed ON sr.id = ed.solution_release_id
WHERE sr.customer = 'ACME'
AND ed.status = 'SUCCESS'
ORDER BY cr.component_key, ed.environment, ed.started_at DESC;
```

### 13.2 Deployment Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Deployment Success Rate** | > 95% | (Successful deployments / Total deployments) × 100 |
| **Mean Time to Deploy (MTTD)** | < 30 minutes | Average duration from pipeline start to success |
| **Mean Time to Recovery (MTTR)** | < 15 minutes | Average time to rollback from failed deployment |
| **Deployment Frequency** | SIT: Daily, PreProd: Weekly, Prod: Monthly | Number of deployments per time period |
| **Change Failure Rate** | < 5% | (Failed deployments / Total deployments) × 100 |
| **Rollback Rate** | < 2% | (Rollbacks / Total deployments) × 100 |

**SQL Queries for Metrics:**

```sql
-- Deployment Success Rate (last 30 days)
SELECT 
  customer,
  environment,
  COUNT(*) as total_deployments,
  SUM(CASE WHEN status = 'SUCCESS' THEN 1 ELSE 0 END) as successful_deployments,
  ROUND(100.0 * SUM(CASE WHEN status = 'SUCCESS' THEN 1 ELSE 0 END) / COUNT(*), 2) as success_rate
FROM env_deploy
WHERE started_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY customer, environment
ORDER BY customer, environment;

-- Mean Time to Deploy (last 30 days)
SELECT 
  customer,
  environment,
  ROUND(AVG(duration_seconds) / 60, 2) as avg_deploy_time_minutes
FROM env_deploy
WHERE started_at >= CURRENT_DATE - INTERVAL '30 days'
AND status = 'SUCCESS'
GROUP BY customer, environment
ORDER BY customer, environment;

-- Failed Deployments (last 7 days)
SELECT 
  sr.release_tag,
  ed.customer,
  ed.environment,
  ed.started_at,
  ed.error_message,
  ed.deployed_by
FROM env_deploy ed
JOIN solution_release sr ON ed.solution_release_id = sr.id
WHERE ed.status = 'FAILED'
AND ed.started_at >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY ed.started_at DESC;
```

### 13.3 Alerting & Notifications

#### Alert Triggers

| Alert | Condition | Severity | Recipients |
|-------|-----------|----------|------------|
| **Deployment Failed** | env_deploy.status = 'FAILED' | High | DevOps team, Release Manager |
| **Deployment Timeout** | duration_seconds > 3600 (1 hour) | Medium | DevOps team |
| **Rollback Executed** | env_deploy.status = 'ROLLED_BACK' | High | DevOps team, Release Manager, Customer |
| **Production Deployment** | environment = 'prod' AND status = 'SUCCESS' | Info | All stakeholders |
| **Low Success Rate** | Success rate < 90% (last 7 days) | Medium | Release Manager, DevOps Lead |

#### Notification Channels

- **Slack:** Real-time deployment notifications
- **Email:** Deployment summaries, failure alerts
- **Dashboard:** Visual status updates
- **Ticketing System:** Auto-create tickets for failures

#### Slack Notification Example

```bash
#!/bin/bash

SLACK_WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

send_slack_notification() {
  local status=$1
  local release=$2
  local customer=$3
  local environment=$4
  local pipeline_url=$5
  
  local color="good"
  local emoji="✅"
  
  if [ "$status" = "FAILED" ]; then
    color="danger"
    emoji="❌"
  elif [ "$status" = "IN_PROGRESS" ]; then
    color="warning"
    emoji="🔄"
  fi
  
  curl -X POST $SLACK_WEBHOOK_URL \
    -H 'Content-Type: application/json' \
    -d "{
      \"attachments\": [{
        \"color\": \"$color\",
        \"title\": \"$emoji Deployment $status\",
        \"fields\": [
          {\"title\": \"Release\", \"value\": \"$release\", \"short\": true},
          {\"title\": \"Customer\", \"value\": \"$customer\", \"short\": true},
          {\"title\": \"Environment\", \"value\": \"$environment\", \"short\": true},
          {\"title\": \"Status\", \"value\": \"$status\", \"short\": true}
        ],
        \"actions\": [{
          \"type\": \"button\",
          \"text\": \"View Pipeline\",
          \"url\": \"$pipeline_url\"
        }]
      }]
    }"
}

# Usage
send_slack_notification "SUCCESS" "SL-ACME-006" "ACME" "sit" "https://jenkins.example.com/job/142"
```

### 13.4 Deployment Reports

#### Weekly Deployment Report

**Contents:**
- Summary of all deployments
- Success/failure breakdown
- Average deployment time
- Issues and resolutions
- Upcoming deployments

**SQL Query:**
```sql
-- Weekly deployment summary
SELECT 
  ed.customer,
  ed.environment,
  COUNT(*) as total_deployments,
  SUM(CASE WHEN ed.status = 'SUCCESS' THEN 1 ELSE 0 END) as successful,
  SUM(CASE WHEN ed.status = 'FAILED' THEN 1 ELSE 0 END) as failed,
  ROUND(AVG(ed.duration_seconds) / 60, 2) as avg_duration_minutes,
  STRING_AGG(DISTINCT sr.release_tag, ', ') as releases_deployed
FROM env_deploy ed
JOIN solution_release sr ON ed.solution_release_id = sr.id
WHERE ed.started_at >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY ed.customer, ed.environment
ORDER BY ed.customer, ed.environment;
```

#### Monthly Release Report

**Contents:**
- All releases created
- Components per release
- Deployment status per environment
- Customer-specific customizations
- Trends and insights

<div style="page-break-after: always;"></div>

---

## 14. Naming Conventions & Standards

### 14.1 Naming Convention Summary

| Artifact | Format | Example |
|----------|--------|---------|
| **Solution Release Tag** | `SL-<CUSTOMER>-<NNN>` | `SL-ACME-006` |
| **CRU Tag** | `CUST-<TENANT>-R<NN>` | `CUST-ACME-R07` |
| **Docker Image** | `<service>:<customerVer>-<CRU>` | `lcm:2.3.0-CUST-ACME-R07` |
| **Customer Repo Branch** | `<product>-<tenant>-<version>` | `lcm-acme-2.3.0` |
| **Environment Branch** | `cust/<TENANT>/<env>` | `cust/ACME/sit` |
| **Helm Release Name** | `<tenant>-<service>` | `acme-lcm` |
| **Kubernetes Namespace** | `<tenant>-<env>` | `acme-sit` |
| **Product Version** | `v<major>.<minor>.<patch>` | `v1.12.3` |
| **Customer Version** | `<major>.<minor>.<patch>` | `2.3.0` |

### 14.2 Detailed Naming Rules

#### Solution Release Tag

**Format:** `SL-<CUSTOMER>-<NNN>`

**Rules:**
- Prefix: `SL` (Solution)
- Customer: Uppercase, 3-10 characters
- Number: Zero-padded to 3 digits, sequential

**Examples:**
- `SL-ACME-001` - First release for ACME
- `SL-ACME-006` - Sixth release for ACME
- `SL-BETA-012` - Twelfth release for BETA

#### CRU Tag

**Format:** `CUST-<TENANT>-R<NN>`

**Rules:**
- Prefix: `CUST` (Customer)
- Tenant: Uppercase, matches customer name
- Release: `R` followed by zero-padded 2-digit number

**Examples:**
- `CUST-ACME-R01` - First CRU for ACME
- `CUST-ACME-R07` - Seventh CRU for ACME
- `CUST-BETA-R15` - Fifteenth CRU for BETA

#### Docker Image Tag

**Format:** `<service>:<customerVer>-<CRU>`

**Rules:**
- Service: Lowercase, matches component key
- Customer Version: Semantic version
- CRU: Full CRU tag

**Examples:**
- `lcm:2.3.0-CUST-ACME-R07`
- `identity:2.3.0-CUST-ACME-R07`
- `scheduler:2.3.0-CUST-ACME-R07`

#### Git Branch Names

**Customer Repo Branch:** `<product>-<tenant>-<version>`

**Examples:**
- `lcm-acme-2.3.0`
- `identity-beta-1.5.2`

**Environment Branch:** `cust/<TENANT>/<env>`

**Examples:**
- `cust/ACME/sit`
- `cust/ACME/preprod`
- `cust/ACME/prod`

### 14.3 Version Numbering

#### Product Version (Semantic Versioning)

**Format:** `v<major>.<minor>.<patch>`

**Rules:**
- Major: Breaking changes
- Minor: New features, backward compatible
- Patch: Bug fixes, backward compatible

**Examples:**
- `v1.12.3` - Version 1, minor 12, patch 3
- `v2.0.0` - Major version 2 (breaking changes)

#### Customer Version

**Format:** `<major>.<minor>.<patch>`

**Rules:**
- Starts from customer's initial version
- Increments independently from product version
- Includes customizations

**Examples:**
- `2.3.0` - Customer version 2.3.0
- `2.3.1` - Patch release for customer

### 14.4 File and Directory Naming

**Helm Chart Directory:** `helm/charts/<service>`

**Examples:**
- `helm/charts/lcm`
- `helm/charts/identity`

**Helm Values File:** `helm/values/<CUSTOMER>/<env>/<service>.values.yaml`

**Examples:**
- `helm/values/ACME/sit/lcm.values.yaml`
- `helm/values/ACME/prod/identity.values.yaml`

**Release Pointer File:** `cust/<CUSTOMER>/<env>/release.yaml`

**Examples:**
- `cust/ACME/sit/release.yaml`
- `cust/BETA/prod/release.yaml`

<div style="page-break-after: always;"></div>

---

## 15. Appendices

### Appendix A: Quick Reference Guide

**Create Solution Release:**
```sql
INSERT INTO solution_release (solution_key, release_tag, customer, cru_tag, product_version, customer_version, status, created_by)
VALUES ('HumainOS-Core', 'SL-ACME-006', 'ACME', 'CUST-ACME-R07', '1.12.3', '2.3.0', 'DRAFT', 'rm@visionwaves.com');
```

**Add Component:**
```sql
INSERT INTO component_release (solution_release_id, component_key, is_customized, code_repo_url, code_ref_type, code_ref_value, image_uri, deploy_repo_url, deploy_ref_type, deploy_ref_value, helm_chart_path, helm_values_path, helm_release_name)
VALUES (1, 'lcm', true, 'https://github.com/visionwaves/customer-acme-lcm', 'tag', 'lcm-acme-2.3.0', 'registry.visionwaves.com/lcm:2.3.0-CUST-ACME-R07', 'https://github.com/visionwaves/deployments', 'branch', 'cust/ACME/sit', 'helm/charts/lcm', 'helm/values/ACME/sit/lcm.values.yaml', 'acme-lcm');
```

**Deploy Release:**
```bash
git checkout cust/ACME/sit
echo "solutionRelease: \"SL-ACME-006\"" > cust/ACME/sit/release.yaml
echo "environment: \"sit\"" >> cust/ACME/sit/release.yaml
git add cust/ACME/sit/release.yaml
git commit -m "Deploy SL-ACME-006 to ACME SIT"
git push origin cust/ACME/sit
```

**Rollback:**
```bash
git checkout cust/ACME/sit
echo "solutionRelease: \"SL-ACME-005\"" > cust/ACME/sit/release.yaml
echo "environment: \"sit\"" >> cust/ACME/sit/release.yaml
git add cust/ACME/sit/release.yaml
git commit -m "ROLLBACK: Revert to SL-ACME-005"
git push origin cust/ACME/sit
```

### Appendix B: Glossary

| Term | Definition |
|------|------------|
| **Atomic Deployment** | All components deploy together or none deploy |
| **Canary Deployment** | Gradual rollout to subset of users |
| **CRU** | Configuration Release Unit - tagged configuration bundle |
| **Helm** | Kubernetes package manager |
| **Idempotent** | Operation that produces same result when repeated |
| **Rollback** | Reverting to previous stable release |
| **Smoke Test** | Basic functionality test after deployment |
| **UAT** | User Acceptance Testing |

### Appendix C: Contact Information

| Role | Contact | Email | Slack |
|------|---------|-------|-------|
| **Release Manager** | John Doe | rm@visionwaves.com | @john.doe |
| **DevOps Lead** | Bob Johnson | devops@visionwaves.com | @bob.johnson |
| **LCM Product Owner** | Alice Smith | lcm.owner@visionwaves.com | @alice.smith |
| **Identity Product Owner** | Charlie Brown | identity.owner@visionwaves.com | @charlie.brown |
| **Support Team** | Support | support@visionwaves.com | #support |

### Appendix D: Related Documents

- **Testing Strategy** - QA/Testing-Strategy.md
- **Infrastructure Guide** - DevOps/Infrastructure-Guide.md
- **Security Procedures** - Security/Security-Procedures.md
- **Incident Response** - Operations/Incident-Response.md
- **Customer Onboarding** - Customer-Success/Onboarding-Guide.md

### Appendix E: Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-11-09 | Visionwaves Engineering | Initial release |

---

**Document End**

For questions or feedback, please contact: release-management@visionwaves.com


<div style="page-break-after: always;"></div>

---

## 16. BOM (Bill of Materials) Management

### 16.1 BOM Concept

A **Bill of Materials (BOM)** is a Maven/Gradle artifact that centralizes dependency version management across multiple modules and products.

**Key Benefits:**
- ✅ **Centralized Version Management** - Single source of truth for all dependency versions
- ✅ **Consistency** - All modules use the same versions of shared libraries
- ✅ **Simplified Updates** - Update BOM version to update all dependencies
- ✅ **Reduced Conflicts** - Eliminates version mismatches across modules
- ✅ **Easier Maintenance** - Manage versions in one place

### 16.2 Enttribe Platform BOM Structure

Based on the actual Enttribe platform codebase:

```
enttribe-platform-services-bom (v7.0.12)
│
├── Platform Core JARs (6 modules)
│   ├── enttribe-core
│   ├── enttribe-mysql-impl
│   ├── enttribe-cql-impl
│   ├── one-platform-api
│   ├── one-platform-service
│   └── one-platform-microservice
│
├── Commons Libraries (4 modules)
│   ├── core-utility
│   ├── spring-utility
│   ├── spring-boot-utility
│   └── http-utility
│
└── Third-Party Dependencies
    ├── Spring Boot 3.5.3
    ├── Spring Cloud 2023.0.3
    ├── Hibernate 6.2.3.Final
    ├── MariaDB Driver 3.4.1
    └── Keycloak 26.2.5
```

**BOM Hierarchy:**

```
┌──────────────────────────────────────────────────────────────┐
│  platform-services-bom (Master BOM)                          │
│  Version: 7.0.12                                             │
│  GroupId: com.enttribe.platform                              │
└────────────────────────┬─────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┬──────────────────┐
         │               │               │                  │
         ▼               ▼               ▼                  ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Platform    │  │ Commons     │  │ Spring Boot │  │ Third-Party │
│ Core JARs   │  │ Libraries   │  │ BOM         │  │ Libraries   │
│ v7.0.12     │  │ v6.0.7      │  │ v3.5.3      │  │ Various     │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
      │                │                │                  │
      ▼                ▼                ▼                  ▼
┌──────────┐    ┌──────────┐    ┌──────────┐      ┌──────────┐
│ core     │    │ core-    │    │ spring-  │      │ hibernate│
│ mysql    │    │ utility  │    │ boot-    │      │ mariadb  │
│ cql      │    │ spring-  │    │ starter  │      │ keycloak │
│ api      │    │ utility  │    │ web/jpa  │      │ lombok   │
└──────────┘    └──────────┘    └──────────┘      └──────────┘
```

### 16.3 Complete BOM Definition (Factual)

#### Platform Services BOM - Complete pom.xml

This BOM is based on the **actual Enttribe platform codebase** (version 7.0.12):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.enttribe.platform</groupId>
    <artifactId>platform-services-bom</artifactId>
    <version>7.0.12</version>
    <packaging>pom</packaging>

    <name>Enttribe Platform Services BOM</name>
    <description>
        Bill of Materials (BOM) for Enttribe Platform Services.
        Centralizes dependency version management for all platform, commons, and service modules.
    </description>

    <properties>
        <java.version>21</java.version>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

        <!-- Spring Boot & Cloud Versions -->
        <spring-boot.version>3.5.3</spring-boot.version>
        <spring-cloud.version>2023.0.3</spring-cloud.version>

        <!-- Enttribe Platform Core Versions (All use same version) -->
        <enttribe.platform.version>7.0.12</enttribe.platform.version>

        <!-- Enttribe Commons Versions -->
        <enttribe.commons.version>6.0.7</enttribe.commons.version>

        <!-- Third-party Dependencies -->
        <hibernate.version>6.2.3.Final</hibernate.version>
        <mariadb.connector.version>3.4.1</mariadb.connector.version>
        <lombok.version>1.18.30</lombok.version>
        <jackson.version>2.15.3</jackson.version>
        <commons-lang3.version>3.18.0</commons-lang3.version>
        <commons-io.version>2.14.0</commons-io.version>
        <commons-fileupload.version>1.6.0</commons-fileupload.version>
        <keycloak.version>26.2.5</keycloak.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- ========================================== -->
            <!-- Spring Boot BOM (Import) -->
            <!-- ========================================== -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- ========================================== -->
            <!-- Spring Cloud BOM (Import) -->
            <!-- ========================================== -->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- ========================================== -->
            <!-- Enttribe Platform Core Libraries -->
            <!-- ========================================== -->
            <dependency>
                <groupId>com.enttribe.platform</groupId>
                <artifactId>enttribe-core</artifactId>
                <version>${enttribe.platform.version}</version>
            </dependency>

            <dependency>
                <groupId>com.enttribe.platform</groupId>
                <artifactId>enttribe-mysql-impl</artifactId>
                <version>${enttribe.platform.version}</version>
            </dependency>

            <dependency>
                <groupId>com.enttribe.platform</groupId>
                <artifactId>enttribe-cql-impl</artifactId>
                <version>${enttribe.platform.version}</version>
            </dependency>

            <dependency>
                <groupId>com.enttribe.platform</groupId>
                <artifactId>one-platform-api</artifactId>
                <version>${enttribe.platform.version}</version>
            </dependency>

            <dependency>
                <groupId>com.enttribe.platform</groupId>
                <artifactId>one-platform-service</artifactId>
                <version>${enttribe.platform.version}</version>
            </dependency>

            <dependency>
                <groupId>com.enttribe.platform</groupId>
                <artifactId>one-platform-microservice</artifactId>
                <version>${enttribe.platform.version}</version>
            </dependency>

            <!-- ========================================== -->
            <!-- Enttribe Commons Libraries -->
            <!-- ========================================== -->
            <dependency>
                <groupId>com.enttribe.commons</groupId>
                <artifactId>core-utility</artifactId>
                <version>${enttribe.commons.version}</version>
            </dependency>

            <dependency>
                <groupId>com.enttribe.commons</groupId>
                <artifactId>spring-utility</artifactId>
                <version>${enttribe.commons.version}</version>
            </dependency>

            <dependency>
                <groupId>com.enttribe.commons</groupId>
                <artifactId>spring-boot-utility</artifactId>
                <version>${enttribe.commons.version}</version>
            </dependency>

            <dependency>
                <groupId>com.enttribe.commons</groupId>
                <artifactId>http-utility</artifactId>
                <version>${enttribe.commons.version}</version>
            </dependency>

            <!-- ========================================== -->
            <!-- Third-Party Dependencies -->
            <!-- ========================================== -->
            
            <!-- Hibernate -->
            <dependency>
                <groupId>org.hibernate</groupId>
                <artifactId>hibernate-core</artifactId>
                <version>${hibernate.version}</version>
            </dependency>

            <dependency>
                <groupId>org.hibernate</groupId>
                <artifactId>hibernate-envers</artifactId>
                <version>${hibernate.version}</version>
            </dependency>

            <!-- Database Drivers -->
            <dependency>
                <groupId>org.mariadb.jdbc</groupId>
                <artifactId>mariadb-java-client</artifactId>
                <version>${mariadb.connector.version}</version>
            </dependency>

            <!-- Lombok -->
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
                <scope>provided</scope>
            </dependency>

            <!-- Jackson -->
            <dependency>
                <groupId>com.fasterxml.jackson.core</groupId>
                <artifactId>jackson-databind</artifactId>
                <version>${jackson.version}</version>
            </dependency>

            <dependency>
                <groupId>com.fasterxml.jackson.core</groupId>
                <artifactId>jackson-core</artifactId>
                <version>${jackson.version}</version>
            </dependency>

            <dependency>
                <groupId>com.fasterxml.jackson.core</groupId>
                <artifactId>jackson-annotations</artifactId>
                <version>${jackson.version}</version>
            </dependency>

            <!-- Apache Commons -->
            <dependency>
                <groupId>org.apache.commons</groupId>
                <artifactId>commons-lang3</artifactId>
                <version>${commons-lang3.version}</version>
            </dependency>

            <dependency>
                <groupId>commons-io</groupId>
                <artifactId>commons-io</artifactId>
                <version>${commons-io.version}</version>
            </dependency>

            <dependency>
                <groupId>commons-fileupload</groupId>
                <artifactId>commons-fileupload</artifactId>
                <version>${commons-fileupload.version}</version>
            </dependency>

            <!-- Keycloak -->
            <dependency>
                <groupId>org.keycloak</groupId>
                <artifactId>keycloak-core</artifactId>
                <version>${keycloak.version}</version>
            </dependency>

        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

<div style="page-break-after: always;"></div>

### 16.4 Product Service Using BOM - Complete Example

#### LCM Service pom.xml (Using Platform BOM)

This example shows how a product service (LCM) uses the platform BOM:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- ========================================== -->
    <!-- Product Service Coordinates -->
    <!-- ========================================== -->
    <groupId>com.visionwaves.product</groupId>
    <artifactId>lcm-service</artifactId>
    <version>2.3.0</version>
    <packaging>jar</packaging>

    <name>LCM Service</name>
    <description>
        Lifecycle Management Service for ACME Corporation.
        Uses Enttribe Platform Services BOM for dependency management.
    </description>

    <properties>
        <java.version>21</java.version>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

        <!-- ========================================== -->
        <!-- Single BOM Version Property -->
        <!-- Change this ONE property to upgrade ALL platform dependencies -->
        <!-- ========================================== -->
        <enttribe.platform.bom.version>7.0.12</enttribe.platform.bom.version>

        <!-- Customer-specific metadata -->
        <customer.name>ACME</customer.name>
        <customer.version>CUST-ACME-R07</customer.version>
        <solution.release>SL-ACME-006</solution.release>
    </properties>

    <!-- ========================================== -->
    <!-- Dependency Management: Import Platform BOM -->
    <!-- ========================================== -->
    <dependencyManagement>
        <dependencies>
            <!-- Import Enttribe Platform Services BOM -->
            <!-- This provides versions for ALL platform, commons, and third-party dependencies -->
            <dependency>
                <groupId>com.enttribe.platform</groupId>
                <artifactId>platform-services-bom</artifactId>
                <version>${enttribe.platform.bom.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- ========================================== -->
    <!-- Dependencies: NO VERSIONS NEEDED! -->
    <!-- All versions are managed by the BOM above -->
    <!-- ========================================== -->
    <dependencies>

        <!-- ========================================== -->
        <!-- Enttribe Platform Core Dependencies -->
        <!-- ========================================== -->
        <dependency>
            <groupId>com.enttribe.platform</groupId>
            <artifactId>enttribe-core</artifactId>
            <!-- Version managed by BOM -->
        </dependency>

        <dependency>
            <groupId>com.enttribe.platform</groupId>
            <artifactId>enttribe-mysql-impl</artifactId>
            <!-- Version managed by BOM -->
        </dependency>

        <dependency>
            <groupId>com.enttribe.platform</groupId>
            <artifactId>one-platform-api</artifactId>
            <!-- Version managed by BOM -->
        </dependency>

        <!-- ========================================== -->
        <!-- Enttribe Commons Dependencies -->
        <!-- ========================================== -->
        <dependency>
            <groupId>com.enttribe.commons</groupId>
            <artifactId>core-utility</artifactId>
            <!-- Version managed by BOM -->
        </dependency>

        <dependency>
            <groupId>com.enttribe.commons</groupId>
            <artifactId>spring-utility</artifactId>
            <!-- Version managed by BOM -->
        </dependency>

        <dependency>
            <groupId>com.enttribe.commons</groupId>
            <artifactId>spring-boot-utility</artifactId>
            <!-- Version managed by BOM -->
        </dependency>

        <!-- ========================================== -->
        <!-- Spring Boot Dependencies -->
        <!-- Versions managed by Spring Boot BOM (imported by platform BOM) -->
        <!-- ========================================== -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <!-- Version from Spring Boot BOM -->
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
            <!-- Version from Spring Boot BOM -->
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
            <!-- Version from Spring Boot BOM -->
        </dependency>

        <!-- ========================================== -->
        <!-- Database & Third-Party Dependencies -->
        <!-- Versions managed by platform BOM -->
        <!-- ========================================== -->
        <dependency>
            <groupId>org.mariadb.jdbc</groupId>
            <artifactId>mariadb-java-client</artifactId>
            <scope>runtime</scope>
            <!-- Version managed by BOM -->
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
            <!-- Version managed by BOM -->
        </dependency>

        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <!-- Version managed by BOM -->
        </dependency>

        <dependency>
            <groupId>commons-io</groupId>
            <artifactId>commons-io</artifactId>
            <!-- Version managed by BOM -->
        </dependency>

        <!-- ========================================== -->
        <!-- Customer-Specific Dependencies -->
        <!-- These are NOT in the BOM, so we specify versions explicitly -->
        <!-- ========================================== -->
        <dependency>
            <groupId>com.acme.custom</groupId>
            <artifactId>acme-integration</artifactId>
            <version>1.5.0</version>
        </dependency>

        <dependency>
            <groupId>com.acme.custom</groupId>
            <artifactId>acme-workflow-extensions</artifactId>
            <version>2.1.0</version>
        </dependency>

        <!-- ========================================== -->
        <!-- Test Dependencies -->
        <!-- ========================================== -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <!-- Version from Spring Boot BOM -->
        </dependency>

    </dependencies>

    <!-- ========================================== -->
    <!-- Build Configuration -->
    <!-- ========================================== -->
    <build>
        <finalName>${project.artifactId}-${project.version}</finalName>
        
        <plugins>
            <!-- Spring Boot Maven Plugin -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>3.5.3</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <!-- Maven Compiler Plugin -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                </configuration>
            </plugin>

            <!-- Maven Surefire Plugin (for tests) -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
            </plugin>
        </plugins>
    </build>

    <!-- ========================================== -->
    <!-- Distribution Management -->
    <!-- ========================================== -->
    <distributionManagement>
        <repository>
            <id>visionwaves-releases</id>
            <name>Visionwaves Release Repository</name>
            <url>https://artifact.visionwaves.com/releases</url>
        </repository>
        <snapshotRepository>
            <id>visionwaves-snapshots</id>
            <name>Visionwaves Snapshot Repository</name>
            <url>https://artifact.visionwaves.com/snapshots</url>
        </snapshotRepository>
    </distributionManagement>

</project>
```

<div style="page-break-after: always;"></div>

### 16.5 BOM Update Strategy

#### Scenario: Upgrading from BOM 7.0.12 to 7.1.0

**Step 1: Platform Team Updates BOM**

```xml
<!-- Update BOM version -->
<groupId>com.enttribe.platform</groupId>
<artifactId>platform-services-bom</artifactId>
<version>7.1.0</version> <!-- Updated from 7.0.12 -->

<properties>
    <!-- Update platform libraries -->
    <enttribe.platform.version>7.1.0</enttribe.platform.version>
    
    <!-- Update commons if needed -->
    <enttribe.commons.version>6.1.0</enttribe.commons.version>
    
    <!-- Update Spring Boot if needed -->
    <spring-boot.version>3.5.4</spring-boot.version>
</properties>
```

**Step 2: Product Owners Update Services**

```xml
<!-- In lcm-service/pom.xml -->
<properties>
    <!-- Change ONE property to upgrade ALL dependencies -->
    <enttribe.platform.bom.version>7.1.0</enttribe.platform.bom.version>
</properties>
```

**Step 3: Build and Test**

```bash
# Build with new BOM version
mvn clean install

# Run tests
mvn test

# Create Docker image
docker build -t lcm-service:2.4.0 .
```

**Step 4: Create New Solution Release**

```sql
INSERT INTO solution_release (
    solution_key, release_tag, customer, cru_tag,
    product_version, customer_version,
    status, created_by, created_at
) VALUES (
    'NetsinularityOS-Core', 'SL-ACME-007', 'ACME', 'CUST-ACME-R08',
    '7.1.0', '2.4.0',
    'DRAFT', 'rm@visionwaves.com', NOW()
);
```

### 16.6 BOM Deployment Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Platform Team Creates/Updates BOM                      │
│ - Update platform-services-bom/pom.xml                         │
│ - Update version properties                                     │
│ - Build: mvn clean install                                      │
│ - Deploy: mvn deploy                                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Product Owners Update Services                         │
│ - Update enttribe.platform.bom.version property                │
│ - Build: mvn clean install                                      │
│ - Test: mvn test                                                │
│ - Create Docker image                                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Release Manager Creates Solution Release               │
│ - INSERT INTO solution_release                                  │
│ - Add component_release entries                                 │
│ - Mark status = 'READY'                                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Deploy to Environments                                  │
│ - Deploy to SIT (test BOM compatibility)                        │
│ - Deploy to PreProd (validate)                                  │
│ - Deploy to Production                                          │
│ - Lock release                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 16.7 BOM Best Practices

#### ✅ DO

1. **Use Single BOM Version Property**
   ```xml
   <properties>
       <enttribe.platform.bom.version>7.0.12</enttribe.platform.bom.version>
   </properties>
   ```

2. **Import BOM in dependencyManagement**
   ```xml
   <dependencyManagement>
       <dependencies>
           <dependency>
               <groupId>com.enttribe.platform</groupId>
               <artifactId>platform-services-bom</artifactId>
               <version>${enttribe.platform.bom.version}</version>
               <type>pom</type>
               <scope>import</scope>
           </dependency>
       </dependencies>
   </dependencyManagement>
   ```

3. **Omit Versions for BOM-Managed Dependencies**
   ```xml
   <dependency>
       <groupId>com.enttribe.platform</groupId>
       <artifactId>enttribe-core</artifactId>
       <!-- No version - managed by BOM -->
   </dependency>
   ```

4. **Test After BOM Updates**
   ```bash
   mvn clean test
   mvn verify
   ```

5. **Document BOM Version in Release Notes**
   - Include BOM version in solution release documentation
   - Track which services use which BOM version

#### ❌ DON'T

1. **Don't Specify Versions for BOM-Managed Dependencies**
   ```xml
   <!-- BAD: Version conflicts with BOM -->
   <dependency>
       <groupId>com.enttribe.platform</groupId>
       <artifactId>enttribe-core</artifactId>
       <version>7.0.11</version> <!-- Don't do this! -->
   </dependency>
   ```

2. **Don't Mix Different BOM Versions**
   ```xml
   <!-- BAD: Inconsistent versions -->
   <enttribe.platform.bom.version>7.0.12</enttribe.platform.bom.version>
   <enttribe.commons.bom.version>6.0.5</enttribe.commons.bom.version>
   ```

3. **Don't Skip Testing After BOM Updates**
   - Always run full test suite
   - Test in SIT before production

4. **Don't Update BOM Without Documentation**
   - Document BOM changes in release notes
   - Communicate BOM updates to all teams

### 16.8 BOM Verification Commands

```bash
# 1. Show dependency tree with versions
mvn dependency:tree

# 2. Show effective POM (with resolved versions from BOM)
mvn help:effective-pom

# 3. List all dependencies with versions
mvn dependency:list

# 4. Check for dependency conflicts
mvn dependency:analyze

# 5. Verify BOM is being used
mvn dependency:tree | grep "com.enttribe"

# Expected output:
# [INFO] +- com.enttribe.platform:enttribe-core:jar:7.0.12:compile
# [INFO] +- com.enttribe.platform:enttribe-mysql-impl:jar:7.0.12:compile
# [INFO] +- com.enttribe.commons:core-utility:jar:6.0.7:compile
```

### 16.9 BOM Checklist

#### For Platform Team (Creating/Updating BOM)

- [ ] All platform JARs use same version (e.g., 7.0.12)
- [ ] All commons JARs use same version (e.g., 6.0.7)
- [ ] Spring Boot version is compatible
- [ ] Java version is specified correctly
- [ ] Third-party dependencies are up-to-date
- [ ] BOM builds successfully: `mvn clean install`
- [ ] BOM deployed to Maven repository: `mvn deploy`
- [ ] Release notes created

#### For Product Owners (Using BOM)

- [ ] BOM version property updated in pom.xml
- [ ] No version conflicts with BOM
- [ ] All platform dependencies use BOM versions
- [ ] Build successful: `mvn clean install`
- [ ] Tests pass: `mvn test`
- [ ] Docker image created successfully
- [ ] Tested in local environment
- [ ] Ready for SIT deployment

#### For Release Managers (Tracking BOM)

- [ ] BOM version documented in release notes
- [ ] All component releases use same BOM version
- [ ] Deployment tested in SIT
- [ ] Release notes include BOM version
- [ ] Rollback plan includes BOM version

### 16.10 Troubleshooting BOM Issues

#### Issue 1: Dependency Version Not Resolved

**Symptom:**
```
[ERROR] 'dependencies.dependency.version' for com.enttribe.platform:enttribe-core:jar is missing.
```

**Solution:**
```xml
<!-- Ensure BOM is imported with correct scope and type -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.enttribe.platform</groupId>
            <artifactId>platform-services-bom</artifactId>
            <version>7.0.12</version>
            <type>pom</type>          <!-- Required -->
            <scope>import</scope>      <!-- Required -->
        </dependency>
    </dependencies>
</dependencyManagement>
```

#### Issue 2: Wrong Version Being Used

**Symptom:**
```
[INFO] com.enttribe.platform:enttribe-core:jar:7.0.11:compile
```
Expected: 7.0.12

**Solution:**
Check dependency order. First BOM imported wins:
```xml
<dependencyManagement>
    <dependencies>
        <!-- This BOM takes precedence -->
        <dependency>
            <groupId>com.enttribe.platform</groupId>
            <artifactId>platform-services-bom</artifactId>
            <version>7.0.12</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

#### Issue 3: BOM Not Found

**Symptom:**
```
[ERROR] Failed to read artifact descriptor for com.enttribe.platform:platform-services-bom:pom:7.0.12
```

**Solution:**
```bash
# 1. Check if BOM exists in local repository
ls ~/.m2/repository/com/enttribe/platform/platform-services-bom/7.0.12/

# 2. If not found, install BOM locally
cd platform-services-bom
mvn clean install

# 3. Or deploy to remote repository
mvn deploy -DaltDeploymentRepository=nexus::default::https://artifact.visionwaves.com/releases
```

---

**End of Section 17: BOM Management**

