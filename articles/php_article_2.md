---
title: "【PHPUnit】スナップショットテストにおけるjson構文エラーの解決方法"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["PHP", "PHPUnit"]
published: true
publication_name: "team_delta"
---
## 問題
PHPUnitでスナップショットテストを実行する際に、以下のようなjson構文エラーが発生しました。

```php
Tests\Unit\Infrastructure\Query\Hoge\HogeTest > Check_Hoge
PHPUnit\Framework\ExceptionWrapper

Syntax error

at vendor/spatie/phpunit-snapshot-assertions/src/Drivers/JsonDriver.php:34
30▕ {
31▕ if (is_string($actual)) {
32▕     $actual = json_decode($actual, true, 512, JSON_THROW_ON_ERROR);
33▕ }
➜ 34▕ $expected = json_decode($expected, true, 512, JSON_THROW_ON_ERROR);
35▕ Assert::assertJsonStringEqualsJsonString(json_encode($expected), json_encode($actual));
36▕ }
37▕ }
38▕
```

##  実行したコマンド
```bash
php artisan test tests/Unit/Infrastructure/Query/Hoge/HogeTest.php -d --update-snapshots
```
使用されているオプション：

- -d（--debugの省略形）：デバッグモードでテストを実行し、詳細な実行情報を表示
- --update-snapshots：既存のスナップショットを現在の実行結果で更新

## 原因
このエラーは主に以下の原因で発生するそうです：
1. スナップショットファイル内のjsonが破損している
2. jsonの構文が正しくない
3. 既存のスナップショットを更新する際に、既存のものが期待される形式と異なる形式で保存されている

例えば、以下のような不完全なjsonが含まれている場合もこのエラーが発生する原因となるようです。
```json
{
    "queries": []
}
```
私に起こったエラーの原因はoriginとのmergeがうまくいかず、1が原因でした。

## 対策
以下の順でエラーを解消し、無事にスナップショットを更新しました。
1. 問題のありそうな既存のスナップショットファイルを特定
2. 既存のスナップショットを削除するもしくは、修復する（自分の場合は、mainブランチの最新変更を取り込む際に発生したコンフリクトが正しく解消されていなかったため、改めて解消しました）
3. 上記のコマンドを実行し、スナップショットを生成・更新する

## まとめ
スナップショットテストにおけるjson構文エラーは、主にスナップショットファイルの破損や不正な形式が原因で発生することがわかりました。
スナップショットの更新に成功しても、必ず生成されたスナップショットの中身をみて正しい形式で生成されているか確認してください。
また、以下の点に注意することでエラーを予防できます：
- mainブランチの最新変更を取り込む際にはスナップショットの中身までもコンフリクトの解消が正しく行われているか確認する
- スナップショットファイルの手動編集は避ける
- 変更時は必ずテストを実行して確認する
