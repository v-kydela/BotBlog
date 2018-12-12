# Bot Analytics: Behind the Scenes
##### by Kyle Delaney <div style="text-align: right">November 13, 2018</div>

[Application Insights](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-overview) is a great way to gather analytic information from all kinds of applications, including bots. An Application Insights resource in Azure provides an [Analytics](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-analytics) tool that allows you to run queries with a form of the [Kusto Query Language](https://docs.microsoft.com/en-us/azure/kusto/query/tutorial), also known as Log Analytics Query Language. 

![Analytics](https://docs.microsoft.com/en-us/azure/application-insights/media/app-insights-analytics/001.png)

To get an idea of what queries you might want to run for a bot, you can switch away from the Application Insights resource and have a look at the Analytics blade in your bot resource (Web App Bot, Functions Bot, or Bot Channels Registration). That will show you some charts that look like the following screenshots. The queries are provided that are being run behind the scenes to populate these charts.

![User retention](https://docs.microsoft.com/en-us/azure/bot-service/media/analytics-retention.png)

	let usersByDay = customEvents
		| where name == 'Activity' and timestamp >= startofday(ago(10d)) and timestamp <= startofday(now())
		| extend from = customDimensions['from']
		| extend from = tostring(iff(isnull(from), customDimensions['From ID'], from))
		| extend channel = customDimensions['channel']
		| extend channel = tostring(iff(isnull(channel), customDimensions['Channel ID'], channel))
		| summarize results = dcount(from) by bin(timestamp, 1d), from, channel;
	usersByDay
		| join kind = leftouter usersByDay on from and channel
		| where timestamp <= timestamp1 and timestamp<startofday(ago(1d))
		| project timestamp, from = from, channel = channel, activity_time = timestamp1
		| summarize event_count = dcount(from) by timestamp, activity_time, channel
		| order by timestamp asc

![Users](https://i.stack.imgur.com/Z0deg.jpg)

Timeline:

	customEvents
		| where name == 'Activity'
		| extend from = customDimensions['from']
		| extend from = tostring(iff(isnull(from), customDimensions['From ID'], from))
		| extend channel = customDimensions['channel']
		| extend channel = tostring(iff(isnull(channel), customDimensions['Channel ID'], channel))
		| extend activityType = customDimensions['Activity type']
		| where activityType != 'conversationUpdate'
		| summarize event_count = dcount(from) by bin(timestamp, 1d), channel
		| order by timestamp asc
		| project timestamp = timestamp, customDimensions_channel = channel, event_count

Totals:

	customEvents
		| where name == 'Activity'
		| extend from = customDimensions['from']
		| extend from = tostring(iff(isnull(from), customDimensions['From ID'], from))
		| extend channel = customDimensions['channel']
		| extend channel = tostring(iff(isnull(channel), customDimensions['Channel ID'], channel))
		| extend activityType = customDimensions['Activity type']
		| where activityType != 'conversationUpdate'
		| summarize event_count = dcount(from) by channel
		| project customDimensions_channel = channel, event_count

![Activities](https://i.stack.imgur.com/8TUnX.jpg)

Timeline:

	customEvents
		| where name == 'Activity'
		| extend channel = customDimensions['channel']
		| extend channel = tostring(iff(isnull(channel), customDimensions['Channel ID'], channel))
		| summarize event_count = count() by bin(timestamp, 1d), channel
		| order by timestamp asc
		| project timestamp = timestamp, customDimensions_channel = channel, event_count

Totals:

	customEvents
		| where name == 'Activity'
		| extend channel = customDimensions['channel']
		| extend channel = tostring(iff(isnull(channel), customDimensions['Channel ID'], channel))
		| summarize event_count = count() by channel
		| project customDimensions_channel = channel, event_count

You can run these queries yourself by switching back to the Analytics tool in Application Insights. Then you can modify them and adapt them to your specific needs.

![Application Insights](https://i.stack.imgur.com/9WaVD.jpg)

One example of what you might want to accomplish is finding data related to a specific user ID. You may notice that the 10 data tables available to you (**traces**, **pageViews**, etc.) all have a **user_Id** column, but those columns are unused in the case of bots. Most of the tables are unused in this case as well. The only two tables where you should expect to find useful bot data are **customEvents** and **exceptions**. And of particular interest is the object-typed **customDimensions** column that stores a lot of bot-specific information in the **customEvents** table. We can see from the provided Bot Analytics queries that a user ID can be retrieved from the 'from' or 'From ID' field in a **customDimensions** object. Once you've retrieved such an ID you can query data related to that ID. It might look something like this:

	customEvents
		| extend from = customDimensions['from']
		| extend from = tostring(iff(isnull(from), customDimensions['From ID'], from))
		| extend activityType = customDimensions['Activity type']
		| where from == "some_user_id"

There are many possibilities of what you can achieve with these kinds of queries. Hopefully you will find yourself empowered to use Application Insights to suit your needs.

[//]: # (
[![Users][1]][1]
[![Activities][2]][2]
[![User retention][3]][3]
  [1]: https://i.stack.imgur.com/Z0deg.jpg
  [2]: https://i.stack.imgur.com/8TUnX.jpg
  [3]: https://i.stack.imgur.com/ynx5N.jpg
)

[//]: # (
Examples of who this blog is for:
https://stackoverflow.com/questions/53149454/in-azure-bot-analytics-blade-how-do-i-get-data-on-a-specific-user-id
)
