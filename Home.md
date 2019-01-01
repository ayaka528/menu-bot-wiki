# Menu-Bot 
This bot demonstrates several concepts at once. At a high level, it offers a fully guided flow so that the user is always aware of their options on any given conversation turn. When bots leave conversations open-ended, users often don't know what to do or say - guiding conversations with buttons (even if you also support natural language) helps mitigate this issue. 

## Welcoming the User


## Waterfall Dialogs and Prompts
This sample uses the botbuilder's waterfall dialog and prompt abstractions to model a conversation. At a high level, a waterfall dialog is an array of functions to run on each conversation turn. Waterfall dialogs are great for expressing rigid conversations, where a specific series of steps should always occur. They're not so good for expressing conversations with lots of context switching. In this case our "food bank bot" should follow a fairly regimented flow to get users to the information they need, so waterfall dialogs are a great choice!

Before setting up our dialogs, we need to make sure to have a place for them to save their state (where they are in a conversation). Let's take a step back to show how we plumb our state management - if you've already set this up, feel free to skip ahead to the "Creating a Waterfall Dialog" section. 

### Setting up State 
We create our `ConversationState` instance in our index.js, by first creating a `MemoryStorage` instance and passing it to the `ConversationState` constructor:
```js
const memoryStorage = new MemoryStorage();
const conversationState = new ConversationState(memoryStorage);
```
We can pass the `MemoryStorage` constructor a storage provider (Azure Tables, Azure Cosmos, etc.) if we like. `ConversationState` would then get and set state in that external provider. For now we'll keep it empty, which will instead store everything in memory. 

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




