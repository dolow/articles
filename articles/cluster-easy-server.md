---
title: "黒い画面を触らずにサーバーを（できればタダで）作ろう！"
emoji: "🌮"
type: "tech"
topics: ["cluster", "javascript", "cloudfunctions", "gas", "gcp"]
published: false
---

こんにちは Smith です、クラスター株式会社のプラットフォーム事業部でプロダクトマネージャーをやっています。
この記事の絵文字はタコスですが、ジョジョの奇妙な冒険というマンガでは「タコス」と言いながらやられるモブがいました、確か2部です。

この記事は クラスター Advent Calendar 2023 の 17日目の記事です。
もしも 12/17 中に公開されていたらたくさん褒めてあげて下さい。


# 本記事について

ご存知ないかも知れませんが、実はクラスター株式会社では、cluster というバーチャル空間で好きなように自分のワールドを作れたり、誰かが作った様々なワールドやイベントを訪れて楽しめるサービスを提供しています。すごいでしょう。

https://cluster.mu/

cluster では Cluster Creator Kit と呼ばれるワールドやワールド内のアイテム、アバターのアクセサリーが作れるツールを Unity の Package Manager で配布しています。
本記事では、そんな Cluster Creator Kit とは無縁のはずのサーバーっぽいものを作るための手法を幾つか紹介します。
明後日くらいには役に立つかも知れません。

なお、いずれも 2023/12 時点の情報ですので、未来にこの記事を参照されている方は各サービスの公式ドキュメントも併せて参照していただけると助かります。

# 想定読者

様々なバックグラウンドを持つ方に読んでいただけるようにしています。
できるだけ噛み砕いて書いたり、あえて正確性を犠牲にしている箇所もありますが、それでも内容に難易度の差が生じています。
各項目に難易度の目安を付けましたので、ちょっと難しい内容にもチャレンジいただけると幸いです。
また、記載内容の誤りについての指摘なども謹んで大歓迎です。

## 難易度表

- 🌶️ => 日常的なインターネットリテラシーで大丈夫
- 🌶️🌶️ => ちょっとインターネットや Web に詳しい必要がある
- 🌶️🌶️🌶️ => ある程度の専門知識が必要


# 前提知識 🌶️~🌶️🌶️

## サーバーってなんですか

サーバーという言葉自体は広義ですが、本記事ではサーバーを以下の定義に限定します。

- HTTP リクエストやレスポンス（後述）を送受信できる、いわゆる HTTP サーバー
- 受け取ったリクエストのパラメーター（後述）に応じでレスポンスの内容を変えることができる

例えばブラウザで https://cluster.mu という URL を指定して開いた時、URL の先にはサーバーが居ます。
サーバーは URL に応じて表示すべき内容をブラウザに返します。

同じ様に、custer のワールド内のメニューから自分のワールド一覧を表示しようとした時は、 cluster アプリがクライアント（後述）として cluster のサーバーに自分のワールドの情報を返すようにリクエストしています。


## クライアントってなんですか

サーバーからデータを送ってくれるように要求する役割を指します。
Chrome や Safari のような Web ブラウザは、もっとも身近なクライアントのひとつです。
cluster アプリも cluster のサーバーに対してリクエストしているのでクライアントとなります。

サーバーやクライアントはデータの送受信の関係上の役割を指すものであるため、ひとつのソフトウェアやプログラムが両方の役割を持つこともあります。


## HTTP リクエストやレスポンスってなんですか

ブラウザで任意の URL を開いた時、裏ではブラウザが「このデータをください」と、HTTP リクエストを送信しています。
HTTP リクエストとは、サーバーに対して「このデータをください」のような要求することです。
それに対してレスポンスとは、サーバーがリクエストに対してデータを返すことです。


#### じゃあ、HTTP ってなんですか

リクエストする側もレスポンスを返す側も、リクエストやレスポンスの内容を同じ知識で書いたり解釈するために共通の取り決めが用いられています。
その一連の取り決めの名称が HTTP です。

HTTP は、正式名称の **H**yper**t**ext **T**ransfer **P**rotocol を略したものです。


#### URL は http://〜 じゃなくて https://〜 ってなってるけど？

良いところに気が付きましたね。流石です。

HTTPS は、 **H**yper**t**ext **T**ransfer **P**rotocol **S**ecure を略したもので、その名の通り HTTP のセキュリティを高めたバージョンです。
通信内容は暗号化されるため、 http://〜 で接続される URL よりもサーバーとやりとりするデータの安全性が高いとされています。

ここ 10年ほどで Web の安全性への意識が大きく高まったことにより、今ではほとんどの Web サイトやサービスが HTTPS で提供されています。

参考:
https://developer.mozilla.org/ja/docs/Glossary/HTTPS

## パラメーターってなんですか

HTTP リクエストの文脈におけるパラメーターは、状況に応じて変えることのできる付加的な送信情報のことを指します。
普段から目にすることのできる例としては、URL に `?` から始まる文字列としてが付与されているパラメーターが多いかと思います。

この記事を掲載している zenn というサイトには、 PC であれば一番上の方に記事の検索窓が表示されていると思いますが、その検索窓に任意の文字列を入力して検索を実行した時、末尾に検索語句が含まれた URL がブラウザのアドレスバーに表示されると思います。
この `?` 以降の文字列がパラメーターの一種です。

例) `golang` というワードで検索した場合

![GAS](/images/cluster-easy-server/httpmethod1.png =250x)

他にも、何らかの Web サイトにログインするような時は、 URL に表示されるものとは別の形でサーバーへのリクエストにパラメーターが付与されています。


## HTTP リクエストメソッドってなんですか

HTTP リクエストが、どんなことをしたいのかを表すのがメソッドです。
HTTP リクエストで送信されるデータの中にメソッドを指定する情報が含まれます。
メソッドは文字列で表現され、 `GET` や `POST` などを始めとした多くの種類が予め取り決められています。

例えば `GET` メソッドは、単にサーバーからデータが欲しい場合に用いられます。
ブラウザで任意の URL を開くとき、フォームの送信などの特殊なケースを除いたほとんどの場合は `GET` メソッドを指定した HTTP リクエストでデータを取得しようとしています。

`POST` メソッドは、主にクライアントからサーバーに対してデータを送信したい場合に用いられます。
ブラウザの場合、フォーム入力内容をサーバーに保存して欲しい場合などで使われています。

今回、サーバーを作るためのサービスを探す条件として `POST` メソッドが扱えることを条件にしています。


参考:
https://developer.mozilla.org/ja/docs/Web/HTTP/Methods


# サーバーを作ろう！（できればタダで）

本記事ではサーバーを(できればタダで)作る手段として、以下の条件に基づいてサービスを選出しました。

- 無料か、限りなく安く使えること
- 黒い画面を叩かなくでも済むこと
- POST メソッドを扱えること

この条件下で2つほど選出したので、以降はそれぞれの利用方法を解説していきます。
奇しくもいずれも Google 製品ですが私は Google の回し者ではありません、普段は AWS を使っています。

- Google App Script
- Cloud Functions (Google Cloud Platform)



## Google App Script 🌶️~🌶️🌶️

Google App Script （以下、GAS）は、Google アカウントを持っていれば無料で利用できるサービスです。

https://script.google.com/home

任意の処理をスクリプトで記述し、柔軟な実行設定をすることであらゆるタスクをこなすことができます。
Google Spread Sheet や Google Forms など、他の Google のサービスとの連携が容易なところが大きな利点ですが、サーバーとして利用とした場合の Google Cloud Platform （以下、GCP）の設定は慣れない人にとっては煩雑です。（私もスマホアプリ開発で触る機会がありますが、未だに慣れません）
本記事では GCP の設定方法も掲載していますが、あくまでもとりあえずサーバーとして動かすための設定ですのでご了承下さい。

なお、ここで記載する手順はすでに Google アカウントを持っていることを前提とします。

### 手順

GAS は Google Drive 上のファイルの一つとしても作成できます。
今回は Google Drive から新しく GAS を作成します。

![google drive new GAS](/images/cluster-easy-server/drive2.png)

作成された GAS を開くとスクリプトエディタが表示されます。
ここに HTTP サーバーとしての処理を記述します。
GAS を記述するために用いる言語は JavaScript で、 CCK の ScriptableItem で用いている言語と同じです。

![GAS](/images/cluster-easy-server/gas1.png)

今回は単に `Hello!` と返すだけの処理を返していきます。
関数内の処理は自由に実装できますが、関数名の `doGet` は変更しないで下さい。

```js
function doGet(e) {
  const output = ContentService.createTextOutput();
  output.setMimeType(ContentService.MimeType.TEXT);
  output.setContent("Hello!!");
  return output;
}
```

:::details TIPS: ちょっとスクリプトの解説を・・・

JavaScript を1から説明すると、この記事ではとても収まりきらない長い物語になるので、ここでは主に CCK の ScriptableItem との差分で解説していきます。

##### 1行目: 関数


```js
function doGet(e) {
    ...
}
```

`function` から始まる記述は関数宣言です。
書き方は異なりますが、ScriptableItem で `$.onInteract()` の引数として書いている内容も関数です。

例えば下記の3つは、書き方は異なるもののやっていることはほぼ同じです。

```js
$.onInteract(() => {
  $.log("hello!");
});
```

```js
const callback = () => {
  $.log("hello!");
};

$.onInteract(callback);
```

```js
function callback() {
  $.log("hello!");
}

$.onInteract(callback);
```

`function` から始まる関数宣言と `() => {}` のようなアロー関数は似て非なるものですが、今のところ CCK ではその違いを意識する必要はあまりありません。
また、ちゃんと解説するととても長くなるためこの記事では解説を割愛します。
いい感じに使い分けられるとかっこいいので、興味があればぜひ MDN のリファレンスから読んでみて下さい。

参考:
https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Functions


##### 2~4行目: レスポンスの生成

```js
const output = ContentService.createTextOutput();
output.setMimeType(ContentService.MimeType.TEXT);
output.setContent("Hello!!");
```

`ContentService.createTextOutput()` は GAS で予め用意されている API です。
素の JavaScript にはありません。
CCK の ScriptableItem でも、 `$` だったり `Vector3` など、素の JavaScript には存在しないクラスやオブジェクトが多く提供されています。

GAS の `ContentService.createTextOutput()` の公式リファレンスは下記を参照して下さい。

https://developers.google.com/apps-script/reference/content/content-service#createTextOutput()


##### 5行目: レスポンスの返却

```js
return output;
```

GAS の場合、 関数の返り値がレスポンスのデータとして利用されます。
ここでは 2~4行目で生成したレスポンスを返しています。

:::


:::details TIPS: `doGet` などのメソッド名指定について

本来、JavaScript の関数名は任意で定義できますが、GAS でサーバーを作る場合は `doGet` や `doPost` など、`do` に続けてキャメライズした HTTP メソッド名を連結してください。
同様に、引数も所定のオブジェクトが渡されることになっています。

GAS の仕様として、 HTTP リクエストを受け取った際に実行する関数がこの様に指定されているためです。

:::


スクリプトをコピペし終わったら、右上のデプロイボタンを押して `新しいデプロイ` を選択します。

![GAS](/images/cluster-easy-server/gas2.png)

表示されたダイアログから歯車アイコンを押下し、`実行可能 API` をクリックします。

![GAS](/images/cluster-easy-server/gas4.png =500x)

はじめて GAS でサーバーを作ろうとしている場合、ここでプロジェクトの種類の変更を求められます。
この場合、`プロジェクトの種類の変更` ボタンを押下します。

![GAS](/images/cluster-easy-server/gas5.png =500x)

遷移先の下部にある `プロジェクトの変更` ボタンを押下します。

![GAS](/images/cluster-easy-server/gas6.png)

:::details 変更先のプロジェクトがまだ無い場合

#### 変更先のプロジェクトがまだ無い場合

ここはプロジェクトがまだない場合の手順です。
すでにある場合は次の `変更先のプロジェクトがある場合` までスキップしてください。

プロジェクトの変更手順が表示されるので、 **手順1** の `こちら` のリンクをクリックします。

![GAS](/images/cluster-easy-server/gas7.png =500x)

Google Cloud のサイトに遷移します。
左上の Google Cloud の右にあるプロジェクト選択タブを押下します。

![GCP](/images/cluster-easy-server/gcp1.png)

表示されたダイアログの右上の `新しいプロジェクト` を押下します。

![GCP](/images/cluster-easy-server/gcp2.png =500x)

任意のプロジェクト名を入力します。
その他はデフォルトの値で大丈夫です。
`作成` ボタンを押下します。

![GCP](/images/cluster-easy-server/gcp3.png =500x)

左上の Google Cloud の右にあるプロジェクト選択タブの表示が、いま作成したプロジェクト名になったことを確認します。
この状態で、左上のメニューから `API とサービス` > `OAuth 同意画面` を選択します。

![GCP](/images/cluster-easy-server/gcp4.png =400x)

OAuth 同意画面の作成ページに遷移します。
`内部`、`外部` の選択はどちらを選択しても良いです。
`作成` ボタンを押下します。

![GCP](/images/cluster-easy-server/gcp5.png =500x)

任意のアプリ名を入力します。
その下のユーザーサポートメールと、最下部のデベロッパーの連絡先情報は任意のメールアドレスを入力します。
その後、最下部の `保存して次へ` を押下します。

![GCP](/images/cluster-easy-server/gcp6.png =500x)

スコープの設定画面は特に変更する項目は無いので、最下部の `保存して次へ` ボタンを押下します。

![GCP](/images/cluster-easy-server/gcp8.png =500x)

テストユーザーの設定画面も特に変更する項目は無いので、最下部の `保存して次へ` ボタンを押下します。

![GCP](/images/cluster-easy-server/gcp9.png =500x)

`OAuth 同意画面` の設定が完了しました。
左上のメニューから `Cloud の概要` > `ダッシュボード` を選択し、プロジェクト情報のペインに表示されるプロジェクト番号を控えます。
この番号は、先程の GAS のデプロイ設定に使用します。

![GCP](/images/cluster-easy-server/gcp10.png)

GAS の画面に戻り `プロジェクトの変更` ボタンを押下します。

:::

表示されるプロジェクト番号入力フィールドに、プロジェクト番号を入力してプロジェクトを変更します。

![GAS](/images/cluster-easy-server/gas8.png =500x)

デプロイ設定ウィンドウから `ウェブアプリ` を選択します。

![GAS](/images/cluster-easy-server/gas9.png =500x)

ウェブアプリの項目の `次のユーザーとして実行` を自分にし、`アクセスできるユーザー` を `全員` にします。
実行可能 API の項目の `アクセスできるユーザー` は `自分のみ` のままです。

![GAS](/images/cluster-easy-server/gas10.png =500x)

デプロイボタンを押下すると利用可能な URL が発行されます。

![GAS](/images/cluster-easy-server/gas11.png =500x)


`ウェブアプリ` の項目の URL に実際にアクセスして `Hello!` と表示されたら成功です。
GAS でサーバー処理が実行されています。

![GAS](/images/cluster-easy-server/gas12.png =500x)

以上が GAS を用いたサーバー作成の手順です。


:::details TIPS: curl での動作確認 🌶️🌶️🌶️

（ここは黒い画面の話です、すいません）

GAS で公開したサーバーは POST で送信したリクエストを受け付けることができますが、その場合は `doGet` ではなく `doPost` として処理を記述します。
動作確認は `curl` コマンドでも行えますが、GAS で発行される URL はリダイレクトを一度挟んでおり、またリダイレクト後は POST リクエストは受け付けないようです。
そのため、 `curl` コマンドのオプションにワークアラウンドが必要です。
メソッドを明示せずにペイロードを指定する、というオプションの指定の仕方をすることでリクエストを成功させることができます。

```sh
curl -d "<data>" -L <endpoint>
```

:::

## Cloud Functions (Google Cloud Platform) 🌶️🌶️~🌶️🌶️🌶️

Cloud Functions は GCP で提供されているサービスのひとつです。
サーバレスで任意のコンピューティングを行うことができ、実装言語の選択肢も豊富です。
本格的なサービスを構築する環境として適していますが、その分 GCP での設定が煩雑だったり必要な知識が多いことがネックです。
Cloud Functions は有料サービスですが、今回のお試しで使う範囲であれば課金されることはないでしょう。
本記事執筆時点で月次の最初の 200万リクエストは無料で、それ以降も 100万リクエストあたり $0.40 と非常に安価です。
最新の料金表は [こちら](https://cloud.google.com/functions/pricing?hl=ja) を参照してください。


### 手順

Google アカウントを持っていることを前提とします。
[GCP のサイト](https://console.cloud.google.com/welcome) にアクセスします。

![Cloud Functions](/images/cluster-easy-server/cf1.png)

必要に応じて新しいプロジェクトを作成します。手順は GAS で示した手順と同じです。
左上の　Google Cloud ロゴの右のプルダウンで意図通りのプロジェクトが選択されていることを確認し，左上のメニューの中から `Cloud Functions` を選択します。
`Cloud Functions` は最下部の `その他のプロダクト` を開いてスクロールした下の方にある場合があります。

:::details 「課金を有効にすると〜」 と表示されている場合

`Cloud Functions` のメニューに遷移した際、画面上部に `課金を有効にすると〜` と表示されている場合は支払い方法を設定する必要があります。
その場合は左上のメニューの中から `お支払い` を選択して下さい。

![Cloud Functions](/images/cluster-easy-server/cf9.png)

このプロジェクトに請求先が設定されていない旨が表示されるため、 `請求先アカウントをリンク` から新しい請求先を作成します。
請求先を作成したら `Cloud Functions` のメニューに戻り、右上の `課金を有効にする` から請求先アカウントをこのプロジェクトに紐つけます。

![Cloud Functions](/images/cluster-easy-server/cf24.png)

:::

上部中央の `ファンクションを作成` を押下します。

![Cloud Functions](/images/cluster-easy-server/cf25.png)

初回のファンクション作成の場合は `必要な API の有効化` ダイアログが表示されるため、有効にします。

![Cloud Functions](/images/cluster-easy-server/cf26.png =500x)

その後の設定画面です。
リージョンを `asia-northeast1 (東京)` にします。
`トリガー` の設定で `HTTPS` のトリガーの `未認証の呼び出しを許可` にチェックを入れます。

![Cloud Functions](/images/cluster-easy-server/cf27.png =500x)

その他はデフォルト値で大丈夫です。
画像ではタイムアウトを 5秒としています。
`次へ` を押下します。

![Cloud Functions](/images/cluster-easy-server/cf28.png =500x)

初回の場合、ここでも `必要な API の有効化` ダイアログが表示されるため、有効にします。

![Cloud Functions](/images/cluster-easy-server/cf29.png =500x)

任意の言語でソースコードを編集できます。
CCK の ScriptableItem で用いている JavaScript を利用したい場合、左上の `ランタイム`　のプルダウンから `Node.js` を選択して下さい。
Node.js の横についている番号はバージョン番号です、特にこだわりがなければ最新版の選択で良いと思います。
スクリプトエディタには、デフォルトで最低限動作するコードが予め入力されています。
このままデプロイすることで簡単な動作を確認できます。

```js
const functions = require(`@google-cloud/functions-framework`);

functions.http('helloHttp', (req, res) => {
  res.send(`Hello ${req.query.name || req.body.name || 'World'}!`);
});
```

:::details TIPS: 続・ちょっとスクリプトの解説を・・・

ここも普段の CCK のスクリプトで見かける記述とは異なるものがあるので、その辺りを解説していきます。

##### 1行目: require

```js
const functions = require(`@google-cloud/functions-framework`);
```

`reuire` は外部のライブラリやモジュールを読み込むための構文です。
CCK では構文自体はサポートしていますが、実際に読み込ませることはサポートしていません。


##### 4行目: テンプレートリテラル (``` `` ```)

```js
res.send(`Hello ${req.query.name || req.body.name || 'World'}!`);
```

文字列はシングルクォートやダブルクォート以外に、バックティック (``` `` ```) でも定義できます。
シングルクォートやダブルクォートとの違いは、文字列中で変数を連結する方法が提供されていることです。
文字列中に `${}` で変数を囲うことによって連結できます。
また、上記の実装例のように条件に合わせた値の連結も可能です。

下記の 2 つは同じことをしています。

```js
const s = "world";
const phrase = "hello " + s + "!";
```

```js
const s = "world";
const phrase = `hello ${s}!`;
```

:::

左下の `デプロイ` を押下します。

![Cloud Functions](/images/cluster-easy-server/cf30.png)

画面が遷移し、しばらくするとデプロイが完了します。
画面上部に表示されている URL にアクセスすることで、サーバーが動作していることが確認できます。

![Cloud Functions](/images/cluster-easy-server/cf34.png =500x)

# まとめ

本記事では2つのサーバーを（できればタダで）作る手段を紹介しました。
それぞれの方法の手順のサマリとイチオシポイントは下記のとおりです。

## Google App Script 🌶️~🌶️🌶️

- Google Drive で GAS を作る
- GAS にスクリプトを書く
- GCP でプロジェクトを作成する
- GCP で作成したプロジェクトの OAuth 同意画面を設定する
- GAS で GCP プロジェクトを設定する
- GAS をデプロイする

### イチオシポイント

- 身近なツールで実現できる
- スプレッドシートなどとの連携が取りやすい

## CloudFunctions 🌶️🌶️~🌶️🌶️🌶️

- GCP でプロジェクトを作成する
- お支払情報を設定する
- GCP で作成したプロジェクトにお支払情報を紐つける
- CloudFunctions の設定をする
- CloudFunctions のスクリプトを書く
- CloudFunctions をデプロイする

### イチオシポイント

- 他の GCP サービスとの連携が容易
- タイムアウトなど、よりテクニカルな設定ができる


# むすびに

cluster では Cluster Creator Kit と呼ばれるワールドやワールド内のアイテム、アバターのアクセサリーが作れるツールを Unity の Package Manager で配布しています。
本記事では、そんな Cluster Creator Kit とは無縁のはずのサーバーっぽいものを作るための手法を幾つか紹介しました。
もしかしたら明後日くらいには役に立つかも知れません。

ピタゴラスイッチのように、本来は別々のもの同士を統合して作り手の目的に適った動作をさせられることはデベロッパーの醍醐味です。
CCK を用いたクリエイター活動でも、何かと何かを組み合わせて思い通りにうまく動いた！という瞬間は楽しいですよね。
これからも組み合わせ用の素材というクリエイターにとってのエンドコンテンツの提供を通して、私達も想像できなかったようなクリエイティビティを目の当たりにしていきたいと思います。

