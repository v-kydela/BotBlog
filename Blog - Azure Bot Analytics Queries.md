# Bot Analytics: Behind the Scenes
##### by Kyle Delaney <div style="text-align: right">November 13, 2018</div>

[Application Insights](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-overview) is a great way to gather analytic information from all kinds of applications, including bots. An Application Insights resource in Azure provides an [Analytics tool](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-analytics) that allows you to run queries using the [Kusto Query Language](https://docs.microsoft.com/en-us/azure/kusto/query/tutorial), also known as Log Analytics Query Language. 

![Analytics](https://docs.microsoft.com/en-us/azure/application-insights/media/app-insights-analytics/001.png)

To get an idea of what queries you might want to run for a bot, you can switch away from the Application Insights resource and have a look at the [Analytics blade](https://docs.microsoft.com/en-us/azure/bot-service/bot-service-manage-analytics?view=azure-bot-service-4.0) in your bot resource (Web App Bot, Functions Bot, or Bot Channels Registration). That will show you the **retention**, **users**, and **activities** charts. (Note that the current analytics visuals are not reproducible using App Insights.)

## Retention

Each row in this chart shows data concerning the users who messaged the bot on a specific day. Each column shows how many users who messaged on that day returned to message the bot again a specific number of days later. For example, in this screenshot we can see that 43 users messaged the bot on April 25th, and 5% of those users returned to message the bot again 4 days later. 

![User retention](https://docs.microsoft.com/en-us/azure/bot-service/media/analytics-retention.png)

The chart shows data for the 10 days leading up to the day before yesterday, because that's the most recent day it can be determined that a user returned. For the users from yesterday, it is unknown whether they will return today or not.

For your benefit, here is the Kusto query used to generate the data for this chart:

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

## Users

This chart shows the total number of users who have messaged your bot on each channel, both for individual days and for all days combined. We can see these all-time grand totals on the bottom. Also on the bottom, each channel is given a color which represents that channel in the pie chart on the left. Those colors also represent the channels in the timeline on the right, where we can see the day-by-day changes in how many users messaged the bot on each channel on each day.

For example, in this screenshot we can see that 6 users total have messaged the bot on Web Chat, and 2 users have messaged the bot on each other channel. Web Chat is represented by the color orange in this case so the orange slice of the pie chart is 3 times the size of each of the other 5 slices. On the timeline, we can see that 2 users messaged the bot on Web Chat on October 29th.

![Users](https://i.stack.imgur.com/Z0deg.jpg)

For your benefit, here is the query used to generate the data for the timeline portion of this chart:

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

And here is the query used to generate the data for the grand totals at the bottom of the chart:

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

## Activities

This chart follows the same format as the previous chart. The difference is that this chart deals with the number of activities (messages, events, etc.) that were sent to the bot whereas the previous chart dealt with the number of users who were sending messages. It's expected that there will be many more activities than users. We can see in this screenshot that 70 total activities were sent to the bot through the Web Chat channel and 28 of those activities were on October 29th.

![Activities](https://i.stack.imgur.com/8TUnX.jpg)

For your benefit, here is the query used to generate the data for the timeline portion of this chart:

	customEvents
		| where name == 'Activity'
		| extend channel = customDimensions['channel']
		| extend channel = tostring(iff(isnull(channel), customDimensions['Channel ID'], channel))
		| summarize event_count = count() by bin(timestamp, 1d), channel
		| order by timestamp asc
		| project timestamp = timestamp, customDimensions_channel = channel, event_count

And here is the query used to generate the data for the grand totals at the bottom of the chart:

	customEvents
		| where name == 'Activity'
		| extend channel = customDimensions['channel']
		| extend channel = tostring(iff(isnull(channel), customDimensions['Channel ID'], channel))
		| summarize event_count = count() by channel
		| project customDimensions_channel = channel, event_count

## Running Queries

You can run these queries yourself by switching back to the Analytics tool in Application Insights. Then you can modify them and adapt them to your specific needs.

![Application Insights](https://i.stack.imgur.com/9WaVD.jpg)

One example of what you might want to accomplish is finding data related to a specific user ID. You may notice that the 10 data tables available to you (**traces**, **pageViews**, etc.) all have a **user_Id** column, but those columns are unused in the case of bots. Most of the tables are unused in this case as well. The only two tables where you should expect to find useful bot data are **customEvents** and **exceptions**. And of particular interest is the object-typed **customDimensions** column that stores a lot of bot-specific information in the **customEvents** table. We can see from the provided Bot Analytics queries that a user ID can be retrieved from the 'from' or 'From ID' field in a **customDimensions** object. Once you've retrieved such an ID you can query data related to that ID. It might look something like this:

	customEvents
		| extend from = customDimensions['from']
		| extend from = tostring(iff(isnull(from), customDimensions['From ID'], from))
		| extend activityType = customDimensions['Activity type']
		| where from == "some_user_id"

There are many possibilities of what you can achieve with these kinds of queries. Hopefully you will find yourself empowered to use Application Insights to suit your needs.

Happy Making!

Kyle Delaney and the Azure Bot Service Team

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
