# semigroup-commutative

## 問題

交換法則に満たす`concat`の例とそうでない例を見つけられますか？

## 答え

### 交換法則に満たす

```ts
import { Semigroup } from 'fp-ts/Semigroup'

const SemigroupSum: Semigroup<number> = {
  concat: (first, second) => first + second
}
```

加算は交換法則を満たすため、`concat(a, b) = a + b = b + a = concat(b, a)` が成り立ちます。

### 交換法則に満たさない

```ts
import { Semigroup } from 'fp-ts/Semigroup'

const first = <A>(): Semigroup<A> => ({
  concat: (first, _second) => first
})
```

`concat(a, b) = a != concat(b, a)`
