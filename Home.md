# Menu-Bot 
This bot demonstrates several concepts at once. At a high level, it offers a fully guided flow so that the user is always aware of their options on any given conversation turn. When bots leave conversations open-ended, users often don't know what to do or say - guiding conversations with buttons (even if you also support natural language) helps mitigate this issue. 

## Welcoming the User


## Waterfall Dialogs and Prompts
This sample uses the botbuilder's waterfall dialog and prompt abstractions to model a conversation. At a high level, a waterfall dialog is an array of functions to run on each conversation turn. Waterfall dialogs are great for expressing rigid conversations, where a specific series of steps should always occur. They're not so good for expressing conversations with lots of context switching. In this case our "food bank bot" should follow a fairly regimented flow to get users to the information they need, so waterfall dialogs are a great choice!

Before creating our `WaterfallDialog` we must first create a `DialogSet`, which is created with `dialogState` (a library for all dialogs to store things over time), which is using `conversationState` to get and set that state. This might sound like a lot of concepts, but it's mostly boilerplate code that we should only need to do once. 

### State Management
In our `index.js` we create our `MemoryStorage` and `ConversationState`. 
```js
const memoryStorage = new MemoryStorage();
const conversationState = new ConversationState(memoryStorage);
```


```js
constructor(conversationState) {
    this.conversationState = conversationState;

    // Configure dialogs
    this.dialogState = this.conversationState.createProperty(DIALOG_STATE_PROPERTY);
    this.dialogs = new DialogSet(this.dialogState);
```

A waterfall dialog has a unique string name for persisting and reusing it. In this case that unique string name is identified by the `MENU_DIALOG` variable: 

```js
this.dialogs.add(new WaterfallDialog(MENU_DIALOG, [
    this.promptForMenu,
    this.handleMenuResult,
    this.resetDialog,
]));
```

This waterfall dialog is composed of three functions, each called a `step`. We could declare these functions anonymously (inline and without a name), but chose to refer to them this way to better organize our code. Let's take a look at the first function: 

```js
async promptForMenu(step) {
    return step.prompt(MENU_PROMPT, {
        choices: ["Donate Food", "Find a Food Bank"],
        prompt: "Do you have food to donate or do you need to find a food bank?",
        retryPrompt: "I'm sorry, that wasn't a valid response. Are you looking to donate food or find a food bank?"
    });
``` 

In this function, we prompt the user with two choices, "Donate Food" and "Find a Food Bank". Just as our dialog had a name, we're calling our choice prompt by the name `MENU_PROMPT`. We defined this choice prompt and other dialogs/prompts in our bot's constructor: 
```js
this.dialogs.add(new ChoicePrompt(MENU_PROMPT));
```


