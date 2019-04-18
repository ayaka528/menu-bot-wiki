このWikiはNode menu-botにある概念とパターンを説明しています。ボットのコードを見るには、このレポジトリの[javascript_nodejs](https://github.com/ryanvolum/menu-bot/tree/master/javascript_nodejs) ディレクトリを参照してください。

## Guiding Conversations 

このプロジェクトでは、メニューシステムを使用してボットが会話を誘導する方法を示します。エンドユーザーには常にオプションの個別のセットが表示されるため、ユーザーができることについてあいまいさはありません。当然のことながら、ボット開発について考えるとき、私たちはすぐに自然言語処理（NLP）を使用してユーザー_intent_を理解し、関連する_entities_を抽出することを考えます。ただし、NLPに全面的に依存している場合、ユーザーは特定のステップで何が言えるのか困惑してしまいます。私たちのモダリティ（この場合はWebクライアント）を用いるときは、ボタン、カード、カルーセルなどの豊富なコントロールを使ってできることを_show_する必要があります。NLPと組み合わせることで会話の周期を短くすることができます。

ガイド付き会話は、最初にユーザーを歓迎し、次にウォーターフォールダイアログとユーザーとのコミュニケーションを促すプロンプトを使用して構築します。これが、このボットの会話の流れの1つをちょっとのぞいてみたものです。

![DonateDialog GIF](https://github.com/ryanvolum/menu-bot/blob/master/wiki_assets/DonateDialog.gif)

## ユーザーを歓迎する
もしボットがプロアクティブにユーザーを歓迎しなければ、ユーザーはそのボットが何をしてくれるのか分かりません。詳細は[BotBuilder document](https://docs.microsoft.com/ja-jp/azure/bot-service/bot-builder-send-welcome-message?view=azure-bot-service-4.0&tabs=csharp%2Ccsharpmulti%2Ccsharpwelcomeback)を確認してください。
この場合、 `OnTurn`関数で入ってくる`ConversationUpdate`アクティビティを監視し、新しいメンバーが追加されたことを確認し、メッセージを送信し、そしてダイアログを開始することでユーザーを歓迎します。

```js
if (turnContext.activity.type === ActivityTypes.ConversationUpdate) {
    if (this.memberJoined(turnContext.activity)) {
        await turnContext.sendActivity(`Hey there! Welcome to the food bank bot. I'm here to help orchestrate 
                                        the delivery of excess food to local food banks!`);
        await dialogContext.beginDialog(MENU_DIALOG);

```

メンバージョイン関数は、メンバーが会話に参加したかどうかを（ボットとは対照的に）判断する複雑な部分を抽象化するための単なるヘルパーです。

```js
memberJoined(activity) {
    return ((activity.membersAdded.length !== 0 && (activity.membersAdded[0].id !== activity.recipient.id)));
}
```

## ウォーターフォールダイアログとプロンプト
このサンプルでは、​​ボットビルダーのウォーターフォールダイアログとプロンプトという概念を用いて会話をモデル化しています。ウォーターフォールダイアログは、会話の各ターンに段階的に実行するための関数の配列です。ウォータフォールダイアログは、特定の一連のステップが常に発生する定型の会話を表現するのに最適です。たくさんの文脈の切り替えが発生する会話を表現するのにはあまり適していません。この場合、私たちの[food bank bot](https://github.com/ryanvolum/food-bank-bot)は、ユーザーが必要な情報を入手するために、きちんと管理されたフローをたどることを期待しているので、ウォーターフォールダイアログは最適な選択です。

ダイアログを設定する前に、私たちはそれらの状態を保存するための場所があることを確認する必要があります。特に、ダイアログは正しいステップを開始するために、会話のどこにいるのかを知る必要があります。状態の管理をどのようにして行うのかを説明しましょう。すでに設定しているのであれば、[Creating a Waterfall](https://github.com/ryanvolum/menu-bot/wiki/C%23-Menu-Bot-Wiki#creating-a-waterfall-dialog)に進んでください。

### 状態の設定
最初に `MemoryStorage`インスタンスを作成し、それを` ConversationState`コンストラクタに渡すことでStartup.csに `ConversationState`インスタンスを作成します。

```js
const memoryStorage = new MemoryStorage();
const conversationState = new ConversationState(memoryStorage);
```
`MemoryStorage`コンストラクターにストレージプロバイダー（Azure Tables、Azure Cosmosなど）を渡して、`ConversationState`がその外部プロバイダーに対して状態を取得および設定できるようにします。今のところは空のままにしておき、代わりにすべてをメモリに保存します。

次に、その `ConversationState`インスタンスでボットを初期化します。

```js
const myBot = new FoodBot(conversationState);
```

### ウォーターフォールダイアログの作成
ボットの状態を保存する場所ができたので、 `dialogState`プロパティと`DialogSet`を作成します。ボットクラスのこれら両プロパティを使用して、ボットを最適に編成します。

```js
constructor(conversationState) {
     this.conversationState = conversationState;
     this.dialogState = this.conversationState.createProperty(DIALOG_STATE_PROPERTY);
     this.dialogs = new DialogSet(this.dialogState);
```
これでウォーターフォールダイアログを作成して `DialogSet`に追加することができます。ウォーターフォールダイアログには、それを永続化して再利用するための一意の文字列名があります。この場合、ダイアログの名前として `MENU_DIALOG`変数（コードの先頭で宣言されています）を使用しています。

```js
        this.dialogs.add(new WaterfallDialog(MENU_DIALOG, [
            this.promptForMenu,
            this.handleMenuResult,
            this.resetDialog,
        ]));
```
前述のように、 `WaterfallDialog`は連続して実行される単なる関数の配列です。このウォーターフォールダイアログは、それぞれ`step`と呼ばれる3つの関数で構成されています。これらの関数を匿名で宣言することもできますが（インラインでも名前なしでも）、コードを整理するためにこのように参照することにしました。最初の関数を見てみましょう。

```js
    async promptForMenu(step) {
        return step.prompt(MENU_PROMPT, {
            choices: ["Donate Food", "Find a Food Bank", "Contact Food Bank"],
            prompt: "Do you have food to donate, do you need food, or are you contacting a food bank?",
            retryPrompt: "I'm sorry, that wasn't a valid response. Please select one of the options"
        });
    }
``` 

この関数では、「寄付する」、「フードバンクを探す」、「フードバンクに連絡する」の3つの選択肢でユーザーを促します。ダイアログに名前が付いているように、プロンプトにも名前を付ける必要があります。この場合、この選択プロンプトを `MENU_PROMPT`という名前で呼び出しています。また、この選択プロンプトをボットのコンストラクタに登録して、ボット全体で再利用できるようにします。

```js
this.dialogs.add(new ChoicePrompt(MENU_PROMPT));
```
プロンプトを呼び出すとき、私たちは3つのプロパティ（`choices`、`prompt`、 `retryPrompt`）を定義しました。 `choices`はオプションの配列で、最終的には1ターンだけ存在するボタン`suggestedActions`としてレンダリングされます。 `prompt`は実際にユーザーにプロンプ​​トを出したいテキストです。 `retryPrompt`は、ユーザーがボタンのテキスト以外のもので答えた場合にボットが使うテキストです。

次のダイアログステップでは、ユーザーの回答を解析します。

```js
    async handleMenuResult(step) {
        switch (step.result.value) {
            case "Donate Food":
                return step.beginDialog(DONATE_FOOD_DIALOG);
            case "Find a Food Bank":
                return step.beginDialog(FIND_FOOD_DIALOG);
            case "Contact Food Bank":
                return step.beginDialog(CONTACT_DIALOG);
        }
        return step.next();
    }
```

ご覧のとおり、ユーザーの応答の値は `step.result.value`に表示されます。私たちはスイッチケースを使って、彼らがどの答えを出したかを判断し、必要に応じて次のダイアログを始めます。私たちのスイッチケースには `default`ハンドラがないことに注意してください。これは、ユーザーが有効な入力の1つを入れた（またはボタンをクリックした）場合にのみ、ダイアログがこのステップに到達するようにするためです。

どのボタンがクリックされたかに応じて、定義した他のダイアログの名前を付けて `step.beginDialog`を呼び出します。 [コンポーネントダイアログの作成](https://github.com/ryanvolum/menu-bot/wiki#creating-component-dialogs)でこれらの他のダイアログをどのように定義したかについて説明します。しかし、今のところ `beginDialog`は別のダイアログをダイアログスタックにプッシュして、後続のメッセージはそれが終了するまで新しいダイアログに送信されるようにするだけで十分です。そのダイアログが終了すると、メインメニューダイアログの最後のステップに戻ります。

```js
async resetDialog(step) {
    return step.replaceDialog(MENU_DIALOG);
}
```
この1行の機能により、「メッセージループ」を構築することができます。基本的に、会話の流れが終わるとこのステップにたどり着き、メインメニューの最初に戻ります。これを実現するには、 `step.replaceDialog`関数を使って、現在のメインメニューダイアログを自分自身で置き換えます。ボットはメインメニューダイアログの最初のステップから始まり、特定のターンに何ができるかについてユーザーが混乱しないように明確にします。

### OnTurnAsync関数からのトップレベルダイアログの調整

`WaterfallDialog`を作成して`DialogSet`に追加、ボットにいつ開始そして続行するかを知らせる必要があります。このコードの大部分は、ダイアログを使用するボットにとってかなり定型的なものになるでしょう。手短に復習すると、 `onTurn`関数は1ターンごとに呼ばれる関数です。つまり、ユーザーからメッセージを受け取るたびに、 `onTurn`関数を実行します。 `onTurn`はパラメータとしてコンテキストオブジェクトを受け取ります。これは入ってくるアクティビティと会話を調整するためのいくつかの会話ヘルパーをまとめたものです。それでは、このボットの `onTurn`関数を見てみましょう。

`dialogContext`を作成することから始めます。

```js
    async onTurn(turnContext) {
        const dialogContext = await this.dialogs.createContext(turnContext);
```
このダイアログコンテキストには、ダイアログの状態を評価するためのヘルパーが含まれています。この場合は、アクティブなダイアログがあるかどうかを判断し、ある場合はそれを続行するために使用します。これは、ユーザーからメッセージを受信したとき（Message Activities）にのみ行われることに注意してください。

```js
        if (turnContext.activity.type === ActivityTypes.Message) {
            if (dialogContext.activeDialog) {
                await dialogContext.continueDialog();
...
```

アクティブなダイアログがない場合は、メインメニューダイアログを起動します。

```js
            } else {
                await dialogContext.beginDialog(MENU_DIALOG);
            }
```

`onTurn`からダ​​イアログを開始または継続すると、ダイアログは適切なステップを実行し（ウォーターフォール関数）、をその後戻ります。したがって、 `onTurn`関数の最後に、すべての状態変化を保存することが重要です。 `ConversationState`インスタンスを通して、ダイアログの状態を保存していることを覚えておいてください。実際に `conversationState.saveChanges`を呼び出さないと、それらは保持されず、その後のダイアログステップに進むことはありません。

```js
        await this.conversationState.saveChanges(turnContext);
```
`onTurn` 関数の残りの部分は、 すでに[Welcoming the User](https://github.com/ryanvolum/menu-bot/wiki/C%23-Menu-Bot-Wiki/Home/_edit#welcoming-the-user)で確認したウェルカムコードを含んでいます。

### コンポーネントダイアログの作成

トップレベルのウォーターフォールダイアログを見てきたので、このボットの他のダイアログを見てみましょう。プロジェクトディレクトリを移動すると、 `DonateFoodDialog.js`、`FindFoodDialog.js`、 `ContactDialog.js`の3つのファイルを持つ`dialogs`という名前のフォルダが見つかります。いくつかの理由で、これらのダイアログを `bot.js`の外側で宣言します。 どのひとつには、1つのファイルにすべての会話フローを構築するのは難しいためです。同じファイル内で他の開発者と共同作業することはほぼ不可能です。ダイアログを分離することで、それらを再利用可能なモジュールとして扱うこともできます。同じボット内でそれらを複数回使用することも、他のボットで使用するために公開することもできます。このモジュールの振る舞いを実現するために、私たちはダイアログまたは複数のダイアログのためのモジュールとして機能する `ComponentDialogs`を使用します。 `FindFoodDialog`を見てみましょう。

私たちのダイアログは `ComponentDialog`を継承し、dialogIdを取ります。

```js
class DonateFoodDialog extends ComponentDialog {
    constructor(dialogId) {
        super(dialogId);
```
それから `this.initialDialogId`をウォーターフォールダイアログの名前に設定します。

```js
// ID of the child dialog that should be started anytime the component is started.
this.initialDialogId = dialogId;
```
この場合、ウォーターフォールダイアログはコンポーネントダイアログの唯一のダイアログであるため、コンポーネントダイアログと同じ名前になります。明示的に `this.initialDialogId`を設定しないことを選択した場合、それは自動的にComponentDialogに追加された最初のダイアログに設定されます。次に、メインメニューダイアログと同じように、ウォーターフォールダイアログで使用する `ChoicePrompt`を追加します。

```js
this.addDialog(new ChoicePrompt('choicePrompt'));
```
そしてウォーターフォールダイアログを追加します。

```js
this.addDialog(new WaterfallDialog(dialogId, [
    async function (step) {
        return await step.prompt('choicePrompt', {
            choices: getValidPickupDays(),
            prompt: "What day would you like to pickup food?",
            retryPrompt: "That's not a valid day! Please choose a valid day."
        });
    },
    async function (step) {
        const day = step.result.value;
        let filteredFoodBanks = filterFoodBanksByPickup(day);
        let carousel = createFoodBankPickupCarousel(filteredFoodBanks)
        return step.context.sendActivity(carousel);
    }
]));
```
ウォーターフォールのステップをクラスのメンバーとして宣言するのではなく、それらを無名関数としてインラインでコーディングしたことに注意してください。これらの関数は1回しか使用しないことがわかっているので、コード・フットプリントがかなり小さいので、ここではこのアプローチを採用しました。

メインメニューダイアログと同様に、このウォーターフォールは機能の配列です。最初のものはユーザーに日数の配列を要求します（JSONスケジュールを処理するヘルパーメソッドによって決定されます）。 2番目のカードは、選択した日にフードバンクが開いていることを示すカードのカルーセルを作成することによって応答を処理します。カードとカルーセルを作成するために、 `botbuilder`パッケージの`CardFactory`を使います。完全な実装を見るために `schedule-helpers`を確認し、[メッセージにメディアを追加する](https://docs.microsoft.com/ja-jp/azure/bot-service/bot-builder-howto-add-media-attachments?view=azure-bot-service-4.0&tabs=csharp) もチェックしてください。ヒーローカードとカルーセルの詳細については、Azureのドキュメントを参照してください。

### ダイアログを介した状態の存続
後のステップで使用するために、ダイアログが情報を保持する必要がある場合があります。以下の連絡先に関する会話フローを見てください。

![ContactDialog GIF](https://github.com/ryanvolum/menu-bot/blob/master/wiki_assets/ContactDialog.gif)

**注意**: このフローは実際にフードバンクにメッセージを送るわけではないので、自分でボットをテストしてください。

ユーザーのメールアドレス、送信したいメッセージ、送信先のフードバンクの名前が収集されていることがわかります。私たちはその情報をすべて使ってフードバンクにメッセージを送ります。しかし、対話の存続期間を通してこの情報をどのようにして永続化しているのでしょうか？ 

`ContactDialog`ダイアログを見てみましょう。他のコンポーネントダイアログと同様に、ボットが一度に1つずつ実行する関数の配列を匿名で作成しています。

```
this.addDialog(new WaterfallDialog(dialogId, [
...
]
``` 
これらの関数は、（`ChoicePrompt`を使って）連絡したいフードバンクの名前、（`TextPrompt`を使って）メールアドレス、そして（`TextPrompt`を使って）送信したいメッセージの入力を促します。また、 `ConfirmPrompt`を使用してメッセージを送信したいかどうかを確認するようにユーザーに求めます。あとで必要になることがわかっている部分をユーザーから収集したら、それを `step.values`辞書に保存します。

```
// Persist the email address for later waterfall steps to be able to access it
step.values.email = step.result;
```
そしてあとで、同じ `step.values`辞書でそのプロパティにアクセスできます。
```
sendFoodBankMessage(step.result.foodBankName, step.result.message, step.result.email);
```