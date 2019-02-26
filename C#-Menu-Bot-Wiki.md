# Menu-Bot 
This bot demonstrates several concepts at once. At a high level, it offers a fully guided flow so that the user is always aware of their options on any given conversation turn. When bots leave conversations open-ended, users often don't know what to do or say - guiding conversations with buttons (even if you also support natural language) helps mitigate this issue. We construct the guided conversation by first welcoming the user, and then by using waterfall dialogs and prompts to communicate with our user. Here's a quick peek at one of this bot's conversation flows: 

![DonateDialog GIF](https://github.com/ryanvolum/menu-bot/blob/master/wiki_assets/DonateDialog.gif)

## Welcoming the User
If our bot doesn't proactively welcome a user, then the user won't have any sense for what the bot is capable of doing. Take a look at in-depth [BotBuilder document](https://docs.microsoft.com/en-us/azure/bot-service/bot-builder-send-welcome-message?view=azure-bot-service-4.0&tabs=csharp%2Ccsharpmulti%2Ccsharpwelcomeback) for more details. In this case we welcome our user by listening for any incoming `ConversationUpdate` activities in our `OnTurnAsync` function, validating that a new member was added, sending a message, and starting a dialog: 

```csharp
if (turnContext.Activity.Type == ActivityTypes.ConversationUpdate)
{
    if (this.MemberJoined(turnContext.Activity))
    {
        await turnContext.SendActivityAsync("Hey there! Welcome to the food bank bot. " +
            "I'm here to help orchestrate the delivery of excess food to local food banks!");
        await dialogContext.BeginDialogAsync(FoodBotDialogSet.StartDialogId, null, cancellationToken);
    }
}
```

The member joined function is just a helper to abstract away some of the complexity of determining whether a member joined the conversation (as opposed to the bot): 

```csharp
private bool MemberJoined(Activity activity)
{
    return activity.MembersAdded.Count != 0 && (activity.MembersAdded[0].Id != activity.Recipient.Id);
}
```

## Waterfall Dialogs and Prompts
This sample uses the botbuilder's waterfall dialog and prompt abstractions to model its conversation. At a high level, a waterfall dialog is an array of functions to run step-by-step on each conversation turn. Waterfall dialogs are great for expressing rigid conversations, where a specific series of steps should always occur. They're not so good for expressing conversations with lots of context switching. In this case our "food bank bot" should follow a fairly regimented flow to get users to the information they need, so waterfall dialogs are a great choice!

Before setting up our dialogs, we need to make sure to have a place for them to save their state. Specifically, dialogs need to know where they are in a conversation in order to kick off the right step. Let's take a step back to show how we plumb our state management - if you've already set this up, feel free to skip ahead to the [Creating a Waterfall](https://github.com/ryanvolum/menu-bot/wiki/C%23-Menu-Bot-Wiki#creating-a-waterfall-dialog) section. 

### Setting up State 
We create our `ConversationState` instance in our Startup.cs by first creating a `MemoryStorage` instance and passing it to the `ConversationState` constructor:

```csharp
IStorage dataStore = new MemoryStorage();
// Create Conversation State object.
// The Conversation State object is where we persist anything at the conversation-scope.
var conversationState = new ConversationState(dataStore);
```
We could have passed the `MemoryStorage` constructor a storage provider (Azure Tables, Azure Cosmos, etc.), which would allow `ConversationState` to then get and set state against that external provider. For now we'll keep it empty, which will instead store everything in memory. 

Next, we ensure our bot can get access to that `ConversationState` instance, by registering it in the services collection which allows it to be passed into the IBot constructor through dependency injection: 

```csharp
// Register conversation state.
services.AddSingleton<BotState>(conversationState);
```
All of our dialogs must be created and executed within a `DialogSet`, and the constructor of a DialogSet requires an `IStatePropertyAccessor<DialogState>`. In this sample, we use a custom DialogSet, which we create and at the same time create the `dialogState` property within our ConversationState.

```csharp
// The dialogset will need a state store accessor. Since the state accessor is only going to be used by the FoodBotDialogSet, use  
// AddSingleton to register and inline create the FoodBotDialogSet, at the same time creating the the property accessor in the
// conversation state
services.AddSingleton(sp => new FoodBotDialogSet(conversationState.CreateProperty<DialogState>("FOODBOTDIALOGSTATE")));
```
The custom `DialogSet` class simply serves as the container for all the child dialogs. It contains a single child dialog, `MainMenuDialog`, which defines the top level menu and which itself calls other child dialogs. Each dialog has a unique string name for persisting and reusing it, and in this case we're using the string `mainMenuDialog` as the dialog ID for our MainMenuDialog instance.

```csharp
public class FoodBotDialogSet : DialogSet
{
    public static string StartDialogId => "mainMenuDialog";

    public FoodBotDialogSet(IStatePropertyAccessor<DialogState> dialogStatePropertyAccessor)
        : base(dialogStatePropertyAccessor)
    {
        // Add the top-level dialog
        Add(new MainMenuDialog(StartDialogId));
    }
}
```

Since we have registered our `FoodBotDialogSet` instance, it can be injected into our Bot class using dependency injection along with our `BotState` (which is the ConversationState object we created and registered back in Startup.cs):
```csharp
public class FoodBot : IBot
{
    private readonly BotState _botState;
    private readonly FoodBotDialogSet _foodBotDialogSet;
    private readonly ILogger _logger;
    
    public FoodBot(BotState botState, FoodBotDialogSet dialogSet, ILoggerFactory loggerFactory)
    {
        _botState = botState ?? throw new ArgumentNullException(nameof(botState));
        _foodBotDialogSet = dialogSet ?? throw new ArgumentNullException(nameof(dialogSet));
        ...
```

### Orchestrating our Top Level Dialog from the OnTurnAsync Function
Now that we've created and added our `MainMenuDialog` to our `FoodBotDialogSet` and injected them into our Bot, we need to let our bot know when to start and continue the top-level dialog. Most of this code will be fairly boilerplate for any bot that uses dialogs. As a quick refresher, the `OnTurnAsync` function is the function that gets called on every single turn. That means any time we get a message from our user, we run the `OnTurnAsync` function. `OnTurnAsync` receives a context object as a parameter, which bundles up the incoming activity and several conversational helpers for orchestrating the conversation. Let's take a look at this bot's `OnTurnAsync` function. 

We start by creating a `DialogContext` for our `DialogSet`: 

```csharp
    public async Task OnTurnAsync(ITurnContext turnContext, CancellationToken cancellationToken = default(CancellationToken))
    {
        var dialogContext = await this._foodBotDialogSet.CreateContextAsync(turnContext, cancellationToken);
```
This dialog context contains helpers to assess the state of our dialogs. In this case, we'll use it to determine if there is an active dialog, and to continue it if there is. Note that we only do this when we receive messages from the user (Message Activities):  

```csharp
    if (turnContext.Activity.Type == ActivityTypes.Message)
    {
        if (dialogContext.ActiveDialog != null)
        {
            await dialogContext.ContinueDialogAsync(cancellationToken);
...
```

If there is no active dialog, we go ahead and start our Main Menu dialog, using the string identifier of the main menu dialog exposed through the `FoodBotDialogSet.StartDialogId` static property: 

```csharp
        }
        else
        {
            await dialogContext.BeginDialogAsync(FoodBotDialogSet.StartDialogId, null, cancellationToken);
        }
```

When we start or continue a dialog from our `OnTurnAsync`, the dialog will run the appropriate step(s) (waterfall functions), and then return. It's therefore important for us to save all state changes at the end of our `OnTurnAsync` function. Remember that we're saving our dialog's state through our `ConversationState` instance, accessible here through the `_botState` field. If we don't actually call `_botState_.SaveChangesAsync`, they won't be persisted and we'll never move on to subsequent dialog steps:

```csharp
    // Always persist changes at the end of every turn, here this is the dialog state in the conversation state.
    await _botState.SaveChangesAsync(turnContext, cancellationToken: cancellationToken);
```
The rest of the `OnTurnAsync` function contains the welcome code which we looked at in [Welcoming the User](https://github.com/ryanvolum/menu-bot/wiki/C%23-Menu-Bot-Wiki/Home/_edit#welcoming-the-user). 

### Creating a Waterfall Dialog
Now that our bot has a place to save its state, a `DialogSet` and code to call `BeginDialogAsync` for our top level dialog in our DialogSet, it's time to look at the implementation of that top level dialog. This is implemented as a **Component Dialog**, as are all the child dialogs in this sample. If you navigate through the project directory, you'll find a folder called `Dialogs` with five files: `FoodBotDialogSet.cs` which we've already discussed, `MainMenuDialog.cs`, `DonateFoodDialog.cs`, `FindFoodDialog.cs` and `ContactDialog.cs`. We declare these dialogs outside of our `FoodBot.cs` for a few reasons. For one, building all of our conversation flow in one file would get unmanageable. It would be near impossible to work collaboratively with other developers in that same file. Separating dialogs also allows us to treat them as reusable modules - we could use them multiple times in the same bot, or even publish them to be used in other bots. In order to achieve this modular behavior, we rely on `ComponentDialogs`, which act as a module for a dialog or multiple dialogs. 

Our `MainMenuDialog` inherits from `ComponentDialog` and takes a dialogId: 

```csharp
public class MainMenuDialog : ComponentDialog
{
    public MainMenuDialog(string dialogId) : base(dialogId)
    {
        ...
```
The vlaue of the dialogId is the name of our ComponentDialog and is set at instantiation, in this case by the `FoodBotDialogSet` class This is of course the same dialogId used in the bots' `OnTurnAsync` method to specify the dialog to start . 

The constructor then sets `this.initialDialogId` to the name of our dialog:

```csharp
    // ID of the child dialog that should be started anytime the component is started.
    this.InitialDialogId = dialogId;
```

Now we can create our waterfall dialog and add it to the `ComponentDialog`. A waterfall dialog also has a unique string name for persisting and reusing it. In this case our waterfall dialog will have the same name as our Component Dialog, since it's the only dialog in our Component Dialog. If we choose not to explicitly set `this.initialDialogId`, it will automatically be set to the first dialog added to the ComponentDialog. 

```csharp
    this.AddDialog(new WaterfallDialog(
        dialogId,
        new WaterfallStep[]
        {
            this.PromptForMenuAsync,
            this.HandleMenuResultAsync,
            this.ResetDialogAsync
        }));
```
As mentioned above, a `WaterfallDialog` is just an array of functions that will run in series. This waterfall dialog is composed of three functions, each called a `step`. We could declare these functions anonymously (inline and without a name), but chose to refer to them this way to better organize our code. Let's take a look at the first function: 

```csharp
    private async Task<DialogTurnResult> PromptForMenuAsync(WaterfallStepContext stepContext, CancellationToken cancellationToken)
    {
        return await stepContext.PromptAsync(MENUPROMPT, 
            new PromptOptions
            {
                Choices = ChoiceFactory.ToChoices(new List<string> { "Donate Food", "Find a Food Bank", "Contact Food Bank" }),
                Prompt = MessageFactory.Text("Do you have food to donate, do you need food, or are you contacting a food bank?"),
                RetryPrompt = MessageFactory.Text("I'm sorry, that wasn't a valid response. Please select one of the options")
            },
            cancellationToken);
    }
```
In this function, we prompt the user with three choices, "Donate Food", "Find a Food Bank" and "Contact Food Bank". Just as our dialog had a name, we need to name prompts as well. In this case, we're calling this choice prompt by the name `MENUPROMPT` which is declared at the top of the class. We also register this choice prompt in our bot's constructor, so that we can reuse it throughout the bot:

```csharp
    this.AddDialog(new ChoicePrompt(MENUPROMPT));
```
When we call our prompt, we defined three properties: `Choices`, `Prompt`, and `RetryPrompt`. `Choices` is the array of options, which will ultimately render as `SuggestedActions`: buttons that only exist for one turn. `Prompt` is the text we actually want to prompt the users with. `RetryPrompt` is the text that the bot will use if a user replies with something other than the button text. 

In our next dialog step, we parse the user's answer: 

```csharp
    private async Task<DialogTurnResult> HandleMenuResultAsync(WaterfallStepContext stepContext, CancellationToken cancellationToken)
    {
        switch (((FoundChoice)stepContext.Result).Value)
        {
            case "Donate Food":
                return await stepContext.BeginDialogAsync(DONATE_FOOD_DIALOG, null, cancellationToken);
            case "Find a Food Bank":
                return await stepContext.BeginDialogAsync(FIND_FOOD_DIALOG, null, cancellationToken);
            case "Contact Food Bank":
                return await stepContext.BeginDialogAsync(CONTACT_DIALOG, null, cancellationToken);
            default:
                break;
        }
        return await stepContext.NextAsync(cancellationToken: cancellationToken);
    }
```
As you can see, the value of the user's response shows up in `stepContext.Result.Value`. We use a switch case to determine which answer they gave and begin the next dialog as necessary. Note that we have no `default` handler logic in our switch case. This is because the users' input is validated by the `ChoicePrompt` and our dialog will only ever get to this method if the user entered one of the valid inputs (or clicked a button). 

Depending on which button was clicked, we call the `BeginDialogAsync` method with the name of other dialogs that we've defined. We'll get into how we defined those other dialogs in [Creating the other Component Dialogs](https://github.com/ryanvolum/menu-bot/wiki/C%23-Menu-Bot-Wiki#creating-the-other-component-dialogs), but for now suffice it to say that `BeginDialogAsync` pushes another dialog onto a dialog stack such that subsequent messages get sent to the new dialog until it finishes. Once that dialog finishes, we come back to the last step of our Main Menu dialog: 

```csharp
    private async Task<DialogTurnResult> ResetDialogAsync(WaterfallStepContext stepContext, CancellationToken cancellationToken)
    {
        return await stepContext.ReplaceDialogAsync(this.InitialDialogId, null, cancellationToken);
    }
```
This one-line function enables us to build a "message loop". Basically, when we've finished a conversation flow we get to this step, which takes us back to the beginning of the main menu. We accomplish this by replacing the current Main Menu dialog with itself, using the `ReplaceDialogAsync` function. The bot will now start back at the first step of the Main Menu dialog, accomplishing our goal of never leaving a user in the dark about what they can do on a specific turn. 


### Creating the other Component Dialogs
Now that we've taken a look at our top-level waterfall dialog, let's dive into the other dialogs in this bot. Let's take a look at the `FindFoodDialog`. 

Our dialog inherits from `ComponentDialog` and takes a dialogId: 

```csharp
public class DonateFoodDialog : ComponentDialog
{
    public DonateFoodDialog(string dialogId) : base(dialogId)
    {
        ...
```
The constructor then sets `this.initialDialogId` to the name of our waterfall dialog:

```csharp
    // ID of the child dialog that should be started anytime the component is started.
    this.InitialDialogId = dialogId;
```

Next, we add a `ChoicePrompt` that we will be using in our waterfall dialog, just as we did in our Main Menu dialog: 

```csharp
    this.AddDialog(new ChoicePrompt("choicePrompt"));
```

And we then add our waterfall dialog: 

```csharp
this.AddDialog(
    new WaterfallDialog(dialogId, new WaterfallStep[]
        {
            async (stepContext, ct) =>
            {
                return await stepContext.PromptAsync(
                    "choicePrompt",
                    new PromptOptions
                    {
                        Choices = ChoiceFactory.ToChoices(ScheduleHelpers.GetValidDonationDays()),
                        Prompt = MessageFactory.Text("What day would you like to donate food?"),
                        RetryPrompt= MessageFactory.Text("That's not a valid day! Please choose a valid day.")
                    },
                    ct
                );
            },
            async (stepContext, ct) =>
            {
                var day = ((FoundChoice)stepContext.Result).Value;
                var filteredFoodBanks = ScheduleHelpers.FilterFoodBanksByDonation(day);
                var carousel = ScheduleHelpers.CreateFoodBankDonationCarousel(filteredFoodBanks).AsMessageActivity();

                // Create the activity and attach a set of Hero cards.
                await stepContext.Context.SendActivityAsync(carousel);
                return await stepContext.EndDialogAsync();
            }
        }
    )
```

Note that instead of declaring the steps of the waterfall as members of the class, we just coded them inline as anonymous functions. We took this approach here since we know we'll only use these functions once, and because the code footprint is fairly small. 

As with our Main Menu dialog, this waterfall is an array of functions. The first prompts users for an array of days (determined by a helper method that process a JSON schedule). The second handles the response by creating a carousel of cards that display food banks open on the selected day. To create cards and carousels, we use the `CardFactory` in the `BotBuilder` package. See `UXHelpers.cs` to see the full implementation, and check out the [Add Media to Messages](https://docs.microsoft.com/en-us/azure/bot-service/bot-builder-howto-add-media-attachments?view=azure-bot-service-4.0&tabs=csharp) Azure doc for more information about Hero Cards and carousels. 

### Persisting State throughout a Dialog
Sometimes a dialog needs to persist some information to be used on a later step. Take a look at the Contact conversation flow below:

![ContactDialog GIF](https://github.com/ryanvolum/menu-bot/blob/master/wiki_assets/ContactDialog.gif)

 **Note**: this flow isn't actually sending a food bank a message, so feel free to test the bot yourself!

You can see that we gather the user's email address, the message they want to send and the name of the food bank they want to send it to. We then use all that information to send a message to a food bank. But how are we persisting this information throughout the lifetime of the dialog? 

Let's take a look at the `ContactDialog` dialog. Like our other Component Dialogs, we're anonymously creating an array of functions that the bot will run through one at a time: 

```csharp
    this.AddDialog(
        new WaterfallDialog(dialogId, new WaterfallStep[]
            {
                ...
```
These functions prompt users for the name of the Food Bank they want to contact (using `ChoicePrompt`), their email address (using `TextPrompt`) and the message they want to send (also using `TextPrompt`). It also asks them to confirm that they want to send a message, which uses `ConfirmPrompt`. When we gather a piece from the user that we know we'll need later, we save it to the **WaterfallStepContext** `values` dictionary: 

```csharp
    const string EMAILADDRESS = "emailAddress";
    ...
    stepContext.Values.Add(EMAILADDRESS, (string)stepContext.Result);
```

Then on later turns we can access that property on the same `values` dictionary!

```csharp
    ScheduleHelpers.SendFoodbankMessage(
        (string)stepContext.Values[FOODBANKNAME], 
        (string)stepContext.Values[EMAILADDRESS], 
        (string)stepContext.Values[MESSAGE]);
```
