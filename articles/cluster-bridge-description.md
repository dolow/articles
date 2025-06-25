---
title: ""
emoji: "🌉"
type: "tech"
topics: ["cluster", "javascript"]
published: false
---

こんにちは Smith です、クラスター株式会社のプラットフォーム事業部でプロダクトマネージャーをやっています。
クラスター株式会社では、cluster というバーチャル空間で好きなように自分のワールドを作れたり、誰かが作った様々なワールドやイベントを訪れて楽しめるサービスを提供しています。

https://cluster.mu/

cluster では定期的にワールド作りのイベントを開催しています。
本稿は 10/18~20 にかけて行われた「[ゆるゲームジャム](https://note.com/cluster_official/n/n8019e27786ff)」で作成・公開したワールドの中身について解説します。
48時間の制限時間で作成したワールドですので、お見苦しい部分も多々あるかと思いますがご容赦ください。

# ワールドについて

この記事で取り扱うワールドはこちらです。

https://cluster.mu/w/a2236219-56db-48b2-bd54-f4ebdf5ee166

github でもプロジェクト一式を公開しています。

https://github.com/dolow/ItemVisibilityFilterSample


このワールドは、見えない橋を渡ることに挑戦するというゲームワールドです。
挑戦中のユーザー以外のユーザーには橋が見えているので、スイカ割りの要領で指示を出しながら遊びます。
（そのため、ソロプレイだと適度なマゾゲーになります）

# ワールドの構造

## 主要要素

ワールドの中でスクリプトで制御されている要素は以下の通りです。

| 役割 | アタッチされている GameObject | スクリプト |
| --- | --- | --- |
| スタートボタン処理、ゲーム制御 | Arena/Start/Pedestal/ButtonItem | game_manager.js |
| スタートライン処理 | Arena/Start/Bars | bars.js |
| ギブアップ処理 | Course/GiveUp/ButtonItem | finish.js |
| ゴール処理 | Course/Goal/ButtonItem | finish.js |
| 単一の見えない足場の制御 | Course/Steps/* | step.js |
| 挑戦者のリスポーン | Course/ChallengerRespawn | challenger_despawn.js |
| オーディオ再生 | BGM/* | audio.js |
| サブノートのテキスト変更 | Arena/Start/Display | display.js |
| パーティクル再生 | Course/GoalParticles/* | particle.js |
| スコアの read/write | Arena/Start/Pedestal/ButtonItem | pcx.js |
| 回転する | Arena/SampleStep | rotate.js |
| スコアボードの表示と姿勢制御 | - | score.js |

スコアボードはシーン上には存在せず `Assets/Resourrces/Prefab/Score.prefab` として prefab から読み込まれます。

## 依存関係

本稿においての依存関係は「ある要素が別の要素の振る舞いを前提にしていること」を指します。

スクリプトの依存関係は下記のようになっています。





## 難易度表

- 🌶️ => 日常的なインターネットリテラシーで大丈夫
- 🌶️🌶️ => ちょっとインターネットや Web に詳しい必要がある
- 🌶️🌶️🌶️ => ある程度の専門知識が必要