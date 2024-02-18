---
title: "【Kaggle】実践！Titanicコンペティション"
emoji: "🚢"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kaggle", "python", "ai"]
published: false
publication_name: "team_delta"
---
## はじめに
Kaggleのアカウントを発行して間もないですが、早速Kaggleでモデルを構築したい気持ちが抑えきれないので、Competitionにチャレンジしてみました。

とは言っても、モデルを構築するにあたって右も左もわからない私みたいな人はどうすればいいのって話です。

実はKaggleには、このようにレベルに応じたチュートリアルのCompetitionが用意されているんです！

![](/images/kaggle_article_2/image1.png)

もちろん私はBeginnerなので、それを選択すると、３つのCompetitionが表示されました。

![](/images/kaggle_article_2/image2.png)

その中でも一番端にある、参加Team数が一番多いTitanicを選びました。

## Notebook
実際に開発に入る前に覚えていたら便利な用語があります。

それがNotebook（呼び方：ノートブック）です。

これは、PythonやR言語をブラウザ上で操作できる開発環境のことで、一から自分のローカル環境に環境構築しなくともデータサイエンスや機械学習でよく使うパッケージなどがインストールされています（Google Colabみたいなイメージです）。

各コンペに以下のようにあり、そのボタンを押すと、あらかじめモデルを作成する際に使うであろうライブラリがimportされてたり、ファイルなどのpathなどが記載されていたりします（下図はTitanicコンペでのイメージ）。

![](/images/kaggle_article_2/image3.png)

コンペからでなくてもNotebookを作成することも可能です。

以下がTitaninコンペでNotebookを初めて開いた時のイメージです。

![](/images/kaggle_article_2/image4.png)

勝手に書いたNotebookは自動保存される仕組みですが、手動で保存も可能です。

## モデル作成の流れ
以下では、実際にTitinicコンペでのモデルの作成の流れを記述していきます。

### 1. パッケージやライブラリをインポート
まず、配列の数値演算を行うnumpy、データフレームによるデータ加工を行うpandas、グラフを描写するmatplotlibとその拡張ライブラリであるseabornをimportします。

![](/images/kaggle_article_2/image5.png)

記載し終わったら、左側の再生ボタンを押すか、shift+enterで実行してください。

![](/images/kaggle_article_2/image6.png)

ちなみにNotebookでの開発はGoogle ColabやJupyter Notebookと同様のセルコーディングです（今回は説明を省きますね。＊[参考](https://qiita.com/hinastory/items/e179361ae806e8776c70)）。

### 2. 教師データを読み込む
事前に用意されている（TitanicコンペからNotebookを開いたらすでに書かれている）、３つのcsvファイルをpandasを用いて読み込みます。

![](/images/kaggle_article_2/image6.png)

### 3. 探索的データ分析（EDA）
予測度が高いモデルを作成するためには、データの特徴を読み取って、適切なアルゴリズムを当てはめることが大切です。

そこで、データの特徴を前もって確認することを探索的データ分析（EDA）といいます。

先ほどの2.で読み込んだtrain.csvファイルはモデルを学習する用のcsvファイルです。

Titanicに乗船した乗客の情報や乗客の生存情報などが載っています。

test.csvファイルにはtrain.csvを用いて作成されたモデルのtestするためのファイルです。

Titanicに乗船した乗客の情報は入っていますが、予測対象である、生存したらどうかは入っておりません。

最後に読み取ったgender_submission.csvは提出用のcsvファイルです。

今回は以下のように、各乗客idと生存したか否かをまとめたファイルにしてくださいということです。

![](/images/kaggle_article_2/image7.png)

#### ・describe
また、以下はdescribe関数を用いて主要な統計指標を確認している様子です。

![](/images/kaggle_article_2/image8.png)

#### ・pandas_profiling
次にpandas_profilingというライブラリも、よく使うライブラリです。

使用するにはimportします。

![](/images/kaggle_article_2/image9.png)

実行の際は、pandas_profilingのprofile_report()関数を用います。

![](/images/kaggle_article_2/image10.png)

以下が実行結果です。

![](/images/kaggle_article_2/image11.png)

ちょっと読みづらい・わかりずらいとこだけの用語説明：
- Overview : 全体の概要
    - Dataset statistics : データの構造を表すもの
    - Variable types : 変数の型
- Variables : 列ごとの詳細な情報の確認
    - unifrom : 値が一様であること
    - unique : 値の重複を許してないこと
    - Pclass : 乗客の部屋のランク
    - Missing : 不詳なデータがあるということ
    - Parch : 親子の数
    - Embarked : 停まった港の名前

### 4. 特徴的エンジニアリング
今回は目的変数＝Survived(生きたか死んだかの値)とし、各特徴量とのヒストグラムを作ります。

以下は、横軸をPclass(乗客の部屋のランクを表す変数)とし、ヒストグラムを作った例です。

![](/images/kaggle_article_2/image12.png)

部屋のランクが低い３等客室の人の死亡率が高いことがわかります。

このような感じで、他の特徴量を横軸として、何らかのデータの傾向を読み取ります。

データ分析は仮説を立てて検証を行うといったものの繰り返しなので、気付きがきったらどんどんメモしていきましょう。

ちなみにヒストグラムはmatplotlibの関数を用いてカスタマイズすることで、見やすくもできます。

（カスタマイズ前）

![](/images/kaggle_article_2/image13.png)

（カスタマイズ後）

![](/images/kaggle_article_2/image14.png)

最終的にデータを分析した結果からアルゴリズムを適応させる、特徴的エンジニアリングを行います。

教師データとテストデータの双方にエンジニアリングを行うことから、一旦一つに結合します。

![](/images/kaggle_article_2/image15.png)

次に、欠損値（値がnullなど）の数をisnull()関数を用いて確認します。

![](/images/kaggle_article_2/image16.png)

欠損値を適当な文字などに置き換えて、それらを数値に変換します(どんな値で補完するかによっても結果は変わってくる)。

以下の例はEmbarked変数である、SとCとQという港のイニシャルを0、1、２に置き換えている様子です（One-Hotエンコーディング）。

![](/images/kaggle_article_2/image17.png)

ここまできたら影響の少ない特徴量は一旦削除する。

![](/images/kaggle_article_2/image18.png)

最後に、連結させていた教師データとテストデータをスライス関数で分離させます。

![](/images/kaggle_article_2/image19.png)

ここまできたらデータが完成したので、モデルを作成していきましょう。

### 5. アルゴリズムの学習

### 6. 予測の実施

## おわりに

## 過去の記事

## 参考文献
- https://www.udemy.com/course/kagglepython-ai/