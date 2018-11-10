> [!IMPORTANT]
> プライマリ ボット ターンが完了すると、それを処理していたスレッドによってコンテキスト オブジェクトの破棄処理が行われます。 **いずれのアクティビティ呼び出しに対しても必ず `await` を実行**します。これにより、プライマリ スレッドでは生成されたアクティビティで待機してから、その処理が終了され、ターン コンテキストの破棄が行われます。 そうしないと、応答 (そのハンドラーも含まれる) にかなりの時間がかかり、コンテキスト オブジェクトに基づいた処理が試みられた場合、"_コンテキスト破棄済み_" エラーが返されることがあります。