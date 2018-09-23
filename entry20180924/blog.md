Task Queue の実行レート制御
===

GAE の Task Queue は Queue に存在するタスクを全て一気に実行するのではなく、あらかじめ設定しておいた実行レートに従ってバックエンドの App Engine インスタンスにリクエストを投げてくれます。
設定値には[様々なパラメータ](https://cloud.google.com/appengine/docs/standard/go/config/queueref)があり少し煩雑に見えますが、実行レートを制御するには以下の3つを抑えればokです。

* bucket_size
* rate
* max_concurrent_requests

今回はこの3つのパラメータに関してまとめていきたいと思います。

# ベースとなる技術 - Token Bucket アルゴリズム

パラメータの解説の前に Task Queue の実行レート制御のベースとなっている [Token Bucket](https://ja.wikipedia.org/wiki/%E3%83%88%E3%83%BC%E3%82%AF%E3%83%B3%E3%83%90%E3%82%B1%E3%83%83%E3%83%88)というアルゴリズムを見てみましょう。

Token Bucket はネットワークに流れるトラフィックを一定量以下になるように調整するアルゴリズムであり、[Amazon API Gateway](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/api-gateway-request-throttling.html) でもクライアントへの Rate Limit として使われています。

Task Queue 上での Token Bucket アルゴリズムのルールはシンプルです。

ここに画像貼る

* タスクを実行する際にはトークンを一つ消費する
* バケットにトークンが存在しなければタスクの実行は待たされる
* トークンは一定のレートでバケットに補充される(バケットサイズ分だけ貯められる)

なぜ「秒間最大x件のタスクを実行させる」という単純な制御にしていないのでしょうか。
それは一時的なバーストは許可したいが、定常的に最大スループットを出したくない

Task Queue は一度にドバっと Queue に入れられることがあります。

上記のパラメータのうち `bucket_size` と `rate` は Token Bucket アルゴリズムに関連した設定値です。

# パラメータその1: bucket_size

その名の通りバケットのサイズを決める設定値です。
このバケットサイズ分トークンを貯め込むことができるので、そのサイズ分のバーストを許可するという意味になります。

たまに混乱してしまいますが、Queue に貯められるタスクのサイズとは関係ありません。
Queue のサイズは[ドキュメント](https://cloud.google.com/appengine/quotas#Task_Queue)によると、最大100億タスクまで保持できるようです。

# パラメータその2: rate

バケットにトークンを補充するレートです。
ここで大事なのはこのレートが

トークンは時間経過によって補充されていきます。実行しているタスクが終わったかどうかは関係ありません。

この `rate` より多くタスクが入ってきたとしても、いずれバケットの中のトークンが0になってしまうので、この rate がキャップとなる。
つまり継続して行われる秒間の最大実行可能数。

ちなみに Task Queue だと `bucket_size` 以上の `rate` を指定しても
例えば `bucket_size: 1`, `rate: 5/s` で試すと、実際の実行数は 5 req/s となっていました。

ここに画像を貼る

# パラメータその3: max_concurrent_requests

これが非常に厄介なパラメータです。
`bucket_size` と `rate` は Token Bucket アルゴリズムに関連したパラメータですが、`max_concurrent_requests` はそれとは全く関係ありません。

逆にこの設定によってキャップがかかってしまい思うような実行レートにならないことも考えられるため、基本は `rate < max_concurrent_requests` にしておくといいと思います。

つまり `bucket_size` と `rate` は秒間での最大実行**開始**数を定義しますが、`max_concurrent_requests` はある瞬間の最大実行数を定義します。

# FAQ
