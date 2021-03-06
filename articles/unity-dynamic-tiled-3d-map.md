---
title: "タイル形式のマップの移動表現 / Unity における A* 経路検索と 3D タイルマップ動的生成"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unity","pathfinding", "tilemap"]
published: true
---

こんにちは、Smith ([@do_low](https://twitter.com/do_low)) です。

本稿では、先日開催された Unity 1 week game jam に公開したゲームに実験的に実装した 3D 空間へのタイルマップ生成と、ついでに実装したタイル上の A* 経路検索について詳解します。

Unity room に公開したゲーム
[https://unityroom.com/games/taxi-manhattan](https://unityroom.com/games/taxi-manhattan)

github での Unity プロジェクトの公開も行っています。
[https://github.com/dolow/taxi-manhattan](https://github.com/dolow/taxi-manhattan)

ゲームの面白みは・・・ちょっと置いておいてください。

---

# 本稿でつくるもの

昔のドラゴンクエストの町中のようなマップを想像してください。
そう、1マスずつ東西南北の 4方向に移動するようなマップです、本稿ではそのマップを 3D 空間上に作ることが目的です。
草原や道などは 1マスずつタイルを配置して表現されます。
おっと、最近だとアップデートで追加のマップが配信されることがありますね。
毎回、職人が作り込んだマップを配信するのも魅力的ですが、今回はポータブルなフォーマットで表現されたデータからマップを生成したいと思います。
また、昔のゲームであればコントローラー入力で移動を行いますが、近代のゲームの入力インターフェースにはタッチスクリーンやマウス入力なども含まれます。
タッチ入力でどのように移動するか？画面上に表示された HUD を操作する方式でも良いですが、ちょっと垢抜けません。
画面上の任意の地点を押下した際、現在地からその地点までの経路検索を行い、キャラクターをそこに移動させるようにしましょう。

さて、色々と要約すると本稿で実現する要件は下記となります。

- 実行時にデータからマップを生成する
- マップはタイル単位のパーツで形成される
- ユーザ入力はマウス及びタッチを想定
- タッチしたタイルまでの経路検索を行いキャラクターを動かす

# 前提環境

- macOS 10.15.7
- Unity 2019.4.17f1
- Chrome 88

---

# 事前準備

## プロジェクト設定

今回、タイルを組み合わせて地形を表現するのでアンチエイリアスを切っておきます。
設定は以下の場所にあります。

File > Build Settings... > Player Settings... > Quality > Anti Alias > Disabled

![anti alias 設定](https://storage.googleapis.com/zenn-user-upload/dm108kkyq1zszjtmc1w3pdvsin14)

# 素材

下記のような個別のタイルを敷き詰めて平面のマップを表現します。

![個別タイル画像](https://storage.googleapis.com/zenn-user-upload/086hxvynhan6dz6q52hsk4wdm9y2)

# データ設計

## マップデータ

タイル状に敷き詰められたマップを表現する場合、どのテクスチャのタイルをどの位置に配置するかの情報が必要です。
タイルのテクスチャ毎に ID を割り振り、その ID を並べてマップデータを表現します。
今回は一次元配列で表現します。

```
fields = [ 2, 7, 7, 9, 7, 3, 0,14, 7, 7, 7, 9, 7, 7, 3, 8, 0, 0, 8, 0, 8, 0, 0, 0, 0, 0, 8, 0, 0, 8, 8, 0, 0, 8, 0, 8, 0, 0, 0, 0, 0, 8, 0, 0, 8,12, 7, 7, 4, 0,12, 7, 7, 3, 0, 0, 8, 0, 0, 8, 8, 0, 0, 0, 0, 8, 0, 0, 8, 0, 0,12, 7, 7,10, 8, 0, 0, 0, 0, 8, 0, 0, 8, 0, 0, 8, 0, 0, 8, 5, 7, 7, 7, 7, 4, 0, 0,12, 7, 7,11, 7, 9, 4, 0, 0, 0, 0, 0, 0, 0, 0, 8, 0, 0, 0, 0, 8, 0, 2, 7, 7, 7, 7, 3, 0, 0, 8, 0, 1, 0, 2,11, 3, 8, 0, 0, 0, 0, 8, 0, 0, 8, 0, 0, 0, 8, 1, 8, 5, 7, 7, 7, 7,11, 7, 7,11,16, 0,14,11, 7, 4]
```

これに対して、一つあたりの列にいくつのタイルが並べられるかの情報が与えられれば、二次元配列な位相が表現できます。

```
width = 15
fields = [
   2, 7, 7, 9, 7, 3, 0,14, 7, 7, 7, 9, 7, 7, 3,
   8, 0, 0, 8, 0, 8, 0, 0, 0, 0, 0, 8, 0, 0, 8,
   8, 0, 0, 8, 0, 8, 0, 0, 0, 0, 0, 8, 0, 0, 8,
  12, 7, 7, 4, 0,12, 7, 7, 3, 0, 0, 8, 0, 0, 8,
   8, 0, 0, 0, 0, 8, 0, 0, 8, 0, 0,12, 7, 7,10,
   8, 0, 0, 0, 0, 8, 0, 0, 8, 0, 0, 8, 0, 0, 8,
   5, 7, 7, 7, 7, 4, 0, 0,12, 7, 7,11, 7, 9, 4,
   0, 0, 0, 0, 0, 0, 0, 0, 8, 0, 0, 0, 0, 8, 0,
   2, 7, 7, 7, 7, 3, 0, 0, 8, 0, 1, 0, 2,11, 3,
   8, 0, 0, 0, 0, 8, 0, 0, 8, 0, 0, 0, 8, 1, 8,
   5, 7, 7, 7, 7,11, 7, 7,11,16, 0,14,11, 7, 4
]
```

## テクスチャ

タイルの種類毎に一枚ずつテクスチャを用意してもよいのですが、それだと下記のようなパフォーマンス的な問題を生むことがあります。

- サイズがべき乗でないタイルの場合、メモリ上に無駄が生まれる
- マテリアルのテクスチャ、あるいはマテリアルそのものを付け替える処理のオーバーヘッド

そこでいわゆるスプライトシートのように、一枚のテクスチャに複数のタイル画像を並べるようにします。
そのためには下記の決め事が必要です。

- タイル一枚のサイズ
- スプライトシートのサイズ (=一列に何枚タイル画像を敷き詰めるか)

また、タイルの継ぎ目が滲んで見えてしまうのを回避するため、テクスチャの Filter Mode を `Point (nofilter)` に設定するようにします。

![filter mode](https://storage.googleapis.com/zenn-user-upload/8xurp8s6025hlg50u8bhv4g0gtvm)

今回はタイル描画の色の境界線を滑らかにする必要はありませんが、ゲームの世界観やルックのコンセプトによっては `Point (nofilter)` が好ましくない場合もあると思います。
その場合、ゲーム上で描画する以上のタイル画像を作成し、タイル間にスペースを空けることで多少は改善されます。

----

さて、データと素材の設計を行ったので、それらをランタイムで取り扱うための設計を行います。


# システム設計と実装

データ読み込みから表示までの流れを確認しましょう。

* 外部マップデータ読み込み
* ランタイムでのデータ表現に変換
* スプライトシートからテクスチャID に応じた UV を決定
* 表示

ここからはこの流れの部品毎に詳解します。

## 外部マップデータ読み込み

外部マップデータはどのようなシリアライザで定義しても良いですが、本稿では JSON として取り扱います。

```json
{
  "width": 15,
  "fields": [ 2, 7, 7, 9, 7, 3, 0,14, 7, 7, 7, 9, 7, 7, 3, 8, 0, 0, 8, 0, 8, 0, 0, 0, 0, 0, 8, 0, 0, 8, 8, 0, 0, 8, 0, 8, 0, 0, 0, 0, 0, 8, 0, 0, 8,12, 7, 7, 4, 0,12, 7, 7, 3, 0, 0, 8, 0, 0, 8, 8, 0, 0, 0, 0, 8, 0, 0, 8, 0, 0,12, 7, 7,10, 8, 0, 0, 0, 0, 8, 0, 0, 8, 0, 0, 8, 0, 0, 8, 5, 7, 7, 7, 7, 4, 0, 0,12, 7, 7,11, 7, 9, 4, 0, 0, 0, 0, 0, 0, 0, 0, 8, 0, 0, 0, 0, 8, 0, 2, 7, 7, 7, 7, 3, 0, 0, 8, 0, 1, 0, 2,11, 3, 8, 0, 0, 0, 0, 8, 0, 0, 8, 0, 0, 0, 8, 1, 8, 5, 7, 7, 7, 7,11, 7, 7,11,16, 0,14,11, 7, 4]
}
```

外部マップデータの読み込み自体は、 Resources 配下であれば `Resource.Load<T>(string)` で簡便に行えるため割愛します。

## ランタイムでのデータ表現に変換

今回はタイルの一つひとつは多くの情報を有さないため、マップ全体を表す構造体を用意します。

```cs
[Serializable]
public struct Map
{
    public int width;
    public int height;
    public int[] fields;
}
```

Unity は `JsonUtility` という JSON 文字列と Serializable な class や struct インスタンスを相互に変換するモジュールを提供しています。
あいにく、ネストされた配列の取り扱いはできないため注意してください。
代替案として .NET の `DataContractJsonSerializer` の利用も有効ですが、本稿の前提環境においては Unity の WebGL ランタイムにおいて、 JSON マーシャル時にエラーが発生することを確認しています。

`JsonUtility` を用いる場合は、下記のようにして JSON 文字列を任意の構造体にマーシャルできます。

```cs
TextAsset asset = Resources.Load<TextAsset>(path);
Map map = JsonUtility.FromJson<RawMap>(asset.text);
```

上記に示した構造体の `Map` のうち `height` は補足情報ですが、 `width` と `fields` の配列長を元にシンプルな演算で後から導出可能です。

```cs
map.height = (int)Mathf.Floor(map.fields.Length / map.width) + 1;
```

## スプライトシートからテクスチャID に応じた UV を決定

さて、ここで唐突に UV が出てきましたが、特にシェーダーを書く必要はありません。
`Material` の `SetTextureOffset` や `SetTextureScale` で十分に対応可能で、データ設計の節で定めたタイルのサイズや、テクスチャ内のサイズの数を利用します。
スプライトシート内のXY軸におけるタイルの配置数を `Vector2 tileLength` としたとき、 `SetTextureScale` の指定は下記のようになります。

```cs
Renderer renderer = this.GetComponent<Renderer>();
Vector2 tileScale = new Vector2(1.0f / tileLength.x, 1.0f / tileLength.y);
renderer.material.SetTextureScale("_MainTex", tileScale);
```

スプライトシート上のタイル画像の位相や一枚のサイズがわかるようになると、描画すべき UV のオフセットが決定できるようになります。
入力値として、左から x 番目、上から y 番目のタイルを表示するという意味の `Vector2 tilePos` を取る場合は下記のようになります。

```cs
Renderer renderer = this.GetComponent<Renderer>();
Vector2 offset = new Vector2(tilePos.x * tileScale.x, 1.0f - tileScale.y - tilePos.y * tileScale.y);
renderer.material.SetTextureOffset("_MainTex", offset);
```

ひとまとめにすると下記のようになるでしょう。

```cs
public void SetUV(Vector2 tileLength, Vector2 tilePos)
{
    Renderer renderer = this.GetComponent<Renderer>();

    Vector2 tileScale = new Vector2(1.0f / tileLength.x, 1.0f / tileLength.y);
    Vector2 offset = new Vector2(tilePos.x * tileScale.x, 1.0f - tileScale.y - tilePos.y * tileScale.y);

    renderer.material.SetTextureScale("_MainTex", tileScale);
    renderer.material.SetTextureOffset("_MainTex", offset);
}
```

---

JSON からの入力値に含まれる `fields` プロパティには表示すべきタイルの ID が羅列されているので、これをイテレーションするだけで全てのタイルの表示すべき UV オフセットが決定できます。
イテレーションついでに `transform.position` も決定してしまっても良いでしょう。
下記の `Tile` は、 `SetUV` が実装されている便宜的なコンポーネントとしています。

```cs
int tilesInRow = 16;
Vector2 tileLength = new Vector2(16.0f, 16.0f);
for (int i = 0; i < map.fields.Length; i++)
{
    int tileId = map.fields[i];
    Vector2 tileUv = new Vector2(tileId % tilesInRow, Mathf.Floor(tileId / tilesInRow));
    Vector3 tilePos = new Vector3(i % map.width, 0.0f, -Mathf.Floor(i / map.width));

    Tile tile = GameObject.Instantiate<Tile>(this.tilePrefab);
    tile.SetUV(tileLength, tileUv);
    tile.transform.position = tilePos;
}

```

---

これで、外部データに定義されたタイル配列から、レンダラーにタイル描画領域を反映するまでの一連の流れができました。
この時点での制作物を、記事用に `ArticleScene` というシーンで確認できるようにしたブランチを作成しましたのでよろしければご参照ください。
これまで記載したコードは説明用であるため、ブランチ上にあるソースコードとは若干異なります。

[feature/article_step1](https://github.com/dolow/taxi-manhattan/tree/feature/article_step1)

# 補足

## Material 設定

Material の設定は Standard Shader で十分事足りますので深く言及はしません。
本稿では下記のみデフォルト設定から変更しています。

- Rendering Mode を Cut Off に変更
- Albedo にスプライトシートのテクスチャを設定
- Metallic -> Smoothness はお好みで


## スプライトシートの作成

Unity においては色々と手法があるようですが、いまいち枯れている手法がない印象です。
本稿ではどのバージョンの Unity でもいいように(Unity に依存せず)、 [imagemagick](https://imagemagick.org/index.php) を用いて作成しています。

```sh
% cd ./TileTextures
% magick montage tile_[1-9].png tile_[1-9][0-9].png -tile 16x16  -geometry 64x64 -background transparent tiles.png
```

メモリ効率の点では、タイル内の余白を切り詰めたり回転させたりなどの余地がまだありますが、本稿では特に行いません。

## アニメーション

スプライトシートのフレームを連続で切り替えることによってパラパラ漫画のようなアニメーションを実現できます。
アニメーション表現となるフレーム範囲またはフレームの順序がわかっていれば、`Material#SetTextureOffset()` のオフセットを周期的に更新することでアニメーーションが実現できます。
アニメーションの速度は経過時間や経過フレーム数で指定すると良いでしょう。

---

# 最適化

さて、タイルを並べたは良いのですが、このままだとタイル 1枚1枚にドローコールがかかってしまいます。

![チューニング前](https://storage.googleapis.com/zenn-user-upload/klcmcl8regt3p3g1ameqxdrxzi92)

同じようなタイルを表示している割には非常に非効率です、ここでメッシュを結合してパフォーマンス改善を試みます。
まずは同じ UV を表示しているタイルの集合を作りましょう、個別のタイルインスタンス生成直後で良いと思います。

```cs
Dictionary<int, List<MeshFilter>> filters = new Dictionary<int, List<MeshFilter>>();
for (int i = 0; i < this.map.fields.Length; i++)
{
    int tileId = map.fields[i];
    Vector3 tilePos = new Vector3(i % map.width, 0.0f, -Mathf.Floor(i / map.width));
    // prefab の active はデフォルトで false
    Tile tile = GameObject.Instantiate<Tile>(this.tilePool);

    if (!filters.ContainsKey(tileId))
    {
        filters.Add(tileId, new List<MeshFilter>());
    }

    MeshFilter meshFilter = tile.GetComponent<MeshFilter>();
    meshFilter.transform.rotation = Quaternion.Euler(90.0f, 0.0f, 0.0f);
    meshFilter.transform.position = tilePos;
    filters[tileId].Add(tile.GetComponent<MeshFilter>());
}
```

この時点で、3D 空間上の座標も設定してしまいます。

次にタイルの集合を結合します。
結合用のタイルインスタンスを作成し、結合元のインスタンスは削除します。
結合用のタイルインスタンスのメッシュに、結合したメッシュをセットします。

```cs
foreach (KeyValuePair<int, List<MeshFilter>> kv in filters)
{
    List<MeshFilter> filterList = kv.Value;
    CombineInstance[] combine = new CombineInstance[filterList.Count];
    for (int i = 0; i < filterList.Count; i++)
    {
        combine[i].mesh = filterList[i].sharedMesh;
        combine[i].transform = filterList[i].transform.localToWorldMatrix;
        combine[i].subMeshIndex = 0;
        GameObject.Destroy(filterList[i].gameObject);
    }

    Article.Tile combinedTile = GameObject.Instantiate<Article.Tile>(this.tilePool);
    MeshFilter filter = combinedTile.GetComponent<MeshFilter>();
    filter.mesh = new Mesh();
    filter.mesh.CombineMeshes(combine);
    // 表示タイルの設定をタイル ID から行えるようにしたメソッド
    combinedTile.SetTilePos(kv.Key);
    combinedTile.gameObject.SetActive(true);
}
```

タイル初期化時に一手間加えるだけでドローコールが大幅に削減できました。

![チューニング後](https://storage.googleapis.com/zenn-user-upload/7boqtm0ci5qzufinvajf08fp226k)

# 経路検索

生成したマップは歩くためのものです。
ここからは A* をもちいた経路検索を実装していきます。

## 地形データ

マップ上のどこが歩けるかを地形データとして定義する必要があります、今回も一次元配列を用いましょう。
歩けるかどうかは真偽値で表したほうがメモリ効率が良いですが、本稿では可読性のため number で表現します。
JSON で `terrain` という名称で通行可能かどうかを表現したプロパティを追加します。

```json
"terrain": [
  1,1,1,1,1,1,1,1,
  1,0,0,0,1,0,0,1,
  1,1,1,1,1,1,1,1,
  1,0,0,1,0,0,0,1,
  1,1,1,1,1,1,1,1
]
```

C# 上でマーシャルする構造体の `Map` にも定義を追加しておきましょう。

```cs
[Serializable]
public struct Map
{
    public int width;
    public int height;
    public int[] fields;
    public int[] terrain;
}
```

これでマップ上の地形を表現し、ランタイムで扱う準備ができました。

## A*

A* 自体はそこまで複雑なアルゴリズムではないため、パフォーマンスを気にしなければ実装自体は容易です。
[A* の Wikipedia](https://en.wikipedia.org/wiki/A*_search_algorithm) の擬似コードをそのまま持ってきて C# っぽく実装したのが [コチラ](https://github.com/dolow/taxi-manhattan/blob/main/Assets/Scripts/Global/Astar.cs) です。
これと、先程の `terrain` フィールドを用いて経路検索を行っていきます。

## ゴール地点

スタート地点はゲーム任意の場所で良いでしょう、ゴール地点はユーザ入力に応じて設定する必要があります。
カメラからタップやクリックした座標に `Raycast` を飛ばしゴール地点の判定を行います。
メインカメラに `PhysicsRaycaster` を追加し、任意の `GameObject` に `Standalone Input Module` コンポーネントを追加することで、タイルで ユーザインタラクションによる `Raycast` に応じた処理を行えるようにします。

```cs
private UnityAction<Tile, BaseEventData> callback = null;

public void InitCollider(UnityAction<Tile, BaseEventData> callback)
{
    this.callback = callback;

    EventTrigger trigger = this.GetComponent<EventTrigger>();
    EventTrigger.Entry entry = new EventTrigger.Entry();
    entry.eventID = EventTriggerType.PointerDown;
    entry.callback.AddListener(this.OnPointerDown);

    trigger.triggers.Add(entry);
}

private void OnPointerDown(BaseEventData data)
{
    this.callback?.Invoke(this, data);
}
```

先程の結合したメッシュの prefab に `EventTrigger` コンポーネントを追加し、初期化時に、この `InitCollider` を実行するようにします。
また、 `MeshCollider` にも結合したメッシュを適用します。
無駄な処理になってしまうので、結合前のメッシュに対しては行いません。
一旦、タッチ/クリックした箇所が正しい内容であるか、ログ出力で確認します。

```cs
foreach (KeyValuePair<int, List<MeshFilter>> kv in filters)
{
    ...

    MeshCollider collider = combinedTile.GetComponent<MeshCollider>();
    collider.sharedMesh = filter.sharedMesh;

    combinedTile.InitCollider(this.OnPhysicsRaycasterHit);
}

private void OnPhysicsRaycasterHit(Tile tile, BaseEventData data)
{
    // アンカーポイント的にスケールの半分の大きさ分、位置判定をずらす
    // タイル配置時に調整してもよい
    Vector3 tileScale = tile.transform.localScale;
    Vector3 hitPos = ((PointerEventData)data).pointerCurrentRaycast.worldPosition;
    int x = (int)Mathf.Floor(hitPos.x + tileScale.x * 0.5f);

    this.TryWalk(x, y);
}

private void TryWalk(int x, int y)
{
    // ちゃんとゴール地点が指定できているか確認
    Debug.Log("x: " + x + ", y: " + y);
}
```

ログ出力が確認できたら、実際に `GameObject` を移動してみましょう。
今回はマップを XZ 方向に展開し、 Z が負の方向を手前としているため、移動対象の座標移動も XZ の変更で行います。
これは今回そうしているだけなので、作りやすいようにアレンジしていただいて大丈夫です。

```cs
[SerializeField]
private GameObject walker = null;

private void TryWalk(int x, int y)
{
    Vector3 basePosition = this.walker.transform.position;
    basePosition.x = x;
    basePosition.z = -y;
    this.walker.transform.position = basePosition;
}
```

`walker` に適当に `Cube` などを割り当てて実行すると、タップした場所にオブジェクトが移動することが確認出来ます。

## 経路で transform を更新する

現状だと、ゴール地点が正しく設定できるかの確認が取れただけなので、ここからは経路を探索し、経路通りに walker の transform を更新していきたいと思います。
移動するオブジェクトに経路を渡したら自分自身の `transform` を更新できるようにしましょう。
コンポーネントひとつをまるっと書く形になるので、解説はコード中のコメントで代えさせていただきます。

```cs
public class Walker : MonoBehaviour
{
    // 移動するスピード
    public float walkSpeed = 4.0f;
    // 現在地を表す terrain 上のインデックス
    public int index = 0;

    // 移動開始地点、経路全体ではなく経路の terrain のインデックス単位で更新される
    private Vector3 walkFrom = Vector3.zero;
    // walkFrom からどれくらい移動すべきか、経路の terrain のインデックス単位で更新される
    private Vector3 walkVector = Vector3.zero;
    // 経路全体を移動するために必要な移動量が terrain のインデックス単位で含まれる List
    private List<Vector3> walkDirectionQueue = new List<Vector3>();

    private void Update()
    {
        this.UpdateWalkPosition(Time.deltaTime);

        this.UpdateWalkVector();
    }

    public bool IsWalking()
    {
        // 移動量が設定されていなければ移動していないとみなす
        return this.walkVector != Vector3.zero;
    }

    public void AppendWalkDirections(List<Vector3> directions)
    {
        //  経路を追加する
        this.walkDirectionQueue.AddRange(directions);
    }

    private void UpdateWalkPosition(float dt)
    {
        // 移動中でなければ更新しない
        if (!this.IsWalking())
            return;

        // 移動量と移動スピード、デルタタイムでこのフレームの移動先を決定する
        this.transform.position += this.walkVector * this.walkSpeed * dt;
        Vector3 dest = this.walkFrom + this.walkVector;
        float estimatedDistance = Vector3.Distance(this.walkFrom, dest);
        float totalDistance = Vector3.Distance(this.walkFrom, this.transform.position);
        // terrain のインデックス間に必要な移動量を満たしていたら移動超過を修正する
        if (totalDistance >= estimatedDistance)
        {
            this.transform.position = this.walkFrom + this.walkVector;
            this.walkVector = Vector3.zero;
        }
    }

    // 現在の経路の terrain インデックス間の移動量を更新する
    private void UpdateWalkVector()
    {
        // 経路がなければ何もしない
        if (this.walkDirectionQueue.Count == 0)
            return;

        // 移動中の場合は更新しない
        if (this.IsWalking())
            return;

        this.walkFrom = this.transform.position;

        // キューの先頭を取得し、キューから削除する
        Vector3 nextDirection = this.walkDirectionQueue[0];
        this.walkDirectionQueue.RemoveAt(0);
        this.walkVector = nextDirection;
    }
}
```

`transform` を自律的に更新するコンポーネントが出来たので、これを利用するようにします。
先程、ゴール地点確認に利用した `Cube` にこのコンポーネントをアタッチします。

## 経路を渡す

先程は瞬間移動するように実装していた `TryWalk` で、`Walker` に対して経路を渡すようにします。
ここで、 A* で経路として返される `terrain` のインデックス情報を、移動量の `Vector3` に変換する必要があります。

```cs
[SerializeField]
private Walker walker = null;
// GameObject から Walker に変更する
// private GameObject walker = null;

private void TryWalk(int x, int y)
{
    if (this.walker.IsWalking())
    {
        return;
    }

    int destIndex = x + y * this.map.width;

    List<int> route = Astar.Exec(this.walker.index, destIndex, this.map.terrain, this.map.width);
    List<Vector3> directions = this.AddressesToDirections(route, this.map.width, this.map.height);
    this.walker.AppendWalkDirections(directions);
    // 論理上のインデックスはゴール地点に移動したとみなす
    this.walker.index = destIndex;
}

public List<Vector3> AddressesToDirections(List<int> addresses, int width, int height)
{
    List<Vector3> directions = new List<Vector3>();

    for (int i = 0; i < addresses.Count - 1; i++)
    {
        int address = addresses[i];
        int nextAddress = addresses[i + 1];
        Vector3 direction;
        if (nextAddress == address + width)
            direction = new Vector3(0.0f, 0.0f, -1.0f);
        else if (nextAddress == address - width)
            direction = new Vector3(0.0f, 0.0f, 1.0f);
        else if (nextAddress == address + 1)
            direction = new Vector3(1.0f, 0.0f, 0.0f);
        else if (nextAddress == address - 1)
            direction = new Vector3(-1.0f, 0.0f, 0.0f);
        else
            continue;
        directions.Add(direction);
    }

    return directions;
}
```

これで `Walker` がアタッチされた `Cube` が、ゴール地点に向かって最短距離で移動するアニメーションするようになりました。
また、通行できない箇所をタップすると何もしないことも確認できます。

ここまでの内容を `ArticleScene` として確認できるようにしたブランチを切っていますのでご参照ください。

[feature/article_step2](https://github.com/dolow/taxi-manhattan/tree/feature/article_step2)


# まとめ

本稿では Unity 3D 空間におけるタイルマップ生成と A* による経路検索及び移動アニメーションを取り扱いました。
タイルマップではメモリや描画パフォーマンスに留意し、経路検索での移動では経路を 3次元空間の移動量に置き換えました。
Unity の達人であればもっと良い方法を提案してくれるかもしれませんが、何も情報が無いよりはと思い記事を公開させていただきました。
ゲーム制作はまだまだ情報が閉じている印象です、本稿や github 公開している Unity プロジェクトが皆様のゲームづくりの参考になれば幸いです。
