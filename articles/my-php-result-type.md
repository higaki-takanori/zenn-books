---
title: "PHPでResult型やってみる"
emoji: "🐘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["php", "カンファレンス", "PHPカンファレンス関西", "Result型"]
published: true
---

# はじめに

こんにちは。ひがきです。

[PHPカンファレンス関西2025](https://2025.kphpug.jp/)の「[PHPでResult型（クラス）やってみよう](https://fortee.jp/phpcon-kansai2025/proposal/6d3a6fd6-f8e1-4362-adad-4f34548b7a9f)」で時間の関係で省略せざるを得ない部分の補足資料となります。

どうやってPHPでResult型を実装するかの部分についての説明をこの記事に記載しておきます。

# Special Thanks

PHPでResult型を実装する際に[tadsan](https://x.com/tadsan)に相談に乗っていただきました。本当にありがとうございました。

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
    /**
     * @param T $value
     */
    public function __construct(
        private mixed $value,
    ) {}
    
    // 各関数を実装
}
```

```php
/**
 * @template E
 * @implements Result<never, E>
 */
final readonly class Err implements Result {
    /**
     * @param E $value
     */
    public function __construct(
        private mixed $value,
    ) {}
    
    // 各関数を実装
}
```

## isOkの実装

`isOk`は`Result`の中身が`Ok`かチェックする関数です。

言い換えれば、ある処理の結果が成功したことを確認する関数です。

### Result interface

```php
interface Result
{
    /**
     * @phpstan-assert-if-true Ok<T> $this
     * @phpstan-assert-if-false Err<E> $this
     */
    public function isOk(): bool;
}
```

`@phpstan-assert-if-true`と`@phpstan-assert-if-false`で型のnarrowingを行なっています。

### Ok

```php
final readonly class Ok implements Result
{
    // ...

    public function isOk(): true
    {
        return true;
    }
}
```

### Err

```php
final readonly class Err implements Result
{
    // ...
    
    public function isOk(): false
    {
        return false;
    }
}
```


## isErrの実装

`isErr`は`Result`の中身が`Err`かチェックする関数です。

### Result interface

```php
interface Result
{
    // ...
    
    /**
     * @phpstan-assert-if-true Err<E> $this
     * @phpstan-assert-if-false Ok<T> $this
     */
    public function isErr(): bool;
```

`@phpstan-assert-if-true`と`@phpstan-assert-if-false`で型のnarrowingを行なっています。

### Ok

```php
final readonly class Ok implements Result
{
    // ...

    public function isErr(): false
    {
        return false;
    }
}
```

### Err

```php
final readonly class Err implements Result
{
    // ...

    public function isErr(): true
    {
        return true;
    }
}
```

## unwrapの実装

`unwrap`は`Ok`の値を返却する関数です。

`Result`の中身が`Err`の時に`unwrap`を実行すると`LogicException`を投げてnever型を返すようにしました。

これにより、後続の処理を記載した際にPHPStanがエラーを吐いてくれるので、ある処理が失敗した時に成功時の値を取得しようしていることに気づけます。

### Result interface

```php
interface Result
{
    // ...

    /**
     * @return ($this is Result<T, never> ? T : never)
     */
    public function unwrap(): mixed;
}
```

interfaceでnever型か成功時の値が返ってくることを明示しました。

### Ok

```php
final readonly class Ok implements Result
{
    /**
     * @return T
     */
    public function unwrap(): mixed
    {
        return $this->value;
    }
}
```

### Err

```php
final readonly class Err implements Result
{
    // ...

    /**
     * @return never
     */
    public function unwrap(): never
    {
        throw new \LogicException('called Result->unwrap() on an err value');
    }
}
```


## unwrapErrの実装

`unwrapErr`は`Err`の値を返却する関数です。

`Result`の中身が`Ok`の時に`unwrapErr`を実行すると`LogicException`を投げてnever型を返すようにしました。

これにより、後続の処理を記載した際にPHPStanがエラーを吐いてくれるので、ある処理が成功した時に失敗時の値を取得しようしていることに気づけます。

### Result interface

```php
interface Result
{
    // ...

    /**
     * @return ($this is Result<never,E> ? E : never)
     */
    public function unwrapErr(): mixed;
}
```

interfaceでnever型か失敗時の値が返ってくることを明示しました。

### Ok

```php
final readonly class Ok implements Result
{
    // ...
    
    /**
     * @return never
     */
    public function unwrapErr(): never
    {
        throw new \LogicException('called Result->unwrapErr() on an ok value');
    }
}
```

### Err

```php
final readonly class Err implements Result
{
    // ...

    /**
     * @return E
     */
    public function unwrapErr(): mixed
    {
        return $this->value;
    }
}
```

## unwrapOrの実装

`unwrapOr`は`Result`の型が`Ok`の場合は`Ok`の値を返却し、`Err`の場合は、引数に指定したものを返却します。

### Result interface

```php
interface Result
{
    // ...

    /**
     * @template D
     * @param D $default
     * @return ($this is Result<T, E> ? T|D : ($this is Result<never, E> ? D : T))
     */
    public function unwrapOr(mixed $default): mixed;
}
```

`@return T|D`で充分だと思うんですが、 型のnarrowingも行うようにしました。

### Ok

```php
final readonly class Ok implements Result
{
    // ...
    
    /**
     * @template D
     * @param D $default
     * @return T
     */
    public function unwrapOr(mixed $default): mixed
    {
        return $this->value;
    }
}
```

### Err

```php
final readonly class Err implements Result
{
    // ...

    /**
     * @template D
     * @param D $default
     * @return D
     */
    public function unwrapOr(mixed $default): mixed
    {
        return $default;
    }
}
```

# 補足

PHPStanのAllowedSubtypesを使用することでPHPStanに`Result`のinterfaceを実装するクラスは`Ok`と`Err`の2つだけであることを伝えることができます。

https://phpstan.org/developing-extensions/allowed-subtypes


あと、[phpstan-sealed](https://github.com/phpstan/phpstan-src/pull/4095)がリリースされると、AllowedSubTypesClassReflectionExtensionの代わりに、以下の記載だけで済むみたいです！

```php
/** 
 *  @phpstan-sealed Ok|Err
 */
interface Result {
    // ...
}
```

# まとめ

PHPでResult型を実装する方法について説明しました。

この記事がPHPでResult型を実装する際の参考になれば幸いです。
