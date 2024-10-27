# Turing Challenge

## Top-level diagram

![top level diagram](images/top-level-diagram.png)

### 1. Security

- The client registers in Azure API Management.
- The client make a request to Azure API Management API endpoint with **subscription key**.
- The Function App can validate **which products** the subscriber has access to before any other action.

```
GET https://turing-challenge.azure-api.net/api/endpoint
Ocp-Apim-Subscription-Key: <subscription_key>
```

### 2. Function App

- The Function App executes a Durable Function with [**Function chaining pattern**](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=isolated-process%2Cnodejs-v3%2Cv1-model&pivots=csharp#chaining)
- Each function will do what is necessary under its own scope (call third-party services, store or read data in a database, store files, ...)
- The Function App could be *Consuption plan* because it supports
  - 1.000.000 monthly request for free
  - Up to 200 instances
  - Up to 5 minutes timeout

Azure Function example:

```csharp
[Function("evaluate")]
public static async Task<object> EvaluateAsync(
    [OrchestrationTrigger] TaskOrchestrationContext context,
    string address)
{
    try
    {
        var plot = await context.CallActivityAsync<object>("GetPlotAsync", address);
        var satellite = await context.CallActivityAsync<object>("GetGoogleSatelliteAsync", plot.Coordinates);
        var street = await context.CallActivityAsync<object>("GetGoogleStreetAsync", plot.Coordinates);

        // ...

        var prediction = await context.CallActivityAsync<object>("GetPredictionAsync", plot.Coordinates);

        await context.CallActivityAsync<object>("SaveImageAsync", satellite);
        await context.CallActivityAsync<object>("SaveImageAsync", street);

        // ...

        return  await context.CallActivityAsync<object>("BuildAnswer", (plot, satellite, street, prediction));
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
    "imageSasUri: "https://accountName.blob.core.windows.net/xxx.png?sv=xxx&se=xxx&sr=xxx&sig=xxx"
}
```

### Health checks

