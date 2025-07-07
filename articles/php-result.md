---
title: "PHPでResult型やってみる"
emoji: "🐘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["php", "カンファレンス", "PHPカンファレンス関西", "Result型"]
published: false
publication_name: "levtech"
---

# はじめに

こんにちは。レバテックのひがきです。

[PHPカンファレンス関西2025](https://2025.kphpug.jp/)の「[PHPでResult型（クラス）やってみよう](https://fortee.jp/phpcon-kansai2025/proposal/6d3a6fd6-f8e1-4362-adad-4f34548b7a9f)」で時間の関係で省略せざるを得ない部分の補足資料となります。

@[speakerdeck](370920ae44cb45bf97dde8d29440ab32)

どうやってPHPでResult型を実装するかの部分についての説明をこの記事に記載しておきます。

# どうやってPHPでResult型を実現するか

## 基本方針

基本方針として以下としました。（抽象クラスで実装しても同じことができるはずです）
- `Result`のinterfaceを作成
- `Ok`クラスで`Result`のinterfaceを実装する
- `Err`クラスで`Result`のinterfaceを実装する
- `Result`で`Ok`と`Err`の具体は`@template`で受け取る

```php
/**
 * @template T
 * @template E
 */
interface Result {
    // 各関数を定義
}
```

```php
/**
 * @template T
 * @implements Result<T, never>
 */
final readonly class Ok implements Result {
    // 各関数を実装
}
```

```php
/**
 * @template E
 * @implements Result<never, E>
 */
final readonly class Err implements Result {
    // 各関数を実装
}
```

## isOkの実装

`isOk`は`Result`の中身が`Ok`かチェックする関数です。

## isErrの実装

`isErr`は`Result`の中身が`Err`かチェックする関数です。

## unwrapの実装

`unwrap`は`Ok`の値を返却する関数です。

`Result`の中身が`Err`の時に`unwrap`を実行すると`RuntimeException`を投げるようにしました。

## unwrapErrの実装

`unwrapErr`は`Err`の値を返却する関数です。

`Result`の中身が`Ok`の時に`unwrapErr`を実行すると`RuntimeException`を投げるようにしました。

## unwrapOrの実装

`unwrapOr`は`Result`の型が`Ok`の場合は`Ok`の値を返却し、`Err`の場合は、引数に指定したものを返却します。

この記事で`unwrapOr`の実装は以下の2通りの方法を記載します。

- interfaceで定義した場合
- PHPStanのAllowedSubtypesを使って実装する場合

interfaceで定義した場合、PHPStanの型推論で`unwrapOr`の返り値を正しく返せないため、PHPStanのAllowedSubtypesを使って実装する方法も記載しました。
（正しく型推論できる方法をご存知の方はコメント欄に書いてくださいる非常に助かります）

### unwrapOrをinterfaceで定義した場合

ある程度型推論を妥協しても良い場合は、以下のような実装になるんじゃないかと思います。

```php
    /**
     * @template D
     * @param D $default
     * @return T|D
     */
    public function unwrapOr(mixed $default): mixed;
```


最初、以下の形で型推論できると思っていたのですがPHPStanのエラーが出ました。

```php
    /**
     * @template D
     * @param D $default
     * @return (T is never ? D : T)
     */
    public function unwrapOr(mixed $default): mixed;
```

`phpstan: Condition "T is never" in conditional return type is always false.`

interface側で`T`は常にneverでないと

### unwrapOrをPHPStanのAllowedSubtypesを使って実装する場合




# まとめ・宣伝


