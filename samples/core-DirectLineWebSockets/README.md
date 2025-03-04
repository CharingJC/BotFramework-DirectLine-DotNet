# Direct Line Bot Sample (using client WebSockets)

A sample bot and a custom client communicating with each other using the Direct Line API and WebSockets.

### Prerequisites

The minimum prerequisites to run this sample are:
* The latest update of Visual Studio 2017. You can download the community version [here](http://www.visualstudio.com) for free.
* The Bot Framework Emulator. To install the Bot Framework Emulator, download it from [here](https://emulator.botframework.com/). Please refer to [this documentation article](https://github.com/microsoft/botframework-emulator/wiki/Getting-Started) to know more about the Bot Framework Emulator.
* Register your bot with the Microsoft Bot Framework. Please refer to [this](https://docs.microsoft.com/en-us/bot-framework/portal-register-bot) for the instructions. Once you complete the registration, update the [Bot's appsettings.json](DirectLineBot/appsettings.json) file with the registered config values (MicrosoftAppId and MicrosoftAppPassword)
* For the client in this sample, The botId should be coming from app.config: https://github.com/microsoft/BotFramework-DirectLine-DotNet/blob/main/samples/core-DirectLineWebSockets/DirectLineClient/App.config#L5

#### Direct Line API
Credentials for the Direct Line API must be obtained from the Bot Channels Registration in the Azure portal, and will only allow the caller to connect to the bot for which they were generated.
In the Bot Framework developer portal, enable Direct Line in the channels list and then, configure the Direct Line secret and update its value in the [client's App.config](DirectLineClient/App.config#L4-L5) file alongside with the Bot Id. Make sure that the checkbox for version 3.0 is checked. Refer to [this](https://docs.microsoft.com/en-us/bot-framework/portal-configure-channels) for more information on how to configure channels.

![Configure Direct Line](images/outcome-configure.png)

#### Publish
Also, in order to be able to run and test this sample you must [deploy your bot, for example to Azure](https://docs.microsoft.com/en-us/azure/bot-service/bot-builder-deploy-az-cli). Alternatively, you can use [Ngrok to interact with your local bot in the cloud](https://blog.botframework.com/2017/10/19/debug-channel-locally-using-ngrok/). 

### Code Highlights

The Direct Line API is a simple REST API for connecting directly to a single bot. This API is intended for developers writing their own client applications, web chat controls, or mobile apps that will talk to their bot. The [Direct Line v3.0 Nuget package](https://www.nuget.org/packages/Microsoft.Bot.Connector.DirectLine) simplifies access to the underlying REST API.

Each conversation on the Direct Line channel must be explicitly started using the `DirectLineClient.Conversations.StartConversationAsync`.
Check out the client's [Program.cs](DirectLineClient/Program.cs#L25-L27) class which creates a new `DirectLineClient` and starts a new conversation.

You'll see that we are using the Direct Line secret to [obtain a token](DirectLineClient/Program.cs#L34). This step is optional, but prevents clients from accessing conversations they aren't participating in.

````C#
// Obtain a token using the Direct Line secret
var tokenResponse = await new DirectLineClient(directLineSecret).Tokens.GenerateTokenForNewConversationAsync();

// Use token to create conversation
var directLineClient = new DirectLineClient(tokenResponse.Token);
var conversation = await directLineClient.Conversations.StartConversationAsync();
````

User messages are sent to the Bot using the Direct Line Client `Conversations.PostActivityAsync` method using the `ConversationId` generated in the previous step.

````C#
while (true)
{
    string input = Console.ReadLine().Trim();

    if (input.ToLower() == "exit")
    {
        break;
    }
    else
    {
        if (input.Length > 0)
        {
            Activity userMessage = new Activity
            {
                From = new ChannelAccount(fromUser),
                Text = input,
                Type = ActivityTypes.Message
            };

            await client.Conversations.PostActivityAsync(conversation.ConversationId, userMessage);
        }
    }
}
````

Messages from the Bot are being received using WebSocket protocol. When the conversation was created a `streamUrl` is also returned and it will be the target for the WebSocket connection. Check out the client's WebSocket initialization in [Program.cs](DirectLineClient/Program.cs#L40-L43). 

> Note: In this project we use [websocket-sharp](https://github.com/sta/websocket-sharp) to implement the receiver client but this functionality can be implemented according to the needs of the solution.

````C#
using (var webSocketClient = new WebSocket(conversation.StreamUrl))
{
    webSocketClient.OnMessage += WebSocketClient_OnMessage;
    webSocketClient.Connect();
````

We use the [WebSocketClient_OnMessage](DirectLineClient/Program.cs#L71) method to handle the OnMessage event of the `webSocketClient` witch occurs when the WebSocket receives a message.

````C#
private static void WebSocketClient_OnMessage(object sender, MessageEventArgs e)
{
    var activitySet = JsonConvert.DeserializeObject<ActivitySet>(e.Data);
    var activities = from x in activitySet.Activities
        where x.From.Id == botId
        select x;
````

Direct Line v3.0 has support for Attachments (see [Add media attachments to messages](https://docs.microsoft.com/en-us/bot-framework/dotnet/bot-builder-dotnet-add-media-attachments) for more information about attachments). Check out the `WebSocketClient_OnMessage` method in [Program.cs](DirectLineClient/Program.cs#L88-L105) to see how the Attachments are retrieved and rendered appropriately based on their type.


````C#
if (activity.Attachments != null)
{
    foreach (Attachment attachment in activity.Attachments)
    {
        switch (attachment.ContentType)
        {
            case "application/vnd.microsoft.card.hero":
                RenderHeroCard(attachment);
                break;

            case "image/png":
                Console.WriteLine($"Opening the requested image '{attachment.ContentUrl}'");

                Process.Start(attachment.ContentUrl);
                break;
        }
    }
}
````


### Outcome

To run the sample, you'll need to run both Bot and Client apps.
* Running Bot app
    1. In the Visual Studio Solution Explorer window, right click on the **DirectLineBot** project.
    2. In the contextual menu, select Debug, then Start New Instance and wait for the _Web application_ to start.
* Running Client app
    1. In the Visual Studio Solution Explorer window, right click on the **DirectLineSampleClient** project.
    2. In the contextual menu, select Debug, then Start New Instance and wait for the _Console application_ to start.

To test the Attachments type `show me a hero card` or `send me a botframework image` and you should see the following outcome.

![Sample Outcome](images/outcome.png)

### More Information

To get more information about how to get started in Bot Builder for .NET and Conversations please review the following resources:
* [Bot Builder for .NET](https://docs.microsoft.com/en-us/bot-framework/dotnet/)
* [Bot Framework FAQ](https://docs.microsoft.com/en-us/azure/bot-service/bot-service-resources-bot-framework-faq#i-have-a-communication-channel-id-like-to-be-configurable-with-bot-framework-can-i-work-with-microsoft-to-do-that)
* [Direct Line API - v3.0](https://docs.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-overview/)
* [Direct Line v3.0 Nuget package](https://www.nuget.org/packages/Microsoft.Bot.Connector.DirectLine/)
* [Add media attachments to messages](https://docs.microsoft.com/en-us/bot-framework/dotnet/bot-builder-dotnet-add-media-attachments)
* [Bot Framework Emulator](https://github.com/microsoft/botframework-emulator/wiki/Getting-Started)

