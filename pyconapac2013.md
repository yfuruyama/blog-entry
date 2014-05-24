メモ...

Python and Neutrons - Keynote
------------------------------
Speaker: Georg Brandl(Python 3.3リリースマネージャー, Sphinxドキュメンテーションシステムの作者)

<p><span itemscope itemtype="http://schema.org/Photograph"><img src="http://cdn-ak.f.st-hatena.com/images/fotolife/f/furuyamayuuki/20130914/20130914110812.jpg" alt="f:id:furuyamayuuki:20130914110812j:plain" title="f:id:furuyamayuuki:20130914110812j:plain" class="hatena-fotolife" itemprop="image" width="400px"></span></p>

- State of Python 3.[34]
    - Python3.3からvirtualenvをサポート
    - Python3.4は現在αモード. 2014年の2月にリリース予定
        - tulip, a new async libary
        - Lots of PEPs under discussion
    - http://python3wos.appspot.com/
- neutronの研究している
    - 研究に使う機材のセットアップなどの自動化をPythonで
- Network Instrument Control System(NICOS)
    - 機材をネットワークに接続して使うためのプロトコル/システム
    - http://arxiv.org/pdf/cond-mat/0210433.pdf
    - Pythonは最近データ解析にもよく使われているため、計測スクリプト+データ解析の両者をPythonで書ける
    - ユーザが使用するコマンド(メソッド)には発生した例外(AttributeErrorとか)をわかりやすく説明するために、デコレータでラップ

Test Failed, Then...
----------------------------------------
Speaker: Toru Furukawa(@torufurukawa)

- 共通のコンポーネントを使っていくつもアプリケーションを作っているので、コンポーネント同士はloosely coupledでwell testedされてないといけない
- "Write small services that speak HTTP and work together"
- Pythonにこだわりすぎると疎結合なコンポーネントにならない。HTTPを使った方がコンポーネント同士のコミュニケーションが容易になる.
- Convert requests function to curl
    - requestsモジュールで書いたHTTPコールをcurlで出力してくれる
    - http://torufurukawa.blogspot.jp/2013/04/requests.html

Fabric for fun and profit
--------------------------
speaker: Jair Trejo(@jairtrejo)

- Q & A
    - どうやってテストするか
        - unittest使ったりmockライブラリ使ったり、普通のPythonコードのようにテストする
    - chefやpuppetと比べた時のfabricの利点は？
        - セットアップが少ない(あれれ。。)
