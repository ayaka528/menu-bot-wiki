# Menu-Bot 
This bot demonstrates several concepts at once. At a high level, it offers a fully guided flow so that the user is always aware of their options on any given conversation turn. When bots leave conversations open-ended, users often don't know what to do or say - guiding conversations with buttons (even if you also support natural language) helps mitigate this issue. We construct the guided conversation by first welcoming the user, and then by using waterfall dialogs and prompts to communicate with our user. 

## Welcoming the User
If our bot doesn't proactively welcome a user, then the user won't have any sense for what the bot's function is. Take a look at in-depth [botbuilder document](https://docs.microsoft.com/en-us/azure/bot-service/bot-builder-send-welcome-message?view=azure-bot-service-4.0&tabs=csharp%2Ccsharpmulti%2Ccsharpwelcomeback) for more details. In this case we welcome our user by listening for any incoming `ConversationUpdate` activities, validating that a new member was added, and sending a message: 
```js
if (turnContext.activity.type === ActivityTypes.ConversationUpdate) {
    if (this.memberJoined(turnContext.activity)) {
        await turnContext.sendActivity(`Hey there! Welcome to the food bank bot. I'm here to help orchestrate 
                                        the delivery of excess food to local food banks!`);
```

The member joined function is just a helper to abstract away some of the complexity of determining whether a member joined the conversation (as opposed to the bot): 
```js
memberJoined(activity) {
    return ((activity.membersAdded.length !== 0 && (activity.membersAdded[0].id !== activity.recipient.id)));
}
```

## Waterfall Dialogs and Prompts
This sample uses the botbuilder's waterfall dialog and prompt abstractions to model its conversation. At a high level, a waterfall dialog is an array of functions to run step-by-step on each conversation turn. Waterfall dialogs are great for expressing rigid conversations, where a specific series of steps should always occur. They're not so good for expressing conversations with lots of context switching. In this case our "food bank bot" should follow a fairly regimented flow to get users to the information they need, so waterfall dialogs are a great choice!

Before setting up our dialogs, we need to make sure to have a place for them to save their state. Specifically, dialogs need to know where they are in a conversation in order to kick off the right step. Let's take a step back to show how we plumb our state management - if you've already set this up, feel free to skip ahead to the "Creating a Waterfall Dialog" section. 

### Setting up State 
We create our `ConversationState` instance in our index.js by first creating a `MemoryStorage` instance and passing it to the `ConversationState` constructor:
```js
const memoryStorage = new MemoryStorage();
const conversationState = new ConversationState(memoryStorage);
```
We can pass the `MemoryStorage` constructor a storage provider (Azure Tables, Azure Cosmos, etc.) if we like. `ConversationState` would then get and set state against that external provider. For now we'll keep it empty, which will instead store everything in memory. 

Next, we initialize our bot with that `ConversationState` instance: 
```js
const myBot = new FoodBot(conversationState);
```

### Creating a Waterfall Dialog
Now that our bot has a place to save its state, we create a `dialogState` property and a `DialogSet`. We make both of these properties of our bot class to best organize our bot: 

```js
constructor(conversationState) {
     this.conversationState = conversationState;
     this.dialogState = this.conversationState.createProperty(DIALOG_STATE_PROPERTY);
     this.dialogs = new DialogSet(this.dialogState);
```
Now we can create our waterfall dialog and add it to the `DialogSet`. A waterfall dialog has a unique string name for persisting and reusing it. In this case that unique string name is identified by the `MENU_DIALOG` variable: 

```js
        this.dialogs.add(new WaterfallDialog(MENU_DIALOG, [
            this.promptForMenu,
            this.handleMenuResult,
            this.resetDialog,
        ]));
```
As mentioned above, a `WaterfallDialog` is just an array of functions that will run in series. This waterfall dialog is composed of three functions, each called a `step`. We could declare these functions anonymously (inline and without a name), but chose to refer to them this way to better organize our code. Let's take a look at the first function: 

```js
async promptForMenu(step) {
    return step.prompt(MENU_PROMPT, {
        choices: ["Donate Food", "Find a Food Bank"],
        prompt: "Do you have food to donate or do you need to find a food bank?",
        retryPrompt: "I'm sorry, that wasn't a valid response. Are you looking to donate food or find a food bank?"
    });
``` 
In this function, we prompt the user with two choices, "Donate Food" and "Find a Food Bank". Just as our dialog had a name, we're calling our choice prompt by the name `MENU_PROMPT`. We also defined this choice prompt in our bot's constructor, so that we can reuse it throughout the bot:
```js
this.dialogs.add(new ChoicePrompt(MENU_PROMPT));
```
When we call our prompt, we defined three properties: `choices`, `prompt`, and `retryPrompt`. `choices` is the array of options, which will ultimately render as `suggestedActions`: buttons that only exist for one turn. `prompt` is the text we actually want to prompt the users with. `retryPrompt` is the text that the bot will use if a user replies with something other than the button text. 

In our next dialog step, we parse the user's answer: 
```js
async handleMenuResult(step) {
    switch (step.result.value) {
        case "Donate Food":
            return step.beginDialog(DONATE_FOOD_DIALOG);
        case "Find a Food Bank":
            return step.beginDialog(FIND_FOOD_DIALOG);
    }
    return step.next();
}
```
As you can see, the value of the user's response shows up in `step.result.value`. We use a switch case to determine which answer they gave and begin the next dialog as necessary. Note that we have no `default` handler in our switch case. This is because our dialog will only ever get to this step if user entered one of the valid inputs (or clicked a button). 

Depending on which button was clicked, we call `step.beginDialog` with the name of other dialogs that we've defined. We'll get into how we defined those other dialogs in "Creating Component Dialogs", but for now suffice it to say that `beginDialog` pushes another dialog onto a dialog stack such that subsequent messages get sent to the new dialog until it finishes. Once that dialog finishes, we come back to the last step of our Main Menu dialog: 

```js
async resetDialog(step) {
    return step.replaceDialog(MENU_DIALOG);
}
```
This one-line function enables us to build a "message loop". Basically, when we've finished a conversation flow we get to this step, which takes us back to the beginning of the main menu. This helps accomplish our goal of never leaving a user in the dark about what they can do on a specific turn. 


