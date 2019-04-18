# メニューボット

このボットを通していくつかの概念を説明します。ここでは、このボットが完全な誘導フローを提示し、それによってユーザーはどんな返答にも選択肢を受け取ります。ボットが会話を自由回答形式のままにしてしまうと、ユーザーは何をすべきか、何を言うべきか分からなくなってしまうため、 会話をボタンによって誘導することで（あなたが自然言語もサポートしていても）この問題を軽減するのに役立ちます。ガイド付き会話は、最初にユーザーを歓迎し、次にウォーターフォールダイアログとユーザーとのコミュニケーションを促すプロンプトを使用して構築します。これが、このボットの会話の流れの1つをちょっとのぞいてみたものです。

![DonateDialog GIF](https://github.com/ryanvolum/menu-bot/blob/master/wiki_assets/DonateDialog.gif)

## ユーザーを歓迎する
もしボットがプロアクティブにユーザーを歓迎しなければ、ユーザーはそのボットが何をしてくれるのか分かりません。詳細は[BotBuilder document](https://docs.microsoft.com/ja-jp/azure/bot-service/bot-builder-send-welcome-message?view=azure-bot-service-4.0&tabs=csharp%2Ccsharpmulti%2Ccsharpwelcomeback)を確認してください。
この場合、 `OnTurnAsync`関数で入ってくる`ConversationUpdate`アクティビティを監視し、新しいメンバーが追加されたことを確認し、メッセージを送信し、そしてダイアログを開始することでユーザーを歓迎します。

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

メンバージョイン関数は、メンバーが会話に参加したかどうかを（ボットとは対照的に）判断する複雑な部分を抽象化するための単なるヘルパーです。

```csharp
private bool MemberJoined(Activity activity)
{
    return activity.MembersAdded.Count != 0 && (activity.MembersAdded[0].Id != activity.Recipient.Id);
}
```

## ウォーターフォールダイアログとプロンプト
このサンプルでは、​​ボットビルダーのウォーターフォールダイアログとプロンプトという概念を用いて会話をモデル化しています。ウォーターフォールダイアログは、会話の各ターンに段階的に実行するための関数の配列です。ウォータフォールダイアログは、特定の一連のステップが常に発生する定型の会話を表現するのに最適です。たくさんの文脈の切り替えが発生する会話を表現するのにはあまり適していません。この場合、私たちの[food bank bot](https://github.com/ryanvolum/food-bank-bot)は、ユーザーが必要な情報を入手するために、きちんと管理されたフローをたどることを期待しているので、ウォーターフォールダイアログは最適な選択です。

ダイアログを設定する前に、私たちはそれらの状態を保存するための場所があることを確認する必要があります。特に、ダイアログは正しいステップを開始するために、会話のどこにいるのかを知る必要があります。状態の管理をどのようにして行うのかを説明しましょう。すでに設定しているのであれば、[Creating a Waterfall](https://github.com/ryanvolum/menu-bot/wiki/C%23-Menu-Bot-Wiki#creating-a-waterfall-dialog)に進んでください。

### 状態の設定 

最初に `MemoryStorage`インスタンスを作成し、それを` ConversationState`コンストラクタに渡すことでStartup.csに `ConversationState`インスタンスを作成します。

```csharp
IStorage dataStore = new MemoryStorage();
// Create Conversation State object.
// The Conversation State object is where we persist anything at the conversation-scope.
var conversationState = new ConversationState(dataStore);
```
`MemoryStorage`コンストラクターにストレージプロバイダー（Azure Tables、Azure Cosmosなど）を渡して、`ConversationState`がその外部プロバイダーに対して状態を取得および設定できるようにします。今のところは空のままにしておき、代わりにすべてをメモリに保存します。

次に、Dependency Injection によって、IBotコンストラクターに渡すことを可能にするservicesコレクションに会話の状態`ConversationState`を登録することで、私たちのボットがその `ConversationState`インスタンスにアクセスできることを確認します。

```csharp
// Register conversation state.
services.AddSingleton<BotState>(conversationState);
```
私たちのすべてのダイアログは `DialogSet`内で作成され実行されなければならず、DialogSetのコンストラクタは` IStatePropertyAccessor <DialogState> `を必要とします。このサンプルでは、​​カスタムDialogSetを使用します。これを作成し、そして同時にConversationState内に `dialogState`プロパティも作成します。

```csharp
// The dialogset will need a state store accessor. Since the state accessor is only going to be used by the FoodBotDialogSet, use  
// AddSingleton to register and inline create the FoodBotDialogSet, at the same time creating the the property accessor in the
// conversation state
services.AddSingleton(sp => new FoodBotDialogSet(conversationState.CreateProperty<DialogState>("FOODBOTDIALOGSTATE")));
```

カスタム `DialogSet`クラスは、シンプルにすべての子ダイアログのコンテナとして機能します。それはトップレベルメニューを定義し、それ自身が他の子ダイアログを呼び出す単一の子ダイアログ `MainMenuDialog`を含みます。各ダイアログはそれを保持し再利用するための一意の文字列名を持ちます。この場合、MainMenuDialogインスタンスのダイアログIDとして文字列 `mainMenuDialog`を使用します。

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

私たちは `FoodBotDialogSet`インスタンスを登録したので、それを`BotState`（これはStartup.csで作成し登録したConversationStateオブジェクトです）と共にDependency Injection を使ってBotクラスにいれることができます。

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

### OnTurnAsync関数からのトップレベルダイアログの調整

`MainMenuDialog`を作成して`FoodBotDialogSet`に追加、そしてそれらをBotに挿入したので、トップレベルのダイアログをいつ開始そして続行するか、このボットに知らせる必要があります。このコードの大部分は、ダイアログを使用するボットにとってかなり定型的なものになるでしょう。手短に復習すると、 `OnTurnAsync`関数は1ターンごとに呼ばれる関数です。つまり、ユーザーからメッセージを受け取るたびに、 `OnTurnAsync`関数を実行します。 `OnTurnAsync`はパラメータとしてコンテキストオブジェクトを受け取ります。これは入ってくるアクティビティと会話を調整するためのいくつかの会話ヘルパーをまとめたものです。それでは、このボットの `OnTurnAsync`関数を見てみましょう。

`DialogSet`の`DialogContext`を作成することから始めます。

```csharp
    public async Task OnTurnAsync(ITurnContext turnContext, CancellationToken cancellationToken = default(CancellationToken))
    {
        var dialogContext = await this._foodBotDialogSet.CreateContextAsync(turnContext, cancellationToken);
```
このダイアログコンテキストには、ダイアログの状態を評価するためのヘルパーが含まれています。この場合は、アクティブなダイアログがあるかどうかを判断し、ある場合はそれを続行するために使用します。これは、ユーザーからメッセージを受信したとき（Message Activities）にのみ行われることに注意してください。

```csharp
    if (turnContext.Activity.Type == ActivityTypes.Message)
    {
        if (dialogContext.ActiveDialog != null)
        {
            await dialogContext.ContinueDialogAsync(cancellationToken);
...
```

アクティブなダイアログがない場合は、 `FoodBotDialogSet.StartDialogId`静的プロパティを通して公開されるメインメニューダイアログの文字列識別子を使用して、メインメニューダイアログを起動します。

```csharp
        }
        else
        {
            await dialogContext.BeginDialogAsync(FoodBotDialogSet.StartDialogId, null, cancellationToken);
        }
```
`OnTurnAsync`からダイアログを開始または続行すると、ダイアログは適切なステップを実行し（ウォーターフォール関数）、その後戻ります。したがって、 `OnTurnAsync`関数の最後に、すべての状態変化を保存することが重要です。 `_botState`フィールドを介してアクセス可能な`ConversationState`インスタンスを通して、ダイアログの状態を保存していることを覚えておいてください。実際に`_botState_.SaveChangesAsync`を呼び出さないと、それらは保持されず、その後のダイアログステップに進むことはありません。

```csharp
    // Always persist changes at the end of every turn, here this is the dialog state in the conversation state.
    await _botState.SaveChangesAsync(turnContext, cancellationToken: cancellationToken);
```
`OnTurnAsync` 関数の残りの部分は、 すでに[Welcoming the User](https://github.com/ryanvolum/menu-bot/wiki/C%23-Menu-Bot-Wiki/Home/_edit#welcoming-the-user)で確認したウェルカムコードを含んでいます。

### ウォーターフォールダイアログの作成
ボットにその状態を保存する場所、 `DialogSet`、そしてDialogSetのトップレベルダイアログ用に`BeginDialogAsync`を呼び出すコードがあるで、そのトップレベルダイアログの実装を見てみましょう。このサンプルのすべての子ダイアログと同様に、これは**コンポーネントダイアログ**として実装されています。プロジェクトディレクトリをナビゲートすると、5つのファイル（`FoodBotDialogSet.cs`（既に解説したもの）、 `MainMenuDialog.cs`、 `DonateFoodDialog.cs`、`FindFoodDialog.cs` 、 `ContactDialog.cs`）がある `Dialogs`というフォルダが見つかります。これらのダイアログは、いくつかの理由で `FoodBot.cs`の外側で宣言しています。そのひとつには、1つのファイルにすべての会話フローを構築するのは難しいためです。同じファイル内で他の開発者と共同作業することはほぼ不可能です。ダイアログを分離することで、それらを再利用可能なモジュールとして扱うこともできます。同じボット内でそれらを複数回使用したり、他のボットで使用するために公開したりすることもできます。このモジュールの振る舞いを実現するために、私たちはダイアログまたは複数のダイアログのためのモジュールとして機能する `ComponentDialogs`を使用します。

私たちの `MainMenuDialog`は`ComponentDialog`を継承し、dialogIdを取ります。

```csharp
public class MainMenuDialog : ComponentDialog
{
    public MainMenuDialog(string dialogId) : base(dialogId)
    {
        ...
```
dialogIdの値はComponentDialogの名前で、インスタンス化する時に設定されます。この場合は `FoodBotDialogSet`クラスによって設定されます。これはもちろん、開始するダイアログを指定するためにbotsの`OnTurnAsync`メソッドで使用されるものと同じdialogIdです。

そしてコンストラクタは `this.initialDialogId`をダイアログの名前に設定します。

```csharp
    // ID of the child dialog that should be started anytime the component is started.
    this.InitialDialogId = dialogId;
```
これでウォーターフォールダイアログを作成して、それを `ComponentDialog`に追加することができます。ウォーターフォールダイアログには、それを永続化して再利用するための一意の文字列名もあります。この場合、ウォーターフォールダイアログはコンポーネントダイアログの唯一のダイアログであるため、コンポーネントダイアログと同じ名前になります。明示的に `this.initialDialogId`を設定しないことを選択した場合、それは自動的にComponentDialogに追加された最初のダイアログに設定されます。 

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
前述のように、 `WaterfallDialog`は連続して実行される単なる関数の配列です。このウォーターフォールダイアログは、それぞれ`step`と呼ばれる3つの関数で構成されています。これらの関数を匿名で宣言することもできますが（インラインでも名前なしでも）、コードを整理するためにこのように参照することにしました。最初の関数を見てみましょう。

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
この関数では、「寄付する」、「フードバンクを探す」、「フードバンクに連絡する」の3つの選択肢でユーザーを促します。ダイアログに名前が付いているように、プロンプトにも名前を付ける必要があります。この場合、この選択プロンプトをクラスの先頭で宣言されている `MENUPROMPT`という名前で呼び出しています。また、この選択プロンプトをボットのコンストラクタに登録して、ボット全体で再利用できるようにします。

```csharp
    this.AddDialog(new ChoicePrompt(MENUPROMPT));
```
プロンプトを呼び出すとき、私たちは3つのプロパティ（`Choices`、`Prompt`、 `RetryPrompt`）を定義しました。 `Choices`はオプションの配列で、最終的には1ターンだけしか存在しないボタン`SuggestedActions`としてレンダリングされます。 `Prompt`は実際にユーザーにプロンプ​​トを出したいテキストです。 `RetryPrompt`は、ユーザーがボタンのテキスト以外のもので答えた場合にボットが使うテキストです。

次のダイアログステップでは、ユーザーの回答を解析します。 

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

ご覧のとおり、ユーザーの応答の値は `stepContext.Result.Value`に表示されます。私たちはスイッチケースを使って、彼らがどの答えを出したか判断し、必要に応じて次のダイアログを始めます。私たちのスイッチケースには `default`ハンドラロジックがないことに注意してください。これは、ユーザーの入力が `ChoicePrompt`によって検証され、ユーザーが有効な入力の1つを入れた（またはボタンをクリックした）場合にのみ、ダイアログがこのメソッドに到達するようにするためです。

どのボタンがクリックされたかに応じて、定義した他のダイアログの名前で `BeginDialogAsync`メソッドを呼び出します。 [他のコンポーネントダイアログの作成](https://github.com/ryanvolum/menu-bot/wiki/C%23-Menu-Bot-Wiki#creating-the-)でこれらの他のダイアログをどのように定義したかについて説明します。しかし、今のところ `BeginDialogAsync`は別のダイアログをダイアログスタックにプッシュして、後続のメッセージはそれが終了するまで新しいダイアログに送信されるようにするだけで十分です。そのダイアログが終了すると、メインメニューダイアログの最後のステップに戻ります。

```csharp
    private async Task<DialogTurnResult> ResetDialogAsync(WaterfallStepContext stepContext, CancellationToken cancellationToken)
    {
        return await stepContext.ReplaceDialogAsync(this.InitialDialogId, null, cancellationToken);
    }
```
この1行の機能により、「メッセージループ」を構築することができます。基本的に、会話の流れが終わるとこのステップにたどり着き、メインメニューの最初に戻ります。これを実現するには、 `ReplaceDialogAsync`関数を使って、現在のメインメニューダイアログを自分自身で置き換えます。ボットはメインメニューダイアログの最初のステップから始まり、特定のターンに何ができるかについてユーザーが混乱しないように明確にします。

### 他のコンポーネントダイアログの作成
トップレベルのウォーターフォールダイアログを見てきたので、このボットの他のダイアログを見ていきます。 `FindFoodDialog`を見てみましょう。

私たちのダイアログは `ComponentDialog`を継承し、dialogIdを取ります。

```csharp
public class DonateFoodDialog : ComponentDialog
{
    public DonateFoodDialog(string dialogId) : base(dialogId)
    {
        ...
```

そしてコンストラクタは `this.initialDialogId`をウォーターフォールダイアログの名前に設定します。

```csharp
    // ID of the child dialog that should be started anytime the component is started.
    this.InitialDialogId = dialogId;
```
次に、メインメニューダイアログと同じように、ウォーターフォールダイアログで使用する `ChoicePrompt`を追加します。

```csharp
    this.AddDialog(new ChoicePrompt("choicePrompt"));
```

そしてウォーターフォールダイアログを追加します。

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

ウォーターフォールのステップをクラスのメンバーとして宣言するのではなく、それらを無名関数としてインラインでコーディングしたことに注意してください。これらの関数は1回しか使用しないことがわかっている上、コード・フットプリントがかなり小さいので、ここではこのアプローチを採用しました。

メインメニューダイアログと同様に、このウォーターフォールは機能の配列です。最初のものはユーザーに日数の配列を要求します（JSONスケジュールを処理するヘルパーメソッドによって決定されます）。 2番目のカードは、選択した日にフードバンクが開いていることを示すカードのカルーセルを作成することによって応答を処理します。カードとカルーセルを作成するために、 `BotBuilder`パッケージの`CardFactory`を使います。完全な実装を見るために `UXHelpers.cs`を確認し、[メッセージにメディアを追加する](https://docs.microsoft.com/ja-jp/azure/bot-service/bot-builder-howto-add-media-attachments?view=azure-bot-service-4.0&tabs=csharp) もチェックしてください。 ヒーローカードとカルーセルの詳細については、Azureのドキュメントを参照してください。

### ダイアログを介した状態の存続 

後のステップで使用するために、ダイアログが情報を保持する必要がある場合があります。以下の連絡先に関する会話フローを見てください。

![ContactDialog GIF](https://github.com/ryanvolum/menu-bot/blob/master/wiki_assets/ContactDialog.gif)

 **注意**: このフローは実際にフードバンクにメッセージを送るわけではないので、自分でボットをテストしてください。


ユーザーのメールアドレス、送信したいメッセージ、送信先のフードバンクの名前が収集されていることがわかります。私たちはその情報をすべて使ってフードバンクにメッセージを送ります。しかし、対話の存続期間を通してこの情報をどのようにして永続化しているのでしょうか？ 


`ContactDialog`ダイアログを見てみましょう。他のコンポーネントダイアログと同様に、ボットが一度に1つずつ実行する関数の配列を匿名で作成しています。

```csharp
    this.AddDialog(
        new WaterfallDialog(dialogId, new WaterfallStep[]
            {
                ...
```

これらの関数は、（`ChoicePrompt`を使って）連絡したいフードバンクの名前、（`TextPrompt`を使って）メールアドレス、そして（`TextPrompt`を使って）送信したいメッセージの入力を促します。また、 `ConfirmPrompt`を使用してメッセージを送信したいかどうかを確認するようにユーザーに求めます。あとで必要になることがわかっている部分をユーザーから収集したら、それを**WaterfallStepContext** `values`辞書に保存します。

```csharp
    const string EMAILADDRESS = "emailAddress";
    ...
    stepContext.Values.Add(EMAILADDRESS, (string)stepContext.Result);
```

そしてあとで、同じ `values`辞書でそのプロパティにアクセスできます。

```csharp
    ScheduleHelpers.SendFoodbankMessage(
        (string)stepContext.Values[FOODBANKNAME], 
        (string)stepContext.Values[EMAILADDRESS], 
        (string)stepContext.Values[MESSAGE]);
```
