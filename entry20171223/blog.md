GAE ローカル開発のログをいい感じに表示してくれる gaelv というツールを作りました
===

App Engine のアプリをローカルで開発していると、標準だとログがこんな感じに表示されてしまって視認性が悪かったりしますよね。

[ここに画像を]

そこで GCP にアプリをデプロイした時と同じように、Stackdriver っぽい見た目でログを見れる gaelv というツールを作ってみました。

[ここに画像を]
https://github.com/addsict/gaelv

## インストール

go get 一発でインストールできます。インストールすると `gaelv` というコマンドが使えるようになります。

```
go get -u github.com/addsict/gaelv/...
```

## 使い方

まずいつも通り dev_appserver.py で GAE アプリケーションを立ち上げます。  
その際に `--logs_path=<path>` というオプションをつけてください(path部分は適当なファイルパスでokです)。

```
dev_appserver.py app.yaml --logs_path=/tmp/log.db
```

次に以下のコマンドで Log Viewer のサーバを立ち上げます。

```
gaelv --logs_path=/tmp/log.db
```

最後にブラウザで http://localhost:9090/ を開くと Log Viewer が表示されます。
この状態でアプリケーションにアクセスすると、リアルタイムにログが流れてきます。
(※ただし後述の注意点にあるように、ログのバッファリングをOFFにした場合のみ)

[ここに画像を]

または `--console` オプションでログをコンソールに直接表示することもできます。

```
gaelv --logs_path=/tmp/log.db --console
```

[ここに画像を]

## 注意点

`dev_appserver.py` は標準だとログを5秒間バッファリングしてしまい、すぐには Log Viewer の方に表示されません。
また厄介なことに5秒経ったら自動的にフラッシュされるのではなく、次にリクエストが来た時にフラッシュする仕組みになっています。
残念ながらこのバッファリングはオプションで OFF にすることができないため、App Engine SDK のソースコードを直接いじってしまう方法以外は、現状だと回避策はなさそうです。

バッファリングを OFF にするには `/usr/local/google-cloud-sdk/platform/google_appengine/google/appengine/api/logservice/logservice_stub.py` というファイルの `_MIN_COMMIT_INTERVAL` という変数を `0` にすればOKです。

diff を貼っておきますので参考にしてください。

```diff
--- a/logservice_stub.py
+++ b/logservice_stub.py
@@ -86,7 +86,7 @@ class LogServiceStub(apiproxy_stub.APIProxyStub):
-  _MIN_COMMIT_INTERVAL = 5
+  _MIN_COMMIT_INTERVAL = 0
```

## 仕組み

gaelv の仕組みを少し紹介します。

全体のアーキテクチャ

[ここに画像を]

### ログの取得方法
App Engine は Log Service という API でリクエストログとアプリケーションログの取得を行うことができます。
ログはデプロイされたアプリでは Stackdriver に入っていきますが、ローカルではそれをシミュレートするために SQLite の DB に入っていきます。
使い方で説明した `--logs_path` というオプションがその DB ファイルを指し示しています。
今回はその SQLite の DB を直接参照する形で実装しました。

リクエストログのテーブル構造
```mysql[
CREATE TABLE IF NOT EXISTS RequestLogs (
  id INTEGER NOT NULL PRIMARY KEY,
  user_request_id TEXT NOT NULL,
  app_id TEXT NOT NULL,
  version_id TEXT NOT NULL,
  module TEXT NOT NULL,
  ip TEXT NOT NULL,
  nickname TEXT NOT NULL,
  start_time INTEGER NOT NULL,
  end_time INTEGER DEFAULT 0 NOT NULL,
  method TEXT NOT NULL,
  resource TEXT NOT NULL,
  http_version TEXT NOT NULL,
  status INTEGER DEFAULT 0 NOT NULL,
  response_size INTEGER DEFAULT 0 NOT NULL,
  user_agent TEXT NOT NULL,
  url_map_entry TEXT DEFAULT '' NOT NULL,
  host TEXT NOT NULL,
  referrer TEXT,
  task_queue_name TEXT DEFAULT '' NOT NULL,
  task_name TEXT DEFAULT '' NOT NULL,
  latency INTEGER DEFAULT 0 NOT NULL,
  mcycles INTEGER DEFAULT 0 NOT NULL,
  finished INTEGER DEFAULT 0 NOT NULL
);
```

### ブラウザへのストリーミング
サーバからブラウザへは Server Sent Events (SSE) を使ってログをストリーミングとして送っています。 
HTTP の標準の仕組みで実装できるので、サーバ・クライアント共に実装がシンプルになります。
golang での実装は [kljensen/golang-html5-sse-example](https://github.com/kljensen/golang-html5-sse-example) を参考にさせて頂きました。

## 最後に

ログを人間にいかにわかりやすく見せるかは、開発効率の向上に大きく繋がってくると思っています。
まだ荒削りなクオリティではありますが、簡単に使えるので是非使ってみてください！
