Task Queue と Token Bucket アルゴリズム
===

GAE の Task Queue (Push Queue) は Queue に入れられたタスクを全て一気に実行するのではなく、あらかじめ設定しておいた実行レートに従って、バックエンドの App Engine インスタンスにリクエストを投げてくれます。
この実行レート制御のベースとなっているのが [Token Bucket](https://ja.wikipedia.org/wiki/%E3%83%88%E3%83%BC%E3%82%AF%E3%83%B3%E3%83%90%E3%82%B1%E3%83%83%E3%83%88) というアルゴリズムです。

今回はその Token Bucket アルゴリズムと、[Task Queue の設定値](https://cloud.google.com/appengine/docs/standard/go/config/queueref)である

* bucket_size
* rate
* max_concurrent_requests

にどのような関連性があるか、まとめてみたいと思います。

## Token Bucket アルゴリズム

Token Bucket はネットワークに流れるトラフィックを一定量以下になるように調整するアルゴリズムであり、[Amazon EBS の IOPS のバースト](https://aws.amazon.com/jp/blogs/aws/new-ssd-backed-elastic-block-storage/) や [Amazon API Gateway での Rate Limit](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/api-gateway-request-throttling.html) でも使われています。

Task Queue 上での Token Bucket アルゴリズムのルールは以下のようになります。

ここに画像貼る

* タスクを実行する際にはトークンを一つ消費する
* トークンがなくなるまでタスクはディスパッチされ続ける
* バケットにトークンが存在しなければタスクの実行は待たされる
* トークンは一定のレートでバケットに補充される(バケットサイズ分だけ貯められる)

なぜ「秒間最大x件のタスクを実行させる」という単純な制御にしていないのでしょうか。
トークンがなければタスクは待たされる = バッファリングされるため、Queue に入ってくる
それはトラフィックをシェーピングできるようにするためです。これは後で例を用いて説明します。

それは一時的なバーストは許可したいが、定常的にはそれより低いスループットで処理させるようにしたいというのがあります。

Task Queue は非同期処理に使われるという性質上、処理しなければいけないタスク量が流動的なことが多いと思います。
いつもは最大 100 tasks/sec で処理したいけれども、たまに 200 tasks が積まれることもある。そういった場合でも遅延なく処理したいという

それでは上記のパラメータについて見ていきます。

## Task Queue の設定値
### bucket_size

その名の通りバケットのサイズを決める設定値です。
このバケットサイズ分トークンを貯め込むことができるので、そのサイズ分のバーストを許可するという意味になります。

たまに混乱してしまいますが、Queue に貯められるタスクのサイズとは関係ありません。
Queue のサイズは[ドキュメント](https://cloud.google.com/appengine/quotas#Task_Queue)によると、最大100億タスクまで保持できるようです。

# パラメータその2: rate

トークンの補充レートです。
ここで大事なのはこのレートが

トークンは時間経過によって補充されていきます。実行しているタスクが終わったかどうかは関係ありません。

この `rate` より多くタスクが入ってきたとしても、いずれバケットの中のトークンが0になってしまうので、この rate がキャップとなる。
つまり継続して行われる秒間の最大実行可能数。

# パラメータその3: max_concurrent_requests

これが非常に厄介なパラメータです。
`bucket_size` と `rate` は Token Bucket アルゴリズムに関連したパラメータですが、`max_concurrent_requests` はそれとは全く関係ありません。

逆にこの設定によってキャップがかかってしまい思うような実行レートにならないことも考えられるため、基本は `rate < max_concurrent_requests` にしておくといいと思います。

つまり `bucket_size` と `rate` は秒間での最大実行**開始**数を定義しますが、`max_concurrent_requests` はある瞬間の最大実行数を定義します。

# 例で理解する

既に Queue にタスクが積まれていて、それらを秒間最大500リクエストで処理したいとします。
またバケットに対してトークンは200ms毎に補充されると仮定します(つまり秒間5回補充のタイミングがある)。

## bucket_size: 500, rate: 500/s の場合

`bucket_size` と `rate` を両方共500にするとどうなるか見てみましょう。

ここにグラフ貼る

タスクはバケットにトークンがある分だけ実行されてしまうので、500個のトークンがあれば500個のタスクが一度に実行されてしまい、スパイクのようなリクエストが飛んでしまいます。
確かにこれでも秒間500リクエスト処理していると言えますが、もう少しなめらかに実行してもらいたいです。

## bucket_size: 100, rate: 500/s の場合

今度は `rate` は 500/s のまま、`bucket_size` を100にしてみます。

ここにグラフ貼る

すると、1秒内でのタスクの実行数が平滑化され、同じ秒間500リクエストでもより安定してリクエストを送れていると思います。
このまま更に `bucket_size` を小さくすればより平滑化されるように見えますが、実際は内部的なトークンの補充間隔に左右されてしまうので、[公式ドキュメント](https://cloud.google.com/appengine/docs/standard/go/config/queueref)で推奨されているように `rate/5` の数値にしておくのがいいと思います。

# まとめ

Task Queue の設定値である、

* bucket_size
* rate
* max_concurrent_requests

の内容についてまとめました。

