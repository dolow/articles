---
title: "関心事を分離する / Unity におけるコンポーネント設計事例"
emoji: "🍣"
type: "tech"
topics: ["unity"]
published: true
---

こんにちは、Smith ([@do_low](https://twitter.com/do_low)) です。
本稿では 2020/12 に実施された [Unity 1 Week Game Jam](https://unityroom.com/unity1weeks) に投稿した ManMachine という作品をベースに、Unity におけるコンポーネント設計の一例を紹介します。

[ManMachine: Unity 1 Week Game Jam](https://unityroom.com/games/man-machine)
[ManMachine: dolow.github.io](https://dolow.github.io/games/unity1week/ManMachine/)

Web アプリと違って鉄板が無いように見えるゲームアプリの設計ですが、本稿が Unity を用いた開発の一助と慣れば幸いです。

ソースコードは unity プロジェクトごと github に push しています。
[https://github.com/dolow/ManMachine](https://github.com/dolow/ManMachine)

ライセンスの都合上、一部のアセットが除かれているのでそのままでは動きません。

# 前提

## 環境

本稿では下記の環境を前提に記述します。

- Unity 2019.4.14f1
- MacOS 10.15.7
- Chrome 87.0.4280.88

## 題材のゲームについて

ManMachine というゲームについての前提知識です。
ManMachine は 3D のゲームで、自動的にスポーンする NPC をうまくゴールに導くことができればステージクリア、というゲームです。
プレイヤーは TPS 視点でプレイヤーキャラクターを操作し、ステージ上の各所に設置されたギミックを操作して NPC を導きます。
ただし、ステージ上では TPS 視点では見えにくい箇所なども存在するため、俯瞰視点のカメラに切り替えながらギミックを操作する、というテクニックも求められます。

以下で、本稿で触れる範囲のゲーム仕様について説明します。
(本稿執筆時点での詳細仕様です)

### ユーザ入力

以下の入力インターフェースに対応しています。

- キーボード
- タッチスクリーン
- マウス

また、直接の入力インターフェース以外に、ゲーム内には HUD も用意しています。
ユーザー操作によって作用する効果は下記のとおりです。

#### プレイヤーの操作

キーボードはUS配列のみしか想定していません、すいません。

| 操作 | キーボード | タッチスクリーン | マウス |
| --- | --- | --- | --- |
| 前進、後退 | W,S | 縦スワイプ | 縦ドラッグ |
| 回転| J,K | 横スワイプ | 横ドラッグ |
| 左右移動 | A,D | - | - |
| インタラクト | スペース | タップ | 左クリック |
| 主カメラ切り替え  | U | (HUD) | (HUD) |
| TPS カメラ左右切り替え | I | (HUD) | (HUD) |

#### HUD とカメラ切り替え

| ボタン | 操作 | 切り替え前 | 切り替え後 |
| --- | --- | --- | --- |
| ![主カメラ切り替え](https://storage.googleapis.com/zenn-user-upload/im6agqdrzhg7ilabhfx85jljsuno =64x) | 主カメラ切り替え | ![TPS](https://storage.googleapis.com/zenn-user-upload/cbf1nn6ck19ftot71bjxq92402n1 =180x) | ![俯瞰](https://storage.googleapis.com/zenn-user-upload/zc0wbe52ursis1p4khscv8y8dvuo =180x) |
| ![TPS カメラ左右切り替え](https://storage.googleapis.com/zenn-user-upload/7tkkk7g9rdotl49djcu96zgs4ogd =64x) | TPS カメラ左右切り替え | ![右](https://storage.googleapis.com/zenn-user-upload/cbf1nn6ck19ftot71bjxq92402n1 =180x) | ![左](https://storage.googleapis.com/zenn-user-upload/1u1vo0n10vicej3cf5nbws9k6sr0 =180x) |
| ![ステージリスタート](https://storage.googleapis.com/zenn-user-upload/ig8aixls8my2yes5z9vw8l9t5m2j =64x) | ステージリスタート | - | - |

### ギミック

ギミックは、本稿執筆時点では下記の 2種類です。

#### ドアの開閉

今回の Unity 1 Week Game Jam のテーマが「あける」だったため取り入れた要素です。
プレイヤーが錠前のアイコンにインタラクトすると開閉できます。
ステージ開始直後だと NPC の導線を塞いだりしています。

| インタラクト | 開閉 |
| --- | --- |
| ![インタラクト](https://storage.googleapis.com/zenn-user-upload/lj5vmc23b1vb81e3783svwost3kh =128x) | ![開閉](https://storage.googleapis.com/zenn-user-upload/riwsrivfu5epm2evumht13xi2qks =180x) |

#### 向きの変更

プレイヤーや NPC が踏むと進行方向が変更されるギミックです。
回転しているようなアイコンにプレイヤーがインタラクトすると向きを変更できます。

| インタラクト | 向きの変更 |
| --- | --- |
| ![インタラクト](https://storage.googleapis.com/zenn-user-upload/6jfjfj8c7eewbcxbf91iovyyo0il =128x) | ![向きの変更](https://storage.googleapis.com/zenn-user-upload/54bj8lry6bwfl502097nm8xndvqg =300x) |


# 設計

ここからはコンポーネント設計について、主な要素ごとに考えていきたいと思います。
本稿では大まかに2つの領域について触れます。

__ユーザインターフェース__

ここでのユーザインターフェースは、いわゆる HUD ではなくキーボードなどのユーザー入力のインターフェースを指します。
ゲームの世界の外からゲーム内の要素を操作しユーザにフィードバックするという一連の流れです。
入力から出力までの流れが非常に長く、複雜になりがちな部分であるため、ちょっと気を使いたい部分です。


__ゲーム仕様__

ゲームの仕様はそれぞれ独創性があってよいのですが、できるだけメンテナンシビリティとトレードオフにはしたくありません。
コンポーネント指向に基づいた、ある程度の拡張性が担保できそうな設計にしたいと思います。

## ユーザインターフェース編

最近ではクロスプラットフォームでのリリースは当たり前になっているので、今回のゲームもモバイルと PC でのプレイを想定して 3種類の入力インターフェースに対応するようにしたいと思います。 (リリースする予定はありませんが・・・)
マウスでもキーボードでもプレイヤーの操作ができるようにし、併用も可能にします。

- キーボード
- タッチスクリーン
- マウス

### 予想される問題

ユーザインターフェースは、例えば「キーボードの W を押したら前進する」というように、どんなインターフェースであれ入力に対して出力が伴います。
しかし、「W」のキー入力と「前進する」という出力とはそもそも全くの別物です。
「前進する」という振る舞いの実現自体は難しくはありませんが、キー入力と「前進する」ことについて混同すると、そのメンテナンシビリティは低くなりがちです。
キー入力や前進させることの責任の所在が不明確であるということは、誰が何にどういう指示や要求を送るべきかが明確ではない、ということになります。
今回、フォーカスしたい問題はおおまかに 2点です。

- ユーザ入力とゲームでの作用は別物にする
- 誰が、誰に、いつ、どのようなメッセージを送ればよいか明確にする

1つめの問題を解決するため、まずは知識のレイヤーを敷きたいと思います。

| 論理名 | 知識 |
| --- | --- |
| Primitive | キー入力などのユーザ入力と、それらの独自の加工方法 |
| Interaction Mediator | ユーザ入力とゲーム知識の相関 |
| Game Logic | ゲームの制御に関わる知識 |

こうすることで、そのレイヤーでは扱えない知識が明確になり、扱える知識への変換という役務が新たに求められることになります。
知識の変換を担うのは、2番目の Interaction Mediator です。

また、メッセージングを整理するためにコンポーネント間でのコミュニケーション導線を図にしました。
Primitive から Game Logic や Presentation に一足飛びにメッセージングするようなことがないことが確認できます。

![interaction overview](https://storage.googleapis.com/zenn-user-upload/miqlr7lsxrha06qvx2dmgh4yqerz)

以降、個別の領域について詳解していきます。


### Primitive

![primitive diagram](https://storage.googleapis.com/zenn-user-upload/q87uglv1ys6hr945o7g9pwlhagzc =320x)

Unity API を用いて直接、何らかの入力を受け取って認識し、上位に渡す層です。
単純に入力値を横流しするのではなく、入力値を定義されたインタラクションの最小単位に加工します。
例えばタップであれば、 `Input.touches` で知り得る情報を、下記のように一般的なアプリケーションで求められる定義に解釈させます。

- タッチ開始
- タッチ中
- タッチ終了
- タップ

同様に、定義に付随する情報も提供します。

- タッチ開始座標
- タッチ中のタッチ開始からの移動距離
- タッチの経過時間
- etc...

入力値の定義や加工に関心を持ちますが、入力の加工結果は利用せず他のオブジェクトにメッセージングします。
ゲームの知識は持たないため、そこそこポータブルに扱えると思います。

### Game Domain

ゲームドメインの知識を知り得る層です。
例えば、「前進する」などのゲームならではのセマンティックが理解できます。

#### Interaction Mediator

Interaction Mediator は、Primitive Layer からの入力の情報をゲームドメイン知識のセマンティックに変換し、伝搬する部分です。

![interaction mediator diagram](https://storage.googleapis.com/zenn-user-upload/9ngmnbmawkulerb0l863vp9qu6q7)

Interaction Mediator がサポートする入力インターフェースは選択できるようにします。
これにより、本当に必要なコンポーネントのみをアタッチできるようにします。
つまり、図中の "X to game semantics" と "X Interaction" は Interaction Mediator によって使役される関係となります。

![inspector interaction mediator](https://storage.googleapis.com/zenn-user-upload/ftwcg29akf8j49mj2zrv0v9k3rub =280x)

##### to game semantics

入力インターフェースからの情報のセマンティックへの変換は、インターフェースの種類ごとに専門のコンポーネントを設けます。
変換とは、例えばキー入力の W を「前進」というセマンティックに置き換えるようなことです。
Interaction Mediator は変換後のセマンティックのみを扱い、変換処理そのものは行いません。
複数のインターフェースから受け取ったセマンティックを整理し、ユーザ入力の知識を持たない(必要のない)レイヤーに要求として伝える役務を持ちます。
例として、以下にセマンティックの一部を挙げます。

- 前進
- 後退
- インタラクト
- カメラ切り替え

#### Game Logic

![](https://storage.googleapis.com/zenn-user-upload/p49866adsv2jcrdgrpnv6kjuxqpe)

Interaction Mediator などから「前進したい」「後退したい」などの要求を受け取り、 Presentation 層に反映させる層です。
受け取った要求がユーザ入力に由来するかどうかの関心はなく、その知識も持ちません。
Game Logic は Presentation 層に作用するため、 Scene 内の GameObject などについての知識を持つことができます。
Interaction Mediator に対して何かメッセージングすることはありません。

##### PlayableScene

Interaction Mediator から受け取った要求を処理するコンポーネントです。
要求自体の制御や、要求を自身が知りえる Scene 内の GameObject にメッセージングすることを責務とします。
どんなアニメーションを再生するか、などのような個別の GameObject の具体的な振る舞いは知り得ません。

#### Presentation

![presentation diagram](https://storage.googleapis.com/zenn-user-upload/jitlvltq5nwev2xjtinuypz65jnm =280x)

実際にアニメーションを行ったりする層です。
明確にその役務を負ったコンポーネント以外からは、 Game Logic に対して逆方向のメッセージングをすることはありません。

----

ユーザ入力の知識や責務でシステムを分断しようとした時、そのアウトプットはユーザへのフィードバックにまで及びます。
気付いたらユーザ入力どころかプレゼンテーション層まで切り分けてしまいましたが、結果として明確にユーザ入力に関心がない(知識を持たない)領域がわかりました。
ここからはちょっとだけ実装を見ていきたいと思います。

### ユーザインターフェースの実装

先程までとは逆の順番で、関心事がちゃんと分離されているか確かめるために Presentation 層から追っていきましょう。
最もイメージが付きやすいのはプレイヤーですね、Game Logic 内の `PlayableScene` から、プレイヤーを動かしている部分を見てみましょう。

[PlayableScene](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/Scene/PlayableScene.cs#L181-L192)

```cs:PlayableScene.cs
protected void MovePlayer(float front, float right, float rotate)
{
    if (this.gameFinished)
    {
        return;
    }

    if (this.player != null)
    {
        this.player.Walk(front, right, rotate);
    }
}
```

`MovePlayer()` は `InteractionMediator` のデリゲートメソッドです。
`InteractionMediator` は自身の要求をデリゲートメソッドとして表現しています。
デリゲートメソッドの設定は、同じく `PlayableScene` 自身の `Awake` で設定しています。

[デリゲートメソッドの登録](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/Scene/PlayableScene.cs#L20-L52)

```cs:PlayableScene.cs
InteractionMediator mediator = this.gameObject.GetComponent<InteractionMediator>();
if (mediator == null)
{
    Debug.LogError("InteractionMediator is required");
    return;
}

mediator.RequestMove += this.MovePlayer;
```


`Awake` で `InteractionMediator` を `GetComponent` しているので、 `InteractionMediator` は Scene 内で静的にアタッチされていることがわかります。

`PlayableScene` では、タップやキー入力などの具体的な情報には一切言及していません。
つまり、ユーザ入力を知らなくてもプレイヤーを動かすことができるのです。
この恩恵は、新たに Joy-Con に対応したり、キー配置や操作方法の変更などへの耐性として表れます。
ゲームロジックやプレゼンテーション層は、ユーザ入力に関する変更の影響からは切り離されています。

次いで `InteractionMediator` を見ていきましょう。

#### InteractionMediator

先程の  `PlayableScene` ではプレイヤーを動かすデリゲートメソッドである `RequestMove` に `MovePlayer()` を指定していました。
では、 `InteractionMediator` の `RequestMove` 呼び出し箇所を見てみましょう。

[InteractionMediator.cs](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/GameDomain/InteractionMediator.cs#L117-L129)

```cs:InteractionMediator.cs
if (!this.HasAnyIntention(MoveIntentions))
{
    this.RequestStop?.Invoke();
}
else
{
    this.ClearAnyIntention(MoveIntentions);

    Vector3 movement = this.CompositMoveDirection();
    float rotation = this.CompositRotation();

    this.RequestMove?.Invoke(movement.z, movement.x, rotation);
}
```

`Update()` 内で `this.HasAnyIntention(MoveIntentions)` が真の場合に `RequestMove` を実行しています。
初めてIntent という単語が出てきましたが、和訳すると「意思」や「意図」といった意味合いの単語です。
命名として「クリックされた」などのような入力現象ではなく「前進したい」という要求として表現しています。
`HasAnyIntention()` は引数に指定した意思(Intent)を持っているかを確認するメソッドで、中身は下記のようなものです。

[HasAnyIntention](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/GameDomain/InteractionMediator.cs#L171-L182)

```cs:InteractionMediator.cs
private bool HasAnyIntention(int semantics)
{
    for (int i = 0; i < this.interactionInterfaces.Count; i++)
    {
        if (this.interactionInterfaces[i].HasIntent(semantics))
        {
            return true;
        }
    }

    return false;
}
```

`HasAnyIntention()` 内で参照されている `interactionInterfaces` は、 `AInteractionMediatorInterface` というクラスを要素に持つ `List` です。

```cs:InteractionMediator.cs
private List<AInteractionMediatorInterface> interactionInterfaces = new List<AInteractionMediatorInterface>();
```

`AInteractionMediatorInterface` 継承コンポーネントは、設計の際に図示した "X to game semantics" 相当のコンポーネントで、ユーザ入力をゲーム知識のセマンティックに変換する役割を持つものです。

![interaction mediator diagram](https://storage.googleapis.com/zenn-user-upload/9ngmnbmawkulerb0l863vp9qu6q7)

`AInteractionMediatorInterface` のそれぞれの実装は `Awake()` で初期化されていることが確認できます。

[Awake()](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/GameDomain/InteractionMediator.cs#L93-L113)

```cs:InteractionMediator.cs
private void Awake()
{
   if (this.keyboard)
   {
       this.interactionInterfaces.Add(this.gameObject.AddComponent<InteractionMediatorKeyboard>());
   }
   if (this.touchScreen)
   {
       this.interactionInterfaces.Add(this.gameObject.AddComponent<InteractionMediatorTouch>());
   }
   if (this.mouse)
   {
       this.interactionInterfaces.Add(this.gameObject.AddComponent<InteractionMediatorMouse>());
   }
   if (this.ui)
   {
       InteractionMediatorUI interactionMediatorUi = this.gameObject.AddComponent<InteractionMediatorUI>();
       this.interactionInterfaces.Add(interactionMediatorUi);
       interactionMediatorUi.SetUIInteractionRegistry(this.uiInteractionRegistry);
   }
}
```

`Awake()` では自身のフィールドの真偽値に応じて、使用する入力インターフェースを初期化しています。
ManMachine では、この真偽値を全て静的に真にしています。
(InteractionMediatorUI については後ほど個別に触れます)

このように `InteractionMediator` では、利用する入力インターフェースの初期化と、下位の `AInteractionMediatorInterface` 実装クラスからのメッセージの伝播のみを責務としていることがわかります。
「前進したい」などの Intent (意思) を誰に伝えるかは知識としても責務としても持っていません。
全てデリゲートメソッドの実装者に委譲しているため、 `PlayableScene` などのゲームロジック以外でも用いることが出来ます。

この時点ではまだ具体的なタップなどの参照や加工などは一切行っていませんが、徐々に入力インターフェースに近づいてきました。

#### AInteractionMediatorInterface 継承クラス

`InteractionMediator` のフィールドの `interactionInterfaces` は、 `AInteractionMediatorInterface` を要素に持つ `List` でした。
ここではマウス入力の実装として `AInteractionMediatorInterface` を継承した `InteractionMediatorMouse` を見ていきましょう。

`InteractionMediator` では各入力インターフェースクラスの Intent (意思) を調べていました。
Intent が発生する場所を見てみると、ここでようやく `MouseMove` というメソッド名の「マウスが動いた」という入力から、 `InteractionSemantic.MoveAny` のような「動きたい」「回転したい」といった意思が確認できます。

[InteractionMediatorMouse.cs](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/GameDomain/InteractionMediatorInterface/InteractionMediatorMouse.cs#L21-L29)

```cs:InteractionMediatorMouse.cs
protected void MouseMove(MouseInteraction interaction)
{
    Vector3 moveVector = interaction.GetClickMoveVector(this.screenInteractionMaxX, this.screenInteractionMaxY);
    this.moveDirection.z = moveVector.y;
    this.rotation = moveVector.x;

    this.AddIntent(InteractionSemantic.MoveAny);
    this.AddIntent(InteractionSemantic.RotateAny);
}
```


`MouseMove` は `MouseInteraction` というコンポーネントのデリゲートメソッドとして登録されています。
実は `MouseInteraction` こそ、マウス入力を直接受け取るクラスそのものです。
いずれも `InteractionMediatorMouse` の `Awake()` で登録されていることが確認できます。


```cs:InteractionMediatorMouse.cs
private MouseInteraction mouseInteraction = null;

private void Awake()
{
    this.mouseInteraction = this.gameObject.AddComponent<MouseInteraction>();

    this.mouseInteraction.OnClicking += this.MouseMove;
    this.mouseInteraction.OnClickEnding += this.MouseStop;
    this.mouseInteraction.OnClick += this.MouseAction;
}
```

まだこの層では、 Unity の API からマウス入力を受け取ったり、ドラッグの距離を計測する処理が入っていないことが確認できると思います。
`InteractionMediatorMouse` の関心事は、ユーザ入力の情報をゲームドメインのセマンティックに読み替え、 Intent として上位に伝えることが主務であるためです。

#### MouseInteraction

ユーザインターフェースの旅路も終わりに近づいてきました。
`InteractionMediatorMouse` で `AddComponent` されていた `MouseInteraction`を見ていきましょう。
ここはもはやゲームドメインの外の世界です、そのため「前進」「回転」のような知識だったり、「〜したい」のような意思は持ち合わせていません。
先程の `MouseMove` のデリゲート元である `OnClicking` を見てみましょう。

[MouseInteraction.cs](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/Primitive/Interaction/MouseInteraction.cs#L32-L49)

```cs:MouseInteraction.cs
if (this.device.IsInteracting())
{
    this.positionClickEnd = this.device.InteractingPosition();

    if (!this.clicking)
    {
        this.clicking = true;
        this.positionClickBegan = this.device.InteractingPosition();
        this.clickingDuration = 0.0f;
        this.OnClickBegan?.Invoke(this);
    }
    else
    {
        this.clicking = true;
        this.clickingDuration += Time.deltaTime;
        this.OnClicking?.Invoke(this);
    }
}
```

既にクリック開始していれば `OnClicking`、まだ開始していなければ `OnClickBegan` が呼ばれていることがわかります。
`this.device.IsInteracting()` はマウス入力とタッチ入力を抽象化したクラスのメソッドです。
内部ではようやく `Input.GetMouseButton(0)` をコールしていることがわかります。

[ScreenInteractionDeviceClick.cs](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/Primitive/Interaction/ScreenInteractionDevice/ScreenInteractionDeviceClick.cs#L7-L10)

```cs:ScreenInteractionDeviceClick.cs
public bool IsInteracting()
{
    return Input.GetMouseButton(0);
}
```

`MouseInteraction` では Unity の `Input` からクリック情報を取得しているものの、ゲームのドメイン知識は用いられていません。
このように、知識や責務の範囲を明確にして定義や実装を分離することで、それぞれのクラスの独立したメンテナンシビリティが維持できるようになります。

### Advanced: UI も Primitive に

(この節は私の趣味嗜好なので、読み飛ばしても大丈夫です)

Unity での UI は高級です。
例えば `Button` などは、タッチやクリックをすっ飛ばしてボタン押下後の処理を実行できたりします。
しかしこれではユーザ入力がセマンティックフルになり、上述の思想には適いません。
`Button` は `Button` でユーザ入力の事実のみを扱い、その上位層でセマンティックを解釈するようにします。
そのためには、`MouseInteraction` と同じ層に `UIInteraction` を定義します。
ただし `Input` のようにグローバルに呼べるような低級 API は UI には存在しないため、多少の工夫が必要です。
また、キー入力の `KeyCode` ように、入力した UI を識別する値が必要です。
ユーザ入力から `UIInteraction` を仲立ちする要素として、下記の 2つのコンポーネントを定義します。

- `UIInteractionRegisterer`
- `UIInteractionRegistry`

![ui diagram](https://storage.googleapis.com/zenn-user-upload/7elv0jnsn8n8jawjblosw6rexxao)

これによって、画面上に操作ボタンを出すタイプのゲームではキーコンフィグなどが容易になります。

#### UIInteractionRegisterer

GameObject に `UIInteractionRegisterer` がアタッチされている場合、 `Button` コンポーネントの `onClick` イベントは、そのまま `UIInteractionRegistry` に流されます。
また、上位層がユーザ入力を識別できるように、UI に対する ID が設定されます。

[UIInteractionRegisterer.cs](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/Primitive/Interaction/UIInteractionRegistry/UIInteractionRegisterer.cs)
```cs:UIInteractionRegisterer.cs
public class UIInteractionRegisterer : MonoBehaviour
{
    public int buttonId = -1;
    public UIInteractionRegistry registry = null;

    public void Awake()
    {
        Button button = this.GetComponent<Button>();
        if (button != null)
        {
            button.onClick.AddListener(this.OnButtonPressed);
        }
    }

    public void OnButtonPressed()
    {
        if (this.registry != null)
        {
            this.registry.OnButtonPressed(this);
        }
    }
}
```

#### UIInteractionRegistry

`UIInteractionRegistry` では、 Unity APi の `Input` のように、現在押されているボタンを取得できるようにします。
また、イベントリスナーなどの push 形式ではなく、あくまでも pull する情報として扱います。
ボタンの押下状況のクリアのタイミングは自身だけでは制御できないため、 `ClearPressedButtons()` メソッドを露出させて外部に委譲します。

[UIInteractionRegistry.cs](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/Primitive/Interaction/UIInteractionRegistry/UIInteractionRegistry.cs)

```cs:UIInteractionRegistry.cs
public class UIInteractionRegistry : MonoBehaviour
{
    protected Dictionary<int, UIInteractionRegisterer> pressedButtons = new Dictionary<int, UIInteractionRegisterer>();

    public IEnumerable<UIInteractionRegisterer> PressedButtons
    {
        get
        {
            foreach (KeyValuePair<int, UIInteractionRegisterer> entry in this.pressedButtons)
            {
                yield return entry.Value;
            }
        }
        private set { }
    }

    public void ClearPressedButtons()
    {
        this.pressedButtons.Clear();
    }

    public void OnButtonPressed(UIInteractionRegisterer registerer)
    {
        int id = registerer.GetInstanceID();
        if (!this.pressedButtons.ContainsKey(id))
        {
            this.pressedButtons.Add(id, registerer);
        }
    }
}
```


`UIInteractionRegistry` からユーザ入力を取得する `UIInteraction` 層では、この `PressedButtons` プロパティから、現在押下されている UI ボタンを読み取ります。
実装はマウスと同じ要領なので割愛します。

先程見た `InteractionMediator` の `Awake()` では `InteractionMediatorUI` を初期化していました。
他の入力インターフェースと異なり、 `InteractionMediator` 自身が UI の入力ソースを知らせる必要があるため、利用する `UIInteractionRegistry` を渡しています。


```cs:InteractionMediator.cs
if (this.ui)
{
   InteractionMediatorUI interactionMediatorUi = this.gameObject.AddComponent<InteractionMediatorUI>();
   this.interactionInterfaces.Add(interactionMediatorUi);
   interactionMediatorUi.SetUIInteractionRegistry(this.uiInteractionRegistry);
}
```



`Input` ほどの懐の深さはありませんが、 `UIInteractionRegistry` にさえ登録すれば押下されたボタンを透過的に取得できるようになりました。
正直、ManMachine ではここまでする意義はありません、最初に申し上げたとおり完全に私の趣味嗜好です。

---

ユーザインターフェース編はいかがでしたでしょうか。
構成としては冗長かもしれませんが、関心事や責務の明確化と分離は、チーム開発だったり長く運用するプロダクトで真価を発揮すると思います。

## ゲーム仕様編

ここからはゲーム仕様編で、よりゲームの内容に沿ったものとなります。
そのため事前にゲームを触って頂いておいたほうがイメージしやすくなるかと思います。
(ゲームの面白さはちょっと置いておいてください)
ここでのトピックはギミックにフォーカスしたいと思います。

### 予想される問題

ドアを開閉するギミックを作るとしましょう。
それだけ作るのであれば苦労はありませんが、同じような粒度のギミックを複数作る場合にはある程度の秩序が必要です。
秩序が保たれるように一つひとつのギミックを中央集約的に管理・統制する、いわゆる神クラスを設けても良いのですが、神クラスこそ無秩序の権化です。
ここでもやはり、責任と役割を明確にしたコンポーネント設計を行い、自律的なギミック動作を実現したいと思います。 (ManMachine の世界観にもマッチしていますね)
またユーザインターフェース同様、コンポーネント同士で奔放にメッセージングされると混迷を極めてしまいます。
ここでもやはりコミュニケーション動線は明確にしたいと思います。
ここでフォーカスしたい問題はおおまかに 2点です。

- ギミックの役務の細分化とパターン確立
- メッセージング方向の遵守

### ギミックの要素分解

どのような要素があればギミックは自律的に動作するか。
ギミック共通の登場人物は 3つと考えられたので、まずはこの 3つを分類したいと思います。

| 論理名 | 役割 |
| --- | --- |
| Gimmicks/Activator | ギミックを起動できるコンポーネント |
| Gimmicks/Worker | ギミックとして動くコンポーネント |
| Gimmicks/Reactor | ギミックが作用する対象のコンポーネント |

ドアの開閉の例であれば、 Activator はプレイヤー、 Worker は錠前アイコン、 Reactor はドアです。
この 3つにギミックの役務を分散することで、あらゆるタイプのギミックへの耐性が得られそうです。
ギミックは Unlock や Spawn など、 What を表現する名詞をベースに上記の 3つの形質に分かれます。

ただし、これらのコンポーネントだけでは機械的なギミック動作しか作れません。
ある程度、環境やゲーム仕様を理解して俯瞰的にギミックを制御できる層が必要です。
その層を便宜的に Roles とします。

| 論理名 | 役割 |
| --- | --- |
| Roles | ギミックの実行をコントロールする、ギミック以外のゲーム要素も扱う |

Roles はギミックのためだけのコンポーネントではなく、特定の役割が課せられた GameObject の特性を包括的に制御するコンポーネントとして扱います。
具体的にはプレイヤーやドアなど、 Who/Which が表現されるユーザにとって直感的な存在です。

---

あらかた登場人物を出しきりました、ここで一度、開閉するドアをベースに図を書いてみましょう。

![gimmicks overview](https://storage.googleapis.com/zenn-user-upload/anewbvuqnzdr4s8rm6bptt9ifqhg)

- Game Logic は個別のギミックではなくそれを制御する Roles に対してメッセージングする
- メッセージを受け取った Roles は Gimmicks/Activator を起動する
- Gimmicks/Activator はメッセージング可能な Gimmicks/Worker にメッセージを送る
- Gimmicks/Worker は Roles などの上層にコールバックを提供する
- Gimmicks/Worker はメッセージング可能な Gimmicks/Reactor にメッセージを送る
- Gimmicks/Reactor は Roles などの上層にコールバックを提供する

この図ではギミックのコンポーネントは 3つのオブジェクトに分かれていますが、実際は 2つや 3つ全てを単一の GameObject は有していても問題ありません。
ギミックの各コンポーネントでコールバックが提供されていますが、それらの役割は後処理だったり、ギミックの実行に必要な情報の要求だったりと、ギミックごとに任意の用途で定義します。

### 実装

コンポーネントのアウトラインが引けたので、ここからはちょっと実装を見ていきましょう、開閉するドアの実装を例に取ります。

先程、ギミックのコンポーネントを 3つに分けましたが、個別のコンポーネントには具体的な命名が必要です。
事前知識として命名規則を決めておきます。


| 論理名 | 命名規則 | ドアの開閉ギミックの命名
| --- | --- | --- |
| Gimmicks/Activator | *able | Unlockable |
| Gimmicks/Worker | *er | Unlocker |
| Gimmicks/Reactor | *ee | Unlockee |

---

開閉するドアは、ユーザ入力をトリガーにして動作します。
Game Logic の `PlayableScene` に、 `Player` にメッセージングしている箇所があるのでそこから見ていきましょう。

[PlayableScene](https://github.com/dolow/ManMachine/blob/5624f5f2a505743bdb9b04a01805881ed694dd8e/Assets/Scripts/Scene/PlayableScene.cs#L180-L183)

```cs:PlayableScene.cs
private void Action()
{
    this.player.TryAction();
}
```

これまで見てきたように、 Game Logic 層ではユーザ入力は隠蔽され「~したい」という Intent だけが渡されます。
ドアのギミック操作も Intent をトリガーにしたデリゲートメソッド経由で実行されますが、Intent は「ドアを開けたい」ではなく「アクションしたい」という抽象的な表現になっています。

```cs:PlayableScene.cs
mediator.RequestAction += this.Action;
```

これは、ユーザ入力からは「ドアの開閉」ほど粒度の細かいセマンティックは読み取れないためです。
仮にドアの開閉専用キーがあれば別ですが、ギミックごとにボタンがあるなんてユーザにとっては不親切でしょう。

さて、`Player` の `TryAction()` を見てみましょう。

[TryAction](https://github.com/dolow/ManMachine/blob/5624f5f2a505743bdb9b04a01805881ed694dd8e/Assets/Scripts/GameDomain/Roles/Player.cs#L79)

```cs:Player.cs
public void TryAction()
{
    if (this.actionables.ContainsKey(ActionableType.Unlock))
    {
        GameObject go = this.actionables[ActionableType.Unlock];
        Unlocker unlocker = go.GetComponent<Unlocker>();
        unlocker.Unlock(this.GetComponent<Unlockable>());
    }
    else if (this.actionables.ContainsKey(ActionableType.Rotate))
    {
        GameObject go = this.actionables[ActionableType.Rotate];
        Rotator rotator = go.GetComponent<Rotator>();
        TurnSwitch turnSwitch = rotator.GetComponent<TurnSwitch>();
        if (turnSwitch == null)
        {
            // unknown rotator
            return;
        }
        rotator.Rotate(this.GetComponent<Rotatable>());
    }
}
```

少し複雜ですね。
`Player` は複数のギミックで様々な役割を持っているため、「アクションしたい」場合にはどのアクションを実行するかの制御が必要です。
実際の `Player` のコンポーネントは下記のようになっており、ギミックとしては `Redirectable` と `Rotatable`, `Unlockable` を有しています。

![player inspector](https://storage.googleapis.com/zenn-user-upload/k1opd5g687764mli5a6gpdh0syzd =280x)

この中のどれをどの様に実行するかは全て、 `Player` コンポーネントに判断されます。

さて、 `Unlock` の条件となる `this.actionables` も見ておきましょう。
`this.actionables` に要素が追加されている箇所は、同じ `Player` クラスの `AddActionableIfEligible()` 内となります。

[AddActionableIfEligible()](https://github.com/dolow/ManMachine/blob/5624f5f2a505743bdb9b04a01805881ed694dd8e/Assets/Scripts/GameDomain/Roles/Player.cs#L113-L130)

```cs:Player.cs
private bool AddActionableIfEligible<Capability, Effectability>(GameObject worker, ActionableType type)
{
    Capability c = this.GetComponent<Capability>();
    if (c == null)
    {
        return false;
    }

    Effectability e = worker.GetComponent<Effectability>();
    if (e == null)
    {
        return false;
    }

    this.actionables.Add(type, worker);

    return true;
}
```


上記の `AddActionableIfEligible()` の呼び出し元はこちら。
コリジョンのトリガー判定を元に行っているようです。

[OnTriggerEnter()](https://github.com/dolow/ManMachine/blob/5624f5f2a505743bdb9b04a01805881ed694dd8e/Assets/Scripts/GameDomain/Roles/Player.cs#L101-L105)

```cs:Player.cs
private void OnTriggerEnter(Collider other)
{
    this.AddActionableIfEligible<Unlockable, Unlocker>(other.gameObject, ActionableType.Unlock);
    this.AddActionableIfEligible<Rotatable, Rotator>(other.gameObject, ActionableType.Rotate);
}
```

`Player` の `Unlockable` としての仕事はドアの開閉が実行可能かどうかを判断するのが主務であり、その判断はコリジョンのトリガーによるものでした。

ギミックの動作は、関連するコンポーネント間だけでなく環境からも影響を受けます。
最初に設計したときの図にはもう少し続きがあったようです、ゲーム AI 然としてきましたね。

![gimmick overview with environment](https://storage.googleapis.com/zenn-user-upload/ir2mom7smyk31uvdccdl622s2zkw)



次いで Worker である `Unlocker` の実装も見ていきましょう。

[Unlocker](https://github.com/dolow/ManMachine/blob/5624f5f2a505743bdb9b04a01805881ed694dd8e/Assets/Scripts/Components/Gimmicks/Unlocker.cs)


```cs:Unlocker.cs
public class Unlocker : MonoBehaviour
{
    public delegate void Unlocked(Unlockable unlockable);

    public Unlocked OnUnlocked = null;
    public Unlockee unlockee = null;

    public void Unlock(Unlockable unlockable)
    {
        this.unlockee.RequestUnlock(unlockable, this);
        this.OnUnlocked?.Invoke(unlockable);
    }
}
```

`Unlockee` に対して Unlock をリクエストしています。
`Unlocker` が Unlock リクエストを送るのになにか条件があれば、デリゲートメソッドなどを介して聞いてみるべきですが、現状は特別な事情はないようです。
`OnUnlocked` デリゲートは Roles である `Key` で実装されていますが、別のギミックの Activator である `Feedbackable` に処理を行わせているようでした。
`Feedbackable` のコード参照は割愛しますが、画面の明滅でユーザにフィードバックを与える処理を行っています。

```cs:Player.cs
private void OnUnlocked(Unlockable unlockable)
{
    UserFeedbackable feedbackable = this.GetComponent<UserFeedbackable>();
    if (feedbackable == null)
    {
        return;
    }

    feedbackable.Feedback(FeedbackType.Unlock);
}
```

Roles では、このようにギミックの実処理は排除して制御に徹している部分がほとんどです。
こうすることで、Roles はギミックの具体処理に関心を持たずに済んでいます。

最後に Reactor である `Unlockee` も見ておきましょう。
`Unlockee` は、その実処理を Roles である `Door` に委譲しています。
結局の所、何をすればよいか `Unlockee` 単体ではわからないためです。

[Unlockee](https://github.com/dolow/ManMachine/blob/edcf5bed588a9b8810c9802cdd7e39948465c6ef/Assets/Scripts/Components/Gimmicks/Unlockee.cs)

```cs:Unlockee.cs
public class Unlockee : MonoBehaviour
{
    public delegate void Exec(Unlockable unlockable, Unlocker unlocker);
    public Exec OnUnlock = null;

    public void RequestUnlock(Unlockable unlockable, Unlocker unlocker)
    {
        this.OnUnlock?.Invoke(unlockable, unlocker);
    }
}
```

`Unlockee` の `OnUnlock` デリゲートは `Door` の `Awake()` で設定されています。

[Door](https://github.com/dolow/ManMachine/blob/edcf5bed588a9b8810c9802cdd7e39948465c6ef/Assets/Scripts/GameDomain/Roles/Door.cs#L39-L40)

```cs:Door.cs
Unlockee unlockee = this.GetComponent<Unlockee>();
unlockee.OnUnlock += this.Toggle;
```

```cs:Door.cs
private void Toggle(Unlockable unlockable, Unlocker unlocker)
{
    LinearTransform linear = this.GetComponent<LinearTransform>();
    if (linear == null)
    {
        return;
    }

    if (linear.IsFinished())
    {
        linear.reveresed = !linear.reveresed;
    }

    // TODO: UserFeedbackable
    AudioCache.GetInstance().OneShot(audioCacheName);
}
```


実処理は Transform の遷移やサウンド再生などの `Door` 固有のものなので、いずれも Gimmicks で行うには無理のある処理ばかりでした。
ここまでを振り返ると、おおよそ当初の設計通りにコンポーネントが分離し、メッセージングがされていることが確認できました。

![gimmicks overview](https://storage.googleapis.com/zenn-user-upload/anewbvuqnzdr4s8rm6bptt9ifqhg)

ほとんどのギミックは MonoBehavior のライフサイクルを利用しません。
`OnTriggerEnter()` などのライフサイクルは実体を前提としますが、ギミックは GameObject の実体に関心を持っていません。
ギミックの役務は下記の 3つに集中すると行ってもいいでしょう。

- ギミック処理のデリゲートの提供
- ギミック前後のコールバックの提供
- メッセージング先の形質の担保

----

ここまでギミックの設計と実装を見てきました。
この構成は単一のギミックで考えると冗長に見えますが、複数のギミックを取り扱う場合には、その複雜性のほとんどを取り除く効果があります。
先程挙げた `Player` にアタッチされたコンポーネントの画像でも分かる通り、ManMachine では実際に `Player` に `Redirectable`, `Unlockable`, `Rotatable`, `Redirectee` という複雜な組み合わせでギミックが登録されています。
これらの Gimmicks の全ての処理を、ギミックコンポーネントに細分化せず `Player` という Role で担っていたとしたら、近い将来破綻するでしょう。
Gimmicks の役務を 3つに分け、それぞれがフォーカスすべきデリゲートメソッドを明確にすることで、その Gimmicks を持つ Role が実装すべき処理は何なのかということが、コード上でも気持ち的にも整理されると思います。
また、新しいギミックの追加が容易になったり、真新しい Role も、作用したいギミックのコンポーネントを有していればギミックのフローに組み込むことが容易にできます。

# 良い設計？悪い設計？

設計は、ある意味ではそのシステムに課したフレームワークとも言えます。
時にはそのフレームワークに従うように実装を再考しなければならないときもありますが、いい感じにハマるとイテレーション速度が飛躍的に上がります。
とはいえ、どこから始めたらよいかわからない、というジレンマは新しい作品に挑むたびに感じるでしょう。
仮組みした設計が良いか悪いかの判断もなかなか付きづらいものです。

筆者がよく意識するのは「◯◯ ツクール」足り得るかどうかです。
「◯◯ ツクール」とは、組み換えや組み合わせが容易なほどに依存関係が少なく、可能であればエンジニアでなくてもいじれるようなもの。
小さなシステムでもツクール化してしまえば、ゲームの面白みに対する試行錯誤の繰り返し速度も向上します。

## 今回、良かった点

- インタラクションの整理で、タップなどのしょーもないステート管理を端っこにまとめられた
- バグトラッキングが比較的容易になった気がする
- ギミックは拡張容易性があって、追加開発したいモチベーションになる
- 全体的に1ファイルのコード行数が少ない、行っても `InteractionMediator` の 200行ちょい

## 今回、悪かった点

- インスペクター上で静的に設定すべきものと動的に処理すべきものが未整理
- ギミックの中身が薄い割にファイルが多い
- こんな記事を書くくらいには、全体像把握に時間がかかりそう
- パフォーマンスそんなに考えてない

# まとめ

本稿では Unity における関心事を分離したコンポーネント設計事例を紹介しました。
ここで詳解した事例は一概に全てのゲームに適用できるものでもないと思いますが、関心事の分離や責務への専念などはどのようなゲームにも適用できうる考え方かと思います。
筆者はゲームの作り方について講釈垂れることができるほどのゲーム開発仙人ではないですが、世の中、相対的にゲーム開発に関する情報はまだまだ少ないと感じていますので、このような記事でも何かしらのインスピレーションにつながれば幸いです。
