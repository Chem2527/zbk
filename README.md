## Section 1: Cloud Infrastructure and Environment Setup Guide

This section outlines how to create the necessary Azure telemetry resources, copy the required connection strings, and establish local and automated CI/CD environment files to connect the application.

### 1. Azure Resource Provisioning Steps
To setup a fresh ingestion telemetry database in the Azure Portal, follow these steps:

#### Step A: Create a Log Analytics Workspace
1. Log in to the Azure Portal at portal.azure.com.
2. In the top search bar, type Log Analytics workspaces and select it from the results.
3. Click the Create button.
4. Fill in the following details:
   * Subscription: Choose your active Azure subscription.
   * Resource Group: Select or create your target resource group (e.g., rg-saikrishna-3312).
   * Name: e.g., zenus-pp-log-analytics.
   * Region: Select your closest deployment region (e.g., East US 2).
5. Click Review + Create at the bottom, wait for validation, and click Create.

#### Step B: Create the Application Insights Instance
1. In the top search bar of the Azure Portal, search for Application Insights and select it.
2. Click the Create button.
3. Fill in the following details:
   * Subscription and Resource Group: Select the same subscription and resource group used in Step A.
   * Name: e.g., PP-Admin-Webapp-Insights.
   * Region: Select the same deployment region (e.g., East US 2).
   * Resource Mode: Ensure Workspace-based is selected.
   * Log Analytics Workspace: Select the workspace you created in Step A (zenus-pp-log-analytics).
4. Click Review + Create, wait for validation to succeed, and click Create.

---

### 2. How to Retrieve the Connection String
Once the Application Insights instance has finished deploying, you can retrieve the connection string:

1. Navigate to your newly created Application Insights resource (PP-Admin-Webapp-Insights).
2. In the left-hand sidebar menu of the resource, scroll down to the Configure section and select Properties.
3. Locate the Connection String field.
4. Click the Copy to Clipboard icon next to the Connection String value.
5. The copied value will look similar to this:
   `InstrumentationKey=6445c96b-9a6d-4f5e-a33e-c1e1c3de0c86;IngestionEndpoint=https://eastus2-3.in.applicationinsights.azure.com/;LiveEndpoint=https://eastus2.livediagnostics.monitor.azure.com/;ApplicationId=7b73d40e-e5e2-4f50-b870-e2ee51d0f7e3`

---

### 3. Local Environment Setup
To connect your local development environment to the newly created Azure telemetry resources:

1. At the root of your project directory (the same folder containing package.json), create a new file named **.env**.
2. Open **.env** and paste your copied Connection String as a variable:
   ```env
   REACT_APP_APPINSIGHTS_CONNECTION_STRING="InstrumentationKey=6445c96b-9a6d-4f5e-a33e-c1e1c3de0c86;IngestionEndpoint=https://eastus2-3.in.applicationinsights.azure.com/;LiveEndpoint=https://eastus2.livediagnostics.monitor.azure.com/;ApplicationId=7b73d40e-e5e2-4f50-b870-e2ee51d0f7e3"
   ```
3. To prepare for production staging builds, create another file named **.env.prod** at the root of your project directory.
4. Open **.env.prod** and add your connection string:
   ```env
   REACT_APP_APPINSIGHTS_CONNECTION_STRING="InstrumentationKey=6445c96b-9a6d-4f5e-a33e-c1e1c3de0c86;IngestionEndpoint=https://eastus2-3.in.applicationinsights.azure.com/;LiveEndpoint=https://eastus2.livediagnostics.monitor.azure.com/;ApplicationId=7b73d40e-e5e2-4f50-b870-e2ee51d0f7e3"
   ```

*Note: Both **.env** and **.env.prod** are correctly listed in **.gitignore** to prevent secrets from being pushed to the remote git repository.*

---

### 4. GitHub CI/CD Pipeline Automation Setup
To allow the automated GitHub Actions staging pipeline to compile the React application with this connection string:

1. Open your repository on GitHub.
2. Go to Settings ➔ Secrets and variables ➔ Actions.
3. Click the New repository secret button.
4. Set the name to exactly: **`REACT_APP_APPINSIGHTS_CONNECTION_STRING`**
5. Set the value to your copied Azure Connection String:
   `InstrumentationKey=6445c96b-9a6d-4f5e-a33e-c1e1c3de0c86;IngestionEndpoint=https://eastus2-3.in.applicationinsights.azure.com/;LiveEndpoint=https://eastus2.livediagnostics.monitor.azure.com/;ApplicationId=7b73d40e-e5e2-4f50-b870-e2ee51d0f7e3`
6. Click Add secret.
7. Push your local Git staging commits. The modified staging workflow will automatically pick up this secret, write it to **.env.prod** on the runner, and bundle it securely during `docker build`.

---

### 5. Azure DevOps Pipeline Automation Setup
If your team uses Azure DevOps (Azure Pipelines) instead of or in addition to GitHub Actions, you can configure your secret connection string and inject it during the build task using these steps:

#### Step A: Configure the Secret Variable in Azure DevOps
You can store the Connection String securely as a pipeline secret:
1. Open your project in Azure DevOps.
2. In the left-hand menu, navigate to Pipelines ➔ Library.
3. Select or create a Variable Group (e.g., fintech-staging-variables), or open your Pipeline Editor and click the Variables button at the top right.
4. Click Add or New Variable.
5. Set the Name to exactly:
   `REACT_APP_APPINSIGHTS_CONNECTION_STRING`
6. Set the Value to your copied Azure Connection String:
   `InstrumentationKey=6445c96b-9a6d-4f5e-a33e-c1e1c3de0c86;IngestionEndpoint=https://eastus2-3.in.applicationinsights.azure.com/;LiveEndpoint=https://eastus2.livediagnostics.monitor.azure.com/;ApplicationId=7b73d40e-e5e2-4f50-b870-e2ee51d0f7e3`
7. Check the checkbox or click the padlock icon to Keep this value secret (this encrypts it securely, preventing it from being printed in logs).
8. Click OK or Save.

#### Step B: Inject the Secret in the azure-pipelines.yml File
In Azure DevOps Pipelines, secret variables are not automatically decrypted into script tasks unless they are explicitly mapped in the task's environment block.

To write this secret into **.env.prod** right before your Docker build step runs, add a Bash task in your **azure-pipelines.yml** file:

```yaml
- task: Bash@3
  displayName: 'Inject Telemetry Connection String'
  env:
    CONNECTION_STRING: $(REACT_APP_APPINSIGHTS_CONNECTION_STRING)
  inputs:
    targetType: 'inline'
    script: |
      echo "REACT_APP_APPINSIGHTS_CONNECTION_STRING=$CONNECTION_STRING" > .env.prod
```

Place this task immediately before your Docker build and push task. The Docker build process will then pick up the generated **.env.prod** file and bake the active connection string securely into the React production bundle.

---

## Section 2: Kusto Query Language (KQL) Log Verification Queries

To inspect the telemetry logs in the Azure Portal (Application Insights ➔ Logs), use the following standard, case-sensitive KQL queries:

### 1. Track Failed REST API Invocations
Lists all failed outbound API calls logged by the Axios interceptor in the last 5 minutes:
```kusto
customEvents
| where customDimensions.component == "PP-Admin-Webapp"
| where name == "RestRequestFailed"
| where timestamp > ago(5m)
```

### 2. Monitor Slow API Performance (Latency > 5 seconds)
Identifies API calls that took longer than 5000 milliseconds to complete within the last 5 minutes:
```kusto
customEvents
| where customDimensions.component == "PP-Admin-Webapp"
| where name == "RestRequestProcessed"
| extend duration = todouble(customDimensions.durationMs)
| where duration > 5000
| where timestamp > ago(5m)
```

### 3. Track All Client Page Views
Lists all tracked page views sorted chronologically, showing user navigation paths:
```kusto
pageViews
| where customDimensions.component == "PP-Admin-Webapp"
| order by timestamp desc
```

### 4. Search and View Client Exceptions
Retrieves all error and crash logs recorded by the Application Insights exceptions handler:
```kusto
exceptions
| where customDimensions.component == "PP-Admin-Webapp"
| order by timestamp desc
```

### 5. Retrieve General Custom Event Telemetry
Fetches all custom event types tracked by the web application for auditing:
```kusto
customEvents
| where customDimensions.component == "PP-Admin-Webapp"
| order by timestamp desc
```

---

## Section 6: Telemetry Workbooks and Dashboards

Five custom Azure Monitor Workbooks have been constructed for the `pp-admin-webapp-insights` component in subscription `Azure subscription 1` under resource group `rg-saikrishna-3312`.

### Registered Workbooks List
1. **PP-Admin-Webapp Telemetry Dashboard**
   * ID: `38818cd1-4e59-4edc-8f23-ebf3b180d0c3`
   * Purpose: Centralized diagnostic panel showcasing aggregate application health, page views, and API latency.
2. **error logs-workboks**
   * ID: `38818cd1-4e59-4edc-8f23-ebf3b180d0f7`
   * Purpose: Displays all REST invocation errors, console exceptions, and tracking failures.
3. **Page views - workbooks**
   * ID: `38818cd1-4e59-4edc-8f23-ebf3b180d125`
   * Purpose: Traces route transitions, active page hits, and user session traffic paths.
4. **Slow API Performance - workbook**
   * ID: `38818cd1-4e59-4edc-8f23-ebf3b180d145`
   * Purpose: Flags API endpoints exceeding latency baselines to diagnose backend bottlenecks.
5. **API Failure Spikes**
   * ID: `38818cd1-4e59-4edc-8f23-ebf3b180d162`
   * Purpose: Tracks real-time status code errors (4xx, 5xx) to trigger early warning operational alarms.

### Workbook Screenshots
Below are the visual dashboards captured from the Azure portal:

#### 1. Telemetry Dashboard Overview
<img width="1907" height="306" alt="workbook-telemetry" src="https://github.com/user-attachments/assets/40e21443-dc90-4ab9-8f22-8b94be7abcf2" />


#### 2. Page Views Workbook
<img width="1846" height="603" alt="page views - workbooks" src="https://github.com/user-attachments/assets/cf3ad594-1383-46cf-ac04-0b7604a5fbca" />



#### 3. Error Logs Workbook


---<img width="1880" height="602" alt="error logs - workbooks" src="https://github.com/user-attachments/assets/d83ee2db-2290-4372-a714-31094903816a" />


## Section 3: Azure Alert Rules Configuration

### Creating Alert Rules from Workbook Queries

To create an Azure Monitor Alert Rule based on a query running inside a custom Workbook, you must first enable the external query option inside the Workbook editor. Follow these precise steps:

1. **Enable the External Query Button:**
   * Open the custom Workbook (e.g., `error logs-workboks` or `API Failure Spikes`) in the Azure Portal.
   * Click the **Edit** button in the top menu to enter editing mode.
   * Locate the specific query block you want to configure and click its individual **Edit** button.
   * In the query editing interface, select the **Advanced Settings** tab at the top.
   * Check the checkbox: **`Show open external query button when not editing`**.
   * Click the **Done Editing** button on that query block.
   * Click the **Save** (disk icon) or **Done Editing** button at the very top of the Workbook to apply the change permanently.
     <img width="1367" height="818" alt="alerts-check-mandate" src="https://github.com/user-attachments/assets/d846edfe-609f-4df7-b666-54e3427bfbb3" />


2. **Open the Query in Logs:**
   * When viewing the Workbook in normal (non-editing) mode, hover over the right-hand corner of the query block.
   * You will see a small square icon with an arrow (the **Open query in Logs** button).
   * Click this icon. Azure will launch the Application Insights **Logs** analytical panel with the exact KQL query loaded automatically.

3. **Provision the Alert Rule:**
   * In the Logs interface, click the **+ New alert rule** (or **Create alert rule**) button in the top toolbar.
   * This action will launch the Azure Monitor alert wizard with your target KQL query automatically populated as the **Condition**.
   * Proceed to configure the Measurement (Count of table rows), Alert Logic (e.g., GreaterThan 5), Severity, Actions, and details before saving the rule.

---

### Example Staging Alert Rule Setup
A highly responsive log-search alert rule has been provisioned to monitor staging performance:

* **Alert Rule Name:** `Staging Exception Spike Alert`
* **Target Resource:** `PP-Admin-Webapp-Insights` (Application Insights instance)
* **Resource Group:** `rg-saikrishna-3312`
* **Region:** `eastus2`
* **Severity:** `3 - Informational` (standard staging threshold)
* **Measurement Logic:**
  * **Measure:** Table rows
  * **Aggregation Type:** Count
  * **Aggregation Granularity:** 5 minutes
  * **Evaluation Frequency:** 5 minutes
* **Alert Trigger Condition:** `Static GreaterThan 5` violations within a `5 minutes` evaluation window (1 aggregated point).
* **Self-Healing:** Automatically resolve alerts is enabled to automatically clear active warnings once the failure spike resolves.



