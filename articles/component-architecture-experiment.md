---
title: "関心事を分離する / Unity におけるコンポーネント設計事例"
emoji: "🍣"
type: "tech"
topics: ["unity"]
published: false
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

##### HUD とカメラ切り替え

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
マウスでもキーボードでもプレイヤーの操作ができ、併用も可能にします。

- キーボード
- タッチスクリーン
- マウス

ユーザインターフェースは、例えば「キーボードの W を押したら前進する」というように、どんなインターフェースであれ入力に対して出力が伴います。
しかし、「W」のキー入力と「前進する」という出力とはそもそも全くの別物です。
「前進する」という振る舞いの実現自体は難しくはありませんが、キー入力と「前進する」ことについてのセマンティックを混同すると、そのメンテナンシビリティは低くなりがちです。
キー入力と「前進」という2つを翻訳する媒体が必要となります。

今回は以下のように役割と知識の領域を分けて、それぞれの関心事に専念し、他のオブジェクトへメッセージングするような構成を採ろうと思います。
以降、個別の領域について詳解していきます。

![interaction diagram](https://storage.googleapis.com/zenn-user-upload/8f4h6die12xa5oq8wcj4ob0cizn5)

### Primitive Layer

![primitive layer diagram](https://storage.googleapis.com/zenn-user-upload/q87uglv1ys6hr945o7g9pwlhagzc =320x)

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

#### Interaction Mediator and subsets

Interaction Mediator and subsets は、Primitive Layer からの入力の情報をゲームドメイン知識のセマンティックに変換し、伝搬する部分です。

![interaction diagram](https://storage.googleapis.com/zenn-user-upload/9xzhh7wsucoejakk2eo5cwmar044)

##### Game Interaction Interface ファミリー

Game Interaction Interface ファミリーは、それぞれが責務を持つ入力インターフェースからのメッセージをゲーム内のセマンティックに変換するオブジェクト群です。
たとえば、キー入力の W を「前進」というセマンティックに置き換えて、キー入力の知識を持たない(必要のない)レイヤーに伝える役務を指します。
前進、後退などのゲームドメイン知識であるセマンティックは Interaction Mediator から提供されます。
例として、以下にセマンティックの一部を挙げます。

- 前進
- 後退
- インタラクト
- カメラ切り替え

サポートする入力インターフェースの種類の分、このセマンティックに変換する実装を行います。
Game Interaction Interface ファミリーはセマンティックを知り得ている一方で、実際に Scene 上に配置されている具体的な GameObject などの知識は持ちません。

##### Interaction Mediator

Interaction Mediator は Touch や Mouse など、どのような入力インターフェースを取り扱うかを決定し、Game Interaction Interface ファミリーを使役するコンポーネントです。
Game Interaction Interface ファミリーから受け取った「前進」などのセマンティックを Game Logic に対して「前進したい」などの意思 (Intent) として送信します。

Interaction Mediator には、どのような入力インターフェースに対応するかの情報を静的に設定できるようにします。
設定された入力インターフェースの初期化と、 Game Interaction Interface ファミリーからの情報の集約と伝達が Interaction Mediator の責任範囲です。


![inspector interaction mediator](https://storage.googleapis.com/zenn-user-upload/ftwcg29akf8j49mj2zrv0v9k3rub =280x)

Interaction Mediator を使役する上位層から見た時、 具体的な入力インターフェースは意識されず、ゲーム内のセマンティックに基づいた挙動の意思 (Intent) のみが渡されるようにします。

#### Game Logic

![gamelogic diagram](https://storage.googleapis.com/zenn-user-upload/ofenbb28x8kvlqyuompujr2hkh7t =280x)

「前進したい」「後退したい」などの意思 (Intent) を受け取り、 Presentation 層に反映させる層です。
Presentation 層に作用するため、 Scene 内の GameObject などについての知識を持つことができます。
受け取った Intent がユーザ入力に由来するかどうかの関心はなく、その知識も持ちません。

##### GameController

Interaction Mediator から受け取った Intent を処理するコンポーネントです。
Intent に伴う制御や、Intent を自身が知りえる Scene 内の GameObject にメッセージングすることを責務とします。
どんなアニメーションを再生するか、などのような個別の GameObject の具体的な振る舞いは知り得ません。

#### Presentation

![presentation diagram](https://storage.googleapis.com/zenn-user-upload/jitlvltq5nwev2xjtinuypz65jnm =280x)

実際にアニメーションを行ったりする層です。
各 GameObject は自身のアニメーションなどの状態にのみ責務を持ち、自身の個別のコンポーネントで取り扱わない限りは他の GameObject や上位層の挙動などに関心を持ちません。

----

さて、ユーザ入力どころかプレゼンテーション層まで切り分けてしまいました。
ユーザ入力の知識や責務でシステムを分断しようとした時、そのアウトプットはユーザへのフィードバックにまで及びます。
そのため、プレゼンテーション層への切込みは避けることが出来ません。

結果として、明確にユーザ入力の知識を持たない(必要としない)領域がわかりました。
ここからはちょっとだけ実装を見ていきたいと思います。

### ユーザインターフェースの実装

先程までとは逆の順番で、関心事がちゃんと分離されているか確かめるために Presentation 層から追っていきましょう。
最もイメージが付きやすいのはプレイヤーですね、Game Contoller 相当のクラスである `PlayableScene` からプレイヤーを動かしている部分を見てみましょう。

#### Game Controller

[プレイヤーを動かすメソッド](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/Scene/PlayableScene.cs#L181-L192)

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

デリゲートの設定は、同じく `PlayableScene` 自身の `Awake` で設定しています。
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

[TutorialScene.unity](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scenes/TutorialScene.unity#L2196)

```yml:TutorialScene.unity
m_Script: {fileID: 11500000, guid: b3fa791327bd14316aa202c0e6b10166, type: 3}
```

[InteractionMediator.cs.meta](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/GameDomain/InteractionMediator.cs.meta#L2)
```yml:InteractionMediator.cs.meta
guid: b3fa791327bd14316aa202c0e6b10166
```

つまり `PlayableScene` では、タップやキー入力などの具体的なユーザ入力の情報がなくてもプレイヤーを動かせていることがわかります。
逆に言うと、知る必要はないのです。
これにより、ゲームロジック~プレゼンテーション層ではユーザ入力インターフェースの増減や、キー配置及び操作方法の変更などの影響は一切受けずに開発できるようになります。
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
`HasAnyIntention()` は、現在、引数に指定した意思(Intent)を持っているかを確認するメソッドで、中身は下記のようなものです。

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

`AInteractionMediatorInterface` はキーボードやマウス入力などの個別入力インターフェースの実装の基底となる抽象クラスです。
`AInteractionMediatorInterface` のそれぞれの実装は `Awake()` で初期化されていることが確認できます。

[interactionInterfaces フィールド](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/GameDomain/InteractionMediator.cs#L51)


`Awake()` では自身のフィールドの真偽値に応じて、使用する入力インターフェースを初期化しています。
先程の画像で見たチェックボックスですね。
(UI については後ほど個別に触れます)

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

ManMachine では、この真偽値を全て静的に真にしています。

[inspector 上の値](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scenes/TutorialScene.unity#L2187-L2203)

```yml:TutorialScene.unity
--- !u!114 &1009830954
MonoBehaviour:
  m_ObjectHideFlags: 0
  m_CorrespondingSourceObject: {fileID: 0}
  m_PrefabInstance: {fileID: 0}
  m_PrefabAsset: {fileID: 0}
  m_GameObject: {fileID: 1009830947}
  m_Enabled: 1
  m_EditorHideFlags: 0
  m_Script: {fileID: 11500000, guid: b3fa791327bd14316aa202c0e6b10166, type: 3}
  m_Name:
  m_EditorClassIdentifier:
  keyboard: 1
  touchScreen: 1
  mouse: 1
  ui: 1
  uiInteractionRegistry: {fileID: 1009830951}
```

このように　`InteractionMediator` では、利用する入力インターフェースの初期化と、下位の `AInteractionMediatorInterface` 実装クラスからのメッセージの伝播のみを責務としていることがわかります。
また、「前進したい」などの Intent を誰に伝えるかは知識としても責務としても持っていません。

この時点ではまだ具体的なタップなどの参照や加工などは一切行っていませんが、徐々に入力インターフェースに近づいてきました。

#### AInteractionMediatorInterface 継承クラス

`InteractionMediator` のフィールドの `interactionInterfaces` は、 `AInteractionMediatorInterface` を要素に持つ `List` でした。
`AInteractionMediatorInterface` は抽象クラスであるため、ここではマウス入力の実装として `AInteractionMediatorInterface` を継承した `InteractionMediatorMouse` を見ていきましょう。

`InteractionMediator` では各入力インターフェースクラスの `Intention` と呼ばれるものを見ていました。
Intention とは、和訳すると「意思」や「意図」と呼ばれるものです。
命名として「クリックされた」などのような入力値ではなく「何がしたいのか」を表現しています。
Intention が発生する場所を見てみると、ここでようやく `MouseMove` というメソッド名の「マウスが動いた」という入力から、 `InteractionSemantic.MoveAny` のような「動きたい」「回転したい」といった文脈が確認できます。

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


`MouseMove` は `MouseInteraction` のデリゲートメソッドとして登録されており、 `MouseInteraction` こそ、マウス入力を直接受け取るクラスそのものです。
いずれも `InteractionMediatorMouse` の `Awake()` で登録されています。


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
ユーザ入力から `UIInteraction` を仲立ちする要素として `UIInteractionRegisterer` と `UIInteractionRegistry` というコンポーネントを定義します。

![ui diagram](https://storage.googleapis.com/zenn-user-upload/m5kh4rl7r3utrvp02bkhdhst7h0q)

これによって、画面上に操作ボタンを出すタイプのゲームの場合はキーコンフィグなどが容易にできるようになります。

#### UIInteractionRegisterer

このコンポーネントがアタッチされた GameObject の `Button` コンポーネントの `onClick` イベントは、そのまま `UIInteractionRegistry` に流すようにします。
上位層がユーザ入力を識別できるように、UI に対する ID を設定できるようにします。

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

#### UIInteractionRegistery

`Input` の様に押されているボタンを取得できるようにします。
イベント発火などの push 形式ではなく、あくまでも pull する情報として扱います。
ボタンの押下状況のクリアのタイミングは自身だけでは制御できないため、 `ClearPressedButtons()` メソッドを露出させて外部に委譲します。

[UIInteractionRegistry.cs](https://github.com/dolow/ManMachine/blob/9ab5b2223efab9c8304511c8c11ebdac46c0f5b3/Assets/Scripts/Primitive/Interaction/UIInteractionRegistry/UIInteractionRegistry.cs)

```cs:UIInteractionRegistery.cs
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


`UIInteractionRegistry` からユーザ入力を取得する `UIInteraction` 層以降は、マウスのときと同じ要領なので割愛します。
`InteractionMediator` の `InteractionMediatorUI` の初期化時のみ、利用する `UIInteractionRegistry` を知らせる追加処理が必要です。

```cs:InteractionMediator.cs
interactionMediatorUi.SetUIInteractionRegistry(this.uiInteractionRegistry);
```

`Input` ほどの懐の深さはありませんが、 `UIInteractionRegistry` にさえ登録すれば押下されたボタンを透過的に取得できるようになりました。
正直、ManMachine ではここまでする意義はあまりありません、最初に申し上げたとおり完全に私の趣味嗜好です。

---

ユーザインターフェース編はいかがでしたでしょうか。
構成としては冗長かもしれませんが、関心事や責務を明確にして分離にすることは、チーム開発だったり長く運用するプロダクトで真価を発揮すると思います。

## ゲーム仕様編

ここからはゲーム仕様編で、よりゲームの内容に沿ったものとなるので、事前にゲームを触って頂いておいたほうがイメージしやすくなるかと思います。
(ゲームの面白さはちょっと置いておいてください)
ここでのトピックはギミックにフォーカスしたいと思います。

### コンポーネントを分類する

ドアを開閉するギミックを作るとしましょう。
それだけ作るのであれば苦労はありませんが、同じような粒度のギミックを複数作る場合にはある程度の秩序が必要です。
秩序が保たれるように一つひとつのギミックを中央集約的に管理、統制しても良いのですが、 ManMachine の場合はその世界観の通り、各ギミックができるだけ自律的に動作できるようにしたいと思います。

自律的にギミックを動作させるには何が必要か。
ManMachine ではコンポーネントに下記のような分類をしました。

- Roles
- Gimmicks
  - Activator
  - Worker
  - Reactor

#### Roles

Roles は論理名であり、 ManMachine における具体的なクラス名などではありません。
本稿では GameObject の形質を表すものの総称として便宜的に用います。
Roles は `Player` や `Door` などの Who/Which を表現する、ユーザにとって直感的な存在です。
自身の GameObject が有する Gimmicks から処理のリクエストを受信し、役割に沿った処理を実行します。
本稿では触れませんが、 Roles は Gimmicks 以外のコンポーネントも処理します。

[ソースで言うとこの辺り](https://github.com/dolow/ManMachine/tree/5624f5f2a505743bdb9b04a01805881ed694dd8e/Assets/Scripts/GameDomain/Roles)


#### Gimmicks

Gimmicks も論理名であり、本稿での便宜的な呼称です。
Gimmicks は Unlock や Spawn など、 What を表現する名詞をベースに 3つの形質に分かれます。
ギミックの実行に必要な情報と実行リクエストを送信し、処理の実体は有しません。
また、実行リクエストの送信先が何者かも意図しません。

| 論理名 | 説明 |
| --- | --- |
| Activator | Worker を作動させられるコンポーネント、クラスが *able と命名されているもの |
| Worker | Activator より作動され Reactor に作用するコンポーネント、クラスが *er と命名されているもの |
| Reactor | Worker に作用されるコンポーネント、クラスが *ee と命名されているもの |

[ソースで言うとこの辺り](https://github.com/dolow/ManMachine/tree/5624f5f2a505743bdb9b04a01805881ed694dd8e/Assets/Scripts/Components/Gimmicks)


### 開閉するドアの例

開閉するドアの例で言えば、 Activator は `Unlockable`、 Worker は `Unlocker`、 Reactor は `Unlockee` です。
開閉するドアの　Role は `Door` ですが、そのメカニズムは、 Roles と Gimmicks の組み合わせで説明可能です。

- `Player` は `Unlockable` (Activator) の形質を持ち、 `Unlocker` (Worker) を作動できる
- `Key` は `Unlocker` (Worker) の形質を持ち、 `Unlockable` (Activator) から作動され、 `Unlockee` (Reactor) を対象に開閉する
- `Door` は `Unlockee` (Reactor) の形質を持ち、 `Unlocker` (Worker) に解錠される

![unlock diagram](https://storage.googleapis.com/zenn-user-upload/sc9jdncjbvpdaq2qhbu4qhsvpk4y)

文章だけだとなんのこっちゃって感じですね、具体例と実装を元に見ていきましょう。

開閉するドアは、ユーザ入力をきっかけに動作します。
これまで見てきたように、 Game Logic 層だとユーザ入力は隠蔽され「~したい」という Intent だけが渡されます。
開閉するドアも他聞に漏れず Intent を実行するのみですが、「ドアを開けたい」ではなく「アクションしたい」という抽象的な表現になっています。
これは、ユーザ入力からは「ドアの開閉」ほど粒度の細かいセマンティックは読み取れないためです。
仮にドアの開閉専用キーがあれば別ですが、ギミックごとにボタンがあるなんてユーザにとっては不親切でしょう。

[アクションしたい](https://github.com/dolow/ManMachine/blob/5624f5f2a505743bdb9b04a01805881ed694dd8e/Assets/Scripts/Scene/PlayableScene.cs#L180-L183)

```cs:PlayableScene.cs
private void Action()
{
    this.player.TryAction();
}
```

`TryAction()` を受け取った `Player` の実装を見る前に、Player のコンポーネントを確認しておきましょう。

![player inspector](https://storage.googleapis.com/zenn-user-upload/k1opd5g687764mli5a6gpdh0syzd =280x)

ギミックの作動は Activator (*able) から行われますが、Player は `Redirectable` と `Rotatable`, `Unlockable` を有しているようです。
この中のどれをどの様に実行するか `PlayableScene` では管理せず、全て `Player` に委譲します。
`Player` の `TryAction()` を見てみましょう。

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

`this.actionables` の確認をした後、 Worker である `Unlocker` の処理を委譲しています。
突然出てきた `this.actionables` も見ておきましょう、要素が追加されている箇所は、同じ `Player` クラスの `AddActionableIfEligible()` 内となります。

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

`AddActionableIfEligible()` の呼び出し元はコチラ、トリガー判定を元に行っているようです。
ゲーム上でも、錠前のアイコンに近づいたらドアの開閉が可能になるため、このへんは割と素直に実装されています。

[OnTriggerEnter()](https://github.com/dolow/ManMachine/blob/5624f5f2a505743bdb9b04a01805881ed694dd8e/Assets/Scripts/GameDomain/Roles/Player.cs#L101-L105)

```cs:Player.cs
private void OnTriggerEnter(Collider other)
{
    this.AddActionableIfEligible<Unlockable, Unlocker>(other.gameObject, ActionableType.Unlock);
    this.AddActionableIfEligible<Rotatable, Rotator>(other.gameObject, ActionableType.Rotate);
}
```

`Player` の `Unlockable` としての仕事はドアの開閉が実行可能かどうかを判断するのが主務であり、その判断はコリジョンのトリガーによるものでした。
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

とくに障壁なく `Unlockee` に対して Unlock をリクエストしています。
Gimmick 固有事情があればここで条件分けなどすべきですが、現状は特別な事情はないようです。
`OnUnlocked` デリゲートは `Key` で実装されていますが、別のギミックの Activator である `Feedbackable` に処理を行わせているようでした。
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

Roles では、このように一定の役割を抽象化して個別のコンポーネントに実処理を委譲している箇所がほとんどです。
こうすることで、Roles はギミックの具体的な処理に関与せず、俯瞰的なギミック制御に徹することが出来ます。

最後に Reactor である `Unlockee` も見ておきましょう。
`Unlockee` は、その実処理を委譲しています。
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
ここまでの流れをおさらいすると下記のようになります。

![door diagram](https://storage.googleapis.com/zenn-user-upload/97iqox7w0csh9ivv9re0whyfy4z5)

作用したいギミックのコンポーネントを中心にフローが流れているのがわかると思います。
また、ほとんどのギミックは MonoBehavior のライフサイクルを利用しません。
`OnTriggerEnter()` などのライフサイクルは実体を前提としますが、ギミックは実体の知識を知り得ないのです。
そのため、ギミックはほとんどデリゲートを提供するだけにとどまるのですが、責務の範囲がデリゲートメソッドという単位に切り出されているため、デリゲートメソッドの実装側で集中すべき関心事が明確になります。
また `Key` のデリゲートで見たように、本来 `Key` 自身が関心を持っていないユーザーフィードバックという仕事は、それ専用の別のコンポーネントに処理を放り投げています。

----

この構成は、`Door` 単体で見ると冗長に見えますが、複数のギミックを取り扱う場合はその複雜性のほとんどが取り除かれます。
少し前の inspector の画像でも分かる通り、ManMachine では実際に `Player` に `Redirectable`, `Unlockable`, `Rotatable`, `Redirectee` という複雜な組み合わせでギミックが登録されています。
これらの Gimmicks の全ての処理をコンポーネント化せず `Player` という Role で担っていたとしたら、近い将来破綻するでしょう。
Gimmicks の役務を 3つに分け、それぞれがフォーカスすべきデリゲートメソッドを明確にすることで、その Gimmicks を持つ Role が実装すべき処理は何なのかということが、コード上でも気持ち的にも整理されると思います。
全く新しいギミックの追加も容易になり、詳細不明の GameObject も、作用したいギミックのコンポーネントを有していればギミックのフローに組み込めることが判断できます。

# 良い設計？悪い設計？

設計は、ある意味ではそのシステムに課したフレームワークとも言えます。
時にはそのフレームワークに従うように実装を再考しなければならないときもありますが、いい感じにハマるとイテレーション速度が飛躍的に上がります。
とはいえ、どこから始めたらよいかわからない、というジレンマは新しい作品に挑むたびに感じるでしょう。
仮組みした設計が良いか悪いかの判断もなかなか付きづらいものです。

筆者がよく意識するのは「◯◯ ツクール」足り得るかどうかです。
「◯◯ ツクール」とは、組み換えや組み合わせが容易で、可能であればエンジニアでなくてもいじれる、といったものです。
小さなシステムでもツクール化してしまえば、ゲームの面白みに対する試行錯誤の繰り返し速度も向上します。

## 今回、良かった点

- インタラクションの整理で、タップなどのしょーもないステート管理を端っこにまとめられた
- Game Logic からすると何がしたいか明確なので、バグがあっても原因がわかりやすかった
- ギミックは拡張容易性があって、追加開発したいモチベーションになる
- 全体的に行数が少ない、行っても `InteractionMediator` の 200行ちょい

## 今回、悪かった点

- インスペクター上で静的に設定すべきものと動的に処理すべきものが未整理
- ギミックの中身が薄い割にファイルが多い
- こんな記事を書くくらいには、全体像把握に時間がかかりそう
- パフォーマンスそんなに考えてない

# まとめ

本稿では Unity における関心事を分離したコンポーネント設計例を紹介しました。
ここで詳解した事例は一概に全てのゲームに適用できるものでもないと思いますが、関心事の分離や責務への専念などはどのようなゲームにも適用できうる考え方かと思います。
筆者はゲームの作り方について講釈垂れることができるほどのゲーム開発仙人ではないですが、世の中、相対的にゲーム開発に関する情報はまだまだ少ないと感じていますので、このような記事でも何かしらのインスピレーションにつながれば幸いです。
