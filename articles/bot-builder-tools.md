---
title: Bot Builder ツールを使用してボットを管理する
description: Bot Builder ツールを使用すると、コマンド ラインから直接、ボットのリソースを管理できます。
keywords: botbuilder テンプレート, ludown, qna, luis, msbot
author: ivorb
ms.author: v-ivorb
manager: kamrani
ms.topic: article
ms.service: bot-service
ms.subservice: tools
ms.date: 09/18/2018
monikerRange: azure-bot-service-4.0
ms.openlocfilehash: ef57cdf6a202679f9fc3a83e3e44640b43adb67f
ms.sourcegitcommit: b78fe3d8dd604c4f7233740658a229e85b8535dd
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 10/24/2018
ms.locfileid: "49998362"
---
# <a name="bot-builder-tools"></a>Bot Builder ツール

Bot Builder [ツール][cliTools]は、計画、ビルド、テスト、発行、接続、評価の各フェーズを含むエンド ツー エンドのボット開発ワークフローに対応しています。 これらのツールが開発サイクルの各フェーズでどのように役立つかを見ていきましょう。

[プラン](#plan)
- まず、ボットの[設計ガイドライン](https://docs.microsoft.com/en-us/azure/bot-service/bot-service-design-principles)でベスト プラクティスを確認する
- [Chatdown](#create-mock-conversations-using-chatdown) ツールを使用して模擬会話を作成する

[ビルド](#build)
- [Ludown](#bootstrap-language-understanding-with-ludown) を使用して言語の理解を行う
- [MSBot](#keep-track-of-service-references-using-bot-file) を使用してサービス参照を追跡する
- [LUIS CLI](#create-and-manage-luis-applications-using-luis-cli) を使用して LUIS アプリケーションを作成、管理する
- [QnA Maker CLI](#create-qna-maker-kb-using-qna-maker-cli) を使用して QnA Maker KB を作成する
- [Dispatch CLI](#create-dipsatch-model-using-dispatch-cli) を使用してディスパッチ モデルを作成する

[Test](#test)
- [Bot Framework Emulator V4](https://aka.ms/bot-framework-emulator-v4-overview) を使用してボットをテストする

[[発行]](#publish)
- [Azure CLI][azureCli] を使用して、ボットの作成、ダウンロード、Azure Bot Service への発行を行う

[接続](#configure-channels)
- [Azure CLI][azureCli] を使用してボットを Azure Bot Service チャネルに接続する

## <a name="plan"></a>プラン

### <a name="create-mock-conversations-using-chatdown"></a>Chatdown を使用して模擬会話を作成する

Chatdown は、.chat ファイルを使用して模擬トランスクリプトを生成するトランスクリプト ジェネレーターです。 生成された模擬トランスクリプト ファイルは stdout に出力されます。

正常なアプリケーションや Web サイトをはじめとする適切なボットを作るには、まずサポート対象のシナリオを明確にすることが必要です。 ボットとユーザーの間で行われる会話のモックアップを作成することは、次の点で役立ちます。

- ボットでサポートされるシナリオのフレーム化。
- レビューしてフィードバックを行うビジネスの意思決定者。
- ユーザーとボット .chat ファイル形式の間の会話フローを通じて "ハッピー パス" (およびその他のパス) を定義すると、ユーザーとボットの間で行われる会話のモックアップを作成するのに役立ちます。 Chatdown CLI ツールにより、.chat ファイルは会話トランスクリプト (.transcript ファイル) に変換されます。このファイルは、[Bot Framework Emulator V4](https://github.com/microsoft/botframework-emulator) で表示できます。

`.chat` ファイルの例を次に示します。

```markdown
user=Joe
bot=LulaBot

bot: Hi!
user: yo!
bot: [Typing][Delay=3000]
Greetings!
What would you like to do?
* update - You can update your account
* List - You can list your data
* help - you can get help

user: I need the bot framework logo.

bot:
Here you go.
[Attachment=bot-framework.png]
[Attachment=http://yahoo.com/bot-framework.png]
[AttachmentLayout=carousel]

user: thanks
bot:
Here's a form for you
[Attachment=card.json adaptivecard]

```

### <a name="create-a-transcript-file-from-chat-file"></a>.chat ファイルからトランスクリプト ファイルを作成する
Chatdown コマンドは、次のようになります。

```bash
chatdown sample.chat > sample.transcript
```

これにより、`sample.chat` を使用して `sample.transcript` が出力されます。 詳細については、[Chatdown CLI][chatdown] に関するページを参照してください。

## <a name="build"></a>構築
### <a name="create-a-luis-application-with-ludown"></a>LUDown を使用して LUIS アプリケーションを作成する
LUDown ツールは、LUIS と QnA の両方の新しい .json モデルを作成するために使用できます。  
LUIS ポータルから行うのと同じように、LUIS アプリケーションの[意図](https://docs.microsoft.com/en-us/azure/cognitive-services/luis/add-intents)と[エンティティ](https://docs.microsoft.com/en-us/azure/cognitive-services/luis/add-entities)を定義することができます。

'#\<intent-name\>' には、新しい意図の定義セクションを記述します。 その後の各行は、その意図を表す[発話](https://docs.microsoft.com/en-us/azure/cognitive-services/luis/add-example-utterances)の一覧です。

たとえば、次のように、1 つの .lu ファイルに複数の LUIS 意図を作成できます。 

```LUDown
# Greeting
- Hi
- Hello
- Good morning
- Good evening

# Help
- help
- I need help
- please help
```

### <a name="create-qna-pairs-with-ludown"></a>LUDown を使用して QnA ペアを作成する

.lu ファイル形式では、次の表記を使用して、QnA ペアもサポートしています。 

```LUDown
> comment
### ? question ?
  ```markdown
    answer
  ```

LUDown ツールでは、質問と回答が自動的に QnA Maker の JSON ファイルに分けられます。この JSON ファイルを使用して、新しい [QnaMaker.ai](http://qnamaker.ai) ナレッジ ベースを作成することができます。

```LUDown
### ? How do I change the default message for QnA Maker?
  ```markdown
  You can change the default message if you use the QnAMakerDialog. 
  See [this link](https://docs.botframework.com/en-us/azure-bot-service/templates/qnamaker/#navtitle) for details. 
  ```

また、同じ回答に対して複数の質問を追加することもできます。これは、1 つの回答に対して質問のバリエーションを新しい行で追加するだけでできます。

```LUDown
### ? What is your name?
- What should I call you?
  ```markdown
    I'm the echoBot! Nice to meet you.
  ```

### <a name="generate-json-models-with-ludown"></a>LUDown を使用して .json モデルを生成する

.lu 形式で LUIS または QnA の言語コンポーネントを定義すると、LUIS .json、QnA .json または QnA .tsv ファイルに発行できます。 LUDown ツールは、実行すると、同じ作業ディレクトリ内のすべての .lu ファイルを検索して解析します。 LUDown ツールは、.lu ファイルを使用することで、LUIS または QnA のどちらもターゲットにできるため、一般的なコマンド **ludown parse <Service> -- in <luFile>** を使用して、どの言語サービス用に生成するかを指定するだけで済みます。 

サンプルの作業ディレクトリには、解析する 2 つの .lu ファイル (LUIS モデルを作成する '1.lu' と QnA ナレッジ ベースを作成する 'qna1.lu') があります。

#### <a name="generate-luis-json-models"></a>LUIS .json モデルを生成する

LUDown を使用して LUIS モデルを生成するには、現在の作業ディレクトリで以下を入力するだけです。

```shell
ludown parse ToLuis --in <luFile>
```

#### <a name="generate-qna-knowledge-base"></a>QnA ナレッジ ベースを生成する

同様に、QnA ナレッジ ベースを作成するには、解析の対象を変更するだけです。

```shell
ludown parse ToQna --in <luFile> 
```

その結果として生成される JSON ファイルは、LUIS または QnA のそれぞれのポータルを通じて、または新しい CLI ツールを使用して使用することができます。

詳細については、[LUdown CLI][ludown] の GitHub リポジトリを参照してください。
## <a name="track-service-references-using-bot-file"></a>.bot ファイルを使用してサービス参照を追跡する

新しい [MSBot][msbotCli] ツールを使用すると、**.bot** ファイルを作成できます。これにより、ボットが使用するさまざまなサービスに関するすべてのメタデータを 1 か所に格納できます。 このファイルを使用すると、CLI からこれらのサービスにボットを接続することもできます。 このツールは、npm モジュールとして使用できます。インストールして実行するには、次のコマンドを使用します。

```shell
npm install -g msbot
```

ボット ファイルを作成するには、CLI から「**msbot init**」を入力し、続けてボットの名前とターゲット URL エンドポイントを入力します。次に例を示します。

```shell
msbot init --name TestBot --endpoint http://localhost:9499/api/messages
```
ボットをサービスに接続するには、CLI で「**msbot connect**」を入力し、続けて該当するサービスを入力します。

```shell
msbot connect [Service]
```

サポートされるサービスの一覧を取得するには、[readme][msbotCli] ファイルを参照してください。

## <a name="create-and-manage-luis-applications-using-luis-cli"></a>LUIS CLI を使用して LUIS アプリケーションを作成、管理する

新しいツール セットには、LUIS リソースを個別に管理できるようにする [LUIS 拡張機能][luisCli]が含まれています。 これは、ダウンロード可能な npm モジュールとして使用できます。

```shell
npm install -g luis-apis
```
CLI からの LUIS ツールの基本的なコマンドの使用方法は、次のとおりです。

```shell
luis <action> <resource> <args...>
```
ボットを LUIS に接続するには、**.luisrc** ファイルを作成する必要があります。 これは、アプリケーションが発信呼び出しを行う際に、サービス エンドポイントへの LUIS appID とパスワードをプロビジョニングする構成ファイルです。 このファイルは、次のように **luis init** を実行することで作成できます。

```shell
luis init
```
ツールがファイルを生成する前に、端末で自分の LUIS オーサリング キー、リージョン、appID の入力を求められます。  

このファイルが生成されると、CLI から次のコマンドを使用して、アプリケーションで (LUDown から生成された) LUIS .json ファイルを使用できるようになります。

```shell
luis import application --in luis-app.json | msbot connect luis --stdin
```
詳細については、[LUIS CLI][luisCli] の GitHub リポジトリを参照してください。

## <a name="create-qna-maker-kb-using-qna-maker-cli"></a>QnA Maker CLI を使用して QnA Maker KB を作成する

新しいツール セットには、LUIS リソースを個別に管理できるようにする [QnA 拡張機能][qnaCli]が含まれています。 これは、ダウンロード可能な npm モジュールとして使用できます。

```shell
npm install -g qnamaker
```
QnA Maker ツールを使用すると、ナレッジ ベースの作成、更新、発行、削除、トレーニングができます。 [ludown parse toqna](#generate-qna-knowledge-base) コマンドを通じて生成されたファイルを使用して、ナレッジ ベースを作成/置換することができます。

```shell
qnamaker create --in qnaKB.json --msbot | msbot connect qna --stdin
```

詳細については、[QnA Maker CLI][qnaCli] の GitHub リポジトリを参照してください。

## <a name="create-dispatch-model-using-dispatch-cli"></a>Dispatch CLI を使用してディスパッチ モデルを作成する

Dispatch は、複数のボット モジュール (LUIS モデル、QnA ナレッジ ベースなど) 間で意図をディスパッチするのに使用される LUIS モデルを作成、評価するためのツールです (ディスパッチにファイルの種類として追加)。

次の場合にディスパッチ モデルを使用します。

- ボットが複数のモジュールで構成されていて、これらのモジュールにユーザーの発話をルーティングするうえでサポートが必要で、ボットの統合を評価する場合。
- 単一の LUIS モデルの意図の分類の品質を評価する場合。
- テキスト ファイルからテキスト分類モデルを作成する場合。

ボットで利用する [LUIS アプリケーション][msbotCli-luis]と [QnA Maker ナレッジ ベース][msbotCli-qna]を使用して .bot ファイルをアセンブルしたら、次を使用して簡単にディスパッチ モデルをビルドできます。 

```shell
dispatch create -b <YOUR-BOT-FILE> | msbot connect dispatch --stdin
```
詳細については、[Disptach CLI][dispatchCli] のページを参照してください。

## <a name="test"></a>テスト

[Bot Framework Emulator](https://github.com/Microsoft/BotFramework-Emulator/releases) は、ボット開発者がローカル ホストで、またはトンネルを介したリモート実行時にボットをテストおよびデバッグできるデスクトップ アプリケーションです。

## <a name="publish"></a>[発行]

[Azure CLI][azureCli] を使用して、ボットの[作成](#create-azure-bot-service-bot)、[ダウンロード](#download-azure-bot-service-bot)、Azure Bot Service への[発行](#publish-azure-bot-service-bot)を行うことができます。 次を使用して、ボットの拡張機能をインストールします。 
```shell
az extension add -n botservice
```

## <a name="create-azure-bot-service-bot"></a>Azure Bot Service のボットを作成する

次を使用して、Azure アカウントにログインします。 
```shell
az login
```

ログインすると、次を使用して新しい Azure Bot Service のボットを作成できます。 
```shell
az bot create [options]
```

ボットを作成し、ボット構成で .bot ファイルを更新するには、次を使用します。  
```shell
az bot create [options] --msbot | msbot connect bot --stdin
```

既存のボットがある場合は、次を使用します。  
```shell
az bot show [options] --msbot | msbot connect bot --stdin
```

| オプション                            | 説明                                   |
|-----------------------------------|-----------------------------------------------|
| --kind -k (必須)              | ボットの種類。  使用できる値: function、registration、webapp。|
| --name -n (必須)              | ボットのリソース名。 |
| --appid                           | ボットで使用する msa アカウント ID。   |
| --location -l                     | 場所。 `az configure --defaults location=<location>` を使用して、既定の場所を構成できます。  既定値: westus。|
| --msbot                           | .bot ファイルと互換性のある JSON として出力を表示します。  使用できる値: false、true。|
| --password -p                     | 開発者ポータルからのボットの msa パスワード。 |
| --resource-group -g               | リソース グループの名前。 `az configure --defaults group=<name>` を使用して、既定のグループを構成できます。  既定値: build2018。 |
| --tags                            | ボットに追加する一連のタグ。 |


### <a name="configure-channels"></a>チャネルを構成する

Azure CLI を使用して、ボットのチャネルを管理できます。 
```shell
>az bot -h
Group
   az bot: Manage Bot Services.
    Subgroups:
        directline: Manage Directline Channel on a Bot.
        email     : Manage Email Channel on a Bot.
        facebook  : Manage Facebook Channel on a Bot.
        kik       : Manage Kik Channel on a Bot.
        msteams   : Manage Msteams Channel on a Bot.
        skype     : Manage Skype Channel on a Bot.
        slack     : Manage Slack Channel on a Bot.
        sms       : Manage Sms Channel on a Bot.
        telegram  : Manage Telegram Channel on a Bot.
        webchat   : Manage Webchat Channel on a Bot.

    Commands:
        create    : Create a new Bot Service.
        delete    : Delete an existing Bot Service.
        download  : Download an existing Bot Service.
        publish   : Publish to an existing Bot Service.
        show      : Get an existing Bot Service.
        update    : Update an existing Bot Service.

```

## <a name="additional-information"></a>追加情報
- [Bot Builder ツール][cliTools]

<!-- Footnote links -->

[cliTools]: https://aka.ms/botbuilder-tools-readme
[azureCli]: https://aka.ms/botbuilder-tools-azureCli
[msbotCli]: https://aka.ms/botbuilder-tools-msbot-readme
[msbotCli-luis]: https://aka.ms/botbuilder-tools-msbot-readme#connecting-to-luis-application
[msbotCli-qna]: https://aka.ms/botbuilder-tools-msbot-readme#connecting-to-qna-maker-knowledge-base
[chatdown]: https://aka.ms/botbuilder-tools-chatdown
[ludown]: https://aka.ms/botbuilder-ludown
[luisCli]: https://aka.ms/botbuilder-luis-cli
[qnaCli]: https://aka.ms/botbuilder-tools-qnaMaker
[dispatchCli]: https://aka.ms/botbuilder-tools-dispatch