---
title: "PHPでもっとResult型やってみる"
emoji: "🐘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["php", "Result型"]
published: false
---

# はじめに

こんにちは。ひがきです。

PHPでResult型を実装するにあたり、より便利な関数の説明がしたくなったので、まとめていきます！！

:::message
この記事は[PHPでResult型やってみる](https://zenn.dev/higaki/articles/my-php-result-type)の続編です。
:::

# もっとResult型やってみる

::: details 元々のResult Interface
```php
/**
 * @template T
 * @template E
 */
interface Result
{
    public function isOk(): bool;
    public function isErr(): bool;
    /**
     * @return T
     */
    public function unwrap(): mixed;
    /**
     * @return E
     */
    public function unwrapErr(): mixed;
    /**
     * @template D
     * @param D $default
     * @return O|D
     */
    public function unwrapOr(mixed $default): mixed;

    /**
     * @template F
     * @param callable(O):F $fn
     * @return Result<F, E>
     */
    public function map(callable $fn): Result;

    /**
     * @template U
     * @template F
     * @param callable(O): Result<U, F> $fn
     * @return Result<U, F|E>
     */
    public function flatMap(callable $fn): Result;
}
```
:::

::: details 元々のOkクラス
```php
/**
 * @template T
 * @implements Result<T, never>
 */
final readonly class Ok implements Result
{
    /**
     * @param T $ok
     */
    public function __construct(
        private mixed $ok,
    ) {
    }

    /**
     * @return bool
     */
    public function isOk(): bool
    {
        return true;
    }

    /**
     * @return bool
     */
    public function isErr(): bool
    {
        return false;
    }

    /**
     * @return T
     */
    public function unwrap(): mixed
    {
        return $this->ok;
    }

    /**
     * @return never
     */
    public function unwrapErr(): never
    {
        throw new RuntimeException('called Result->unwrapErr() on an ok value');
    }

    /**
     * @template D
     * @param D $default
     * @return T
     */
    public function unwrapOr(mixed $default): mixed
    {
        return $this->ok;
    }

    /**
     * @template U
     * @param callable(T): U $fn
     * @return Result<U, never>
     */
    public function map(callable $fn): Result
    {
        return new self($fn($this->ok));
    }

    /**
     * @template U
     * @template F
     * @param callable(T): Result<U, F> $fn
     * @return Result<U, F>
     */
    public function flatMap(callable $fn): Result
    {
        return $fn($this->ok);
    }
}
```
:::


::: details 元々のErrクラス
```php
/**
 * @template E
 * @implements Result<never, E>
 */
final readonly class Err implements Result
{
    /**
     * @param E $err
     */
    public function __construct(
        private mixed $err,
    ) {
    }

    /**
     * @return bool
     */
    public function isOk(): bool
    {
        return false;
    }

    /**
     * @return bool
     */
    public function isErr(): bool
    {
        return true;
    }

    /**
     * @return never
     */
    public function unwrap(): never
    {
        throw new RuntimeException('called Result->unwrap() on an err value');
    }

    /**
     * @return E
     */
    public function unwrapErr(): mixed
    {
        return $this->err;
    }

    /**
     * @template D
     * @param D $default
     * @return D
     */
    public function unwrapOr(mixed $default): mixed
    {
        return $default;
    }

    /**
     * @return Result<never, E>
     */
    public function map(callable $fn): Result
    {
        return $this;
    }

    /**
     * @return Result<never, E>
     */
    public function flatMap(callable $fn): Result
    {
        return $this;
    }
}
```
:::


## 基本方針

# 補足

# まとめ

PHPでResult型を実装する方法について説明しました。

この記事がPHPでResult型を実装する際の参考になれば幸いです。
