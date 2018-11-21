---
title: ボットのカスタム ストレージの実装 | Microsoft Docs
description: Bot Builder SDK v4.0 でカスタム ストレージを構築する方法
keywords: カスタム, ストレージ, 状態, ダイアログ
author: johnataylor
ms.author: johtaylo
manager: kamrani
ms.topic: article
ms.service: bot-service
ms.subservice: sdk
ms.date: 10/31/2018
monikerRange: azure-bot-service-4.0
ms.openlocfilehash: b005b9f024c5813ba22cd8663c196a8c3a5bb716
ms.sourcegitcommit: 15f7fa40b7e0a05507cdc66adf75bcfc9533e781
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 11/02/2018
ms.locfileid: "50919008"
---
# <a name="implement-custom-storage-for-your-bot"></a>ボットのカスタム ストレージの実装

ボットの対話は 3 つの領域に分かれています。1 つ目は Azure Bot Service とのアクティビティの交換、2 つ目は Store によるダイアログの状態の読み込みと保存、3 つ目はボットがジョブを完了するために連携する必要があるその他のバックエンド サービスです。

![スケールアウト ダイアグラム](../media/scale-out/scale-out-interaction.png)

この記事では、ボットと Azure Bot Service および Store との対話に関するセマンティクスについて説明します。

Bot Builder Framework には既定の実装が含まれています。ほとんどの場合、この実装は多くのアプリケーションのニーズに合致します。これを使用するために行う必要があるのは、数行の初期化コードで各要素をつなぎ合わせることだけです。 サンプルの多くはそれだけを示しています。

ただし、ここでの目的は、既定の実装のセマンティクスがアプリケーションで思いどおりに機能しない場合に何をすればよいかについて説明することです。 基本的なポイントは、これはフレームワークであり、動作が固定されているあらかじめ用意されたアプリケーションではないということです。つまり、フレームワーク内の多くのメカニズムの実装は既定の実装にすぎず、唯一の実装ではありません。

具体的には、フレームワークは、Azure Bot Service とのアクティビティの交換とボットの状態の読み込み/保存との関係を示しているわけではありません。既定の実装を提供するだけです。 この点をさらに詳しく説明するために、セマンティクスの異なる別の実装を開発します。 この代替ソリューションは、フレームワークに同様に適合し、開発するアプリケーションにより適している可能性もあります。 これはすべてシナリオに左右されます。

## <a name="behavior-of-the-default-botframeworkadapter-and-storage-providers"></a>既定の BotFrameworkAdapter と Storage プロバイダーの動作

まず、次のシーケンス ダイアグラムが示す、フレームワーク パッケージに含まれている既定の実装を確認しましょう。

![スケールアウト ダイアグラム](../media/scale-out/scale-out-default.png)

ボットは、アクティビティを受信すると、この会話に対応する状態を読み込みます。 次に、この状態と受信したアクティビティを使用してダイアログ ロジックを実行します。 ダイアログを実行するプロセスでは、1 つ以上の送信アクティビティが作成され、即座に送信されます。 ダイアログの処理が完了すると、ボットは更新された状態を保存し、新しい状態で古い状態を上書きします。

この動作で問題が発生する可能性があるいくつかの点は検討に値します。

第 1 に、何らかの理由で保存操作が失敗した場合、応答を受け取ったユーザーは状態が前進していると思っていますが、実際にはそうではないため、状態はチャネル上で表示されている内容と暗黙的に一致しなくなっています。 この場合、状態が正常であり、応答メッセージの送受信が成功した場合よりも一般に状況は悪化しています。 これは、会話の設計に影響を及ぼす可能性があります。たとえば、ダイアログに、ユーザーとの追加のあるいは冗長な確認のやり取りが含まれることがあります。 

第 2 に、実装が複数のノードにスケールアウトされて展開されている場合、状態が誤って上書きされる可能性があります。ダイアログで、確認メッセージを伝達するチャネルに複数のアクティビティが送信された可能性があるため、特に混乱が生じるおそれがあります。 ピザ注文ボットの例を考えてみましょう。ユーザーがトッピングをたずねられたときにマッシュルームを追加し、すぐにチーズを追加した場合、複数のインスタンスが実行されているスケールアウト シナリオでは、ボットを実行しているさまざまなマシンに、後続のアクティビティが同時に送信される可能性があります。 この状況が発生すると、あるマシンが別のマシンによって書き込まれた状態を上書きする "競合状態" と呼ばれる状態になります。 しかし、ここでは、応答が既に送信されているため、ユーザーはマッシュルームとチーズの両方が追加されたという確認を受け取っています。 残念ながら、ピザが届いても、含まれているのはマッシュルームまたはチーズだけであり、両方は含まれていません。

## <a name="optimistic-locking"></a>オプティミスティック ロック

ソリューションは、状態に関する何らかのロックを導入することです。 ここで使用するロックの方式は、オプティミスティック ロックと呼ばれます。あらゆるものを、それぞれ実行されている唯一のものであるかのように実行することができ、処理が完了した後で同時実行違反を検出するためです。 これは複雑に思われるかもしれませんが、クラウド ストレージ テクノロジと Bot Framework の適切な拡張ポイントを使用することで簡単に構築できます。

エンティティ タグ (ETag) ヘッダーに基づく標準の HTTP メカニズムを使用します。 このメカニズムを理解しておくことは、この後のコードを理解する上で不可欠となります。 次のダイアグラムはこのシーケンスを示しています。

![スケールアウト ダイアグラム](../media/scale-out/scale-out-precondition-failed.png)

このダイアグラムは、あるリソースの更新を実行する 2 つのクライアントのケースを示しています。 クライアントが GET 要求を発行し、サーバーからリソースが返されるときに、リソースに ETag ヘッダーが添付されます。 ETag ヘッダーは、リソースの状態を表す非透過的な値です。 リソースが変更されると、ETag が更新されます。 クライアントは状態の更新を完了すると、サーバーに POST 要求を送信します。クライアントは、前提条件の If-Match ヘッダーに以前に受け取った ETag 値を添付してこの要求を行います。 この ETag が、(いずれかのクライアントに対するいずれかの応答で) サーバーから最後に返された値と一致しない場合、前提条件チェックは 412 Precondition Failure で失敗します。 このエラーは、リソースが更新されたという POST 要求を行っているクライアントに対するインジケーターです。 このエラーが発生した場合、リソースの GET 要求を再度発行し、必要な更新を適用して、リソースの POST 要求を送信するのがクライアントの標準的な動作です。 他のクライアントがリソースを更新していなければ、この 2 番目の POST は成功します。リソースが更新されている場合、クライアントは再試行する必要があります。

リソースを保持しているクライアントが処理を進めますが、リソース自体は他のクライアントが何の制限もなくアクセスできるという意味で "ロック" されていないため、このプロセスは "オプティミスティック" と呼ばれます。 リソースの状態に対するクライアント間の競合は、処理が完了するまで確認されません。 一般に、分散システムでは、この戦略は対照的な "ペシミスティック" アプローチよりも最適です。

ここで説明したオプティミスティック ロック メカニズムは、プログラム ロジックを安全に再試行できることを前提としています。言うまでもなく、ここで考慮すべき重要事項は、外部サービス呼び出しはどうなるかということです。 この場合、最適なソリューションは、これらのサービスをべき等にすることができるかどうかです。 コンピューター サイエンスでは、べき等操作とは、同じ入力パラメーターを使用して複数回呼び出されても付加的影響がない操作のことです。 GET、PUT、DELETE を実装する純粋な HTTP REST サービスがこれに該当します。 ここでの論拠は直感的です。処理を再試行し、その再試行の一環として必要な呼び出しが再実行されたときに、付加的影響を及ぼさないことが望まれます。 これを説明するために、環境が理想的な状態であり、この記事の冒頭にあるシステム図の右側に示すバックエンド サービスがすべてべき等の HTTP REST サービスであると仮定し、ここからはアクティビティの交換だけに注目していきます。

## <a name="buffering-outbound-activities"></a>送信アクティビティのバッファー処理

アクティビティの送信はべき等操作ではなく、エンド ツー エンドのシナリオではあまり意味がないことも明らかです。 結局のところ、アクティビティは、ビューに追加したり、場合によってはテキスト読み上げエージェントが読み上げたりするメッセージを伝達するだけであることも少なくありません。

アクティビティの送信で回避する必要がある重要なことは、アクティビティを何度も送信することです。 ここでの問題は、オプティミスティック ロック メカニズムでは、場合によってはロジックを何度も再実行する必要があることです。 ソリューションは単純です。ロジックを再実行しないことを確認するまで、ダイアログからの送信アクティビティをバッファー処理する必要があります。 これは、保存操作が正常に完了するまでです。 次のようなフローが必要です。

![スケールアウト ダイアグラム](../media/scale-out/scale-out-buffer.png)

ダイアログ実行について再試行ループを構築できれば、保存操作で前提条件エラーが発生したときに次の動作が得られます。

![スケールアウト ダイアグラム](../media/scale-out/scale-out-save.png)

このメカニズムを適用し、以前の例を再検討すると、注文に追加されるピザのトッピングの誤った肯定応答が発生することはありません。 実際には、展開を複数のマシンにスケールアウトしましたが、オプティミスティック ロック スキームによって、状態の更新を効果的にシリアル化しました。 ピザの注文では、完全な状態を正確に反映するように、項目の追加の確認応答を書き込むこともできるようになります。 たとえば、ユーザーが即座に「チーズ」と入力した場合、ボットが "マッシュルーム" に応答する機会を得る前に、"チーズ付きピザ" とその後の "チーズとマッシュルーム付きピザ" の 2 つの応答が可能になります。

シーケンス ダイアグラムを見ると、保存操作が正常に完了しても、エンド ツー エンド通信のどこかで応答が失われる可能性があることがわかります。 ポイントは、これは状態管理インフラストラクチャが修正できる問題ではないということです。 上位レベルのプロトコルと、場合によっては、チャネルのユーザーを関与させるプロトコルが必要となります。 たとえば、ボットが応答していないように見える場合は、最終的にやり直すか、そうした何らかの操作をユーザーに求めるのが妥当です。 このような一時的な停止が発生することがあるシナリオではこれは妥当ですが、誤った肯定応答や他の意図しないメッセージを除外できることをユーザーに期待するのは妥当性に欠けます。 

新しいカスタム ストレージ ソリューションでは、このすべてをまとめて、フレームワークの既定の実装では実行されない 3 つのことを実行します。 第 1 に、ETag を使用して競合を検出し、第 2 に、ETag エラーが検出されたときに処理を再試行します。第 3 に、保存が正常に完了するまで送信アクティビティをバッファー処理します。 この記事の残りの部分では、この 3 つの部分の実装について説明します。

## <a name="implementing-etag-support"></a>ETag サポートの実装

単体テストをサポートするために、まず、ETag をサポートする新しいストアのインターフェイスを定義します。 インターフェイスがあれば、ネットワークに到達する必要なくメモリ内で実行される単体テスト用と、運用環境用の 2 つのバージョンを記述できます。 このインターフェイスにより、ASP.NET の依存関係挿入メカニズムを簡単に活用できるようになります。

インターフェイスは、Load メソッドと Save メソッドで構成されます。 どちらも状態に使用するキーを取得します。 Load はデータと関連する ETag を返します。 Save はこれらを取得します。 さらに、Save はブール値を返します。 このブール値は、ETag が一致したかどうかと、保存が成功したかどうかを示します。 これは、一般的なエラーのインジケーターではなく、前提条件エラーのインジケーターとして使用することを目的としています。これを中心とする制御フロー ロジックは再試行ループの形で記述するため、例外ではなく、リターン コードとしてこれをモデル化します。

この最下位レベルのストレージ要素はプラグ可能にするので、この要素にはシリアル化の要件を適用しないようにします。ただし、コンテンツの保存は JSON にするよう指定します。これにより、ストア実装でコンテンツの種類を設定できます。 .NET でこれを行う最も簡単で最も自然な方法は、引数の型を使用することです。具体的には、コンテンツ引数を JObject として型指定します。 JavaScript または TypeScript では、これは通常のネイティブ オブジェクトです。  

最終的に作成するインターフェイスを次に示します。

```csharp
public interface IStore
{
  Task<(JObject content, string eTag)> LoadAsync(string key);
  Task<bool> SaveAsync(string key, JObject content, string eTag);
}
```
Azure Blob Storage に対してこれを実装するのは簡単です。
```csharp
public class BlobStore : IStore
{
  private CloudBlobContainer _container;

  public BlobStore(string myAccountName, string myAccountKey, string containerName)
  {
    var storageCredentials = new StorageCredentials(myAccountName, myAccountKey);
    var cloudStorageAccount = new CloudStorageAccount(storageCredentials, useHttps: true);
    var client = cloudStorageAccount.CreateCloudBlobClient();
    _container = client.GetContainerReference(containerName);
  }

  public async Task<(JObject content, string eTag)> LoadAsync(string key)
  {
    var blob = _container.GetBlockBlobReference(key);
    try
    {
      var content = await blob.DownloadTextAsync();
      var obj = JObject.Parse(content);
      var eTag = blob.Properties.ETag;
      return (obj, eTag);
    }
    catch (StorageException e)
      when (e.RequestInformation.HttpStatusCode ==
        (int)HttpStatusCode.NotFound)
    {
      return (new JObject(), null);
    }
  }

  public async Task<bool> SaveAsync(string key, JObject obj, string eTag)
  {
    var blob = _container.GetBlockBlobReference(key);
    blob.Properties.ContentType = "application/json";
    var content = obj.ToString();
    if (eTag != null)
    {
      try
      {
        await blob.UploadTextAsync(content,
          new AccessCondition { IfMatchETag = eTag },
          new BlobRequestOptions(),
          new OperationContext());
      }
      catch (StorageException e)
        when (e.RequestInformation.HttpStatusCode ==
          (int)HttpStatusCode.PreconditionFailed)
      {
        return false;
      }
    }
    else
    {
      await blob.UploadTextAsync(content);
    }
    return true;
  }
}
```

ご覧のとおり、ここでは Azure Blob Storage が実際の作業を行っています。 特定の例外のキャッチと、呼び出し元のコードで想定されている内容に合わせて、キャッチがどのように変換されているかに注意してください。 つまり、読み込み時の Not Found 例外では null を返し、保存時の Precondition Failed 例外ではブール値を返します。

このソース コードはすべて、対応する[サンプル](https://aka.ms/scale-out)で提供されます。このサンプルには、メモリ ストア実装が含まれます。

## <a name="implementing-the-retry-loop"></a>再試行ループの実装
ループの基本的な形は、シーケンス ダイアグラムに示されている動作から直接派生します。

アクティビティの受信時に、その会話の対応する状態のキーを作成します。 アクティビティと会話の状態の関係は変更しないので、状態の既定の実装とまったく同じ方法でキーを作成します。

適切なキーを作成したら、対応する状態の読み込みを試みます。 次に、ボットのダイアログを実行し、保存を試みます。 その保存が成功した場合は、ダイアログを実行した結果として実行される送信アクティビティを送信します。 それ以外の場合は、元に戻って読み込み前からプロセス全体を繰り返します。 読み込みを再実行すると、新しい ETag が提供されるので、うまくいけば、次回は保存が成功します。

最終的に作成する OnTurn 実装は次のようになります。
```csharp
public async Task OnTurnAsync(ITurnContext turnContext,
  CancellationToken cancellationToken = default(CancellationToken))
{
  // Create the storage key for this conversation.
  string key = $"{turnContext.Activity.ChannelId}/conversations/{turnContext.Activity.Conversation?.Id}";

  // The execution sits in a loop because there might be a retry if the save operation fails.
  while (true)
  {
    // Load any existing state associated with this key
    var (oldState, etag) = await _store.LoadAsync(key);

    // Run the dialog system with the old state and inbound activity,
    // resulting in a new state and outbound activities.
    var (activities, newState) = await DialogHost.RunAsync(_rootDialog, turnContext.Activity, oldState);

    // Save the updated state associated with this key.
    bool success = await _store.SaveAsync(key, newState, etag);

    // Following a successful save, send any outbound Activities, otherwise retry everything.
    if (success)
    {
      if (activities.Any())
      {
        // This is an actual send on the TurnContext we were given and so will actual do a send this time.
        await turnContext.SendActivitiesAsync(activities);
      }
      break;
    }
  }
}
```
ダイアログ実行を関数呼び出しとしてモデル化していることに注意してください。 さらに高度な実装では、インターフェイスを定義し、この依存関係を挿入可能にすることも考えられますが、ここでは、すべてのダイアログを静的関数の背後に置くことで、このアプローチの機能的特性を強調しています。 一般に、重要な部分が機能するように実装を構成すると、ネットワーク上で正常に動作させることに関して非常に良好な環境になります。


## <a name="implementing-outbound-activity-buffering"></a>送信アクティビティのバッファー処理の実装 

次の要件は、保存が正常に実行されるまで送信アクティビティをバッファー処理することです。 これには、カスタム BotAdapter の実装が必要になります。 このコードでは、アクティビティを送信するのではなくリストに追加する、抽象 SendActivity 関数を実装します。 ホストするダイアログに変わりはありません。
このシナリオでは、UpdateActivity 操作と DeleteActivity 操作はサポートされていないので、これらのメソッドから Not Implemented がスローされるだけです。 また、SendActivity からの戻り値も考慮していません。 これは、アクティビティの更新を送信する必要があるシナリオ (チャネルで表示されているカードのボタンの無効化など) で、一部のチャネルによって使用されます。 特に状態が必要な場合、これらのメッセージ交換は複雑になる可能性がありますが、これについてはこの記事では取り上げません。 カスタム BotAdapter の完全な実装は次のようになります。

```csharp
public class DialogHostAdapter : BotAdapter
{
  private List<Activity> _response = new List<Activity>();

  public IEnumerable<Activity> Activities => _response;

  public override Task<ResourceResponse[]> SendActivitiesAsync(ITurnContext turnContext,
    Activity[] activities, CancellationToken cancellationToken)
  {
    foreach (var activity in activities)
    {
      _response.Add(activity);
    }
    return Task.FromResult(new ResourceResponse[0]);
  }

  public override Task DeleteActivityAsync(ITurnContext turnContext,
    ConversationReference reference, CancellationToken cancellationToken)
  {
    throw new NotImplementedException();
  }

  public override Task<ResourceResponse> UpdateActivityAsync(ITurnContext turnContext,
    Activity activity, CancellationToken cancellationToken)
  {
    throw new NotImplementedException();
  }
}
```
あと残っている作業は統合だけです。これらのさまざまな新しい要素をつなぎ合わせ、フレームワークの既存の要素に組み込みます。 メインの再試行ループは、IBot OnTurn 関数内にあります。 これには、テストのために依存関係を挿入可能にしたカスタム IStore 実装が保持されています。 ここでは、単一のパブリック静的関数を公開する DialogHost というクラスに、すべてのダイアログ ホスティング コードを配置しました。 この関数は、受信アクティビティと古い状態を取得し、結果としてのアクティビティと新しい状態を返すように定義されています。

この関数で最初に行うことは、既に紹介したカスタム BotAdapter を作成することです。 その後は、DialogSet と DialogContext を作成し、通常の Continue または Begin フローを実行して、通常とまったく同様にダイアログを実行するだけです。 ここで説明しなかった唯一の要素は、カスタム アクセサーの必要性です。 これは、ダイアログの状態をダイアログ システムに渡すことを容易にする非常に単純な shim であることがわかります。 アクセサーでは、ダイアログ システムを操作するときに参照セマンティクスを使用するので、必要なのはハンドルを渡すことだけです。 さらにわかりやすくするために、使用するクラス テンプレートを参照セマンティクスに制限しました。

階層化は慎重に行います。実装によってそれぞれ異なる方法でシリアル化する可能性がある場合に、JsonSerialization がプラグ可能なストレージ層内にあるのは望ましくないため、ホスティング コードにインラインで配置します。

ドライバー コードを次に示します。
```csharp
public class DialogHost
{
  private static readonly JsonSerializer StateJsonSerializer = new JsonSerializer()
    { TypeNameHandling = TypeNameHandling.All };

  public static async Task<Tuple<Activity[], JObject>> RunAsync(Dialog rootDialog,
    Activity activity, JObject oldState)
  {
    // A custom adapter and corresponding TurnContext that buffers any messages sent.
    var adapter = new DialogHostAdapter();
    var turnContext = new TurnContext(adapter, activity);

    // Run the dialog using this TurnContext with the existing state.
    JObject newState = await RunTurnAsync(rootDialog, turnContext, oldState);

    // The result is a set of activities to send and a replacement state.
    return Tuple.Create(adapter.Activities.ToArray(), newState);
  }

  private static async Task<JObject> RunTurnAsync(Dialog rootDialog,
    TurnContext turnContext, JObject state)
  {
    if (turnContext.Activity.Type == ActivityTypes.Message)
    {
      // If we have some state, deserialize it. (This mimics the shape produced by BotState.cs.)
      var dialogState = state?[nameof(DialogState)]?.ToObject<DialogState>(StateJsonSerializer);

      // A custom accessor is used to pass a handle on the state to the dialog system.
      var accessor = new RefAccessor<DialogState>(dialogState);

      // The following is regular dialog driver code.
      var dialogs = new DialogSet(accessor);
      dialogs.Add(rootDialog);

      var dialogContext = await dialogs.CreateContextAsync(turnContext);
      var results = await dialogContext.ContinueDialogAsync();

      if (results.Status == DialogTurnStatus.Empty)
      {
        await dialogContext.BeginDialogAsync("root");
      }

      // Serialize the result, and put its value back into a new JObject.
      return new JObject
      {
        { nameof(DialogState), JObject.FromObject(accessor.Value, StateJsonSerializer) }
      };
    }

    return state;
  }
}
```
最後に、カスタム アクセサーについては、状態が参照に基づくため、実装する必要があるのは Set だけです。
```csharp
public class RefAccessor<T> : IStatePropertyAccessor<T> where T : class
{
  public RefAccessor(T value)
  {
    Value = value;
  }

  public T Value { get; private set; }

  public string Name => nameof(T);

  public Task<T> GetAsync(ITurnContext turnContext, Func<T> defaultValueFactory = null,
    CancellationToken cancellationToken = default(CancellationToken))
  {
    if (Value == null)
    {
      if (defaultValueFactory == null)
      {
        throw new KeyNotFoundException();
      }
      else
      {
        Value = defaultValueFactory();
      }
    }
    return Task.FromResult(Value);
  }

  public Task DeleteAsync(ITurnContext turnContext,
    CancellationToken cancellationToken = default(CancellationToken))
  {
    throw new NotImplementedException();
  }

  public Task SetAsync(ITurnContext turnContext, T value,
    CancellationToken cancellationToken = default(CancellationToken))
  {
    throw new NotImplementedException();
  }
}
```

## <a name="additional-resources"></a>その他のリソース
この記事で使用する [C#](http://aka.ms/scale-out) サンプル コードは、GitHub で入手できます。
