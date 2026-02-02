# Azure Subscription Spend Information - API Specification Document

**Document Version:** 1.0  
**Date:** February 2, 2026  
**Purpose:** Specification for accessing Azure subscription spend information via REST APIs for Power BI Dashboard integration

---

## 1. Executive Summary

This document outlines the requirements and technical specifications for accessing Azure subscription spend and cost data using Microsoft Azure REST APIs. The data retrieved will be used to populate a Power BI dashboard for cost management and analysis.

---

## 2. API Overview

Microsoft provides multiple APIs for accessing Azure cost and usage data:

| API | Best For | Data Type |
|-----|----------|-----------|
| **Cost Details API** | Automation, detailed cost data | Detailed, unaggregated cost/usage |
| **Consumption APIs** | Legacy usage details | Usage details, reservations, marketplace |
| **Cost Management Query API** | Ad-hoc cost analysis | Aggregated cost data |
| **Exports API** | Large datasets, scheduled reports | Automated exports to storage |

### Recommended Approach

For Power BI dashboards, we recommend:
1. **Power BI Cost Management Connector** - Native integration for direct data access
2. **Cost Details API** - For programmatic automation and custom integrations
3. **Exports API** - For large datasets (>2GB/month)

---

## 3. Authentication Requirements

### 3.1 OAuth 2.0 Authentication

All Azure Cost Management APIs require Azure Active Directory (Microsoft Entra ID) authentication.

#### Required Steps:
1. **Register an Application** in Microsoft Entra ID
2. **Configure API Permissions** for Cost Management
3. **Obtain Access Token** using OAuth 2.0 flow

#### Service Principal Setup:
```bash
# Azure CLI - Register Application
az ad app create --display-name "CostManagement-PowerBI"
az ad sp create --id <app-id>
```

#### Required Permissions:
| Scope | Permission | Description |
|-------|------------|-------------|
| Billing Account | Billing Reader | Read billing data |
| Subscription | Cost Management Reader | Read cost data at subscription level |
| Resource Group | Reader | Access resource-level cost data |

### 3.2 Supported Account Types

| Account Type | Supported | Notes |
|--------------|-----------|-------|
| Enterprise Agreement (EA) | ✅ Yes | Full support, requires EA Administrator |
| Microsoft Customer Agreement (MCA) | ✅ Yes | Billing profile access required |
| Web Direct (Pay-as-you-go) | ⚠️ Limited | Use Exports or Consumption API |
| CSP Azure Plan | ✅ Yes | Partner Center API may be required |
| Sponsored/Internal | ❌ No | Not supported |

---

## 4. Cost Details API (Recommended)

### 4.1 API Endpoint

**Base URL:** `https://management.azure.com`

**API Version:** `2024-08-01` (or latest available)

### 4.2 Workflow

The Cost Details API uses an **asynchronous workflow**:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  1. Create      │    │  2. Poll for    │    │  3. Download    │
│     Report      │───▶│     Status      │───▶│     Report      │
│     (POST)      │    │     (GET)       │    │     (CSV)       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 4.3 Step 1: Create Report Request

**Endpoint:**
```http
POST https://management.azure.com/subscriptions/{subscriptionId}/providers/Microsoft.CostManagement/generateCostDetailsReport?api-version=2024-08-01
```

**Request Headers:**
```http
Content-Type: application/json
Authorization: Bearer {access-token}
```

**Request Body:**
```json
{
  "metric": "ActualCost",
  "timePeriod": {
    "start": "2026-01-01",
    "end": "2026-01-31"
  }
}
```

#### Request Parameters:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `metric` | string | No | `ActualCost` (default) or `AmortizedCost` |
| `timePeriod.start` | date | Conditional | Start date (YYYY-MM-DD) |
| `timePeriod.end` | date | Conditional | End date (YYYY-MM-DD) |
| `billingPeriod` | string | Conditional | EA only, format: YYYYMM |
| `invoiceId` | string | Conditional | MCA only, specific invoice |

> **Note:** Use only ONE of: `timePeriod`, `billingPeriod`, or `invoiceId`

#### Metric Types:

| Metric | Description |
|--------|-------------|
| `ActualCost` | Charges as they were billed (purchases shown on purchase date) |
| `AmortizedCost` | Spreads reservation/savings plan purchases across their term |

### 4.4 Step 2: Poll for Status

**Initial Response (HTTP 202 Accepted):**
```http
Location: https://management.azure.com/subscriptions/{subscriptionId}/providers/Microsoft.CostManagement/costDetailsOperationStatus/{operationId}?api-version=2024-08-01
Retry-After: 60
```

**Poll Endpoint:**
```http
GET https://management.azure.com/subscriptions/{subscriptionId}/providers/Microsoft.CostManagement/costDetailsOperationStatus/{operationId}?api-version=2024-08-01
Authorization: Bearer {access-token}
```

### 4.5 Step 3: Download Report

**Completed Response (HTTP 200 OK):**
```json
{
  "id": "subscriptions/{subscriptionId}/providers/Microsoft.CostManagement/operationResults/{operationId}",
  "name": "{operationId}",
  "status": "Completed",
  "manifest": {
    "manifestVersion": "2024-08-01",
    "dataFormat": "Csv",
    "blobCount": 1,
    "byteCount": 160769,
    "compressData": false,
    "blobs": [
      {
        "blobLink": "{downloadLink}",
        "byteCount": 32741
      }
    ]
  },
  "validTill": "2026-02-02T08:08:46.1973252Z"
}
```

---

## 5. Cost Details File Schema

### 5.1 Key Fields for Power BI Dashboard

| Field | Type | Description |
|-------|------|-------------|
| `Date` | date | Usage or purchase date |
| `SubscriptionId` | string | Azure subscription GUID |
| `SubscriptionName` | string | Subscription display name |
| `ResourceGroup` | string | Resource group name |
| `ResourceName` | string | Resource name |
| `ResourceId` | string | Full ARM resource ID |
| `MeterCategory` | string | Service category (e.g., Virtual Machines) |
| `MeterSubCategory` | string | Service subcategory (e.g., Dsv3 Series) |
| `MeterName` | string | Meter name |
| `Quantity` | decimal | Units consumed |
| `UnitOfMeasure` | string | Unit of measurement |
| `CostInBillingCurrency` | decimal | Cost before credits/taxes |
| `BillingCurrency` | string | Currency code (e.g., USD) |
| `ChargeType` | string | `Usage`, `Purchase`, or `Refund` |
| `PricingModel` | string | `OnDemand`, `Reservation`, `Spot`, `SavingsPlan` |
| `ServiceFamily` | string | Service family grouping |
| `Tags` | string | Resource tags (JSON format) |

### 5.2 Additional Fields for Analysis

| Field | Description |
|-------|-------------|
| `PublisherType` | `Azure`, `Marketplace`, or `AWS` |
| `ReservationId` | Reservation identifier (if applicable) |
| `ReservationName` | Reservation name (if applicable) |
| `Frequency` | `OneTime`, `Recurring`, `UsageBased` |
| `EffectivePrice` | Negotiated/discounted price |
| `UnitPrice` | List price |
| `PayGPrice` | Pay-as-you-go retail price |
| `CostAllocationRuleName` | Cost allocation rule (if configured) |

---

## 6. Alternative APIs

### 6.1 Consumption Usage Details API (Legacy)

> **Note:** This API is being deprecated. Use Cost Details API for new implementations.

**Endpoint:**
```http
GET https://management.azure.com/{scope}/providers/Microsoft.Consumption/usageDetails?api-version=2024-08-01
```

**Scope Options:**
- Subscription: `/subscriptions/{subscriptionId}`
- Resource Group: `/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}`
- Billing Account: `/providers/Microsoft.Billing/billingAccounts/{billingAccountId}`

**Query Parameters:**

| Parameter | Description |
|-----------|-------------|
| `$filter` | OData filter expression |
| `$top` | Maximum records to return (1-1000) |
| `$expand` | Include additional properties |
| `metric` | `ActualCost` or `AmortizedCost` |

**Example Request:**
```http
GET https://management.azure.com/subscriptions/{subscriptionId}/providers/Microsoft.Consumption/usageDetails?$filter=properties/usageStart ge '2026-01-01' and properties/usageEnd le '2026-01-31'&$top=1000&api-version=2024-08-01
```

### 6.2 Cost Management Query API

For aggregated cost analysis queries:

**Endpoint:**
```http
POST https://management.azure.com/{scope}/providers/Microsoft.CostManagement/query?api-version=2024-08-01
```

**Request Body:**
```json
{
  "type": "ActualCost",
  "timeframe": "MonthToDate",
  "dataset": {
    "granularity": "Daily",
    "aggregation": {
      "totalCost": {
        "name": "Cost",
        "function": "Sum"
      }
    },
    "grouping": [
      {
        "type": "Dimension",
        "name": "ServiceName"
      },
      {
        "type": "Dimension",
        "name": "ResourceGroup"
      }
    ]
  }
}
```

### 6.3 Exports API (For Large Datasets)

For automated, scheduled exports to Azure Storage:

**Create Export:**
```http
PUT https://management.azure.com/{scope}/providers/Microsoft.CostManagement/exports/{exportName}?api-version=2024-08-01
```

**Request Body:**
```json
{
  "properties": {
    "schedule": {
      "status": "Active",
      "recurrence": "Daily",
      "recurrencePeriod": {
        "from": "2026-01-01T00:00:00Z",
        "to": "2026-12-31T00:00:00Z"
      }
    },
    "format": "Csv",
    "deliveryInfo": {
      "destination": {
        "resourceId": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Storage/storageAccounts/{storageAccountName}",
        "container": "costexports",
        "rootFolderPath": "daily"
      }
    },
    "definition": {
      "type": "ActualCost",
      "timeframe": "MonthToDate",
      "dataSet": {
        "granularity": "Daily"
      }
    }
  }
}
```

---

## 7. Power BI Integration Options

### 7.1 Option 1: Native Cost Management Connector (Recommended)

The Power BI Cost Management Connector provides direct integration:

**Steps:**
1. Open Power BI Desktop
2. Get Data → Azure → Azure Cost Management
3. Select scope (EA Enrollment or MCA Billing Account)
4. Authenticate with OAuth 2.0
5. Select data tables (Usage Details, Budgets, etc.)

**Available Tables:**

| Table | Description |
|-------|-------------|
| Usage details | Detailed consumption data |
| Usage details amortized | Amortized cost data |
| Balance summary | EA balance information |
| Budgets | Budget configurations |
| Pricesheets | Meter rates |
| RI recommendations | Reserved Instance suggestions |
| RI usage details | Reservation utilization |
| Marketplace charges | Third-party charges |

**Limitations:**
- Maximum ~$5M of raw cost details
- Data refresh: 8-24 hours lag
- Schedule refreshes: 1-2 times per day recommended

### 7.2 Option 2: REST API with Power Query

Use Power Query to call the Cost Details API:

```powerquery
let
    subscriptionId = "your-subscription-id",
    startDate = "2026-01-01",
    endDate = "2026-01-31",
    
    // Step 1: Create report request
    createUrl = "https://management.azure.com/subscriptions/" & subscriptionId & "/providers/Microsoft.CostManagement/generateCostDetailsReport?api-version=2024-08-01",
    
    requestBody = "{""metric"": ""ActualCost"", ""timePeriod"": {""start"": """ & startDate & """, ""end"": """ & endDate & """}}",
    
    createResponse = Web.Contents(createUrl, [
        Content = Text.ToBinary(requestBody),
        Headers = [
            #"Content-Type" = "application/json",
            #"Authorization" = "Bearer " & accessToken
        ],
        ManualStatusHandling = {202}
    ]),
    
    // Parse Location header for polling URL
    locationHeader = Value.Metadata(createResponse)[Headers][Location],
    
    // Step 2: Poll until complete (implement polling logic)
    // Step 3: Download CSV from blobLink
    
    Source = Csv.Document(Web.Contents(blobLink))
in
    Source
```

### 7.3 Option 3: Export to Azure Storage + Power BI

1. Configure automated exports to Azure Blob Storage
2. Connect Power BI to Azure Blob Storage
3. Schedule Power BI refresh to match export schedule

---

## 8. Best Practices

### 8.1 API Usage

| Best Practice | Description |
|---------------|-------------|
| **Request Frequency** | Maximum once per day; data refreshes every 4 hours |
| **Date Ranges** | Chunk large requests into daily/weekly segments |
| **Caching** | Cache historical data; charges don't change after invoice close |
| **Error Handling** | Implement retry logic with exponential backoff |

### 8.2 Data Considerations

| Consideration | Recommendation |
|---------------|----------------|
| **Large Datasets** | Use Exports API for >2GB monthly data |
| **Real-time Needs** | Data has 8-24 hour latency; plan accordingly |
| **Cost Allocation** | Use Cost Details API (not Usage Details) for allocated costs |
| **Amortization** | Use AmortizedCost metric for reservation/savings plan analysis |

### 8.3 Security

- Store credentials in Azure Key Vault
- Use managed identities where possible
- Apply principle of least privilege for RBAC assignments
- Rotate service principal secrets regularly

---

## 9. Error Handling

### 9.1 Common Error Codes

| HTTP Code | Error | Resolution |
|-----------|-------|------------|
| 400 | Bad Request | Validate request parameters |
| 401 | Unauthorized | Refresh access token |
| 403 | Forbidden | Check RBAC permissions |
| 404 | Not Found | Verify scope/resource exists |
| 429 | Too Many Requests | Implement rate limiting/backoff |
| 500 | Server Error | Retry with exponential backoff |

### 9.2 Error Response Format

```json
{
  "error": {
    "code": "InvalidDateRange",
    "message": "The specified date range exceeds the maximum allowed period of 12 months."
  }
}
```

---

## 10. Sample Implementation Workflow

### 10.1 Daily Data Ingestion Pipeline

```
┌─────────────────┐
│   Trigger       │  (Azure Function Timer / Logic App Schedule)
│   (Daily 6AM)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Get Access     │  (Client Credentials OAuth Flow)
│  Token          │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Call Cost      │  (POST generateCostDetailsReport)
│  Details API    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Poll Until     │  (GET costDetailsOperationStatus)
│  Complete       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Download CSV   │  (GET blobLink)
│  to Storage     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Power BI       │  (Scheduled Refresh)
│  Refresh        │
└─────────────────┘
```

---

## 11. References

| Resource | URL |
|----------|-----|
| Azure Consumption API Reference | https://learn.microsoft.com/en-us/rest/api/consumption/ |
| Cost Management API Reference | https://learn.microsoft.com/en-us/rest/api/cost-management/ |
| Cost Details API | https://learn.microsoft.com/en-us/rest/api/cost-management/generate-cost-details-report |
| Power BI Cost Management Connector | https://learn.microsoft.com/en-us/power-bi/connect-data/desktop-connect-azure-cost-management |
| Understanding Cost Details Fields | https://learn.microsoft.com/en-us/azure/cost-management-billing/automate/understand-usage-details-fields |
| Choosing a Cost Details Solution | https://learn.microsoft.com/en-us/azure/cost-management-billing/automate/usage-details-best-practices |
| Exports Tutorial | https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-export-acm-data |

---

## 12. Appendix: Sample Power BI Dashboard Metrics

### Recommended Visualizations:

| Visualization | Data Source | Description |
|---------------|-------------|-------------|
| **Total Spend Card** | `CostInBillingCurrency` SUM | Current period total cost |
| **Spend Trend** | Daily aggregated costs | Line chart over time |
| **Cost by Service** | Group by `MeterCategory` | Pie/bar chart breakdown |
| **Cost by Resource Group** | Group by `ResourceGroup` | Hierarchical view |
| **Top 10 Resources** | Top N by cost | Table with drill-down |
| **Budget vs Actual** | Budgets API + Usage | Gauge chart |
| **Reservation Utilization** | RI Usage Details | Percentage utilized |
| **Anomaly Detection** | Daily cost variance | Highlight unusual spikes |

---

*Document prepared for Azure Cost Management Power BI Dashboard implementation project.*
