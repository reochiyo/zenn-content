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

とは言っても、モデルを構築するにはどうすればいいのっていう話です。

実はKaggleには、このようにレベルに応じたチュートリアルのCompetitionが用意されているんです！

![](/images/kaggle_article_2/image1.png)

もちろん私はBeginnerなので、それを選択すると、３つのCompetitionが表示されました。

![](/images/kaggle_article_2/image2.png)

その中でも一番左端にある、参加Team数が一番多いTitanicを選びました。

## Notebook
実際に開発に入る前に覚えているといい、便利な用語があります。

それがNotebook（呼び方：ノートブック）です。

これは、PythonやR言語をブラウザ上で操作できる開発環境のことで、一から自分のローカル環境に環境構築しなくともデータサイエンスや機械学習でよく使うパッケージなどがインストールされています（Google Colabみたいなイメージです）。

各コンペに以下のようにあり、そのボタンを押すと、あらかじめモデルを作成する際に使うであろうライブラリがimportされてたり、ファイルなどのpathなどが記載されていたりします（下図はTitanicコンペでのイメージ）。

![](/images/kaggle_article_2/image3.png)

コンペからでなくてもNotebookを作成することも可能です。

以下がTitanicコンペでNotebookを初めて開いた時のイメージです。

![](/images/kaggle_article_2/image4.png)

勝手に書いたNotebookは自動保存される仕組みですが、手動で保存も可能です。

## モデル作成
以下では、実際にTitinicコンペでのモデル作成の流れを記述していきます。

### 1. パッケージやライブラリをインポート
まず、配列の数値演算を行うnumpy、データフレームによるデータ加工を行うpandas、グラフを描写するmatplotlibとその拡張ライブラリであるseabornをimportします。

![](/images/kaggle_article_2/image5.png)

記載し終わったら、左側の再生ボタンを押すか、shift+enterで実行してください。

![](/images/kaggle_article_2/image6.png)

ちなみにNotebookでの開発はGoogle ColabやJupyter Notebookと同様のセルコーディングです（＊[参考](https://qiita.com/hinastory/items/e179361ae806e8776c70)）。

### 2. 教師データを読み込む
事前に用意されている（TitanicコンペからNotebookを開いたらすでに書かれている）、３つのcsvファイルをpandasを用いて読み込みます。

![](/images/kaggle_article_2/image7.png)

### 3. 探索的データ分析（EDA）
予測度が高いモデルを作成するためには、データの特徴を読み取って、適切なアルゴリズムを当てはめることが大切です。

そこで、データの特徴を前もって確認することを探索的データ分析（EDA）といいます。

先ほどの2.で読み込んだtrain.csvファイルはモデルを学習する用のcsvファイルです。

Titanicに乗船した乗客の情報や乗客の生存情報などが載っています。

test.csvファイルにはtrain.csvを用いて作成されたモデルのtestするためのファイルです。

Titanicに乗船した乗客の情報は入っていますが、予測対象である、生存したらどうかは入っておりません。

最後に読み取ったgender_submission.csvは提出用のcsvファイルです。

今回は以下のように、各乗客idと生存したか否かをまとめたファイルにしてくださいということです。

![](/images/kaggle_article_2/image8.png)

#### ・describe
また、以下はdescribe関数を用いて主要な統計指標を確認している様子です。

![](/images/kaggle_article_2/image9.png)

ちなみにmeanは平均値、stdは標準偏差（データのばらつきの度合い）を表しています。

#### ・ydata_profiling（pandas_profiling）
次にydata_profiling（かつてはpandas_profiling）というライブラリは、データをざっと確認するのによく使うライブラリです。

使用するにはimportし、実行の際は、ydata_profilingのprofile_report()関数を用います。

![](/images/kaggle_article_2/image10.png)

以下が実行結果の一部です。

![](/images/kaggle_article_2/image11.png)

- Overview : 全体の概要
    - Dataset statistics : データの構造を表すもの
    - Variable types : 変数の型

![](/images/kaggle_article_2/image12.png)

- Variables : 列ごとの詳細な情報の確認
    - unifrom : 値が一様であること
    - unique : 値の重複を許してないこと
    - Pclass : 乗客の部屋のランク
    - Missing : 不詳なデータがあるということ
    - Parch : 親子の数
    - Embarked : 停まった港の名前

このほかにもデータの概要が出力されています。

### 4. 特徴的エンジニアリング
今回は目的変数＝Survived(生きたか死んだかの値)とし、各特徴量とのヒストグラムを作ります。

以下は、横軸をPclass(乗客の部屋のランクを表す変数)とし、ヒストグラムを作った例です。

![](/images/kaggle_article_2/image13.png)

部屋のランクが低い３等客室の人の死亡率が高いことがわかります。

このような感じで、他の特徴量を横軸として、何らかのデータの傾向を読み取ります。

データ分析は仮説を立てて検証を行うといったものの繰り返しなので、気付きがきったらどんどんメモしていきましょう。

ちなみにヒストグラムはmatplotlibの関数を用いてカスタマイズすることで、見やすくもできます。

（カスタマイズ前）

![](/images/kaggle_article_2/image14.png)

（カスタマイズ後）

![](/images/kaggle_article_2/image15.png)

最終的にデータを分析した結果からアルゴリズムを適応させる、特徴的エンジニアリングを行います。

教師データとテストデータの双方にエンジニアリングを行うことから、一旦一つに結合します。

![](/images/kaggle_article_2/image16.png)

次に、欠損値（値がnullなど）の数をisnull()関数を用いて確認します。

![](/images/kaggle_article_2/image17.png)

欠損値を適当な文字などに置き換えて、それらを数値に変換します(どんな値で補完するかによっても結果は変わってくる)。

テストデータはSurvivedがないため、欠損値としてカウントされています。

以下の例はEmbarked変数である、SとCとQという港のイニシャルを0、1、２に置き換えている様子です（One-Hotエンコーディング）。

![](/images/kaggle_article_2/image18.png)

ここまできたら影響の少ない特徴量は一旦削除する。

![](/images/kaggle_article_2/image18.png)

最後に、連結させていた教師データとテストデータをスライス関数で分離させます。

![](/images/kaggle_article_2/image19.png)

ここまできたらデータが完成したので、モデルを作成していきましょう。

### 5. アルゴリズムの学習
モデルを作成する際は、あらかじめ用意されたアルゴリズムを適用して作っていきます。

以下では機械学習でよく使われるsklearnというライブラリからRandomForestClassifierというアルゴリズムをimportしている例です。

![](/images/kaggle_article_2/image20.png)

RandomForestは決定木を拡張したアルゴリズムで、条件ごとに分岐させながら最終的に予測をするというものです。

最初は一つの条件から枝や葉のように分岐していくことから決定木とは言われています。

それに対してRandomForestは複数の決定木のアンサンブルによって予測を行うアルゴリズムです。

以下はclfにRandomForestを実装している例です。

![](/images/kaggle_article_2/image21.png)

決定木は100で、木の深さは2階層（条件の多さとも言える）であることがわかります。

木の深さは深い程よいと思いがちですが、意外にそうでもありません。

深ければ深いほど、条件が多くなるという意味であり、予測に関係のない条件が追加される恐れがあるからです。

また、深いほど計算処理も遅くなります。

少ないよりかは多い方がいいのは明らかですが、ある程度の深さが求められます。

それでは次に、教師データで学習させましょう。

![](/images/kaggle_article_2/image22.png)

たったこれだけでモデルは完成しました。

### 6. 予測の実施
作成したモデルで予測を行いましょう。

予測の実行はpredict関数で行います。

学習したモデルの変数clfに対して、引数にはテストデータを指定します。

![](/images/kaggle_article_2/image23.png)

以下が結果が格納された変数の中身です。

![](/images/kaggle_article_2/image24.png)

list上に結果が返されていることがわかります。

0がdiedで、1がsurvivedということですが、predict関数の閾値は0.5がデフォルトであるため、0.5以上を1、0.5未満を0として返されています。

### 7. 予測の提出
結果が可視化できたということでsubmitを行います。

![](/images/kaggle_article_2/image25.png)

map関数でlistのそれぞれのデータをint型に変換後、最終的にcsvファイルに出力します(index番号は出力しないためFalse)。

ここまで作業が済んだら、書いたコードをcommitします。

右上のSave Versionというボタンをクリックして、versionを指定して記載したのち、commitボタンを押し、saveします。

![](/images/kaggle_article_2/image26.png)

最後にcompleteと出ればcommitが完了です。

最後にOutputのsubmitボタンを押したら提出完了です。

提出後はスコアを確認できます。

![](/images/kaggle_article_2/image27.png)

私の正答率は約割だったようです。

今回はテストデータの予測を行いましたが、主催者はそのテストデータの答えを持っているので、そのデータと照らし合わせてテストの答えの割合がわかるという感じです。

ここからは、正答率を上げるため、精度の高いモデルを作成していくという流れです。

ですが、今回はモデルを作成する一連の流れを説明したかったので、その作業は次の記事でということにします。

## おわりに
一連のモデルを作成し、結果を出力するという作業を行いました。
ある程度パッケージやライブラリに様々なアルゴリズムの関数が入っているので、全体的なコード量は少なかったです。
今後はこのモデルの精度を上げるための方法や、ライブラリに含まれる使える関数などを紹介していけたらないと思います。

## 過去の記事

## 参考文献
- https://www.udemy.com/course/kagglepython-ai/