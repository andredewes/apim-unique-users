# Plotting "unique users" charts from API Management logs

It is quite a common requirement when working with API gateways (such as [Azure API Management](https://learn.microsoft.com/azure/api-management/api-management-key-concepts)) to understand how many users are actively accessing your applications in the backend. However, the solution for that is not the same for everyone. For instance, a metric exposed out-of-the-box is the total number of HTTP requests... but that doesn't tell us how many unique users we have. Normally, an application running in the client devices generates multiple HTTP requests towards the backend.

So... how can we get the number of active and unique users from Azure API Management data?

# Defining what is a "unique user" in your context

The first question we must ask ourselves is: how do you identify a user in the context of your HTTP requests? Some of the options include:

- *By caller's IP address?*

That is normally not a good option because there is no guarantee that a caller will have an unique IP address. Multiple users behind NAT devices might share the same IP address. Or even the opposite: the same user might be accessing the backend using multiple devices and multiple IP addresses will be logged for that same user

- *By number incoming HTTP requests?*

Also not a good option because an app normally generates multiple HTTP requests for each user actions. This metric give us an idea how much usage our API Management is having across all users and devices but definitely not a good approach to check number of unique and active users

- *By crafting specific API endpoints to log "login" data?*

That is for sure a viable approach: you enforce that calling apps will perform a call to some endpoint defined in your APIs and it will receive the login data and store this somewhere so you can use this information to get some statistics. However, this includes development effort in both frontend and backend to make this happen. I am looking for an "easier" approach.

- **Using auth tokens (JWT) sent by your apps**

Now, let's stop for a moment and analyze the most common authentication/authorization mechanism today: [JWT tokens](https://learn.microsoft.com/azure/active-directory/develop/security-tokens). If your apps make use of JWT tokens and send them to your API Management instance, you might implement something much quicker to get unique users data!
The reasoning is because these tokens normally include the user's email address or some unique ID that represents their identity in its [claims](https://learn.microsoft.com/azure/active-directory/develop/security-tokens#json-web-tokens-and-claims). In case of Microsoft Entra tokens, the [oid](https://learn.microsoft.com/azure/active-directory/develop/id-token-claims-reference#use-claims-to-reliably-identify-a-user) claim can be used to identify an user uniquely within a tenant.

This article will focus on this approach since it is very reliable: extracting the **"oid"** claim from the incoming JWT tokens received in the *Authorization* HTTP header and then plotting a chart of how many active users we have passing through API Management!

## Configuring API Management to export logs

The first step is to make sure you have configured your API Management resource logs to be exported to a Log Analytics workspace. Follow the steps [here](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor#resource-logs):

![diag-settings](/images/diag-settings.png)

The second step is to [enable logging](https://learn.microsoft.com/azure/api-management/api-management-howto-use-azure-monitor#modify-api-logging-settings) in your APIs. And, in that configuration, make sure to include the HTTP header name that you receive your JWT tokens to be saved in the logs too:

![api-logging](/images/api-logging.png)

## Crafting a Kusto query to plot the chart

Now that we have setup API Management to export its logs to Log Analytics, we can [query the data](https://learn.microsoft.com/azure/api-management/api-management-howto-use-azure-monitor#view-diagnostic-data-in-azure-monitor) in Azure Monitor. Make sure that your "Authorization" header is present in the logs by running this query in Azure Monitor:

```
ApiManagementGatewayLogs
| where isnotempty(RequestHeaders)
| project RequestHeaders, TimeGenerated
| order by TimeGenerated desc
| limit 10
```
![requestheaders](/images/monitor-headers.png)

Now you can build the final query that will give you a chart which you can count how many users you have by summarizing the "oid" claim in your JWTs. This Kusto query:

```
let subscriptions = dynamic(['my-app-1', 'my-app-2', 'my-app-3', 'my-app-5', 'my-app-5']);
ApiManagementGatewayLogs
| where isnotempty(RequestHeaders['Authorization'])
| extend encodedPayload = tostring(split(RequestHeaders['Authorization'], '.')[1])   //Extract the part of JWT where we keep the claims
| extend decodedClaims = base64_decode_tostring(encodedPayload)                      //Decode from Base64 to plain text
| extend usernameClaim = tostring(parse_json(decodedClaims)["oid"])                  //Parse the JSON field 'oid'
| project ApimSubscriptionId, usernameClaim, TimeGenerated
| summarize Users = dcount(usernameClaim) by bin(TimeGenerated, 1h), ApimSubscriptionId
| render columnchart
```

This will plot this chart:

![requestheaders](/images/uniqueusers-chart.png)

Now, some important notes about the query and chart:
- In this scenario, each calling application have its own APIM [subscription](https://learn.microsoft.com/azure/api-management/api-management-subscriptions) key. The goal is to see how many users there are **per application** too, not for the entire APIM instance. That's why in the first line I'm filtering only for the apps (subscriptions) I'm interested in: "my-app-1", "my-app-2", "my-app-3", "my-app-4" and "my-app-5" 
- The grouping is done hourly. That is, each chart bar will represent one hour and each bar is sub-divided in the 5 applications. You can change that time frame to daily for example, just adjust from *bin(TimeGenerated, 1h)* to *bin(TimeGenerated, 1d)*
- The *parse_json* Kusto function might not perform well if your dataset is huge. You need to try the query yourself to see if that will be a problem or not. If yes, you might consider a process to extract that claim using APIM policies and send it to some other location such as [Event Hub](https://learn.microsoft.com/azure/api-management/api-management-howto-log-event-hubs)
- Another faster but less accurate way would be to not decode the extract the 'oid' claim from the JWT but just use the raw string from Authorization header as-is for the calculations. However, keep in mind that access and ID tokens change from time to time for the same user and you will likely get higher number of users by using this approach
- Be aware of any implications in regulations and privacy when saving tokens in a data store. Check with your compliance team before doing it

# Conclusion

If you are working with API Management in an organization, it is very likely that soon or later you will be interested (or asked!) to provide a chart or report of how many unique users are using the API Gateway and its applications. Instead of trying to come up with a custom and complex solution, first check if you can leverage your existing data for that!