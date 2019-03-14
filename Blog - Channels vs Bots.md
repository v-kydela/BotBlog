# Understanding the Bot Framework: The Responsibilities of the Channel vs the Responsibilities of the Bot
##### by Kyle Delaney <div style="text-align: right">November 2, 2018</div>

Whether you're new to the Microsoft Bot Framework or you've already got a few bots under your belt, you may find yourself wondering about the capabilities of the framework when it comes to customizing the user's experience of interacting with your bot. As you've probably come to realize, in order to interact with a bot your users depend on a UI provided by a client application associated with a channel. This article is intended to provide more clarity about how much control a bot has when it comes to that UI.

## What is a channel?

At the time of this writing, there are about 15 channels available for bots to communicate through. I say "about" because there may be some ambiguity about the definition of what constitutes a channel when it comes to things like Direct Line and Web Chat and the Bot Emulator. Direct Line for example is meant to provide a means for developers to create their own client applications and so it is unlike other channels which provide client applications.

## What should we expect a bot to be able to do?


[//]: # (
Examples of who this blog is for:
https://github.com/Microsoft/BotBuilder/issues/1287
https://stackoverflow.com/questions/53092401/bot-v4-shows-inconsistent-results-in-diff-channel-for-basic-choiceprompt
https://stackoverflow.com/questions/53262675/open-browser-window-on-click
"Speech recognition Fails for QnA Bot [pearl_test]" - a 11/22/2018 email where Web Chat's microphone wouldn't receive audio input and someone thought this was a bot problem
https://stackoverflow.com/questions/53526447/carousel-in-bot-framework-continuously-updating-from-database-upon-scrolling
https://stackoverflow.com/questions/53844906/how-to-restart-a-connection-to-the-bot-using-sdk-v4-for-node-js
https://stackoverflow.com/questions/54255185/how-to-do-autocomplete-text-suggestion-in-message-box-in-bot
)