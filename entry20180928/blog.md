App Engine と Cloud Tasks で一回限りのバッチを実行する
===

App Engine でアプリケーションを運用していると、たまに単発のバッチジョブを実行したいことがあります。例えば Datastore の Entity のマイグレーションを行ったり、環境セットアップの一環として Datastore に初期データを詰めたりしたいことってありますよね。

もちろん Datastore を操作するだけであれば Cloud Datastore API をスクリプトから使ったり、Cloud Dataflow のバッチジョブで操作するということもできますが、App Engine とのコードの共通化が難しいですし、何より App Engine の開発作法からなるべく外れたくありません。

そこで今回は、本日ベータリリースが発表された [Cloud Tasks](https://cloud.google.com/tasks/) を使って、そのような単発のバッチジョブを App Engine で実行する方法を見てみたいと思います。

## Cloud Tasks とは

[https://cloud.google.com/blog/products/application-development/announcing-cloud-tasks-a-task-queue-service-for-app-engine-flex-and-second-generation-runtimes:embed:cite]

App Engine で非同期タスクを扱う [Task Queue](https://cloud.google.com/appengine/docs/standard/go/taskqueue/) を、App Engine 以外の GCP プロダクトからも利用できるようにしたサービスです。
Task Queue と Cloud Tasks では同一の世界を見ているため、どちらの API を使っても同じ Queue を扱うことができます。

## なぜ単発バッチジョブに Cloud Tasks を使うか

任意のタイミングで実行できるバッチジョブといえば [App Engine cron](https://cloud.google.com/appengine/docs/standard/go/config/cron) がありますが、App Engine cron では定期的に実行するスケジューリングしか出来ないため、単発のバッチジョブには不向きです。

では通常の Task Queue はどうかというと、Task Queue でタスクを作成するためには App Engine からでないとåº来ないため、外から任意のタイミングで発火させるのがやりづらい状況でした。

そこで本日出た Cloud Tasks です。Cloud Tasks は [タスクを作成する API](https://cloud.google.com/tasks/docs/reference/rest/v2beta3/projects.locations.queues.tasks/create) が存在するため、外から任意のタイミングでタスクを作成し、それを契機にバッチジョブを動かすことができます。

## 設定方法

設定と言っても通常の Task Queue を扱う方法と変わりませんが、要点をいくつかまとめます。

#### 1. Handler を定義する

バッチジョブの本体となるロジックを HTTP Handler として定義します。Task Queue からのリクエストは `0.1.0.2` という IP アドレスから来るので、それ以外はブロックするといいでしょう。

以下 Go での例です。

```go
http.HandleFunc("/tasks/some_batch_task", func(w http.ResponseWriter, r *http.Request) {
    if r.RemoteAddr != "0.1.0.2" {
        http.Error(w, http.StatusText(http.StatusForbidden), http.StatusForbidden)
        return
    }

    // do some batch job...

    fmt.Fprintf(w, "finish task\n")
})
```

#### 2. app.yaml に Handler を追加

先程定義した Handler がリクエストを受け付けれるように、app.yaml に Handler を追加します。[公式ドキュメント](https://cloud.google.com/appengine/docs/standard/go/taskqueue/push/creating-handlers#securing_task_handler_urls)に記載されているように handler の設定に `login: admin` を追加しておくと、GCP プロジェクトの管理者(or Task Queue から)でないとアクセスできなくなるので、IPアドレスでのフィルタリングと合わせて設定しておくと安全です。

```yaml
handlers:
- url: /tasks/.*
  login: admin
  script: _go_app
```

#### 3. queue.yaml に Queue を追加

バッチジョブを起動させるための Queue を追加します。

ポイントとしては `bucket_size: 0` にしておくことです。こうしておくことで、なんらかのタイミングで意図せず該当の Queue にタスクが作成されてしまっても、自動でタスクが実行されないようになります。

```yaml
queue:
- name: my-batch-queue
  bucket_size: 0
  rate: 1/s
  target: my-service
```

`bucket_size` に関しては、詳しくは[過去のエントリ](http://addsict.hatenablog.com/entry/2018/09/24/125820)を参照してください。

#### 4. app.yaml と queue.yaml をデプロイ

アプリケーションと Queue の設定をデプロイします。

```sh
gcloud app deploy app.yaml
gcloud app deploy queue.yaml
```

#### 5. Cloud Tasks API を使ってタスクを実行

あとはジョブを実行したい任意のタイミングで [gcloud tasks create-app-engine-task](https://cloud.google.com/sdk/gcloud/reference/beta/tasks/create-app-engine-task) を使ってタスクを作成します。

```
gcloud beta tasks create-app-engine-task --queue=my-batch-queue --url=/tasks/some_batch_task
```

queue.yaml で `bucket_size: 0` と定義した場合は自動でタスクが実行されないので、Cloud Tasks の console 画面から「Run Now」と書かれたボタンを手動でポチッとします。

[f:id:furuyamayuuki:20180928180524p:plain]

バッチジョブは実行に時間がかかることも多いので進捗を知れることが大事ですが、アプリから吐いたログは Stackdriver Logging に都度表示されるので、進捗はログに出しておくと良さげです。以下の例は30秒かかるジョブを実行したときに、4つのログが吐かれている例です。

[f:id:furuyamayuuki:20180928180609p:plain]

## 注意点

Task Queue をバッチジョブとして用いた場合、いくつか注意点があります。

#### デッドラインに注意

App Engine ではリクエストを処理できる時間に制限時間があります。Task Queue からのリクエストであれば automatic scaling の場合は10å、basic or manual scaling の場合は24時間です([参考](https://cloud.google.com/appengine/docs/standard/go/taskqueue/#push_queues_and_pull_queues))。
これ以上の処理時間がかかるようであれば、バッチジョブの作業内容を別々のタスクに分割してこの時間に収めるのがいいでしょう。

#### 処理を冪等にすること

エラーが発生した場合自動でリトライが走ります(ただし `backet_size: 0` の場合はこの限りじゃありません)。処理を冪等にしておくことで何度リトライが走っても大丈夫なようにします。これはバッチジョブ全般にいえるプラクティスですね。

#### 途中でやめることはできない

何かクリティカルな問題が発生したとしても、アプリケーションがエラーレスポンスを返さない限り途中で abort させることはできません。インスタンスを落とせば出来るのかもしれませんが、本番のサã¼ビスでそれを行うことは難しいでしょう...。

## まとめ

本日ベータリリースされた Cloud Tasks を使うことで、App Engine で単発のバッチジョブを実行できることを説明しました。
Cloud Tasks は特に第二世代のランタイムの App Engine ユーザ向けにフィーチャーされてますが、既存の App Engine ユーザでも使い所がありそうですね。
