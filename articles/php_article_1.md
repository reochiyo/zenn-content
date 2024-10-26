---
title: "【PHPUnit】dataProviderで日時を扱う処理で沼ったこと"
emoji: "🎃"
type: "tech"
topics: ["PHP", "PHPUnit"]
published: true
---
## はじめに
TestCaseを継承しているテストコードで、dataProviderを用いている時にsetUpメソッドで現在時刻を固定したのに処理がうまくいかなかった時の対処法です。

## 開発環境
- php : 8.2
- phpunit : 9.0

## テストが落ちるコード
```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use Tests\TestCase;
use Carbon\CarbonImmutable;

class HogeTest extends TestCase
{
    protected function setUp(): void
    {
        parent::setUp();
        CarbonImmutable::setTestNow(CarbonImmutable::parse('2024-01-01')); //時刻を固定
    }

    /**
     * @dataProvider hogeProvider
     */
    public function test_Hogeテスト(CarbonImmutable $reference_date, bool $result)
    {
        $this->assertEquals($reference_date->isPast(), $result);
    }

    public function hogeProvider()
    {
        return [
            '与えられた日時が過去の時trueになる' => [
                CarbonImmutable::now()->subDay(),
                true
            ],
            '与えられた日時が未来の時falseになる' => [
                CarbonImmutable::now()->addDay(),
                false
            ]
        ];
    }
}
```
→出力結果
```
   FAIL  Tests\Unit\HogeTest
  ⨯ hogeテスト with data set "与えられた日時が過去の時trueになる"
  ✓ hogeテスト with data set "与えられた日時が未来の時falseになる"

  ---
```


## デバッグしてみた
```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use Tests\TestCase;
use Carbon\CarbonImmutable;

class HogeTest extends TestCase
{
    protected function setUp(): void
    {
        parent::setUp();
        CarbonImmutable::setTestNow(CarbonImmutable::parse('2024-01-01')); //時刻を固定
    }

    /**
     * @dataProvider hogeProvider
     */
    public function test_Hogeテスト(CarbonImmutable $reference_date, bool $result)
    {
        echo("テストメソッド内の時刻:" . CarbonImmutable::now() . "\n");
        $this->assertEquals($reference_date->isPast(), $result);
    }

    public function hogeProvider()
    {
        echo("dataProvider内の時刻:" . CarbonImmutable::now() . "\n");
        return [
            '与えられた日時が過去の時trueになる' => [
                CarbonImmutable::now()->subDay(),
                true
            ],
            '与えられた日時が未来の時falseになる' => [
                CarbonImmutable::now()->addDay(),
                false
            ]
        ];
    }
}
```

→出力結果
```
dataProvider内の時刻:2024-10-25 10:00:19
テストメソッド内の時刻:2024-01-01 00:00:00
テストメソッド内の時刻:2024-01-01 00:00:00
```

デバッグしてわかるように、dataProviderの中で現在時刻が固定できていないみたいです。
上記のテストで「与えられた日時が未来の時falseになる」の項目だけ通って、「与えられた日時が過去の時trueになる」の項目は落ちるロジックがこれで説明できます。

## 原因
原因は以下のような実行順序にありました。
1. dataProvider→最初に一回のみ実行
2. setUpメソッド→各テストメソッドの実行前に実行


## まとめ（テストが通るコード）
結果的にparseで時刻を固定したら正常に通りました。

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use Tests\TestCase;
use Carbon\CarbonImmutable;

class HogeTest extends TestCase
{
    // setUpメソッドを削除

    /**
     * @dataProvider hogeProvider
     */
    public function test_Hogeテスト(CarbonImmutable $reference_date, bool $result)
    {
        $this->assertEquals($reference_date->isPast(), $result);
    }

    public function hogeProvider()
    {
        $now = '2024-01-01'; // 時刻を固定
        return [
            '与えられた日時が過去の時trueになる' => [
                CarbonImmutable::parse($now)->subDay(), // 修正
                true
            ],
            '与えられた日時が未来の時falseになる' => [
                CarbonImmutable::parse($now)->addDay(), // 修正
                false
            ]
        ];
    }
}
```

→出力結果
```
   PASS  Tests\Unit\HogeTest
  ✓ hogeテスト with data set "与えられた日時が過去の時trueになる"
  ✓ hogeテスト with data set "与えられた日時が未来の時falseになる"

  Tests:  2 passed
  Time:   0.06s
```

## 参考文献
- [PHPUnitのテストクラスのメソッドの動く順番を調べた](https://zenn.dev/naopusyu/articles/42d393c1c1b4bc)
- [【phpunit】dataProvider内のタイムゾーンがUTCな件](https://uma-no-kawa.hateblo.jp/entry/2022/03/30/160217#google_vignette)