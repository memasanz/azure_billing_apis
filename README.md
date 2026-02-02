# Azure Cost Management API

A Python project for retrieving and analyzing Azure subscription spend information using the Azure Cost Management REST APIs.

## Overview

This project demonstrates how to:
- Authenticate with Azure using Service Principal or Azure CLI
- Call the Azure Cost Details API to retrieve subscription spend data
- Parse and analyze cost data with pandas
- Visualize cost trends and breakdowns
- Export data for Power BI dashboards

## Prerequisites

- Python 3.9+
- Azure subscription with Cost Management Reader access
- Service Principal with appropriate permissions (or Azure CLI logged in)

## Setup

### 1. Clone the Repository

```bash
git clone <repository-url>
cd cost_management
```

### 2. Create Virtual Environment (Optional)

```bash
python -m venv venv
venv\Scripts\activate  # Windows
# or
source venv/bin/activate  # Linux/Mac
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure Environment Variables

Copy the sample environment file and fill in your Azure credentials:

```bash
cp .env.sample .env
```

Edit `.env` with your values:

```env
AZURE_SUBSCRIPTION_ID=your-subscription-id
AZURE_TENANT_ID=your-tenant-id
AZURE_CLIENT_ID=your-client-id
AZURE_CLIENT_SECRET=your-client-secret
```

### 5. Set Up Azure Permissions

Your Service Principal needs the **Cost Management Reader** role on the subscription:

1. Go to **Azure Portal** → **Subscriptions** → Select your subscription
2. Click **Access control (IAM)** → **Add role assignment**
3. Select **Cost Management Reader** role
4. Assign to your Service Principal

## Usage

### Jupyter Notebook

Open and run the notebook:

```bash
jupyter notebook azure_cost_details_api.ipynb
```

The notebook walks through:
1. Authentication setup
2. Requesting a cost details report
3. Polling for completion
4. Downloading and parsing results
5. Data analysis and visualization
6. Exporting for Power BI

### API Workflow

The Cost Details API uses an asynchronous workflow:

```
POST /generateCostDetailsReport  →  Poll for status  →  Download CSV
```

## Project Structure

```
cost_management/
├── azure_cost_details_api.ipynb   # Main notebook with API examples
├── Azure_Cost_Management_API_Spec.md  # API specification document
├── requirements.txt               # Python dependencies
├── .env.sample                    # Sample environment variables
├── .env                          # Your credentials (git-ignored)
├── .gitignore                    # Git ignore rules
└── README.md                     # This file
```

## API Reference

| Endpoint | Description |
|----------|-------------|
| `POST /generateCostDetailsReport` | Start async cost report generation |
| `GET /costDetailsOperationStatus/{id}` | Poll for report completion |
| Download from `blobLink` | Get CSV with cost details |

For full API documentation, see [Azure_Cost_Management_API_Spec.md](Azure_Cost_Management_API_Spec.md).

## Power BI Integration

Options for connecting to Power BI:

1. **Native Connector**: Use the built-in Azure Cost Management connector in Power BI Desktop
2. **CSV Import**: Export data from the notebook and import into Power BI
3. **Scheduled Exports**: Use Azure Exports API to automate data delivery to Azure Storage

## Resources

- [Azure Cost Details API Documentation](https://learn.microsoft.com/en-us/rest/api/cost-management/generate-cost-details-report)
- [Azure Consumption API](https://learn.microsoft.com/en-us/rest/api/consumption/)
- [Power BI Cost Management Connector](https://learn.microsoft.com/en-us/power-bi/connect-data/desktop-connect-azure-cost-management)
- [Understanding Cost Details Fields](https://learn.microsoft.com/en-us/azure/cost-management-billing/automate/understand-usage-details-fields)

## License

MIT
