App Engine 第一世代ユーザの Cloud Tasks の使いどころ
===


App Engine と Cloud Tasks で一回限りのバッチを実行する
===

App Engine でアプリケーションを運用していると、たまに単発のバッチジョブを実行したいことがあります。例えば Datastore の Entity のマイグレーションを行ったり、環境セットアップの一環として Datastore に初期データを詰めたり等です。

もちろん Datastore を操作するだけであれば Cloud Datastore API をスクリプトから使ったり、Cloud Dataflow のバッチジョブで操作するということもできますが、App Engine とのコードの共通化が難しいですし、何より App Engine の開発作法からなるべく外れたくありません。

そこで本日ベータリリースが発表された [Cloud Tasks](https://cloud.google.com/tasks/) を使って、そのような単発のバッチジョブを実行する方法を見てみたいと思います。

## Cloud Tasks とは

App Engine に今まで存在していた、非同期キューを実現する [Task Queue](https://cloud.google.com/appengine/docs/standard/go/taskqueue/) を App Engine 以外の GCP サービスからも利用できるようにするものです。
Task Queue と Cloud Tasks では同一の世界を見ているため、どちらの API を使っても同じ queue を扱うことができます。

## なぜ単発バッチジョブに Cloud Tasks を使うか

任意のタイミングで実行できるバッチジョブといえば [App Engine cron](https://cloud.google.com/appengine/docs/standard/go/config/cron) が存在しますが、App Engine cron では繰り返し実行するスケジューリングしか出来ないため、単発のバッチジョブには不向きです。

では Task Queue はどうかというと、Task Queue で enqueue する(Queue にタスクを詰める)ためには App Engine 内からでないと出来ないため、外から任意のタイミングで発火させるのがやりづらいです。

そこで本日出た Cloud Tasks です！Cloud Tasks は Queue に [enqueue する API](https://cloud.google.com/tasks/docs/reference/rest/v2beta3/projects.locations.queues.tasks/create) が存在するため、任意のタイミングでバッチジョブを動かすことができます。

## 設定方法

設定と言っても通常の Task Queue を扱う方法と変わりませんが、要点を簡単にまとめます。

#### 1. Handler を定義する

バッチジョブの内容の本体となるロジックを HTTP Handler として定義します。Task Queue からのリクエストは `0.1.0.2` という IP アドレスから来るので、それ以外はブロックするようにするといいでしょう。

以下 Go での例です。

```go
http.HandleFunc("/tasks/some_batch_task", func(w http.ResponseWriter, r *http.Request) {
    if r.RemoteAddr != "0.1.0.2" && !appengine.IsDevAppServer() { // ローカル開発サーバは許可
        http.Error(w, http.StatusText(http.StatusForbidden), http.StatusForbidden)
        return
    }

    // do some batch job...

    fmt.Fprintf(w, "finish task\n")
})
```

#### 2. app.yaml に Handler を追加

先程定義した Handler がリクエストを受け付けれるように、app.yaml に Handler を追加します。[公式ドキュメント](https://cloud.google.com/appengine/docs/standard/go/taskqueue/push/creating-handlers#securing_task_handler_urls)に記載されているように handler の設定に `login: admin` を追加しておくと、GCP プロジェクトの管理者(or Task Queueから)でないとアクセスできなくなるので安全です。

```yaml
handlers:
- url: /tasks/.*
  login: admin
  script: _go_app
```

#### 3. queue.yaml に Queue を追加

ポイントとしては `bucket_size: 0` にしておくことです。こうしておくことで、なんらかのタイミングで意図せず該当の Queue に enqueue されてしまっても、自動でタスクが実行されないようになります。

```yaml
queue:
- name: my-batch-queue
  bucket_size: 0
  rate: 1/s
  target: my-service
```

`bucket_size` に関して詳しくは[過去のエントリ](http://addsict.hatenablog.com/entry/2018/09/24/125820)を参照してください。

#### 4. app.yaml と queue.yaml をデプロイ

app.yaml をデプロイしアプリケーションを最新にします。
また queue.yaml をデプロイします。

```sh
gcloud app deploy app.yaml
gcloud app deploy queue.yaml
```

#### 5. Cloud Tasks API を使ってタスクを実行

あとはジョブを実行したい任意のタイミングで [gcloud tasks create-app-engine-task](https://cloud.google.com/sdk/gcloud/reference/beta/tasks/create-app-engine-task) を使ってタスクを作成します。

```
gcloud beta tasks create-app-engine-task --queue=my-batch-queue --url=/tasks/some_batch_task
```

queue.yaml で `bucket_size: 0` にしている場合は自動でタスクが実行されないので、Cloud Tasks の console 画面から手動でポチッと実行します。

[ここに画像貼る]

## 注意点

* Task Queue からのリクエストであれば automatic_sccaling の場合: 10分, basic or manual scaling の場合: 24時間の制限時間があります。通常のリクエストは60秒なので、それよりは緩和されている。
* ログはちゃんと見れるよ
* 途中で abort したくなっても出来ないからそれだけ注意(インスタンス落とせばできる)


## まとめ

本日ベータリリースされた Cloud Tasks を使うことで、
