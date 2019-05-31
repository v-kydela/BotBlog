# Using the Facebook Messenger Handover Protocol with the Microsoft Bot Framework

The Facebook Messenger Platform offers a [handover protocol](https://developers.facebook.com/docs/messenger-platform/handover-protocol) that allows your Facebook page to pass "thread control" between primary and secondary receivers. A thread can be thought of like a Facebook Messenger conversation in this case, and the idea of thread control is that one receiver is the "thread owner" that receives all messages and can reply to them or pass control to other receivers. A receiver could be a Facebook app (like the kind your bot communicates through) or it could be the page inbox. This means the end user could be talking to one bot and then in the same conversation could start talking to a different bot instead, or to a human. Even though the conversation remains between one Facebook user and one Facebook page, the page may have multiple receivers to choose from that can listen and respond on its behalf. This is useful if a user wants to escalate the conversation by talking to a live support agent instead of a bot.

![Sample conversation](https://i.stack.imgur.com/JnP0z.png)

If you want to use the Messenger Platform's handover protocol, you can choose your own combination of receivers. That is to say you can use the handover protocol between a bot and the page inbox, or between multiple bots, or between multiple bots and the page inbox. If you're wondering why you would want to do a handover between two bots, consider a situation where a company uses a third-party proprietary bot service that they don't have direct access to, and so they want to use their own bot in addition to a bot made by another company. It's a handy thing to be able to do.

Most of the information in this post can apply to either bot-to-human or bot-to-bot handoffs, and all the information pertaining to the Messenger Platform's API can be applied whether or not you're using the Microsoft Bot Framework, though the code samples use the Bot Builder SDK. [The full sample can be downloaded here.](https://github.com/microsoft/BotFramework-Samples/tree/master/blog-samples/CSharp/FacebookHandover) It's based on Eric Dahlvang's [FacebookEventsBotExpanded](https://github.com/EricDahlvang/FacebookEventsBotExpanded), which is based on the official [Facebook events Bot Builder sample](https://github.com/Microsoft/BotBuilder-Samples/tree/master/samples/csharp_dotnetcore/23.facebook-events).

## Setting up the handover protocol

You will want to follow the instructions for [connecting a bot to Facebook](https://docs.microsoft.com/en-us/azure/bot-service/bot-service-channel-connect-facebook), and you may want to perform those steps more than once in the case where you want to enable a handover between multiple bots.

### Webhooks

In [Facebook for Developers](https://developers.facebook.com/), you will subscribe to webhook events as part of the process of setting up the Messenger section of each Facebook app you want a bot to connect to. In addition to any webhook events you already wanted to subscribe to, for the sake of the handover protocol it's important to subscribe to `standby` and `messaging_handovers` in particular. Each box you check pertains to a specific set of events that Facebook will send to the bot connector, which the bot connector converts to Bot Framework activities before sending to your bot.

![Webhook configuration](https://user-images.githubusercontent.com/41968495/58438151-3da77800-8082-11e9-9340-24809a63f516.png)

Checking `standby` allows your bot to listen to messages that get sent to other receivers when it doesn't have thread control. This is useful if a bot wants to take or request thread control in response to a specific message being sent to a different receiver. Checking `messaging_handovers` means your bot will receive `pass_thread_control`, `take_thread_control`, and `request_thread_control` events, which will be discussed later.

There's nothing stopping you from checking all the boxes, but that may result in your bot receiving more events than you want to get. You can have a look at the [webhook events reference](https://developers.facebook.com/docs/messenger-platform/reference/webhook-events) to learn what each subscription does so that you can decide what to check.

In addition to choosing which webhook events to subscribe to, don't forget to select a page.

![Select a Page](https://user-images.githubusercontent.com/41968495/58440955-e6f66a00-8092-11e9-88f5-dff69804fa5b.png)

### App roles

Once you've selected a page to receive webhook events from in Facebook for Developers, go back to the main [Facebook](https://www.facebook.com/) site. When you view your page's settings, you will see your app show up in the Messenger Platform tab. If you've followed the steps for more than one Facebook app then you should see them all.

![Connected apps](https://user-images.githubusercontent.com/41968495/58441155-6c2e4e80-8094-11e9-9265-35de2618371f.png)

You can click the "Configure" button next to "App settings" to choose which app is the primary receiver and which apps are secondary receivers.

![App settings](https://user-images.githubusercontent.com/41968495/58441056-a9dea780-8093-11e9-872c-cfbff6524cac.png)

You can select the page inbox only as a secondary receiver and not the primary receiver. If you already have a primary receiver selected and you select a different one, the old primary receiver automatically becomes a secondary receiver.

The primary receiver has special privileges that secondary receivers don't. Think of the primary receiver as a sort of administrator app that manages the handover protocol and has the final say in which app gets thread control. The primary receiver is also the default thread owner before thread control gets passed around. If you don't configure any primary or secondary receivers and you've connected multiple apps to your page's Messenger inbox, you can't use the handover protocol and the page will determine which app to relay messages to based on the order that you've generated each app's current page token.

## Using the handover protocol

The Messenger Platform handover protocol includes an [API](https://developers.facebook.com/docs/messenger-platform/reference/handover-protocol) that your bot can call. There are three main actions you can perform: `pass_thread_control`, `take_thread_control`, and `request_thread_control`. If the API is called successfully, an event will be sent to the app specified as the target of the action. For `pass_thread_control` and `take_thread_control`, the bot will receive the event as a `conversationUpdate` activity. For `request_thread_control`, the bot will receive the event as an `event` activity.

Before implementing the API calls in your bot, it can be helpful to experiment using cURL or any other HTTP request tool. In order to verify that your API calls are indeed changing the thread owner, you can call the [Get Thread Owner](https://developers.facebook.com/docs/messenger-platform/handover-protocol/get-thread-owner) API.

Note that in order to call the API you will need to use the page token you generated when connecting your bot to Facebook. If you generate a new page token the old one will still work so don't worry about having to update everything. The page token allows an app to perform actions and communicate on behalf of a page, and it also allows the page to identify what app is performing those actions. When you call the API, you are calling the API on behalf of a specific app, and so the page token determines whether the app you're using to call the API has permission to perform whatever action you're trying to perform. It can be helpful to store the page token in your app settings alongside your Microsoft app ID and password like this:

```json
{
  "MicrosoftAppId": "",
  "MicrosoftAppPassword": "",
  "FacebookPageToken": ""
}
```

Each API call has a common base URL so it's a good idea to use a helper function to handle the code that's common to all three of them. In C# it might look like this:

```c#
public const string GRAPH_API_BASE_URL = "https://graph.facebook.com/v3.3/me/{0}?access_token={1}";

private static readonly HttpClient _httpClient = new HttpClient();

private static async Task<bool> PostToFacebookAPIAsync(string postType, string pageToken, string content)
{
    var requestPath = string.Format(GRAPH_API_BASE_URL, postType, pageToken);
    var stringContent = new StringContent(content, Encoding.UTF8, "application/json");

    // Create HTTP transport objects
    using (var requestMessage = new HttpRequestMessage())
    {
        requestMessage.Method = new HttpMethod("POST");
        requestMessage.RequestUri = new Uri(requestPath);
        requestMessage.Content = stringContent;
        requestMessage.Content.Headers.ContentType = System.Net.Http.Headers.MediaTypeHeaderValue.Parse("application/json; charset=utf-8");

        // Make the Http call
        using (var response = await _httpClient.SendAsync(requestMessage, CancellationToken.None).ConfigureAwait(false))
        {
            // Return true if the call was successfull
            Debug.Print(await response.Content.ReadAsStringAsync().ConfigureAwait(false));
            return response.IsSuccessStatusCode;
        }
    }
}

```

This method takes three parameters. `postType` is the specific API to call and it distinguishes between the three actions you can perform, `pageToken` is of course the Facebook page token that you need to pass to the API, and `content` is JSON data that will be used as the body of the HTTP request. The content always contains a recipient which represents the Facebook user who is communicating with the page, and that allows the API to identify the conversation to perform the operation on.

### Passing thread control

Either the primary receiver or a secondary receiver can [pass thread control](https://developers.facebook.com/docs/messenger-platform/handover-protocol/pass-thread-control) as long as that app is currently the thread owner. Using the `PostToFacebookAPIAsync` helper method, you can call the `pass_thread_control` API in C# like this:

```c#
public static async Task<bool> PassThreadControlAsync(string pageToken, string targetAppId, string userId, string message)
{
    var content = new { recipient = new { id = userId }, target_app_id = targetAppId, metadata = message };
    return await PostToFacebookAPIAsync("pass_thread_control", pageToken, JsonConvert.SerializeObject(content)).ConfigureAwait(false);
}
```

If you're calling this method on behalf of the primary receiver then use `target_app_id` to specify the Facebook app ID of the secondary receiver you want to pass thread control to.

The ID used to represent the page inbox is constant in that it doesn't vary between Facebook pages. It will always be 263902037430900:

```c#
private const string PAGE_INBOX_ID = "263902037430900";
```

So you can pass thread control to the page inbox with your `PassThreadControlAsync` method like this:

```c#
await FacebookThreadControlHelper.PassThreadControlAsync(_configuration["FacebookPageToken"], PAGE_INBOX_ID, turnContext.Activity.From.Id, text);
```

You can use the [`Get Secondary Receivers`](https://developers.facebook.com/docs/messenger-platform/handover-protocol#secondary_receivers_list) API to get the Facebook app ID for all the secondary receivers connected to your page. This can eliminate the need to store the secondary receiver's ID in your primary receiver's app settings. Only the primary receiver can access the list of secondary receivers, so if you try to use a page token from a secondary receiver you will get an error. You can use the `fields` parameter in the URL to specify what information you want to get in the response, and if you omit the `fields` parameter you will get all possible fields: `link`, `name`, and `id`. You can call the API in C# like this:

```c#
public static async Task<List<string>> GetSecondaryReceiversAsync(string pageToken)
{
    var requestPath = string.Format(GRAPH_API_BASE_URL, "secondary_receivers", pageToken);

    // Create HTTP transport objects
    using (var requestMessage = new HttpRequestMessage())
    {
        requestMessage.Method = new HttpMethod("GET");
        requestMessage.RequestUri = new Uri(requestPath);

        // Make the Http call
        using (var response = await _httpClient.SendAsync(requestMessage, CancellationToken.None).ConfigureAwait(false))
        {
            // Interpret response
            var responseString = await response.Content.ReadAsStringAsync().ConfigureAwait(false);
            var responseObject = JObject.Parse(responseString);
            var responseData = responseObject["data"] as JArray;

            return responseData.Select(receiver => receiver["id"].ToString()).ToList();
        }
    }
}
```

This helper method returns a list of strings representing each secondary receiver Facebook app ID. If you have multiple bots as secondary receivers then you may want to return their names as well to help you choose between them. If you want to pass thread control to a secondary receiver bot and there aren't any other secondary receivers other than the page inbox, you can easily identify the bot's Facebook app ID because it will be the only one that isn't the page inbox ID. You can use `GetSecondaryReceiversAsync` along with `PassThreadControlAsync` like this:

```c#
var secondaryReceivers = await FacebookThreadControlHelper.GetSecondaryReceiversAsync(_configuration["FacebookPageToken"]);

foreach (var receiver in secondaryReceivers)
{
    if (receiver != PAGE_INBOX_ID)
    {
        await turnContext.SendActivityAsync($"Primary Bot: Passing thread control to {receiver}...");
        await FacebookThreadControlHelper.PassThreadControlAsync(_configuration["FacebookPageToken"], receiver, turnContext.Activity.From.Id, text);
        break;
    }
}
```

Secondary receivers are effectively unaware of each other and cannot pass thread control to other secondary receivers. Since a secondary receiver can only pass thread control to the primary receiver, `target_app_id` is ignored when calling the `pass_thread_control` API on behalf of a secondary receiver. This eliminates the need to store the primary receiver's Facebook app ID in your secondary receiver's app settings. You can use `PassThreadControlAsync` to pass thread control from a secondary receiver to the primary receiver like this:

```c#
// A null target app ID will automatically pass control to the primary receiver
await FacebookThreadControlHelper.PassThreadControlAsync(_configuration["FacebookPageToken"], null, turnContext.Activity.From.Id, text);
```

If the app that thread control gets passed to is subscribed to `messaging_handovers` webhook events, it will receive a `conversationUpdate` activity. The Facebook payload defined in the [documentation](https://developers.facebook.com/docs/messenger-platform/handover-protocol/pass-thread-control#example_response) will be stored in the activity's channel data. You can check the channel data for a `pass_thread_control` field in order to recognize it as a `pass_thread_control` event.

### Taking thread control

While any receiver can pass thread control, only the primary receiver can [take thread control](https://developers.facebook.com/docs/messenger-platform/handover-protocol/take-thread-control). This makes sense when you think in terms of it requiring more authority to take something than to give it away. If you're the thread owner then thread control is yours to do with as you please, but if you're not the thread owner then you'd have to take something that isn't yours and that is a special privilege. Using the `PostToFacebookAPIAsync` helper method, you can call the `take_thread_control` API in C# like this:

```c#
public static async Task<bool> TakeThreadControlAsync(string pageToken, string userId, string message)
{
    var content = new { recipient = new { id = userId }, metadata = message };
    return await PostToFacebookAPIAsync("take_thread_control", pageToken, JsonConvert.SerializeObject(content)).ConfigureAwait(false);
}
```

There is no need to specify any target app ID in this case because there's only one possible target app for this API. The primary receiver can call this API to ensure that it has thread control even if it's already the thread owner, and that won't result in an error.

In order to know when to take thread control, it can be helpful to listen for [`standby`](https://developers.facebook.com/docs/messenger-platform/reference/webhook-events/standby) events. When a user messages your page, a `standby` event will be sent to each receiver that doesn't have thread control, so long as they've subscribed to those events. The bot connector will turn a `standby` event into a Bot Framework event activity and store the Facebook payload as the activity's value.

Before thread control changes, if the thread owner is subscribed to `messaging_handovers` webhook events then it will receive a `conversationUpdate` activity. The Facebook payload defined in the [documentation](https://developers.facebook.com/docs/messenger-platform/handover-protocol/take-thread-control#example-messaging-handovers-event) will be stored in the activity's channel data. You can check the channel data for a `take_thread_control` field in order to recognize it as a `take_thread_control` event.

### Requesting thread control

While the primary receiver can take thread control, a secondary receiver must [request thread control](https://developers.facebook.com/docs/messenger-platform/handover-protocol/request-thread-control) instead. This means that a secondary receiver can notify the primary receiver that it wants thread control, but the primary receiver must respond to such a notification by passing thread control in order for the secondary receiver to actually become the thread owner. Since the primary receiver would have no need to request thread control, only a secondary receiver is allowed to do it. Using the `PostToFacebookAPIAsync` helper method, you can call the `request_thread_control` API in C# like this:

```c#
public static async Task<bool> RequestThreadControlAsync(string pageToken, string userId, string message)
{
    var content = new { recipient = new { id = userId }, metadata = message };
    return await PostToFacebookAPIAsync("request_thread_control", pageToken, JsonConvert.SerializeObject(content)).ConfigureAwait(false);
}
```

There is no need to specify any target app ID in this case because the app that's requesting thread control is determined by the page token. If the page token is associated with an app that already has thread control then an error will be returned.

In order to know when to request thread control, it can be helpful to listen for [`standby`](https://developers.facebook.com/docs/messenger-platform/reference/webhook-events/standby) events. When a user messages your page, a `standby` event will be sent to each receiver that doesn't have thread control, so long as they've subscribed to those events. The bot connector will turn a `standby` event into a Bot Framework event activity and store the Facebook payload as the activity's value.

If the primary receiver is subscribed to `messaging_handovers` webhook events, it will receive a `request_thread_control` event activity. The Facebook payload defined in the [documentation](https://developers.facebook.com/docs/messenger-platform/handover-protocol/request-thread-control#example-messaging-handovers-event) will be stored in the activity's value. When responding to that event activity, your bot can decide whether or not to pass control. In C# it might look like this:

```c#
string requestedOwnerAppId = facebookPayload.RequestThreadControl.RequestedOwnerAppId;

if (facebookPayload.RequestThreadControl.Metadata == "please")
{
    await turnContext.SendActivityAsync($"Primary Bot: {requestedOwnerAppId} requested thread control nicely. Passing thread control...");

    var success = await FacebookThreadControlHelper.PassThreadControlAsync(
        _configuration["FacebookPageToken"],
        requestedOwnerAppId,
        facebookPayload.Sender.Id,
        "allowing thread control");

    if (!success)
    {
        // Account for situations when the primary receiver doesn't have thread control
        await turnContext.SendActivityAsync("Primary Bot: Thread control could not be passed.");
    }
}
else
{
    await turnContext.SendActivityAsync($"Primary Bot: {requestedOwnerAppId} requested thread control but did not ask nicely."
        + " Thread control will not be passed."
        + " Send any message to continue.");
}
```

That code uses the `metadata` field to decide whether to pass thread control or not, but your bot can use whatever criteria you like. You can even decide to pass thread control every time it's requested.

You can also decide what to do if thread control is requested but the primary receiver can't pass thread control because it's not the thread owner. The standard option is to take thread control before passing it, but this sample code just tries to pass thread control and lets the user know if it couldn't be passed.

Notice that the code is using the turn context to send activities even though it's responding to an event activity. The turn context uses information from the `conversation`, `from`, and `recipient` properties of its `activity` property to send activities to the conversation, so if that data is missing then it won't be able to send activities. However, in this case the event activity contains all the necessary information in its `value` property, so you can use that to populate the `conversation`, `from`, and `recipient` properties yourself.

For the Bot Framework's Facebook channel, the ID associated with the bot is the page's ID, and the ID associated with the user is something called a [page-scoped ID](https://developers.facebook.com/docs/messenger-platform/identity/id-matching/). Both of these values can be retrieved from a handover protocol event. The page ID will be in the `recipient` field and the user ID will be in the `sender` field. A conversation ID is actually composed of both the the user ID and the page ID, with a hyphen in the middle. So if you want to use the turn context to send messages in response to a `request_thread_control` event, you can modify the turn context's activity to include the necessary information. In C# you could do it like this:

```c#
/// <summary>
/// This extension method populates a turn context's activity with conversation and user information from a Facebook payload.
/// This is necessary because a turn context needs that information to send messages to a conversation,
/// and event activities don't necessarily come with that information already in place.
/// </summary>
public static void ApplyFacebookPayload(this ITurnContext turnContext, FacebookPayload facebookPayload)
{
    var userId = facebookPayload.Sender.Id;
    var pageId = facebookPayload.Recipient.Id;
    var conversationId = string.Format("{0}-{1}", userId, pageId);

    turnContext.Activity.From = new ChannelAccount(userId);
    turnContext.Activity.Recipient = new ChannelAccount(pageId);
    turnContext.Activity.Conversation = new ConversationAccount(id: conversationId);
}
```

## The handover protocol user interface

Thread control is interconnected with the folders in a page's inbox. When you pass thread control to the page inbox, the conversation gets moved to the "Main" folder. When the page inbox has thread control, whoever is controlling the page's inbox can click the "Mark as done" button not only to send the conversation to the "Done" folder but also to simultaneously pass thread control to the primary receiver, triggering a `pass_thread_control` event with the metadata "Pass thread control from Page Inbox".

![image](https://user-images.githubusercontent.com/41968495/58518056-89315300-8162-11e9-8804-3bb8384f16c4.png)

When the page inbox doesn't have thread control, you can click the "Move to Main" button to send the conversation to the Main folder and simultaneously trigger a `request_thread_control` event without metadata.

![image](https://user-images.githubusercontent.com/41968495/58518110-b54cd400-8162-11e9-93cd-d0d3d499d161.png)

Of course, because the page inbox can only be a secondary receiver, this means anyone controlling the page inbox can only obtain thread control if the primary receiver passes it to them. Therefore, you can have a conversation in the Main folder even when a bot still has thread control. If the primary receiver takes thread control from the page inbox then it will be sent a `pass_thread_control` event even though thread control is being taken rather than passed. And once the user starts talking to the bot, the conversation will automatically be moved into the Done folder.

## Conclusion

The handover protocol provided by the Facebook Messenger Platform offers useful functionality for escalating conversations and otherwise passing control between receivers. Thanks to recent upgrades to the Microsoft Bot Framework's Facebook connector, bot developers now have access to webhook events that unlock the protocol's potential. Hopefully you will find yourself empowered to use the handover protocol to suit your needs.

Happy Making!

Kyle Delaney and the Azure Bot Service Team
