---
title: Durable Functions のパターンおよび技術的概念 - Azure
description: クラウド内のステートフルなコード実行の利点を活用するために Azure 内で Durable Functions が動作するしくみに関する詳細を示します。
services: functions
author: kashimiz
manager: jeconnoc
keywords: ''
ms.service: azure-functions
ms.devlang: multiple
ms.topic: conceptual
ms.date: 12/06/2018
ms.author: azfuncdf
ms.openlocfilehash: 6eb08af9cdd19bc83d44d29874f6ac58b41ed8c8
ms.sourcegitcommit: f863ed1ba25ef3ec32bd188c28153044124cacbc
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 02/15/2019
ms.locfileid: "56302042"
---
# <a name="durable-functions-patterns-and-technical-concepts"></a>Durable Functions のパターンおよび技術的概念

*Durable Functions* は、サーバーレス環境でステートフル関数を記述できる、[Azure Functions](../functions-overview.md) と [Azure WebJobs](../../app-service/web-sites-create-web-jobs.md) の拡張機能です。 この拡張機能は状態、チェックポイント、再起動を管理します。 この記事では、Azure Functions 向けの Durable Functions 拡張機能の動作と一般的な実装パターンについて、より詳しい情報を提供します。

> [!NOTE]
> Durable Functions は Azure Functions の高度な拡張機能であり、すべてのアプリケーションに適しているわけではありません。 この記事は、開発者が [Azure Functions](../functions-overview.md) のコンセプトを十分に理解しており、サーバーレス アプリケーションの開発に取り組んでいることを前提としています。

## <a name="patterns"></a>パターン

このセクションでは、Durable Functions を使用することでメリットがある、いくつかの典型的なアプリケーションのパターンを示します。

### <a name="chaining"></a>パターン #1: 関数チェーン

*関数チェーン*は、特定の順序で関数のシーケンスを実行するパターンです。 ある関数の出力が、別の関数の入力に適用される必要がある、ということがよくあります。

![関数チェーンの図](./media/durable-functions-concepts/function-chaining.png)

Durable Functions を使用すれば、このパターンをコードで簡潔に実装できます。

#### <a name="c-script"></a>C# スクリプト

```cs
public static async Task<object> Run(DurableOrchestrationContext context)
{
    try
    {
        var x = await context.CallActivityAsync<object>("F1");
        var y = await context.CallActivityAsync<object>("F2", x);
        var z = await context.CallActivityAsync<object>("F3", y);
        return  await context.CallActivityAsync<object>("F4", z);
    }
    catch (Exception)
    {
        // error handling/compensation goes here
    }
}
```

> [!NOTE]
> C# で記述中のプリコンパイル済みの Durable Functions と、前述した C# サンプル スクリプトには、若干の違いがあります。 C# でプリコンパイル済みの関数には、それぞれの属性で修飾される durable パラメーターが必要になります。 例としては、`DurableOrchestrationContext` パラメーターの `[OrchestrationTrigger]` 属性があります。 パラメーターが正しく修飾されていない場合、ランタイムが変数を関数に挿入できなくなり、エラーが発生します。 その他の例については、[サンプル](https://github.com/Azure/azure-functions-durable-extension/blob/master/samples)をご覧ください。

#### <a name="javascript-functions-2x-only"></a>JavaScript (Functions 2.x のみ)

```js
const df = require("durable-functions");

module.exports = df.orchestrator(function*(context) {
    const x = yield context.df.callActivity("F1");
    const y = yield context.df.callActivity("F2", x);
    const z = yield context.df.callActivity("F3", y);
    return yield context.df.callActivity("F4", z);
});
```

"F1"、"F2"、"F3"、"F4" という値は、関数アプリ内の他の関数の名前です。 制御フローは、通常の命令型のコーディング構造で実装されます。 つまり、コードは上から下へ実行され、コードに条件文やループなどの既存言語の制御フロー セマンティクスを含めることができます。  エラー処理ロジックを try ブロック、catch ブロック、finally ブロックに含めることもできます。

`context` パラメーター ([DurableOrchestrationContext]) (\(.NET\)) と `context.df` オブジェクト (JavaScript) は名前によって他の関数を呼び出し、パラメーターを渡して、関数の出力を返すメソッドを提供します。 コードが `await` (C#) または `yield` (JavaScript) を呼び出すたびに Durable Functions フレームワークは、現在の関数インスタンスの進行状況に "*チェックポイントを設定*" します。 プロセスまたは VM が実行途中でリサイクルされる場合、その関数インスタンスは先の `await` または `yield` 呼び出しから再開されます。 この再起動動作については、後で詳しく説明します。

> [!NOTE]
> JavaScript の `context` オブジェクトは、[DurableOrchestrationContext] ではなく、[関数コンテキスト](../functions-reference-node.md#context-object)全体を表しています。

### <a name="fan-in-out"></a>パターン #2: ファンアウト/ファンイン

*ファンアウト/ファンイン*は、複数の関数を並列に実行してすべてが完了するまで待機するパターンです。  複数の関数から返された結果に基づいて集計作業が行われる場合があります。

![ファンアウト/ファンインの図](./media/durable-functions-concepts/fan-out-fan-in.png)

通常の関数では、関数が複数のメッセージを 1 つのキューに送信することでファンアウトが行われます。 しかし、ファンインして戻すことはこれよりずっと難しくなります。 キューによってトリガーされる関数が終了し、関数の出力が格納される時間を追跡するように、コードを記述する必要があります。 Durable Functions 拡張機能は、比較的単純なコードでこのパターンを処理します。

#### <a name="c-script"></a>C# スクリプト

```cs
public static async Task Run(DurableOrchestrationContext context)
{
    var parallelTasks = new List<Task<int>>();

    // get a list of N work items to process in parallel
    object[] workBatch = await context.CallActivityAsync<object[]>("F1");
    for (int i = 0; i < workBatch.Length; i++)
    {
        Task<int> task = context.CallActivityAsync<int>("F2", workBatch[i]);
        parallelTasks.Add(task);
    }

    await Task.WhenAll(parallelTasks);

    // aggregate all N outputs and send result to F3
    int sum = parallelTasks.Sum(t => t.Result);
    await context.CallActivityAsync("F3", sum);
}
```

#### <a name="javascript-functions-2x-only"></a>JavaScript (Functions 2.x のみ)

```js
const df = require("durable-functions");

module.exports = df.orchestrator(function*(context) {
    const parallelTasks = [];

    // get a list of N work items to process in parallel
    const workBatch = yield context.df.callActivity("F1");
    for (let i = 0; i < workBatch.length; i++) {
        parallelTasks.push(context.df.callActivity("F2", workBatch[i]));
    }

    yield context.df.Task.all(parallelTasks);

    // aggregate all N outputs and send result to F3
    const sum = parallelTasks.reduce((prev, curr) => prev + curr, 0);
    yield context.df.callActivity("F3", sum);
});
```

ファンアウト作業が `F2` 関数の複数のインスタンスに配分され、タスクの動的リストを使用してこの作業が追跡されます。 .NET `Task.WhenAll` API または JavaScript `context.df.Task.all` API が呼び出され、すべての呼び出された関数が終了するまで待機します。 次に、`F2` 関数の出力が動的なタスク リストから集計されて、`F3` 関数に渡されます。

`Task.WhenAll` または `context.df.Task.all` の `await` 呼び出しまたは `yield` 呼び出しの際に自動でチェックポイントを設定することで、実行途中でクラッシュや再起動が生じても既に完了したすべてのタスクをやり直す必要がなくなります。

### <a name="async-http"></a>パターン #3: 非同期 HTTP API

3 番目のパターンでもっとも重要なのは、外部クライアントとの間で行われる実行時間の長い操作の状態を調整するという問題です。 このパターンを実装する一般的な方法は、HTTP 呼び出しにより実行時間の長い操作をトリガーし、その操作が完了する時間を照会できる状態エンドポイントにクライアントをリダイレクトするという方法です。

![HTTP API の図](./media/durable-functions-concepts/async-http-api.png)

Durable Functions は、実行時間の長い関数を操作するコードをシンプルにするビルトイン API を提供します。 クイック スタート サンプル ([C#](durable-functions-create-first-csharp.md)、[JavaScript](quickstart-js-vscode.md)) では、新しいオーケストレーター関数のインスタンスを開始するために使用できる、シンプルな REST コマンドについて説明しています。 インスタンスが開始されると、この拡張機能はオーケストレーター関数の状態をクエリする Webhook HTTP API を公開します。 次の例では、オーケストレーターを開始してその状態をクエリする REST コマンドを示します。 わかりやすくするため、この例では細部をいくらか省略しています。

```
> curl -X POST https://myfunc.azurewebsites.net/orchestrators/DoWork -H "Content-Length: 0" -i
HTTP/1.1 202 Accepted
Content-Type: application/json
Location: https://myfunc.azurewebsites.net/admin/extensions/DurableTaskExtension/b79baf67f717453ca9e86c5da21e03ec

{"id":"b79baf67f717453ca9e86c5da21e03ec", ...}

> curl https://myfunc.azurewebsites.net/admin/extensions/DurableTaskExtension/b79baf67f717453ca9e86c5da21e03ec -i
HTTP/1.1 202 Accepted
Content-Type: application/json
Location: https://myfunc.azurewebsites.net/admin/extensions/DurableTaskExtension/b79baf67f717453ca9e86c5da21e03ec

{"runtimeStatus":"Running","lastUpdatedTime":"2017-03-16T21:20:47Z", ...}

> curl https://myfunc.azurewebsites.net/admin/extensions/DurableTaskExtension/b79baf67f717453ca9e86c5da21e03ec -i
HTTP/1.1 200 OK
Content-Length: 175
Content-Type: application/json

{"runtimeStatus":"Completed","lastUpdatedTime":"2017-03-16T21:20:57Z", ...}
```

状態は Durable Functions ランタイムによって管理されるため、独自の状態追跡メカニズムを実装する必要はありません。

Durable Functions 拡張機能には実行時間の長いオーケストレーションを管理するためのビルトイン Webhook がありますが、独自の関数トリガー (HTTP、キュー、またはイベント ハブなど) と `orchestrationClient` バインドを使用してこのパターンを自分で実装できます。 たとえば、キュー メッセージを使用して終了をトリガーできます。  または、生成されたキーを認証で使用するビルトイン Webhook の代わりに、Azure Active Directory の認証ポリシーで保護された HTTP トリガーを使用できます。

#### <a name="c"></a>C#

```cs
// HTTP-triggered function to start a new orchestrator function instance.
public static async Task<HttpResponseMessage> Run(
    HttpRequestMessage req,
    DurableOrchestrationClient starter,
    string functionName,
    ILogger log)
{
    // Function name comes from the request URL.
    // Function input comes from the request content.
    dynamic eventData = await req.Content.ReadAsAsync<object>();
    string instanceId = await starter.StartNewAsync(functionName, eventData);

    log.LogInformation($"Started orchestration with ID = '{instanceId}'.");

    return starter.CreateCheckStatusResponse(req, instanceId);
}
```

#### <a name="javascript-functions-2x-only"></a>JavaScript (Functions 2.x のみ)

```javascript
// HTTP-triggered function to start a new orchestrator function instance.
const df = require("durable-functions");

module.exports = async function (context, req) {
    const client = df.getClient(context);

    // Function name comes from the request URL.
    // Function input comes from the request content.
    const eventData = req.body;
    const instanceId = await client.startNew(req.params.functionName, undefined, eventData);

    context.log(`Started orchestration with ID = '${instanceId}'.`);

    return client.createCheckStatusResponse(req, instanceId);
};
```

> [!WARNING]
> JavaScript でローカルに開発する場合は、環境変数 `WEBSITE_HOSTNAME` を `localhost:<port>` に設定する必要があります。たとえば、 `DurableOrchestrationClient` のメソッドを使用するには、`localhost:7071` に設定します。 この要件の詳細については、[GitHub の問題](https://github.com/Azure/azure-functions-durable-js/issues/28)に関するページを参照してください。

.NET における [DurableOrchestrationClient](https://azure.github.io/azure-functions-durable-extension/api/Microsoft.Azure.WebJobs.DurableOrchestrationClient.html) `starter`パラメーターは、Durable Functions 拡張機能の一部である `orchestrationClient` 出力バインドからの値です。 JavaScript では、`df.getClient(context)` を呼び出すことによって、このオブジェクトが返されます。 これらのオブジェクトは、新しいまたは既存のオーケストレーター関数インスタンスの開始や終了、インスタンスへのイベント送信やクエリ送信のためのメソッドを備えています。

上記の例では、HTTP によってトリガーされる関数は受信 URL から `functionName` 値を受け取り、その値を [StartNewAsync](https://azure.github.io/azure-functions-durable-extension/api/Microsoft.Azure.WebJobs.DurableOrchestrationClient.html#Microsoft_Azure_WebJobs_DurableOrchestrationClient_StartNewAsync_) に渡します。 次に [CreateCheckStatusResponse](https://azure.github.io/azure-functions-durable-extension/api/Microsoft.Azure.WebJobs.DurableOrchestrationClient.html#Microsoft_Azure_WebJobs_DurableOrchestrationClient_CreateCheckStatusResponse_System_Net_Http_HttpRequestMessage_System_String_) バインド API は、`Location` ヘッダーと共に、開始されたインスタンスの状態を後で調べるため、またはインスタンスを終了するために使用するインスタンスの追加情報を含めた応答を返します。

### <a name="monitoring"></a>パターン #4: 監視

モニター パターンは、ワークフローの柔軟な "*繰り返し*" プロセスを参照します。たとえば、特定の条件が満たされるまでポーリングします。 通常の[タイマー トリガー](../functions-bindings-timer.md)は、定期的なクリーンアップ ジョブなどの単純なシナリオに対処できますが、その間隔は静的であり、インスタンスの有効期間の管理は複雑になります。 Durable Functions では、柔軟な繰り返し間隔とタスクの有効期間の管理を実行でき、1 つのオーケストレーションから複数のモニター プロセスを作成する機能も備えています。

例として、前述の非同期 HTTP API のシナリオの逆があります。 外部クライアントのエンドポイントを公開して実行時間の長い操作を監視する代わりに、長時間実行されるモニターで、外部エンドポイントを使用して、何らかの状態変更が発生するまで待機します。

![モニター ダイアグラム](./media/durable-functions-concepts/monitor.png)

Durable Functions を使用して、任意のエンドポイントを観察する複数のモニターを、数行のコードで作成できます。 モニターは、何らかの条件が満たされた場合、または [DurableOrchestrationClient](durable-functions-instance-management.md) によって実行を終了でき、待機間隔は、何らかの条件に基づいて変更できます (つまり指数バックオフを実装します)。次のコードは、基本的なモニターを実装します。

#### <a name="c-script"></a>C# スクリプト

```cs
public static async Task Run(DurableOrchestrationContext context)
{
    int jobId = context.GetInput<int>();
    int pollingInterval = GetPollingInterval();
    DateTime expiryTime = GetExpiryTime();

    while (context.CurrentUtcDateTime < expiryTime)
    {
        var jobStatus = await context.CallActivityAsync<string>("GetJobStatus", jobId);
        if (jobStatus == "Completed")
        {
            // Perform action when condition met
            await context.CallActivityAsync("SendAlert", machineId);
            break;
        }

        // Orchestration will sleep until this time
        var nextCheck = context.CurrentUtcDateTime.AddSeconds(pollingInterval);
        await context.CreateTimer(nextCheck, CancellationToken.None);
    }

    // Perform further work here, or let the orchestration end
}
```

#### <a name="javascript-functions-2x-only"></a>JavaScript (Functions 2.x のみ)

```js
const df = require("durable-functions");
const moment = require("moment");

module.exports = df.orchestrator(function*(context) {
    const jobId = context.df.getInput();
    const pollingInternal = getPollingInterval();
    const expiryTime = getExpiryTime();

    while (moment.utc(context.df.currentUtcDateTime).isBefore(expiryTime)) {
        const jobStatus = yield context.df.callActivity("GetJobStatus", jobId);
        if (jobStatus === "Completed") {
            // Perform action when condition met
            yield context.df.callActivity("SendAlert", machineId);
            break;
        }

        // Orchestration will sleep until this time
        const nextCheck = moment.utc(context.df.currentUtcDateTime).add(pollingInterval, 's');
        yield context.df.createTimer(nextCheck.toDate());
    }

    // Perform further work here, or let the orchestration end
});
```

要求が受信されると、そのジョブ ID 用の新しいオーケストレーション インスタンスが作成されます。 インスタンスは、条件が満たされてループが終了するまで、状態をポーリングします。 ポーリング間隔を制御するために永続的タイマーが使用されます。 さらに作業を実行するか、オーケストレーションを終了できます。 `context.CurrentUtcDateTime` (.NET) または `context.df.currentUtcDateTime` (JavaScript) が `expiryTime` を超えると、モニターが終了します。

### <a name="human"></a>パターン #5: 人による操作

ほとんどのプロセスには、何らかの人による操作が含まれます。 自動化されたプロセスに人による操作が含まれる場合に問題になるのが、人は必ずしもクラウド サービスのように可用性と応答性が高くないということです。 自動化されたプロセスはこのことを許容する必要があり、通常タイムアウトや補正ロジックを使用してそれを行います。

人による操作を含むビジネス プロセスの一例として、承認プロセスがあります。 たとえば、特定の額を超えた経費報告書について上司から承認を得る必要があるなどの場合です。 72 時間以内に上司が承認しない (上司が休暇に入ったなどの) 場合に、他の誰か (おそらくは上司の上司) から承認を得るためにエスカレーション プロセスに入ります。

![人による操作の図](./media/durable-functions-concepts/approval.png)

このパターンは、オーケストレーター関数を使用して実装できます。 オーケストレーターは、[永続タイマー](durable-functions-timers.md)を使用して承認を要求し、タイムアウトした場合にエスカレーションします。 また、人による何らかの操作によって生成された通知である[外部イベント](durable-functions-external-events.md)を待機します。

#### <a name="c-script"></a>C# スクリプト

```cs
public static async Task Run(DurableOrchestrationContext context)
{
    await context.CallActivityAsync("RequestApproval");
    using (var timeoutCts = new CancellationTokenSource())
    {
        DateTime dueTime = context.CurrentUtcDateTime.AddHours(72);
        Task durableTimeout = context.CreateTimer(dueTime, timeoutCts.Token);

        Task<bool> approvalEvent = context.WaitForExternalEvent<bool>("ApprovalEvent");
        if (approvalEvent == await Task.WhenAny(approvalEvent, durableTimeout))
        {
            timeoutCts.Cancel();
            await context.CallActivityAsync("ProcessApproval", approvalEvent.Result);
        }
        else
        {
            await context.CallActivityAsync("Escalate");
        }
    }
}
```

#### <a name="javascript-functions-2x-only"></a>JavaScript (Functions 2.x のみ)

```js
const df = require("durable-functions");
const moment = require('moment');

module.exports = df.orchestrator(function*(context) {
    yield context.df.callActivity("RequestApproval");

    const dueTime = moment.utc(context.df.currentUtcDateTime).add(72, 'h');
    const durableTimeout = context.df.createTimer(dueTime.toDate());

    const approvalEvent = context.df.waitForExternalEvent("ApprovalEvent");
    if (approvalEvent === yield context.df.Task.any([approvalEvent, durableTimeout])) {
        durableTimeout.cancel();
        yield context.df.callActivity("ProcessApproval", approvalEvent.result);
    } else {
        yield context.df.callActivity("Escalate");
    }
});
```

永続タイマーは `context.CreateTimer` (.NET) または `context.df.createTimer` (JavaScript) を呼び出すことで作成されます。 通知は `context.WaitForExternalEvent` (.NET) または `context.df.waitForExternalEvent` (JavaScript) が受け取ります。 そして、エスカレーションする (最初にタイムアウトが発生する) か、承認を処理する (タイムアウト前に承認を得る) かを決定するために `Task.WhenAny` (.NET) または `context.df.Task.any` (JavaScript) が呼び出されます。

外部クライアントは、[組み込みの HTTP API](durable-functions-http-api.md#raise-event) を使用するか、別の関数の [DurableOrchestrationClient.RaiseEventAsync](https://azure.github.io/azure-functions-durable-extension/api/Microsoft.Azure.WebJobs.DurableOrchestrationClient.html#Microsoft_Azure_WebJobs_DurableOrchestrationClient_RaiseEventAsync_System_String_System_String_System_Object_) API を使用して、待機中のオーケストレーター関数にイベント通知を配信できます。

```csharp
public static async Task Run(string instanceId, DurableOrchestrationClient client)
{
    bool isApproved = true;
    await client.RaiseEventAsync(instanceId, "ApprovalEvent", isApproved);
}
```

```javascript
const df = require("durable-functions");

module.exports = async function (context) {
    const client = df.getClient(context);
    const isApproved = true;
    await client.raiseEvent(instanceId, "ApprovalEvent", isApproved);
};
```

## <a name="the-technology"></a>テクノロジ

背後では、永続的タスクのオーケストレーションを構築するための GitHub のオープン ソース ライブラリである [Durable Task Framework](https://github.com/Azure/durabletask) の上に、Durable Functions 拡張機能が構築されています。 Azure Functions が Azure WebJobs のサーバーレスな進化形であるのと同じように、Durable Functions は Durable Task Framework のサーバーレスな進化形です。 Durable Task Framework は、ミッション クリティカルなプロセスを自動化するために Microsoft 内外で頻繁に使用されています。 サーバーレスな Azure Functions 環境には自然に適合します。

### <a name="event-sourcing-checkpointing-and-replay"></a>イベント ソーシング、チェックポイント設定、リプレイ

オーケストレーター関数は、[Event Sourcing](https://docs.microsoft.com/azure/architecture/patterns/event-sourcing) として知られるデザイン パターンを使用して、この関数の実行状態を安定的に維持します。 オーケストレーションの *"現在の"* 状態を直接格納する代わりに、この永続的な拡張機能は追加専用のストアを使用して、関数オーケストレーションがとる *"アクションのすべて"* を記録します。 これには、ランタイムの完全な状態を "ダンプする" のと比べて、パフォーマンス、スケーラビリティ、応答時間の改善など多くのメリットがあります。 他にも、トランザクション データに最終的な一貫性を提供する、完全な監査証跡や監査履歴を維持するなどのメリットもあります。 監査証跡のみで、信頼性の高い補正アクションが可能です。

この拡張機能による Event Sourcing の使用は透過的です。 背後では、オーケストレーター関数内の `await` (C#) または `yield` (JavaScript) 演算子が、Durable Task Framework のディスパッチャーにオーケストレーター スレッドのコントロールを引き渡します。 次に、このディスパッチャーは、オーケストレーター関数がスケジュールした新しいアクション (1 つ以上の子関数を呼び出す、永続タイマーをスケジュールする、など) をストレージにコミットします。 この透過的なコミット アクションは、オーケストレーション インスタンスの *"実行履歴"* に追加されます。 この履歴はストレージ テーブルに格納されます。 次に、このコミット アクションはキューにメッセージを追加して実際の作業をスケジュールします。 この時点では、メモリからオーケストレーター関数をアンロードできます。 それに対する課金は Azure Functions の従量課金プランを使用している場合は停止します。  処理すべき作業がさらにある場合は、関数は再起動され、関数の状態は再構築されます。

オーケストレーション関数にさらに処理すべき作業が与えられる (たとえば、応答メッセージを受け取る、または永続タイマーが期限切れになる) と、ローカルの状態を再構築するためにオーケストレーターがもう一度起動して始めからその関数全体を再実行します。 このリプレイ中にコードが関数を呼び出そうとする (または他の非同期作業を行おうとする) と、Durable Task Framework は現在のオーケストレーションの*実行履歴*を照会します。 [アクティビティ関数](durable-functions-types-features-overview.md#activity-functions)が既に実行されいくつかの結果を生成していることが確認されると、その関数の結果をリプレイしてオーケストレーター コードが実行を継続します。 この動作は、関数コードが完了する、または新しい非同期作業をスケジュールする時点まで継続して行われます。

### <a name="orchestrator-code-constraints"></a>オーケストレーター コードの制約

このリプレイ動作は、オーケストレーター関数に記述できるコード タイプに関する制約を設けます。 たとえば、オーケストレーター コードは複数回リプレイされて毎回同じ結果を生成する必要があるため、確定的である必要があります。 制約に関する完全なリストについては、**チェックポイントの設定と再起動**に関する記事の「[Orchestrator code constraints (オーケストレーター コードの制約)](durable-functions-checkpointing-and-replay.md#orchestrator-code-constraints)」セクションをご覧ください。

## <a name="monitoring-and-diagnostics"></a>監視と診断

Durable Functions 拡張機能は、Application Insights インストルメンテーション キーで関数アプリが構成されるときに、構造化された追跡データを [Application Insights](../functions-monitoring.md) に自動で出力します。 この追跡データは、オーケストレーションの動作や進行状況を監視するために使用できます。

[Application Insights Analytics](../../application-insights/app-insights-analytics.md) を使用すると Application Insights ポータル内で Durable Functions の追跡イベントがどのように表示されるか、その例を次に示します。

![App Insights のクエリ結果](./media/durable-functions-concepts/app-insights-1.png)

各ログ エントリの `customDimensions` フィールドには、有用な構造化データが数多く格納されています。 完全に展開された、そのようなエントリの例を次に示します。

![App Insights クエリの customDimensions フィールド](./media/durable-functions-concepts/app-insights-2.png)

Durable Task Framework ディスパッチャーのリプレイ動作のおかげで、リプレイされたアクションの豊富なログ エントリを確認できます。 これはコア エンジンのリプレイ動作を理解するのに役立ちます。 [診断](durable-functions-diagnostics.md)に関する記事では、リプレイ ログをフィルターで除外したサンプル クエリを示しているため、"リアルタイム" ログを確認できます。

## <a name="storage-and-scalability"></a>ストレージとスケーラビリティ

Durable Functions 拡張機能は、Azure Storage の Queue、Table、BLOB を使用して実行履歴の状態を保持し、関数の実行をトリガーします。 関数アプリの既定のストレージ アカウントを使用、または別のストレージ アカウントを構成できます。 ストレージのスループット制限が原因で、別のアカウントを使用したいと思うこともあるでしょう。 記述するオーケストレーター コードは、これらのストレージ アカウント内のエンティティとやりとりをする必要はありません (というより、すべきではありません)。 エンティティは実装の詳細として、Durable Task Framework により直接管理されます。

オーケストレーター関数はアクティビティ関数をスケジュールし、内部キュー メッセージを介して応答を受け取ります。 Azure Functions の従量課金プランで関数アプリを実行するときは、これらのキューは [Azure Functions Scale Controller](../functions-scale.md#how-the-consumption-plan-works) により監視され、必要に応じてコンピューティング インスタンスが追加されます。 オーケストレーター関数は、複数の VM にスケールアウトされるときに、呼び出すアクティビティ関数を複数の異なる VM で実行しつつ、1 つの VM 上で実行される場合があります。 Durable Functions のスケール動作の詳細については、[パフォーマンスとスケール](durable-functions-perf-and-scale.md)に関する記事をご覧ください。

Table Storage は、オーケストレーター アカウントの実行履歴を格納するために使用されます。 特定の VM 上でインスタンスが復元される場合は常に、そのローカルの状態を再構築できるようにするために、テーブルから実行履歴がフェッチされます。 Table Storage 内で履歴を使用できるようにすることの便利な点の 1 つは、[Microsoft Azure Storage Explorer](../../vs-azure-tools-storage-manage-with-storage-explorer.md) などのツールを使用してオーケストレーションの履歴を確認できることです。

Storage BLOB は、複数の VM におけるオーケストレーション インスタンスのスケールアウトを調整するリース メカニズムとして主に使用されます。 これらは、テーブルまたはキューに直接保存することができない大きなメッセージのデータを保持するためにも使用されます。

![Azure Storage Explorer のスクリーン ショット](./media/durable-functions-concepts/storage-explorer.png)

> [!WARNING]
> Table Storage 内で実行履歴を確認できるのは簡単で便利ですが、このテーブルに依存関係を設定することは避けてください。 これについては、Durable Functions 拡張機能が今後改善されていくうちに変更される可能性があります。

## <a name="known-issues-and-faq"></a>既知の問題とよくあるご質問

すべての既知の問題は、[GitHub の Issue](https://github.com/Azure/azure-functions-durable-extension/issues) 一覧で追跡されています。 問題が発生してその問題が GitHub で見つからない場合は、新しい Issue を開き、その問題の詳細な説明を記載してお知らせください。

## <a name="next-steps"></a>次の手順

Durable Functions の詳細については、「[Overview of function types and features for Durable Functions (Azure Functions)](durable-functions-types-features-overview.md)」(Durable Functions の関数の種類と機能の概要 (Azure Functions)) を参照するか、または次のページを参照してください。

> [!div class="nextstepaction"]
> [最初の Durable Function を作成する](durable-functions-create-first-csharp.md)

[DurableOrchestrationContext]: https://azure.github.io/azure-functions-durable-extension/api/Microsoft.Azure.WebJobs.DurableOrchestrationContext.html