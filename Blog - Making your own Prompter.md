# FormFlow: Making your own Prompter
##### by Kyle Delaney

## Overview

[FormFlow](https://docs.microsoft.com/en-us/azure/bot-service/dotnet/bot-builder-dotnet-formflow) is one of the fastest ways to create a dialog in the [Microsoft Bot Framework](https://dev.botframework.com/) (version 3.x). All you need is a data structure, and FormFlow can turn that data structure into a dialog automatically using [.NET reflection](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/reflection). However, sometimes you'll want less automation and more control, such as specifying a specific prompt for a specific field. And when it comes to control, giving your FormFlow dialog your own custom prompter is perhaps the ultimate level of it. In this blog post, you will learn how to leverage a little-known tool to achieve greater flexibility and customizability than you ever could without it.

## What is a prompter?

A form can be built from a combination of several types of components, and if certain components are missing then defaults are automatically provided. One component that is almost always defaulted is called the **prompter**. It's a delegate that controls how prompts are generated and sent to the user.

You are probably familiar with using the [**`Field()`**](https://docs.microsoft.com/en-us/dotnet/api/microsoft.bot.builder.formflow.formbuilder-1.field) methods to specify **prompts** for specific fields in your form. Specifying a prompter is different. When you specify a prompt you're just choosing what text gets displayed, but when you specify a prompter you get to dynamically perform any action at the time when a prompt is meant to be displayed, or even decide not to display a prompt at all! Knowing this, the possibilities are practically endless and are certainly more versatile than just using a prompt.

Note that there is a class called [**`Prompter<T>`**](https://docs.microsoft.com/en-us/dotnet/api/microsoft.bot.builder.formflow.advanced.prompter-1) in the [**`Microsoft.Bot.Builder.FormFlow.Advanced`**](https://docs.microsoft.com/en-us/dotnet/api/microsoft.bot.builder.formflow.advanced) namespace, but that's not what we're talking about here. You're unlikely to be using that class because it's meant to be used behind the scenes. The prompters we're talking about are delegates.

## How do we make a prompter?

You can use the [**`Prompter()`**](https://docs.microsoft.com/en-us/dotnet/api/microsoft.bot.builder.formflow.formbuilderbase-1.prompter) method to specify a prompter the same way you'd use a `Field()` method to add a field to your form. The `Prompter()` method takes a [**`PromptAsyncDelegate`**](https://docs.microsoft.com/en-us/dotnet/api/microsoft.bot.builder.formflow.advanced.promptasyncdelegate-1) which will contain the logic of your new prompter. 

	public static IForm<MyClass> BuildForm()
	{
		return new FormBuilder<MyClass>()
			.Prompter(PromptAsync)
			.Build();
	}

	/// <summary>
	/// Here is the method we're using for the PromptAsyncDelgate
	/// </summary>
	private static async Task<FormPrompt> PromptAsync(IDialogContext context, FormPrompt prompt, MyClass state, IField<MyClass> field)
	{
		// Your prompter logic goes here
	}

But how do we know what to put in our `PromptAsync()` method? It might be nice to know what the default prompter is so that we have something to build off of and so we know we're not leaving out anything important. Early on in the [FormBuilder source](https://github.com/Microsoft/BotBuilder/blob/master/CSharp/Library/Microsoft.Bot.Builder/FormFlow/FormBuilder.cs), you'll find the following code:

	if (this._form._prompter == null)
	{
		this._form._prompter = async (context, prompt, state, field) =>
		{
			var preamble = context.MakeMessage();
			var promptMessage = context.MakeMessage();
			if (prompt.GenerateMessages(preamble, promptMessage))
			{
				await context.PostAsync(preamble);
			}
			await context.PostAsync(promptMessage);
			return prompt;
		};
	}

You can see that this is where FormFlow checks to see if a custom prompter hasn't been implemented, and where the default prompter is then generated. We can copy this default code into our own prompter like this:

	/// <summary>
	/// Here is the method we're using for the PromptAsyncDelgate
	/// </summary>
	private static async Task<FormPrompt> PromptAsync(IDialogContext context, FormPrompt prompt, MyClass state, IField<MyClass> field)
	{
		var preamble = context.MakeMessage();
		var promptMessage = context.MakeMessage();
		if (prompt.GenerateMessages(preamble, promptMessage))
		{
			await context.PostAsync(preamble);
		}
		await context.PostAsync(promptMessage);
		return prompt;
	}

You will notice that this default prompter has the potential to post two messages, one of them being the "preamble." The preamble is present only when the prompt has attachments and the prompt's text includes multiple lines. When would a prompt have attachments? Consider the button choices generated by a Boolean Field for example:

	public bool ExtraBag { get; set; }

[![Would you like an extra bag? Yes, No][1]][1]

  [1]: https://i.stack.imgur.com/pbCtS.jpg

The attachment is generated by another piece of that default prompter you might have noticed, the [**`GenerateMessages()`**](https://docs.microsoft.com/en-us/dotnet/api/microsoft.bot.builder.formflow.advanced.extensions.generatemessages) extension method. It's used to make messages from the [**`FormPrompt`**](https://docs.microsoft.com/en-us/dotnet/api/microsoft.bot.builder.formflow.advanced.formprompt) it's being called on. If we don't expect to be needing any attachments in our prompts, we can go ahead and make our first change to the prompter by simplifying it down to this:

	/// <summary>
	/// Here is the method we're using for the PromptAsyncDelgate.
	/// </summary>
	private static async Task<FormPrompt> PromptAsync(IDialogContext context, FormPrompt prompt, MyClass state, IField<MyClass> field)
	{
		await context.PostAsync(prompt.Prompt);
		return prompt;
	}

The change can be seen if we encounter prompts that would otherwise have attachments. 

[![Would you like an extra bag?][2]][2]

  [2]: https://i.stack.imgur.com/k6ZQk.jpg

## Why make a prompter?

If getting rid of that prompt's attachments was what we actually wanted to do, we could just have defined a prompt for that Field instead.

	public static IForm<MyClass> BuildForm()
	{
		return new FormBuilder<MyClass>()
			.Field(nameof(ExtraBag), "Would you like an extra bag?")
			.Build();
	}

So what practical applications do custom prompters actually have?

We may run into situations where we can't seem to find any other way around FormFlow's builtin behavior. Suppose we've defined a validator for one of our Fields.

	public static IForm<MyClass> BuildForm()
	{
		return new FormBuilder<MyClass>()
			.Field(nameof(MyDouble), "Please enter a positive number", validate: ValidateAsync)
			.Build();
	}

	private static async Task<ValidateResult> ValidateAsync(MyClass state, object value)
	{
		var result = new ValidateResult();
		
		if ((double)value > 0)
		{
			result.IsValid = true;
			result.Feedback = "Good job";
			result.Value = value;
		}
		else
		{
			result.IsValid = false;
		}

		return result;
	}

While the feedback for a valid entry is optional, FormFlow always expects feedback when you set `result.IsValid` to `false`. Since we're leaving `result.Feedback` as `null` in that case, the user will be sent an empty message when they enter a negative number. 

[![Please enter a positive number][3]][3]

  [3]: https://i.stack.imgur.com/KctKi.jpg

This is an example of when it might be handy to have a custom prompter. We can make our prompter check to see if a message is empty before sending it like this:

	if (!string.IsNullOrWhiteSpace(promptMessage.Text))
	{
		await context.PostAsync(promptMessage);
	}

We can implement that prompter like this:

	public static IForm<MyClass> BuildForm()
	{
		return new FormBuilder<MyClass>()
			.Field(nameof(MyDouble), "Please enter a positive number", validate: ValidateAsync)
			.Prompter(PromptAsync)
			.Build();
	}

	/// <summary>
	/// Here is the method we're using for the PromptAsyncDelgate.
	/// </summary>
	private static async Task<FormPrompt> PromptAsync(IDialogContext context, FormPrompt prompt,
		MyClass state, IField<MyClass> field)
	{
		var preamble = context.MakeMessage();
		var promptMessage = context.MakeMessage();

		if (prompt.GenerateMessages(preamble, promptMessage))
		{
			await context.PostAsync(preamble);
		}

		// Here is where we've made a change to the default prompter.
		if (!string.IsNullOrWhiteSpace(promptMessage.Text))
		{
			await context.PostAsync(promptMessage);
		}

		return prompt;
	}

And then FormFlow won't send the user any more empty messages.

[![Please enter a positive number][4]][4]

  [4]: https://i.stack.imgur.com/b0dQP.jpg

## Conclusion

FormFlow offers a lot of customizability to begin with, and once you unlock the power of prompters then you can attain an even greater level of customizability. There is a lot you can do with prompters, such as sending extra messages, sending fewer messages, and keeping track of what messages the bot is sending. Examining every possibility is outside the scope of this blog post, but ideally you'll be left with the impression that you're ultimately in control of what happens in FormFlow and you'll be empowered to make new and exciting bots.
