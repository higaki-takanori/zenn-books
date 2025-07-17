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

    /**
     * @template F
     * @param callable(T):F $fn
     * @return Result<F, E>
     */
    public function map(callable $fn): Result;

    /**
     * @template U
     * @template F
     * @param callable(T): Result<U, F> $fn
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
    public function unwrap(): mixed
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
    public function unwrapOr(mixed $default): mixed
    {
        return $this->value;
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
    public function unwrapErr(): mixed
    {
        return $this->value;
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


## map実装

mapとは、Rustの実装では

> /// Maps a `Result<T, E>` to `Result<U, E>` by applying a function to a contained [`Ok`] value, leaving an [`Err`] value untouched.

と説明されています。

自分は以下のように解釈しました。

- Okの場合は、`T -> U`になる関数を適用して、`Result<T, E>`を`Result<U, E>`にする

- Errの場合は、何をしない



### Result Interface

### Ok

### Err

# 補足

# まとめ

PHPでResult型を実装する方法について説明しました。

この記事がPHPでResult型を実装する際の参考になれば幸いです。
