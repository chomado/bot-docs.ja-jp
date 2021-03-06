---
title: ボット設定の構成 - Bot Service
description: Azure Portal を使用してボットのさまざまなオプションを構成する方法について説明します。
keywords: ボット設定の構成, 表示名, アイコン, Application Insights, 設定ブレード
author: v-royhar
ms.author: kamrani
manager: kamrani
ms.topic: article
ms.service: bot-service
ms.date: 12/13/2017
ms.openlocfilehash: a34c99e6995835341cb610525d8033567d914e5c
ms.sourcegitcommit: 9d77f3aff9521d819e88efd0fbd19d469b9919e7
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 04/16/2020
ms.locfileid: "81395705"
---
# <a name="configure-bot-settings"></a>ボット設定の構成

ボット設定 (表示名、アイコン、Application Insights など) は、 **[設定]** ブレードで表示および変更できます。

![ボット設定ブレード](~/media/bot-service-portal-configure-settings/bot-settings-blade.png)

以下に示すのは、 **[設定]** ブレードのフィールドの一覧です。

| フィールド | 説明 |
| :---  | :---        |
| アイコン | 各チャネルでボットを視覚的に識別したり、Cortana などのサービスでアイコンとして使用するためのカスタム アイコンです。 このアイコンは PNG 形式である必要があり、サイズは 30 K 以下である必要があります。 この値はいつでも変更できます。 |
| Display name | チャネルやディレクトリでのボットの名前です。 この値はいつでも変更できます。 文字数は最大 35 文字です。 |
| ボット ハンドル | ボットの一意識別子です。 この値は、Bot Service を使ってボットを作成した後には変更できません。 |
| メッセージング エンドポイント | ボットと通信するためのエンドポイントです。 |
| Microsoft アプリ ID | ボットの一意識別子です。 この値は変更できません。 **[管理]** リンクをクリックすると、新しいパスワードを生成できます。 |
| Application Insights インストルメンテーション キー | ボット テレメトリの一意キーです。 このボットのボット テレメトリを受信する場合は、このフィールドに Azure Application Insights キーをコピーします。 この値は省略可能です。 Azure Portal で作成されたボットに対しては、このキーが自動生成されます。 このフィールドについて詳しくは、「[Application Insights キー](~/bot-service-resources-app-insights-keys.md)」をご覧ください。 |
| Application Insights API キー | ボット分析の一意キーです。 ボットに関する分析をダッシュ ボードに表示する場合は、このフィールドに Azure Application Insights API キーをコピーします。 この値は省略可能です。 このフィールドについて詳しくは、「[Application Insights キー](~/bot-service-resources-app-insights-keys.md)」をご覧ください。 |
| Application Insights アプリケーション ID | ボット分析の一意キーです。 ボットに関する分析をダッシュ ボードに表示する場合は、このフィールドに Azure Insights アプリケーション ID キーをコピーします。 この値は省略可能です。 Azure Portal で作成されたボットに対しては、このキーが自動生成されます。 このフィールドについて詳しくは、「[Application Insights キー](~/bot-service-resources-app-insights-keys.md)」をご覧ください。 |

ボットの設定を変更したら、クリックして、ブレードの上部にある **[保存]** ボタンをクリックして、新しいボット設定を保存します。

## <a name="additional-information"></a>追加情報

[az bot update](https://docs.microsoft.com/cli/azure/bot?view=azure-cli-latest#az-bot-update) を使用すると、コマンド ラインからボット設定を更新できます。

## <a name="next-steps"></a>次のステップ

ここでは、ボット サービスの設定を構成する方法について説明しました。次は、音声プライミングを構成する方法について学習しましょう。
> [!div class="nextstepaction"]
> [音声プライミング](bot-service-manage-speech-priming.md)
