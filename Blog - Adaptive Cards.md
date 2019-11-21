# Using Adaptive Cards with the Microsoft Bot Framework

![Adaptive Cards](https://user-images.githubusercontent.com/41968495/59643697-8781fc00-911e-11e9-8547-216b5c507f19.png)

## Introduction

[Adaptive Cards](https://adaptivecards.io/) are a way to present a self-contained UI to a user within a larger UI such as a chat client. They incorporate almost all of the functionality of the Bot Framework's [rich cards](https://github.com/microsoft/botframework-sdk/blob/master/specs/botframework-activity/botframework-cards.md), and they also provide some special functionality of their own. They are supported in Web Chat, Cortana, and Microsoft Teams, and can even be used outside of bots in [Windows Timeline](https://blogs.windows.com/windowsexperience/2017/12/19/announcing-windows-10-insider-preview-build-17063-pc/) and [Outlook Actionable Messages](https://docs.microsoft.com/en-us/outlook/actionable-messages/). You can also render Adaptive Cards in [your own applications](https://docs.microsoft.com/en-us/adaptive-cards/rendering-cards/getting-started). Adaptive Cards are designed to "adapt" to whatever environment they're being used in, because the host application is given ultimate control over presentation. If you're new to Adaptive Cards then this post should help you get started, and if you've already been using Adaptive Cards then this post might teach you a few new tricks.

### Schema

Every Adaptive Card can be represented as a JSON object that follows a specific schema. The schema defines everything that you can put in an Adaptive Card. Every host application that supports Adaptive Cards must be able to interpret the schema in a way that follows certain [rules](https://docs.microsoft.com/en-us/adaptive-cards/rendering-cards/implement-a-renderer), but the nature of Adaptive Cards still allows for a lot of flexibility and variation between applications. There is a handy [schema explorer](https://adaptivecards.io/explorer/) that lets you navigate through the schema's documentation like a language/API reference, and you can also read the schema's JSON directly if you want to be extra thorough.

![Schema Explorer](https://user-images.githubusercontent.com/41968495/59643251-cc0c9800-911c-11e9-9236-4e061e59ddb8.png)

There are currently two versions of the Adaptive Cards schema available: [1.1](https://github.com/microsoft/AdaptiveCards/blob/master/schemas/1.1.0/adaptive-card.json) and [1.2](https://github.com/microsoft/AdaptiveCards/blob/master/schemas/1.2.0/adaptive-card.json). The schemas are cumulative, which means each schema can be used for a card of a lower version because the elements in the schema are all explicit about the version they were introduced in. This means you can still use Adaptive Cards 1.0 even though its original schema is no longer publicly available. It should be noted that 1.0 still has the widest support among channels.

When specifying the schema for an Adaptive Card, you can specify a version-specific path like `https://adaptivecards.io/schemas/1.2.0/adaptive-card.json` or you can use the static path `https://adaptivecards.io/schemas/adaptive-card.json`. The static path originally contained the 1.0 schema but now contains the 1.1 schema, so it may be updated again in the future. Here's a small example of what an Adaptive Card might look like when represented as JSON, including the schema specification:

```json
{
  "$schema": "https://adaptivecards.io/schemas/1.1.0/adaptive-card.json",
  "type": "AdaptiveCard",
  "version": "1.0",
  "body": [
    {
      "type": "TextBlock",
      "text": "Example card"
    }
  ]
}
```

If you want to see a preview of what an Adaptive Card's JSON will look like when rendered then you can copy and paste the JSON right into the [Adaptive Card designer](https://adaptivecards.io/designer/).

![Designer](https://user-images.githubusercontent.com/41968495/59644777-450eee00-9123-11e9-92ab-8f52f4a5d291.png)

You can find more samples of Adaptive Cards [here](https://adaptivecards.io/samples/).

### Packages

Because Adaptive Cards can be represented as JSON strings and sent as HTTP content using the [Bot Framework REST API](https://docs.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-overview), they can be incorporated into a bot built in any programming language. However, there are some optional packages available that can make bot development easier by providing classes that help you build and manipulate your Adaptive Cards programmatically.

If you're building a bot in C# then you'll want the [AdaptiveCards](https://www.nuget.org/packages/AdaptiveCards/) NuGet package. Do *not* use the older [Microsoft.AdaptiveCards](https://www.nuget.org/packages/Microsoft.AdaptiveCards/) package because it's deprecated. Similarly, if you're building a bot in Node.js then you will want [adaptivecards](https://www.npmjs.com/package/adaptivecards) and not [microsoft-adaptivecards](https://www.npmjs.com/package/microsoft-adaptivecards). Note that much of the code in those packages pertains to rendering Adaptive Cards on the client side, but the included types will still help on the bot side. For example, with the types contained in the NuGet package you can construct an Adaptive Card like this rather than just relying on JSON:

```c#
var card = new AdaptiveCard(new AdaptiveSchemaVersion(1, 0))
{
    Body = new List<AdaptiveElement>()
    {
        new AdaptiveTextBlock("Example card"),
    },
};
```

### Body & Actions

The body of an Adaptive Card is where you can display information and gather input from the user. You can use containers to help define the layout for the card elements in the body. A card element is a component of an Adaptive Card that you put in the body like a text block for example. The full range of available elements can be seen in the schema.

A card action is something the card can do when the user clicks on it. Unlike other rich cards, Adaptive Cards originally only supported three action types: `OpenUrl`, `ShowCard`, and `Submit`. Adaptive Cards 1.2 introduced a fourth action: `ToggleVisibility`.

Card actions can be used in the card's body when an element has a `selectAction` property. Adaptive Cards also have an `actions` list that is separate from the body, and those actions will be displayed as buttons in a way that conforms to the style of each channel client. Adaptive Cards 1.2 introduced the ability for the card author to specify where and how they want actions to appear by including an `ActionSet`.

## Tips & Tricks

While Adaptive Cards have not been designed specifically for the Bot Framework, they are very useful for bots and there are some important things to know about when using them.

### Submit Actions

If you look at [`Action.Submit`](https://adaptivecards.io/explorer/Action.Submit.html) in the schema, you will see that a submit action's `data` property can be either a string or an object. A submit action has two possible behaviors that correspond to these two possible data types. If you use a string as a submit action's `data` property then the submit action will behave in a way that can be referred to as the "string" submit action behavior. If you use an object as a submit action's `data` property or if you omit the `data` property then the submit action will behave in a way that can be referred to as the "object" submit action behavior.

A string submit action automatically sends a message from the user to the bot that shows up in the client application's conversation history as though the user typed the message and sent it manually. An object submit action automatically sends an invisible message from the user to the bot that contains hidden data. (To draw an analogy to the actions of other types of rich cards like hero cards and receipt cards, string submit actions and object submit actions roughly correspond to the card actions known as `imBack` and `postBack` respectively.) Here's what it might look like when you click on a string submit action:

![imBack](https://user-images.githubusercontent.com/41968495/60307051-bdca3300-98f7-11e9-89d6-5197cf4561ab.png)

The JSON for that card might look like this (notice the string assigned to the submit action's `data` property):

```json
{
  "$schema": "https://adaptivecards.io/schemas/adaptive-card.json",
  "type": "AdaptiveCard",
  "version": "1.0",
  "actions": [
    {
      "type": "Action.Submit",
      "title": "Click here to say \"Hello\"",
      "data": "Hello"
    }
  ]
}
```

The reason behind the name "submit action" becomes clear when an Adaptive Card contains input fields. A submit action is used to "submit" the information provided in the input fields by combining that information with anything already in the `data` property and sending it to the bot. Because a string cannot have properties added to it, you cannot use a string submit action in a card that contains input fields. If the submit action tries to combine the input fields with the `data` property and the `data` property is a string then the action will fail.

When using the Bot Framework, clicking on a submit action will send a Bot Framework message activity to the bot. A string submit action will simply transfer its string data into the activity's `text` property. An object submit action has a somewhat more complicated process of populating the activity's `value` property while leaving the `text` property empty. Each input field in the object submit action's card will be represented as a property of that `value` object and the properties will be named according to each input field's ID. Any properties already in the object submit action's `data` property will also be present in the `value` object. For example, you might have an Adaptive Card like this:

```json
{
  "$schema": "https://adaptivecards.io/schemas/adaptive-card.json",
  "type": "AdaptiveCard",
  "version": "1.0",
  "body": [
    {
      "type": "Input.Text",
      "id": "id_text"
    },
    {
      "type": "Input.Number",
      "id": "id_number"
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "Submit",
      "data": {
        "prop1": true,
        "prop2": []
      }
    }
  ]
}
```

There are two input fields for the user to fill out:

![Adaptive Card with two input fields](https://user-images.githubusercontent.com/41968495/60207563-11128780-980b-11e9-9749-98db21f96d17.png)

When the user clicks the submit button, an invisible message activity will be sent from the user to the bot. That activity will have the following object as its `value` property:

```json
{
  "id_number": "30",
  "id_text": "Kyle",
  "prop1": true,
  "prop2": []
}
```

Notice how both of the card's input fields are included in this object in addition to the two properties that were already present in the submit action's JSON. Also notice that the number input is represented as a string. You should expect all input fields to be represented as strings when their data is sent to the bot, regardless of what type of data they're meant to represent. Always make sure to test your cards so you know what type conversions you need to make.

Because an object submit action generates an activity of type `message`, you will need a way to distinguish such an activity from an ordinary text-based message activity like the kind that gets sent when the user types something into the chat client. In most cases it's enough to recognize an object submit action's activity by checking if the `value` property is populated and the `text` property isn't. If necessary, you can perform additional checks by seeing if the data inside the `value` object meets your expectations. However, there is usually no way for a bot to distinguish between a message from a string submit action and a message that the user typed into the chat client. This is by design because those messages should be treated the same way.

Taking all of that into consideration, in C# you might retrieve a number from an activity's `value` property like this:

```c#
var txt = turnContext.Activity.Text;

dynamic val = turnContext.Activity.Value;

// Check if the activity came from a submit action
if (string.IsNullOrEmpty(txt) && val != null)
{
    // Retrieve the data from the id_number field
    var num = double.Parse(val.id_number);

    // . . .
}
```

In JavaScript you might do it like this:

```js
var txt = turnContext.activity.text;
var val = turnContext.activity.value;

// Check if the activity came from a submit action
if (!txt && val) {
    // Retrieve the data from the id_number field
    var num = parseFloat(val.id_number);

    // . . .
}
```

Other types of input can be retrieved in a similar fashion.

### Adaptive Cards in Dialogs

The Dialogs library is an essential part of the Bot Framework. While dialogs are typically meant to flow in such a way where each message pertains to the message that came before it, "interruptions" can occur where the user brings up something that doesn't fit into the sequential flow of the dialog and the bot has to figure out the best way to handle that. The way cards behave may seem ill-suited for dialogs because cards are designed to persist in the conversation history so that a user may interact with an old card even when the card has become irrelevant to the current progress of the conversation. This is true for cards in general and not just Adaptive Cards.

Many channels support some form of "suggested actions" which remedy the card problem by showing the buttons only for one turn of the conversation. However, if you use cards appropriately then there may be no reason to think of this as a problem at all. Sometimes you'll want to allow users to click on old cards from a previous point in the conversation, and in the situations when you don't want those old cards to do anything then your bot can choose to ignore them.

Prompts are a very common form of dialog. A prompt allows for validation and automatic type conversion of user input. With a prompt, the dialog will not proceed until the user provides the right kind of information. If you're using an Adaptive Card in a dialog then it's likely that you'll want to use a prompt.

To include any kind of card in a prompt, you can just attach it to the activity that's getting sent in the prompt. For example, you can use an Adaptive Card in a choice prompt in a waterfall step in C# like this:

```c#
private async Task<DialogTurnResult> PromptWithAdaptiveCardAsync(
    WaterfallStepContext stepContext,
    CancellationToken cancellationToken)
{
    // Define choices
    var choices = new[] { "One", "Two", "Three" };

    // Create card
    var card = new AdaptiveCard(new AdaptiveSchemaVersion(1, 0))
    {
        // Use LINQ to turn the choices into submit actions
        Actions = choices.Select(choice => new AdaptiveSubmitAction
        {
            Title = choice,
            Data = choice,  // This will be a string
        }).ToList<AdaptiveAction>(),
    };

    // Prompt
    return await stepContext.PromptAsync(
        CHOICEPROMPT,
        new PromptOptions
        {
            Prompt = (Activity)MessageFactory.Attachment(new Attachment
            {
                ContentType = AdaptiveCard.ContentType,
                // Convert the AdaptiveCard to a JObject
                Content = JObject.FromObject(card),
            }),
            Choices = ChoiceFactory.ToChoices(choices),
            // Don't render the choices outside the card
            Style = ListStyle.None,
        },
        cancellationToken);
}
```

The card (along with the message that gets sent when the user clicks on a choice) will look something like this:

![Choice prompt](https://user-images.githubusercontent.com/41968495/60306896-151bd380-98f7-11e9-98bf-b0d1e02b6ed3.png)

There are a few important things to note about what we're doing here. The reason we're converting the card to a `JObject` is that the Bot Builder SDK preserves type data when it serializes objects into messages. This can result in sending a lot of extra information that we don't care about sending since all we really need to send is a raw JavaScript object that fits the Adaptive Cards schema. The extra type information can even lead to exceptions if the classes you're serializing have custom serialization/deserialization functionality. Converting the card object to a `JObject` is our way of saying we just want a normal raw JavaScript object and its properties without any special type information associated with it.

Also note that we're setting `Style` as `ListStyle.None`. Depending on the list style you've chosen, a choice prompt can automatically display the prompt's choices for you. For example, if the style is "inline" or "list" then the choices will be appended right onto the text of the activity. We don't want that to happen here because we've already included the choices in the card itself, and that's why we're setting "none" as the list style.

If you want to incorporate Adaptive Card input fields into your prompt, you will only be able to use object submit actions and not string submit actions (because of the limitations discussed in the Submit Actions section). Most prompts operate based on an activity's `text` property, and you may recall that `text` will be empty in the case of an object submit action. The trick you can use to have a prompt accept a value from a textless activity is to set the text property yourself before continuing the dialog. If you have multiple input fields, you can combine them into the text property by serializing them into a JSON string. It only makes sense to use a text prompt in this case.

After you assign your own value to the text property of an incoming activity, you will have effectively modified that activity so that a prompt will act as though the user entered that information. As long as that modified activity remains in the turn context, anything you do with the turn context will use that modified activity. So you can assign that turn context to a dialog context and then continue the active dialog and the dialog will use whatever you put in the text property. In C# it might look like this:

```c#
private async Task SendValueToDialogAsync(
    ITurnContext turnContext,
    CancellationToken cancellationToken)
{
    // Serialize value
    var json = JsonConvert.SerializeObject(turnContext.Activity.Value);
    // Assign the serialized value to the turn context's activity
    turnContext.Activity.Text = json;
    // Create a dialog context
    var dc = await _dialogSet.CreateContextAsync(
        turnContext, cancellationToken);
    // Continue the dialog with the modified activity
    await dc.ContinueDialogAsync(cancellationToken);
}
```

If the active dialog is a text prompt, that text prompt will return the information that came from an Adaptive Card's submit action.

### Adaptive Cards in Teams

[There is some special Adaptive Card functionality that is specific to the Microsoft Teams channel.](https://docs.microsoft.com/en-us/microsoftteams/platform/task-modules-and-cards/cards/cards-actions#adaptive-cards-actions) You can supply a special property with the name `msteams` to the object in an object submit action's `data` property in order to access this functionality. By doing so, you can get a submit action to behave like an action of your choosing. The object you put in the `msteams` property must have a `type` property in order to specify the submit action's special behavior.

When the `type` property is ["messageBack"](https://docs.microsoft.com/en-us/microsoftteams/platform/task-modules-and-cards/cards/cards-actions#adaptive-cards-with-messageback-action) the submit action will behave like a `messageBack` card action, which is like a combination of `imBack` and `postBack`. This means you can specify both visible text that will be displayed in the conversation history as well as invisible data that gets sent to the bot behind the scenes.

```json
{
  "type": "Action.Submit",
  "title": "Click me for messageBack",
  "data": {
    "msteams": {
        "type": "messageBack",
        "displayText": "I clicked this button",
        "text": "text to bots",
        "value": "{\"bfKey\": \"bfVal\", \"conflictKey\": \"from value\"}"
    },
    "extraData": {}
  }
}
```

The `displayText` string is what gets displayed in the conversation history. The `text` string will populate the activity's `text` property but will be invisible to the user. The activity's `value` property will get populated in the usual way by combining the values of any input fields with any additional properties in the `data` property's object, but it will also include any properties that have been serialized into the `value` property of the `msteams` property's object.

When the `type` property is ["imBack"](https://docs.microsoft.com/en-us/microsoftteams/platform/task-modules-and-cards/cards/cards-actions#adaptive-cards-with-imback-action) the submit action will behave like a `imBack` card action. This of course will work similarly to a string submit action but it has the advantage of not breaking when the card contains input fields, though the input fields' values will not be sent to bot when the user clicks this action and neither will any additional properties you put in the `data` property's object. Also, Microsoft Teams does not support string submit actions at the time of this writing, so you will need to use this special Teams feature if you want to simulate string submit actions.

```json
{
  "type": "Action.Submit",
  "title": "Click me for imBack",
  "data": {
    "msteams": {
        "type": "imBack",
        "value": "Text to reply in chat"
    },
    "extraData": "(this will be ignored)"
  }
}
```

When the `type` property is ["signin"](https://docs.microsoft.com/en-us/microsoftteams/platform/task-modules-and-cards/cards/cards-actions#adaptive-cards-with-signin-action) the submit action will behave like a `signin` card action. This may not be necessary in most cases because `signin` actions are normally used in very simple cards, so it might make more sense to use a [signin card](https://docs.microsoft.com/en-us/microsoftteams/platform/task-modules-and-cards/cards/cards-reference#signin-card) than an Adaptive Card. All the same, if you'd like to include a `signin` action in an Adaptive Card in Microsoft Teams, just put the sign-in URL in the `value` property of the `msteams` property's object.

```json
{
  "type": "Action.Submit",
  "title": "Click me for signin",
  "data": {
    "msteams": {
        "type": "signin",
        "value": "https://yoursigninurl.com/signinpath?parames=values",
    },
    "extraData": "(this will be ignored)"
  }
}
```

## Conclusion

Adaptive Cards can do many things and serve many purposes. New features have lately been added to Adaptive Cards and will likely continue to be added in the future. It's also likely that more applications will adopt greater support for Adaptive Cards as they gain popularity. With the information in this post I hope that you will be able to expand the way you use Adaptive Cards, especially when you incorporate them into your bots.

Happy Making!

Kyle Delaney and the Azure Bot Service Team
