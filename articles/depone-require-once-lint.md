---
title: "Composerオートロードで冗長になったrequire_onceを静的解析で検出する"
emoji: "🧹"
type: "tech"
topics: ["php", "composer", "静的解析", "oss", "cli"]
published: true
---

## 消せない require_once の山

長く生きている PHP プロジェクトには、Composer 導入以前の名残として `require_once` が大量に残っていることがあります。

```php
<?php

require_once __DIR__ . '/../vendor/autoload.php';
require_once __DIR__ . '/../src/Greeting.php';
require_once __DIR__ . '/../src/Legacy/Mailer.php';
require_once __DIR__ . '/../src/helpers.php';
require_once $config['plugins_dir'] . '/bootstrap.php';

$greeting = new App\Greeting();
echo $greeting->say();
```

かなり雑に言えば、`require_once` は「オートローダがなかった時代の手動オートロード」です。Composer のオートローダが入った時点で、この中のいくつかは役目を終えています。

ただ、実際に消そうとすると、1行ごとに次の問いに答える必要があります。

- このパス式は結局どのファイルを指しているのか(`__DIR__` 連結、`dirname()`、定数…)
- そのファイルが宣言するクラスは、composer.json の autoload 設定で本当にそのファイルへ解決されるのか
- そもそもこのファイル、クラス宣言以外の副作用(手続き的なコード)を持っていないか

ここがつらいところです。1行だけなら数分で確認できますが、レガシープロジェクトにはこれが数百行あります。手作業では現実的に無理で、だから誰も消さず、山は積み上がり続けます。

この判定を自動化する CLI「depone」を作りました。この記事では宣伝よりも、**中でどう判定しているか**と、**作る過程で踏んだバグ**の話をします。

https://github.com/lll-lll-lll-lll/require-once-lint

名前はラテン語の *Rerum curam depone.*(心配事を下ろしなさい)から取りました。depone は「下に置く、手放す」という意味の動詞で、消していいか分からないまま抱え続けてきた `require_once` を安心して手放せるように、という気持ちを込めています。

冒頭のコードに対して実行するとこうなります。

```
$ vendor/bin/depone
redundant_require_once=3
public/index.php:4 => src/Greeting.php
public/index.php:5 => src/Legacy/Mailer.php
public/index.php:6 => src/helpers.php

unresolved_include_require=1
  public/index.php:7 [variable] $config['plugins_dir'] . '/bootstrap.php'
```

## 全体の流れ

やっていることは3段階です。

1. `token_get_all()` で全 PHP ファイル(vendor/ 除く)をトークナイズし、include/require 文と `define()` を拾う
2. require のパス式を小さな評価器で静的に評価し、具体的なファイルパスに解決する
3. そのファイルが composer.json の autoload 設定で「本当に」オートロード可能かを判定し、可能なら冗長と報告する

AST は使っていません。必要なのは「require 文のパス式」と「クラス宣言」だけなので、トークン列を直接歩くほうが依存も少なく済みます(実行時依存は symfony/console だけです)。

ファイルを頭から走査しながら `define()` を見つけたら定数表に積んでいくので、同じファイル内で先に定義された定数は require のパス式で使えます。

```php
// 走査中に発見して $consts に積む
define('LIB_DIR', __DIR__ . '/lib');

// あとで出てきたこの式は LIB_DIR を展開して評価できる
require_once LIB_DIR . '/legacy.php';
```

## パス式の評価器

require のパス式として現実のコードに出てくるのは、おおむねこの範囲です。

```php
require_once __DIR__ . '/../src/Foo.php';
require_once dirname(__FILE__) . '/inc/functions.php';
require_once dirname(__DIR__, 2) . '/bootstrap.php';
require_once APP_ROOT . '/lib/' . 'util.php';
require_once('legacy.php');   // 関数呼び出し風
```

これを再帰下降で評価します。文法はかなり小さくて、

```
expr    := primary ('.' primary)*
primary := 文字列リテラル
         | __DIR__ | __FILE__
         | 既知の定数
         | dirname( expr [, 階層数] )
         | '(' expr ')'
```

これだけです。変数・メソッド呼び出し・`::` が出てきた時点で評価は諦めます。諦めたものをどう扱うかは後述します。

## 踏んだバグ: 括弧で始まる require

ここからが本題です。レガシーコードにはこういう書き方が実在します。

```php
require_once (dirname(__FILE__)) . '/../src/Foo.php';
```

`dirname(__FILE__)` を**さらに括弧で包んで**から連結しています。意味はただのグルーピングで、`__DIR__ . '/../src/Foo.php'` と同じです。

一方で、こういう書き方もよく見ます。

```php
require_once('legacy.php');
```

`require_once` は関数ではないので、この括弧は「関数呼び出し」ではなく式を包んでいるだけなのですが、見た目は関数呼び出しです。

初期実装では、この関数呼び出し風の形を「特別扱い」していました。require の直後のトークンが `(` だったら、対応する閉じ括弧までを引数とみなして読む、という処理です。

```php
// 旧実装のイメージ
if ($firstToken->text === '(') {
    // 対応する ')' までを式として読む
    [$exprTokens] = $this->readUntilMatchingCloseParen($tokens, $cursor);
    return $exprTokens;
}
```

`require_once('legacy.php');` に対しては正しく動きます。しかしグルーピングのケースでは、

```php
require_once (dirname(__FILE__)) . '/../src/Foo.php';
//           ^^^^^^^^^^^^^^^^^^^ ここまでしか読まない
//                               . '/../src/Foo.php' は黙って捨てられる
```

対応する `)` で読むのをやめてしまうので、後ろの連結が**丸ごと無視**されます。評価結果はディレクトリのパスになり、エラーにもならず、間違った対象に対して冗長判定が走っていました。テストも「関数呼び出し風」のケースしかなかったので、緑のまま通過していました。

修正は、特別扱いを消すことでした。字句レベルでは常に `;` まで集めるだけにして、括弧の解釈は評価器の文法(`primary := '(' expr ')'`)に任せます。

```php
public function readIncludeExprTokens(array $tokens, int $index): array
{
    $cursor = $index + 1;
    TokenHelper::skipTrivia($tokens, $cursor);

    $exprTokens = [];
    while ($cursor < $count && $tokens[$cursor]->text !== ';') {
        $exprTokens[] = $tokens[$cursor];
        $cursor++;
    }

    return $exprTokens;
}
```

こうすると `require_once('legacy.php')` は「括弧で包まれた文字列リテラル」として評価器が正しく処理し、グルーピング + 連結のケースも普通の式として評価されます。字句を読む層が式の意味を先取りしようとしたのが敗因で、**層の責務を1つ戻したらコードが減ってバグも消えた**、という教科書みたいな話でした。もちろんこのケースのリグレッションテストを足してあります。

## psr-4 は「プレフィックス一致」では足りない

もう1つ、実装していて一番気を使ったところです。

「require 対象のファイルが psr-4 のディレクトリ配下にあるか」だけで判定すると、誤検出します。たとえば composer.json がこうなっていて、

```json
{
    "autoload": {
        "psr-4": { "App\\": "src/" }
    }
}
```

`src/legacy_helpers.php` がこう書かれていたとします。

```php
<?php
// namespace 宣言なし
class LegacyHelpers { /* ... */ }
```

このファイルは psr-4 ディレクトリの中にありますが、クラス `LegacyHelpers` は `App\` プレフィックスを持たないので、**オートローダはこのファイルに永遠に到達できません**。ここで `require_once .../legacy_helpers.php` を「冗長」と報告してしまうと、ユーザーがそれを信じて消した瞬間にアプリが壊れます。大文字小文字の不一致(`src/specific/` に `App\Specific\` のクラスを置いてしまっているケース)でも同じことが起きます。

なので depone は往復で確認します。

1. ファイルが宣言するクラス名(namespace 込み)をトークン列から抽出する
2. そのクラス名を psr-4 / psr-0 / classmap の規則で**ファイルパスに解決し直す**
3. 解決先が元のファイル自身と一致したときだけ「オートロード可能」に入れる

```php
$resolver = new AutoloadResolver($this->repoRoot);
foreach ($candidateFiles as $filePath) {
    foreach ($classExtractor->extract($filePath) as $className) {
        $resolved = $resolver->resolve($className);
        if ($resolved !== null
            && PathHelper::normalize($resolved) === PathHelper::normalize($filePath)) {
            $autoloadable[$filePath] = true;  // 往復して戻ってきた
            break;
        }
    }
}
```

同名クラスが複数ファイルで宣言されているケース(レガシーあるあるです)でも、オートローダが実際に解決するほうのファイルだけが「オートロード可能」になり、負けたほうのファイルへの require は冗長と報告されません。

## 解決できないものは、理由付きで必ず報告する

変数を含むパスのように評価器が諦めたものは、黙ってスキップせず理由付きで一覧します。

| 理由 | 意味 |
| --- | --- |
| `variable` | 式に変数が含まれる |
| `method_call` | 式にメソッド呼び出しが含まれる |
| `static_access` | 式に `::` アクセスが含まれる |
| `complex` | それ以外で評価器が解決を諦めたもの |

静的解析ツールの信頼性は「何を検査したか」だけでなく「何を検査**できなかった**か」を正直に言うかどうかで決まる、と自分は考えています。「redundant と言われなかった行は安全」ではなく、「unresolved の行は自分の目で見てね」まで含めて初めて削除作業の土台になります。変数を含むパスをどこまで追いかけるべきかは自分もまだ答えを持っていなくて、いまは追いかけずに正直に報告する側に倒すのが現実的な落としどころだと考えています。

## 自動修正はしない

depone はコードを書き換えません。`require_once` が読み込むファイルは、クラス宣言だけでなく手続き的な副作用(定数定義、ini 設定、グローバルへの代入…)を持っていることがあります。つまり「オートロードでカバーされている」は必ずしも「消して安全」を意味しません。機械が「安全」と言い切れない以上、候補の提示までを機械がやり、最終判断は人間に残す。だからこそのレポート専用です。

削除前の確認用に、対象ファイルを誰が require しているかを逆に辿る `--trace` も用意しています。

```
$ vendor/bin/depone --trace src/Legacy/Mailer.php
trace_target=src/Legacy/Mailer.php
direct_callers=1
  - public/index.php
entrypoint_candidates=1
  - public/index.php
trace_paths=1
  1. public/index.php -[r]-> src/Legacy/Mailer.php
```

## おわりに

この話は [PHP Conference Ehime 2026](https://fortee.jp/phpconehime-2026/proposal/f0e20aad-f02d-45ee-9e55-8afffc68a90d)(2026-10-03)でも話します。「Composerオートロードと重複する冗長なrequire_onceを静的解析で自動検出する」というタイトルで、評価器の中身やレガシーコードでの実践をもう少し掘る予定です。

```sh
composer require --dev lll-lll-lll-lll/depone
vendor/bin/depone
```

レガシープロジェクトで試してみて、評価器が諦める変な `require_once` パターンに出会ったら、[issue](https://github.com/lll-lll-lll-lll/require-once-lint/issues) で教えてもらえるとうれしいです。それが一番の貢献です。
