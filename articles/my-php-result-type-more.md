---
title: "PHPでもっとResult型やってみる"
emoji: "🐘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["php", "Result型"]
published: false
---

# はじめに

:::message
この記事は[PHPでResult型やってみる](https://zenn.dev/higaki/articles/my-php-result-type)の続編です。
:::

こんにちは。ひがきです。

PHPでResult型を実装するにあたり、より便利な関数の説明がしたくなったので、まとめていきます！！

以下についてまとめていきます！（順次更新予定）

- map
- andThen(flatMap)

# もっとResult型やってみる

:::message
Rustのコードを参考にしております。
:::

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
     * @template U
     * @param callable(T):U $fn
     * @return Result<U, E>
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
        return new self($fn($this->value));
    }

    /**
     * @template U
     * @template F
     * @param callable(T): Result<U, F> $fn
     * @return Result<U, F>
     */
    public function flatMap(callable $fn): Result
    {
        return $fn($this->value);
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

> Maps a `Result<T, E>` to `Result<U, E>` by applying a function to a contained [`Ok`] value, leaving an [`Err`] value untouched.

と説明されています。

自分は以下のように解釈しました。

- Okの場合は、`T -> U`になる関数を適用して、`Result<T, E>`を`Result<U, E>`にする

- Errの場合は、何をしない



### Result Interface

```php
/**
 * @template T
 * @template E
 */
interface Result
{
    // ...

    /**
     * @template U
     * @param callable(T):U $fn
     * @return Result<U, E>
     */
    public function map(callable $fn): Result;
}
```

`map`の適用で `Result<T, E> -> Result<U, E>`に変化することがinterfaceでわかるかと思います。

### Ok

```php
final readonly class Ok implements Result
{
    // ...
    
    /**
     * @template U
     * @param callable(T): U $fn
     * @return Result<U, never>
     */
    public function map(callable $fn): Result
    {
        return new self($fn($this->value));
    }
}
```


### Err

```php
final readonly class Err implements Result
{
    // ...

    /**
     * @return Result<never, E>
     */
    public function map(callable $fn): Result
    {
        return $this;
    }
}
```

### 使い方

`map`は失敗可能性が**ない**関数をResultのOkのvalueに適用させたい時に使用します。

例）

```php:mapの使い方
$hoge = validateUserId($request['id'])
    ->map(fn(ValidUserId $id) => getUserIdValue($id));
    
\PHPStan\dumpType($hoge); // Result<string, InvalidUserIdException>
```

`getUserIdValue`は`ValidUserId`を渡すと必ず成功して`string`を返す関数です。

このような必ず成功する関数をResult型のOkのvalueに適用させたい時に`map`を使用します。

```php:mapの使い方で使用するサンプル
/**
 * @return Result<ValidUserId, InvalidUserIdException>
 */
function validateUserId(string $id): Result
{
    if (empty($id)) {
        return new Err(new InvalidUserIdException());
    }

    return new Ok(new ValidUserId());
}

class InvalidUserIdException {}

class ValidUserId
{
    public function __construct(public string $value = '') {}
}

function getUserIdValue(ValidUserId $userId): string
{
    return $userId->value;
}
```

## andThen(flatMap)の実装

and_thenとは、Rustの実装では

> Calls `op` if the result is [`Ok`], otherwise returns the [`Err`] value of `self`.

> pub fn and_then<U, F: FnOnce(T) -> Result<U, E>>(self, op: F) -> Result<U, E> 

と説明されています。

ここで`op` に着目すると、`T -> Result<U, E>`を返すものであることがわかります。

Rustでは`and_then`で受け取るErrの型は元々のエラーと同じ型でないといけなさそうです。

これは私の考えですが、PHPで実装する際にはエラーの変換が大変なので、異なるエラー型を受け取れるようにしてそれぞれの可能性を型として持つ方が良いと考えました。

また、入出力表は以下の通りです。

> | method       | self     | function input | function result | output   |
> |--------------|----------|----------------|-----------------|----------|
> | [`and_then`] | `Err(e)` | (not provided) | (not evaluated) | `Err(e)` |
> | [`and_then`] | `Ok(x)`  | `x`            | `Err(d)`        | `Err(d)` |
> | [`and_then`] | `Ok(x)`  | `x`            | `Ok(y)`         | `Ok(y)`  |


自分はPHPで実装する際には以下のようにするのが良いと考えました。

- Okの場合は、`T -> Result<U, F>`になる関数を適用して、`Result<T, E>`を`Result<U, E|F>`にする

- Errの場合は、何をしない



### Result Interface

```php
interface Result
{
    // ...

    /**
     * @template U
     * @template F
     * @param callable(T): Result<U, F> $fn
     * @return Result<U, F|E>
     */
    public function flatMap(callable $fn): Result;
}
```

`$fn`が`T`を受け取って`Result<U, F>`を返す関数`T -> Result<U, F>`であることを`callable(T): Result<U, F>`で示しました。

Rustでは`T -> Result<U, E>`になる関数を適用して、`Result<T, E>`を`Result<U, E>`にしていましたが、

自分は`T -> Result<U, F>`になる関数を適用して、`Result<T, E>`を`Result<U, E|F>`にしました。

### Ok

```php
final readonly class Ok implements Result
{
    // ...

    /**
     * @template U
     * @template F
     * @param callable(T): Result<U, F> $fn
     * @return Result<U, F>
     */
    public function flatMap(callable $fn): Result
    {
        return $fn($this->value);
    }
}
```


### Err

```php
final readonly class Err implements Result
{
    // ...

    /**
     * @return Result<never, E>
     */
    public function flatMap(callable $fn): Result
    {
        return $this;
    }
}
```

### 使い方

`andThen(flatMap)`は失敗可能性が**ある**関数をResultのOkのvalueに適用させたい時に使用します。

例）

```php:andThenの使い方
$hoge = validateUserId($request['id'])
    ->map(fn(ValidUserId $id) => getUserIdValue($id));
    
\PHPStan\dumpType($hoge); // Result<string, InvalidUserIdException>
```

# 補足

# まとめ

PHPでResult型を実装する方法について説明しました。

この記事がPHPでResult型を実装する際の参考になれば幸いです。
