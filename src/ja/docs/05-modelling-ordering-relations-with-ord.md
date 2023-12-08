# `ord` で順序関係をモデリングする (Modeling ordering relations with `Ord`)

前の `Eq` の章では **等価性** の概念を扱いました。この章では **順序付け** の概念を扱います。

全順序関係の概念は、TypeScript で次のように実装できます：

```ts
import { Eq } from 'fp-ts/lib/Eq'

type Ordering = -1 | 0 | 1

interface Ord<A> extends Eq<A> {
  readonly compare: (x: A, y: A) => Ordering
}
```

結果は以下のようになります：

- `x < y` のとき、かつそのときに限り `compare(x, y) = -1`
- `x = y` のとき、かつそのときに限り `compare(x, y) = 0`
- `x > y` のとき、かつそのときに限り `compare(x, y) = 1`

**例**

`number` 型において`Ord` インスタンスを定義してみましょう：

```ts
import { Ord } from 'fp-ts/Ord'

const OrdNumber: Ord<number> = {
  equals: (first, second) => first === second,
  compare: (first, second) => (first < second ? -1 : first > second ? 1 : 0)
}
```

次の法則が成り立たつ必要があります：

1. **反射律**: `A` の任意の元 `x` に対して `compare(x, x) <= 0`
2. **反対称律**: `A` の任意の元 `x`、`y` に対して `compare(x, y) <= 0` かつ `compare(y, x) <= 0` ならば `x = y`
3. **推移律**: `A` の任意の元 `x`、`y`、`z` に対して `compare(x, y) <= 0` かつ `compare(y, z) <= 0` ならば `compare(x, z) <= 0`

`compare` は `Eq` の `equals` 操作とも互換性を持たなければなりません。

つまり、`A` の任意の元 `x`、`y` に対して、`equals(x, y) === true` のとき、かつそのときに限り `compare(x, y) === 0` でなければなりません。

**注**. `equals`は次のようにして`compare`から導出できます：

```ts
equals: (first, second) => compare(first, second) === 0
```

実際、`fp-ts/Ord` モジュールは便利なヘルパー `fromCompare` を export しており、`compare` 関数を定義するだけで `Ord` インスタンスを定義できます：

```ts
import { Ord, fromCompare } from 'fp-ts/Ord'

const OrdNumber: Ord<number> = fromCompare((first, second) =>
  first < second ? -1 : first > second ? 1 : 0
)
```

**クイズ**. じゃんけんで、`move2` が `move1` に勝つ場合、`move1 <= move2` の `Ord` インスタンスを定義することは可能ですか？

`ReadonlyArray` の要素をソートする `sort` 関数を定義して、`Ord`インスタンスの実用的な使用例を見てみましょう。

```ts
import { pipe } from 'fp-ts/function'
import * as N from 'fp-ts/number'
import { Ord } from 'fp-ts/Ord'

export const sort = <A>(O: Ord<A>) => (
  as: ReadonlyArray<A>
): ReadonlyArray<A> => as.slice().sort(O.compare)

pipe([3, 1, 2], sort(N.Ord), console.log) // => [1, 2, 3]
```

**クイズ** (JavaScript). 上記の実装がネイティブの Array `slice`メソッドを利用しているのはなぜですか？

もう一つの `Ord` の実用的な使用例として、二つの値のうち最小の値を返す `min` 関数を定義してみましょう。

```ts
import { pipe } from 'fp-ts/function'
import * as N from 'fp-ts/number'
import { Ord } from 'fp-ts/Ord'

const min = <A>(O: Ord<A>) => (second: A) => (first: A): A =>
  O.compare(first, second) === 1 ? second : first

pipe(2, min(N.Ord)(1), console.log) // => 1
```

## デュアル順序付け

同様に、`concat` 操作を反転させて `reverse` コンビネータを使用して `dual semigroup` を得ることができるように、`compare` 操作を反転してデュアルな順序を得ることもできます。

`Ord` に対して `reverse` コンビネータを定義してみましょう：

```ts
import { pipe } from 'fp-ts/function'
import * as N from 'fp-ts/number'
import { fromCompare, Ord } from 'fp-ts/Ord'

export const reverse = <A>(O: Ord<A>): Ord<A> =>
  fromCompare((first, second) => O.compare(second, first))
```

`reverse` の使用例としては、`min` 関数から `max` 関数を得る操作が挙げられます：

```ts
import { flow, pipe } from 'fp-ts/function'
import * as N from 'fp-ts/number'
import { Ord, reverse } from 'fp-ts/Ord'

const min = <A>(O: Ord<A>) => (second: A) => (first: A): A =>
  O.compare(first, second) === 1 ? second : first

// const max: <A>(O: Ord<A>) => (second: A) => (first: A) => A
const max = flow(reverse, min)

pipe(2, max(N.Ord)(1), console.log) // => 2
```

順序付けの **完全性** （任意の `x` と `y` に対して、`x <= y` または `y <= z` の少なくとも一方が成立する必要があること）は、数値に関しては明らかに思えるかもしれませんが、必ずしもそのようなわけではありません。少し複雑なシナリオを見てみましょう：

```ts
type User = {
  readonly name: string
  readonly age: number
}
```

どのような場合に「ある `User` が、別の `User` 以下である」のか、明確ではありません。

`Ord<User>` インスタンスはどのように定義できるのでしょう？

場合によりますが、ありそうな例としては `User` を年齢で順序付けすることでしょう：

```ts
import * as N from 'fp-ts/number'
import { fromCompare, Ord } from 'fp-ts/Ord'

type User = {
  readonly name: string
  readonly age: number
}

const byAge: Ord<User> = fromCompare((first, second) =>
  N.Ord.compare(first.age, second.age)
)
```

この例においても、`contramap` コンビネータを使用してボイラープレートを取り除くことができます。`Ord<A>` インスタンスと `B` から `A` への関数が与えられると、`Ord<B>` を導出することが可能です：

```ts
import { pipe } from 'fp-ts/function'
import * as N from 'fp-ts/number'
import { contramap, Ord } from 'fp-ts/Ord'

type User = {
  readonly name: string
  readonly age: number
}

const byAge: Ord<User> = pipe(
  N.Ord,
  contramap((_: User) => _.age)
)
```

先ほど定義した `min` 関数によって、二人の `User` のうち最も若い `User` を得ることができます。

```ts
// const getYounger: (second: User) => (first: User) => User
const getYounger = min(byAge)

pipe(
  { name: 'Guido', age: 50 },
  getYounger({ name: 'Giulio', age: 47 }),
  console.log
) // => { name: 'Giulio', age: 47 }
```

**クイズ**. `fp-ts/ReadonlyMap` モジュールでは以下の API が提供されています

```ts
/**
 * Get a sorted `ReadonlyArray` of the keys contained in a `ReadonlyMap`.
 */
declare const keys: <K>(
  O: Ord<K>
) => <A>(m: ReadonlyMap<K, A>) => ReadonlyArray<K>
```

この API はなぜ `Ord<K>` インスタンスを要求するのでしょうか？

最初の問題に戻りましょう。`number` 以外の型に対して2つの半群 `SemigroupMin` と `SemigroupMax` を定義します：

```ts
import { Semigroup } from 'fp-ts/Semigroup'

const SemigroupMin: Semigroup<number> = {
  concat: (first, second) => Math.min(first, second)
}

const SemigroupMax: Semigroup<number> = {
  concat: (first, second) => Math.max(first, second)
}
```

`Ord` 抽象があるので、以下のように実装できます：

```ts
import { pipe } from 'fp-ts/function'
import * as N from 'fp-ts/number'
import { Ord, contramap } from 'fp-ts/Ord'
import { Semigroup } from 'fp-ts/Semigroup'

export const min = <A>(O: Ord<A>): Semigroup<A> => ({
  concat: (first, second) => (O.compare(first, second) === 1 ? second : first)
})

export const max = <A>(O: Ord<A>): Semigroup<A> => ({
  concat: (first, second) => (O.compare(first, second) === 1 ? first : second)
})

type User = {
  readonly name: string
  readonly age: number
}

const byAge: Ord<User> = pipe(
  N.Ord,
  contramap((_: User) => _.age)
)

console.log(
  min(byAge).concat({ name: 'Guido', age: 50 }, { name: 'Giulio', age: 47 })
) // => { name: 'Giulio', age: 47 }
console.log(
  max(byAge).concat({ name: 'Guido', age: 50 }, { name: 'Giulio', age: 47 })
) // => { name: 'Guido', age: 50 }
```

**例**

最後の例で、ここまでの内容をまとめてみましょう。
（出典： [Fantas, Eel, and Specification 4: Semigroup](http://www.tomharding.me/2017/03/13/fantas-eel-and-specification-4/)）

システムを構築することを想定します。データベースに顧客のレコードが以下のように実装されているとします：

```ts
interface Customer {
  readonly name: string
  readonly favouriteThings: ReadonlyArray<string>
  readonly registeredAt: number // エポック秒
  readonly lastUpdatedAt: number // エポック秒
  readonly hasMadePurchase: boolean
}
```

何らかの理由で、同じ人のレコードが重複してしまったとしましょう。

どのようにマージするのかを決めなければなりません。こういうときには半群が持ってこいです！

```ts
import * as B from 'fp-ts/boolean'
import { pipe } from 'fp-ts/function'
import * as N from 'fp-ts/number'
import { contramap } from 'fp-ts/Ord'
import * as RA from 'fp-ts/ReadonlyArray'
import { max, min, Semigroup, struct } from 'fp-ts/Semigroup'
import * as S from 'fp-ts/string'

interface Customer {
  readonly name: string
  readonly favouriteThings: ReadonlyArray<string>
  readonly registeredAt: number // エポック秒
  readonly lastUpdatedAt: number // エポック秒
  readonly hasMadePurchase: boolean
}

const SemigroupCustomer: Semigroup<Customer> = struct({
  // 長い方の名前を保持する
  name: max(pipe(N.Ord, contramap(S.size))),
  // accumulate things
  favouriteThings: RA.getSemigroup<string>(),
  // 最も古いデータを保持する
  registeredAt: min(N.Ord),
  // 最も新しいデータを保持する
  lastUpdatedAt: max(N.Ord),
  // 論理和に基づく boolean 半群
  hasMadePurchase: B.SemigroupAny
})

console.log(
  SemigroupCustomer.concat(
    {
      name: 'Giulio',
      favouriteThings: ['math', 'climbing'],
      registeredAt: new Date(2018, 1, 20).getTime(),
      lastUpdatedAt: new Date(2018, 2, 18).getTime(),
      hasMadePurchase: false
    },
    {
      name: 'Giulio Canti',
      favouriteThings: ['functional programming'],
      registeredAt: new Date(2018, 1, 22).getTime(),
      lastUpdatedAt: new Date(2018, 2, 9).getTime(),
      hasMadePurchase: true
    }
  )
)
/*
{ name: 'Giulio Canti',
  favouriteThings: [ 'math', 'climbing', 'functional programming' ],
  registeredAt: 1519081200000, // new Date(2018, 1, 20).getTime()
  lastUpdatedAt: 1521327600000, // new Date(2018, 2, 18).getTime()
  hasMadePurchase: true
}
*/
```

**クイズ**. 型 `A` が与えられた場合、`Semigroup<Ord<A>>` インスタンスを定義することは可能ですか？それは何を表していると考えられますか？

**デモ**
