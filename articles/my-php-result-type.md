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

@[speakerdeck](51d71abd65834fd08ccdebdb25d6bdca)

どうやってPHPでResult型を実装するかの部分についての説明をこの記事に記載しておきます。

# Special Thanks

PHPでResult型を実装する際に[tadsan](https://x.com/tadsan)に相談に乗っていただきました。本当にありがとうございました。

# どうやってPHPでResult型を実現するか

先に完成系のコードを載せておきます。

::: details 最終的なResult Interface
```php
/**
 * @phpstan-sealed Ok|Err
 * @template T
 * @template-covariant E
 */
interface Result
{
    /**
     * @phpstan-assert-if-true Ok<T> $this
     * @phpstan-assert-if-false Err<E> $this
     */
    public function isOk(): bool;
    
    /**
     * @phpstan-assert-if-true Err<E> $this
     * @phpstan-assert-if-false Ok<T> $this
     */
    public function isErr(): bool;
    
    /**
     * @return ($this is Result<T, never> ? T : never)
     */
    public function unwrap(): mixed;
    
    /**
     * @return ($this is Result<never,E> ? E : never)
     */
    public function unwrapErr(): mixed;
    
    /**
     * @template D
     * @param D $default
     * @return ($this is Result<T, E> ? T|D : ($this is Result<never, E> ? D : T))
     */
    public function unwrapOr(mixed $default): mixed;
}
```
:::

::: details 最終的なOkクラス
```php
/**
 * @template T
 * @implements Result<T, never>
 */
final readonly class Ok implements Result
{
    /**
     * @param T $value
     */
    public function __construct(
        private mixed $value,
    ) {
    }

    public function isOk(): true
    {
        return true;
    }

    public function isErr(): false
    {
        return false;
    }

    /**
     * @return T
     */
    public function unwrap()
    {
        return $this->value;
    }

    public function unwrapErr(): never
    {
        throw new LogicException('called Result->unwrapErr() on an ok value');
    }

    /**
     * @template D
     * @param D $default
     * @return T
     */
    public function unwrapOr(mixed $default)
    {
        return $this->value;
    }
}
```
:::


::: details 最終的なErrクラス
```php
/**
 * @template-covariant E
 * @implements Result<never, E>
 */
final readonly class Err implements Result
{
    /**
     * @param E $value
     */
    public function __construct(
        private mixed $value,
    ) {
    }

    public function isOk(): false
    {
        return false;
    }

    public function isErr(): true
    {
        return true;
    }

    public function unwrap(): never
    {
        throw new LogicException('called Result->unwrap() on an err value');
    }

    /**
     * @return E
     */
    public function unwrapErr()
    {
        return $this->value;
    }

    /**
     * @template D
     * @param D $default
     * @return D
     */
    public function unwrapOr(mixed $default)
    {
        return $default;
    }
}
```
:::

## 基本方針

:::message
Rustのコードを参考にしました。
:::

基本方針として以下としました。（抽象クラスで実装しても同じことができるはずです）
- `Result`のinterfaceを作成
- `Ok`クラスで`Result`のinterfaceを実装する
- `Err`クラスで`Result`のinterfaceを実装する
- `Result`で`Ok`と`Err`の具体は`@template`で受け取る

```php
/**
 * @template T
 * @template-covariant E
 */
interface Result {
    // 各関数を定義
}
```

```php
/**
 * @phpstan-sealed Ok|Err
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
 * @template-covariant E
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
    public function unwrap();
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
    public function unwrap()
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

    public function unwrap(): never
    {
        throw new LogicException('called Result->unwrap() on an err value');
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
    public function unwrapErr();
}
```

interfaceでnever型か失敗時の値が返ってくることを明示しました。

### Ok

```php
final readonly class Ok implements Result
{
    // ...
    
    public function unwrapErr(): never
    {
        throw new LogicException('called Result->unwrapErr() on an ok value');
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
    public function unwrapErr()
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
    public function unwrapOr(mixed $default);
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
    public function unwrapOr(mixed $default)
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
    public function unwrapOr(mixed $default)
    {
        return $default;
    }
}
```

## 補足

PHPStanのAllowedSubtypesを使用することでPHPStanに`Result`のinterfaceを実装するクラスは`Ok`と`Err`の2つだけであることを伝えることができます。

https://phpstan.org/developing-extensions/allowed-subtypes


あと、[phpstan-sealed](https://github.com/phpstan/phpstan-src/pull/4095)がリリースされると、AllowedSubTypesClassReflectionExtensionの代わりに、以下の記載だけで済むみたいです！

（2025/11/09追記）リリースされました
https://github.com/phpstan/phpdoc-parser/releases/tag/2.2.0

```php
/** 
 *  @phpstan-sealed Ok|Err
 */
interface Result {
    // ...
}
```

# Result型で嬉しいところ

複数のエラーの可能性がある場合、`match`文などを使用することで、失敗時の対応の網羅チェックできます。
（PHPStanレベル5以上の場合）

```php
/**
 * @return Result<CompletedPayment, PaymentAmountRuleError|PaymentMethodNotRegistedError>
 */
function completePayment(ProcessingPayment $payment): Result {
    if ($payment->amount <= 0) {
        return new Err(new PaymentAmountRuleError("Payment amount must be positive"));
    }
    if ($payment->method == PaymentMethod::NotRegisterd) {
        return new Err(new PaymentMethodNotRegistedError("Payment amount must be positive"));
    }
    return new Ok(new CompletedPayment($payment->amount));
}
```

上記のコードの場合、`completePayment`関数は`PaymentAmountRuleError`と`PaymentMethodNotRegistedError`の失敗の可能性があります。

```php:呼び出し側
// 呼び出し側
$result = completePayment($payment);
if ($result->isErr()) {
    match (true) {
       $result->unwrapErr() instanceof PaymentAmountRuleError::class => // ... 支払金エラー時の処理
    }
}
// ... 支払い完了時の処理
```

しかし、`PaymentMethodNotRegistedError`がmatch文でチェックできていないので、PHPStanがエラーを出力してくれます。


※ Union型でも網羅チェックはできますが、失敗時と成功時を同列でチェックを行う必要があります。

# まとめ

PHPでResult型を実装する方法について説明しました。

この記事がPHPでResult型を実装する際の参考になれば幸いです。
