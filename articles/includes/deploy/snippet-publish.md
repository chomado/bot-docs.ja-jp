---
ms.openlocfilehash: bb7520e8e99ad6326d7d00d8190dae306bf11afa
ms.sourcegitcommit: a47183f5d1c2b2454c4a06c0f292d7c075612cdd
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 06/19/2019
ms.locfileid: "67252597"
---
ローカルのボットを Azure に公開します。 この手順には時間がかかる場合があります。

```cmd
az bot publish --name <bot-resource-name> --proj-file-path "<project-file-name>" --resource-group <resource-group-name> --code-dir <directory-path> --verbose --version v4
```

| オプション | 説明 |
|:---|:---|
| --name | Azure 内のボットのリソース名。 |
| --proj-file-path | C# の場合は、公開する必要があるスタートアップ プロジェクト ファイル名 (.csproj は含めません) を使用します。 (例: `EnterpriseBot`)。 Node.js の場合は、ボットのメイン エントリ ポイントを使用します。 たとえば、「 `index.js` 」のように入力します。 |
| --resource-group | リソース グループの名前。 |
| --code-dir | ボット コードをアップロードする元のディレクトリ。 |

これが完了すると、"Deployment successful!" (デプロイに成功しました!) メッセージが表示され、ボットは Azure にデプロイされます。
