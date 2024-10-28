# Turing Challenge

## Top-level diagram

![top level diagram](images/top-level-diagram.png)

### 1. Authentication / Authorization

The client registers in **Azure API Management** developer portal and then makes a request to the Azure API Management API endpoint using their **subscription key**. Azure API Management validates the subscription key along with applicable quotas (policies) and forwards the request to the Function App. Before performing any action, the Function App can validate which **products** the subscriber has access to.

Request example:

```
GET https://turing-challenge.azure-api.net/api/endpoint?address=xxx
Ocp-Apim-Subscription-Key: <subscription_key>
```

### 2. Function App

The Function App executes a Durable Function using the Function [**Function chaining pattern**](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=isolated-process%2Cnodejs-v3%2Cv1-model&pivots=csharp#chaining). Each activity performs necessary tasks within its own scope, such as calling third-party services, storing or reading data in a database, or saving files. Each Function also logs telemetry and custom metrics.

![top level diagram](images/function-chaining.png)

#### Considerations

The Function App could be *Consuption plan* because it supports
- 1.000.000 monthly request for free
- Up to 200 instances
- Up to 5 minutes timeout

Azure Function example:

```csharp
[FunctionName("Chaining")]
public static async Task<object> RunAsync(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    try
    {
        var x = await context.CallActivityAsync<object>("F1", null);
        var y = await context.CallActivityAsync<object>("F2", x);
        var z = await context.CallActivityAsync<object>("F3", y);
        return  await context.CallActivityAsync<object>("F4", z);
    }
    catch (Exception)
    {
        // Error handling or compensation goes here.
    }
}
```

Response example:

```json
{
    "propertyOne": "",
    "propertyTwo": "",
    "imageSasUri": "https://accountName.blob.core.windows.net/xxx.png?sv=xxx&se=xxx&sr=xxx&sig=xxx"
}
```
### 3. Telemetry

```csharp
 [FunctionName("Chaining")]
    public static async Task<object> RunAsync(
        [OrchestrationTrigger] IDurableOrchestrationContext context)
    {
        var telemetryClient = new TelemetryClient();

        try
        {
            // Measure time for F1
            var start = DateTime.UtcNow;
            var x = await context.CallActivityAsync<object>("F1", null);
            var elapsed = DateTime.UtcNow - start;
            telemetryClient.TrackMetric("F1Duration", elapsed.TotalMilliseconds);

            // Measure time for F2
            start = DateTime.UtcNow;
            var y = await context.CallActivityAsync<object>("F2", x);
            elapsed = DateTime.UtcNow - start;
            telemetryClient.TrackMetric("F2Duration", elapsed.TotalMilliseconds);

            // Measure time for F3
            start = DateTime.UtcNow;
            var z = await context.CallActivityAsync<object>("F3", y);
            elapsed = DateTime.UtcNow - start;
            telemetryClient.TrackMetric("F3Duration", elapsed.TotalMilliseconds);

            // Measure time for F4
            start = DateTime.UtcNow;
            var result = await context.CallActivityAsync<object>("F4", z);
            elapsed = DateTime.UtcNow - start;
            telemetryClient.TrackMetric("F4Duration", elapsed.TotalMilliseconds);

            return result;
        }
        catch (Exception ex)
        {
            telemetryClient.TrackException(ex);
            throw;
        }
    }
```

### 4. Health checks & Azure Monitor

By include Health checks API in Function App we can monitor availability and performance even third-party integrations.

Example request:

```
GET https://turing-challenge.azure-api.net/api/health-checks
Ocp-Apim-Subscription-Key: <subscription_key>
```

Example response:

```json
{
  "status": "Healthy",
  "details": {
    "AzureCosmosDB": {
      "status": "Healthy",
      "responseTime": "120ms",
      "details": {
        "database": "connected",
        "lastCheck": "2024-10-28T10:30:00Z"
      }
    },
    "AzureStorage": {
      "status": "Degraded",
      "responseTime": "300ms",
      "details": {
        "storageAccount": "connected",
        "lastCheck": "2024-10-28T10:30:00Z",
        "issues": "High response time"
      }
    },
    "GoogleMapsAPI": {
      "status": "Unhealthy",
      "responseTime": "500ms",
      "details": {
        "error": "503 Service Unavailable",
        "lastCheck": "2024-10-28T10:30:00Z",
        "message": "Google Maps API is currently unavailable"
      }
    }
  },
  "timestamp": "2024-10-28T10:30:05Z"
}
```

Now, with the response we have configured using the Health Checks API, we can connect it to the "availability" module in Azure Monitor and configure alerts in case a service malfunctions

![top level diagram](images/azure-monitor-standard-test.png)

With an alert, we can trigger another Azure Function or Logic App (for example) to create a Jira ticket.

### 5. CI/CD

Solution repository should contains a .yml file that triggers when a user completes a "Pull Requests"

The pipeline should
- Build solution
- Execute xUnit tests
- Deploy Function App

![top level diagram](images/cicd.png)

This a more detailed diagram about Gitflow:

![top level diagram](images/gitflow.png)

### 7. Costs

Monthly costs will be around 150€.

| Resource | Tier | Monthly cost |
|--|--|--|
| Azure Function App | Consumption | 0€ |
| Azure Storage Account | Writes up to 200k files and store 200Gb | 5€ |
| Azure API Management | Basic | 130€ |
| Azure Cosmos Db | Serverless | 30€  |

Costs can be optimized with a few considerations:

- We can create budgets in Azure Cost Management and alerts to notify subscription administrators
- Older files can be moved to cold storage
- Minimize the telemetry data we store
