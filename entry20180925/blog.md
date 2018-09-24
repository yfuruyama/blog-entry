Task Queue と Token Bucket アルゴリズム
===

GAE の Task Queue (Push Queue) は Queue に入れられたタスクを全て一気に実行するのではなく、あらかじめ設定しておいた実行レートに従って、バックエンドの App Engine インスタンスにリクエストを投げてくれます。この実行レート制御のベースとなっているのが [Token Bucket](https://ja.wikipedia.org/wiki/%E3%83%88%E3%83%BC%E3%82%AF%E3%83%B3%E3%83%90%E3%82%B1%E3%83%83%E3%83%88) というアルゴリズムです。

今回はその Token Bucket アルゴリズムと、[Task Queue の設定値](https://cloud.google.com/appengine/docs/standard/go/config/queueref) である

* bucket_size
* rate
* max_concurrent_requests

にどのような関連性があるか、まとめてみたいと思います。

## Token Bucket アルゴリズム

Token Bucket はネットワークに流れるトラフィックを一定量以下になるように調整するアルゴリズムであり、[Amazon EBS の IOPS のバースト](https://aws.amazon.com/jp/blogs/aws/new-ssd-backed-elastic-block-storage/) や [Amazon API Gateway での Rate Limit](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/api-gateway-request-throttling.html) でも使われてたりします。

Task Queue 上での Token Bucket アルゴリズムのルールは以下のようになります。

[f:id:furuyamayuuki:20180924114855p:plain:w250]

* タスクを実行する際にはトークンを一つ消費する
* トークンがなくなるまでタスクはディスパッチされ続ける
* バケットにトークンが存在しなければタスクの実行は待たされる
* トークンは一定のレートでバケットに補充される(バケットサイズ分だけ貯められる)

ここで大事なのは、トークンがなければタスクの実行は待たされる = バッファリングされる、という点です。これがあることで、例えタスクが一度に大量に enqueue されても、バックエンドには安定したレートでリクエストが投げられるようになります。これは後で例を用いて説明します。

## Task Queue の設定値

次に Task Queue の各設定値を見てみたいと思います。

#### bucket_size

その名の通りバケットのサイズを決める設定値です。

このバケットサイズ分トークンを貯め込むことができるので、一度に実行が開始されるタスク量はこの値でキャップがかかります(例: バケットサイズが100の場合、一度に実行開始できるタスクは100個まで)。

たまに混乱してしまいますが、Queue に保持できるタスクの量とは関係ありません。
([公式ドキュメント](https://cloud.google.com/appengine/quotas#Task_Queue)によると、課金している場合最大100億タスクまで保持できます。)

#### rate

トークンの補充レートです。

この補充レート以上でタスクが enqueue された場合、例えバケットのサイãºが十分大きかったとしてもいずれバケットの中のトークンは0になってしまうので、長期的に見るとこの値がタスクの最大実行開始レートとなります。

トークンは時間経過によって補充されていきます。実行しているタスクが終わったかどうかは関係ありません。

#### max_concurrent_requests

タスクの最大同時実行数を決める設定値です。

それだけ聞くと `bucket_size` との違いがよくわかりませんが、

* `bucket_size` はある瞬間の最大**実行開始数**を定義する
* `max_concurrent_requests` はある瞬間の最大**同時実行数**を定義する

という違いがあります。

例えば、処理を完了するのに物凄い時間がかかるタスクがあったとします。Token Bucket アルゴリズムでは、今までのタスクが終わってなかろうが、おかまいなしにトークンを補充していくので、時間経過に伴ってタスクが随時実行開始されていき、同時実行しているタスクが積み重なっていくことになります。
そういう場合に `max_concurrent_requests` を設定することで、タスクの同時実行数にキャップをかけることができます。

ちなみに `bucket_size` と `rate` は Token Bucket アルゴリズムに関連したパラメータですが、`max_concurrent_requests` はそれとは全く関係ありません。

## bucket_size の用途

ここからは `bucket_size` の用途を、例を挙げながらもう少し見てみたいと思います。条件として、既に Queue にタスクが十分に積まれていて、それらを秒間最大500リクエストで処理したいとします。またバケットに対してトークンは200ms毎に補充されると仮定します(つまり秒間5回補充のタイミングがある)。

#### bucket_size: 500, rate: 500/s の場合

`bucket_size` と `rate` を両方共500にするとどうなるか見てみましょう。

[f:id:furuyamayuuki:20180924115400p:plain:w400]

横軸が時間で、縦軸が実行開始されるタスクの量です。

タスクはバケットにトークンがある分だけ実行されてしまうので、500個のトークンがあれば500個のタスクが一度に実行されてしまい、スパイクのようなリクエストが飛んでしまいます。
確かにこれでも秒間500リクエスト処理していると言えますが、バックエンドに瞬間的な高負荷がかかってしまいます。

#### bucket_size: 100, rate: 500/s の場合

今度は `rate` は 500/s のまま、`bucket_size` を100にしてみます。

[f:id:furuyamayuuki:20180924115507p:plain:w400]

すると、一回あたりのタスクの実行開始数が100個に抑えられ、小刻みに実行されるようになります。つまり、タスクの実行開始数が平滑化され、同じ秒間500リクエストでもより安定してバックエンドにリクエストを送れるようになります。

このまま更に `bucket_size` を小さくすればより平滑化されるように見えますが、実際は内部的なトークンの補充間隔に左右されてしまうので、[公式ドキュメント](https://cloud.google.com/appengine/docs/standard/go/config/queueref)で推奨されているように `rate` を5で割った `rate/5` の値にしておくのがいいと思われます。

## まとめ

Task Queue の実行レート制御のベースとなる Token Bucket アルゴリズムと、それに関連した3つの設定値

- bucket_size
- rate
- max_concurrent_requests

の役割をまとめました。

適切に設定してバックエンドを突発的な負荷から守るようにしたいですね。
