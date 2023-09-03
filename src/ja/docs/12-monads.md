# モナド

<center>
<img src="../../images/moggi.jpg" width="300" alt="Eugenio Moggi" />

(Eugenio Moggi は、イタリア・ジェノヴァ大学の計算機科学の教授である。彼は、構造化プログラミングにおいて広くモナドを用いることを提言した最初の人物である)

<img src="../../images/wadler.jpg" width="300" alt="Philip Lee Wadler" />

(Philip Lee Wadler はアメリカの計算機科学者であり、プログラミング言語や型理論における功績で知られている)

</center>

前章では、`F` がアプリカティブ関手インスタンスであるならば、作用プログラム `f: (a: A) => F<B>` と、`n` 項の純粋プログラム `g` を合成できるということを見てきました：

| プログラム f | プログラム g     | 合成     |
| --------- | ------------- | --------------- |
| 純粋      | 純粋          | `g ∘ f`         |
| 作用 | 純粋 (単項)  | `map(g) ∘ f`    |
| 作用 | 純粋, `n` 項 | `liftAn(g) ∘ f` |

最後に、非常に一般的なケースである、**両方とも** 作用プログラムである場合を解決しなければなりません：

```ts
f: (a: A) => F<B>
g: (b: B) => F<C>
```

`f` と `g` の合成はどうなるのでしょうか？

## 入れ子問題

どういった状況が未解決に当たるのか、例を見ておきます。

**例** (`F = Array`)

フォロワーのフォロワーを取得したい場合を考えます。

```ts
import { pipe } from 'fp-ts/function'
import * as A from 'fp-ts/ReadonlyArray'

interface User {
  readonly id: number
  readonly name: string
  readonly followers: ReadonlyArray<User>
}

const getFollowers = (user: User): ReadonlyArray<User> => user.followers

declare const user: User

// followersOfFollowers: ReadonlyArray<ReadonlyArray<User>>
const followersOfFollowers = pipe(user, getFollowers, A.map(getFollowers))
```

これはちょっとおかしなことになっています。`followersOfFollowers` の型は `ReadonlyArray<ReadonlyArray<User>>` ですが、本来は `ReadonlyArray<User>` であって欲しいところです。

入れ子になった配列を **平坦化** する必要があります。

The function `flatten: <A>(mma: ReadonlyArray<ReadonlyArray<A>>) => ReadonlyArray<A>` exported by the `fp-ts/ReadonlyArray` is exactly what we need:

```ts
// followersOfFollowers: ReadonlyArray<User>
const followersOfFollowers = pipe(
  user,
  getFollowers,
  A.map(getFollowers),
  A.flatten
)
```

いいですね！ 他の型も見てみましょう。

**例** (`F = Option`)
数列の最初の要素の逆数を求めたい場合を考えます。
```ts
import { pipe } from 'fp-ts/function'
import * as O from 'fp-ts/Option'
import * as A from 'fp-ts/ReadonlyArray'

const inverse = (n: number): O.Option<number> =>
  n === 0 ? O.none : O.some(1 / n)

// inverseHead: O.Option<O.Option<number>>
const inverseHead = pipe([1, 2, 3], A.head, O.map(inverse))
```

おっと、またです。`inverseHead` の型は `Option<Option<number>>` ですが、`Option<number>` であって欲しいです。

入れ子になった `Option` をまた平坦化する必要があります。

`fp-ts/Option` モジュールが expose している関数 `flatten: <A>(mma: Option<Option<A>>) => Option<A>` が使えます：

```ts
// inverseHead: O.Option<number>
const inverseHead = pipe([1, 2, 3], A.head, O.map(inverse), O.flatten)
```

このような関数 `flatten` は偶然現れた訳ではありません。背後に、関数型のパターンが存在します。型コンストラクタ `ReadonlyArray` と `Option` （他にも多くありますが）は、**モナドインスタンス** であり、

> `flatten` はモナドの最も特徴的な操作です。

**注意**. `flatten` は一般的に **join** とも呼ばれます。

では、モナドとは何なのでしょうか？

以下、よくあるな説明を示します……

## モナドの定義

**定義**. モナドは3つの制約があります：

(1) 型コンストラクタ `M` は関手インスタンスであること

(2) 以下のシグネチャを持つ関数 `of` (**pure** や **return** とも呼ばれる) が定義されていること:

```ts
of: <A>(a: A) => M<A>
```

(3) 以下のシグネチャを持つ関数 `chain` (**flatMap** や **bind** とも呼ばれる) が定義されていること:

```ts
chain: <A, B>(f: (a: A) => M<B>) => (ma: M<A>) => M<B>
```

関数 `of` と `chain` は以下の制約を満たす必要があります：

- `chain(of) ∘ f = f` (**左単位元**)
- `chain(f) ∘ of = f` (**右単位元**)
- `chain(h) ∘ (chain(g) ∘ f) = chain((chain(h) ∘ g)) ∘ f` (**結合律**)

ただし、`f` `g` `h` はすべて作用プログラムであって、`*` は普通の関数合成を表します。

私が初めてこの定義を見たとき、多くの疑問が浮かびました：

- なぜ必要とされるのは他でもない `of` と `chain` なのか？ そして、なぜこれらはこういったシグネチャを持たねばならないのか？
- なぜ "pure" や "flatMap" のような同義語が存在するのか？
- なぜこうした制約が課されているのか？ これらは何を意味しているのか？
- `flatten` はモナドにとって重要であるのに、なぜその定義の中に現れないのか？

この章では、これらの問いすべてに対する答えを探していきます。

話を元の問いに戻しましょう： 2つの作用関数 `f` `g` はどうやって合成するのでしょうか？

<img src="../../images/kleisli_arrows.png" alt="two Kleisli arrows, what's their composition?" width="450px" />

<center>(two Kleisli Arrows)</center>

**注意**. 作用関数は **Kleisli arrow** とも呼ばれます。

For the time being I don't even know the **type** of such composition.

But we've already seen some abstractions that talks specifically about composition. Do you remember what we said about categories?

> Categories capture the essence of composition

We can transform our problem into a category problem, meaning: can we find a category that models the composition of Kleisli arrows?

## The Kleisli category

<center>
<img src="images/kleisli.jpg" width="300" alt="Heinrich Kleisli" />

(Heinrich Kleisli, Swiss mathematician)

</center>

Let's try building a category _K_ (called **Kleisli category**) which contains _only_ Kleisli arrows:

- **objects** will be the same objects of the _TS_ category, so all TypeScript types.
- **morphisms** are built like this: every time there is a Kleisli arrow `f: A ⟼ M<B>` in _TS_ we draw an arrow `f': A ⟼ B` in _K_

<center>
<img src="images/kleisli_category.png" alt="above the TS category, below the K construction" width="400px" />

(above the composition in the _TS_ category, below the composition in the _K_ construction)

</center>

So what would be the composition of `f` and `g` in _K_?
It's th red arrow called `h'` in the image below:

<center>
<img src="images/kleisli_composition.png" alt="above the composition in the TS category, below the composition in the K construction" width="400px" />

(above the composition in the _TS_ category, below the composition in the _K_ construction)

</center>

Given that `h'` is an arrow from `A` to `C` in `K`, we can find a corresponding function `h` from `A` to `M<C>` in `TS`.

Thus, a good candidate for the following composition of `f` and `g` in _TS_ is still a Kleisli arrow with the following signature: `(a: A) => M<C>`.

Let's try implementing such a function.

## Defining `chain` step by step

The first point (1) of the monad definition tells us that `M` admits a functor instance, thus we can use the `map` function to transform the function `g: (b: B) => M<C>` into a function `map(g): (mb: M<B>) => M<M<C>>`

<center>
<img src="images/flatMap.png" alt="where chain comes from" width="450px" />

(how to obtain the `h` function)

</center>

We're stuck now though: there is no legal operation for the functor instance that allows us to flatten a value of type `M<M<C>>` into a value of type `M<C>`, we need an additional operation, let's call it `flatten`.

If we can define such operation then we can find the composition we were looking for:

```
h = flatten ∘ map(g) ∘ f
```

By joining the `flatten ∘ map(g)` names we get "flatMap", hence the name!

Thus we can get `chain` in this way

```
chain = flatten ∘ map(g)
```

<center>
<img src="images/chain.png" alt="how chain operates on the function g" width="400px" />

(how `chain` operates on the function `g`)

</center>

Now we can update our composition table

| Program f | Program g     | Composition     |
| --------- | ------------- | --------------- |
| pure      | pure          | `g ∘ f`         |
| effectful | pure (unary)  | `map(g) ∘ f`    |
| effectful | pure, `n`-ary | `liftAn(g) ∘ f` |
| effectful | effectful     | `chain(g) ∘ f`  |

What about `of`? Well, `of` comes from the identity morphisms in _K_: for every identity morphism 1<sub>A</sub> in _K_ there has to be a corresponding function from `A` to `M<A>` (that is, `of: <A>(a: A) => M<A>`).

<center>
<img src="images/of.png" alt="where of comes from" width="300px" />

(where `of` comes from)

</center>

The fact that `of` is the neutral element for `chain` allows this kind of flux control (pretty common):

```ts
pipe(
  mb,
  M.chain((b) => (predicate(b) ? M.of(b) : g(b)))
)
```

where `predicate: (b: B) => boolean`, `mb: M<B>` and `g: (b: B) => M<B>`.

Last question: where do the laws come from? They are nothing else but the categorical laws in _K_ translated to _TS_:

| Law            | _K_                               | _TS_                                                    |
| -------------- | --------------------------------- | ------------------------------------------------------- |
| Left identity  | 1<sub>B</sub> ∘ `f'` = `f'`       | `chain(of) ∘ f = f`                                     |
| Right identity | `f'` ∘ 1<sub>A</sub> = `f'`       | `chain(f) ∘ of = f`                                     |
| Associativity  | `h' ∘ (g' ∘ f') = (h' ∘ g') ∘ f'` | `chain(h) ∘ (chain(g) ∘ f) = chain((chain(h) ∘ g)) ∘ f` |

If we now go back to the examples that showed the problem with nested contexts we can solve them using `chain`:

```ts
import { pipe } from 'fp-ts/function'
import * as O from 'fp-ts/Option'
import * as A from 'fp-ts/ReadonlyArray'

interface User {
  readonly id: number
  readonly name: string
  readonly followers: ReadonlyArray<User>
}

const getFollowers = (user: User): ReadonlyArray<User> => user.followers

declare const user: User

const followersOfFollowers: ReadonlyArray<User> = pipe(
  user,
  getFollowers,
  A.chain(getFollowers)
)

const inverse = (n: number): O.Option<number> =>
  n === 0 ? O.none : O.some(1 / n)

const inverseHead: O.Option<number> = pipe([1, 2, 3], A.head, O.chain(inverse))
```

Let's see how `chain` is implemented for the usual type constructors we've already seen:

**Example** (`F = ReadonlyArray`)

```ts
// transforms functions `B -> ReadonlyArray<C>` into functions `ReadonlyArray<B> -> ReadonlyArray<C>`
const chain = <B, C>(g: (b: B) => ReadonlyArray<C>) => (
  mb: ReadonlyArray<B>
): ReadonlyArray<C> => {
  const out: Array<C> = []
  for (const b of mb) {
    out.push(...g(b))
  }
  return out
}
```

**Example** (`F = Option`)

```ts
import { match, none, Option } from 'fp-ts/Option'

// transforms functions `B -> Option<C>` into functions `Option<B> -> Option<C>`
const chain = <B, C>(g: (b: B) => Option<C>): ((mb: Option<B>) => Option<C>) =>
  match(() => none, g)
```

**Example** (`F = IO`)

```ts
import { IO } from 'fp-ts/IO'

// transforms functions `B -> IO<C>` into functions `IO<B> -> IO<C>`
const chain = <B, C>(g: (b: B) => IO<C>) => (mb: IO<B>): IO<C> => () =>
  g(mb())()
```

**Example** (`F = Task`)

```ts
import { Task } from 'fp-ts/Task'

// transforms functions `B -> Task<C>` into functions `Task<B> -> Task<C>`
const chain = <B, C>(g: (b: B) => Task<C>) => (mb: Task<B>): Task<C> => () =>
  mb().then((b) => g(b)())
```

**Example** (`F = Reader`)

```ts
import { Reader } from 'fp-ts/Reader'

// transforms functions `B -> Reader<R, C>` into functions `Reader<R, B> -> Reader<R, C>`
const chain = <B, R, C>(g: (b: B) => Reader<R, C>) => (
  mb: Reader<R, B>
): Reader<R, C> => (r) => g(mb(r))(r)
```

## Manipulating programs

Let's see now, how thanks to referential transparency and the monad concept we can programmaticaly manipulate programs.

Here's a small program that reads / writes a file:

```ts
import { log } from 'fp-ts/Console'
import { IO, chain } from 'fp-ts/IO'
import { pipe } from 'fp-ts/function'
import * as fs from 'fs'

// -----------------------------------------
// library functions
// -----------------------------------------

const readFile = (filename: string): IO<string> => () =>
  fs.readFileSync(filename, 'utf-8')

const writeFile = (filename: string, data: string): IO<void> => () =>
  fs.writeFileSync(filename, data, { encoding: 'utf-8' })

// API derived from the previous functions
const modifyFile = (filename: string, f: (s: string) => string): IO<void> =>
  pipe(
    readFile(filename),
    chain((s) => writeFile(filename, f(s)))
  )

// -----------------------------------------
// program
// -----------------------------------------

const program1 = pipe(
  readFile('file.txt'),
  chain(log),
  chain(() => modifyFile('file.txt', (s) => s + '\n// eof')),
  chain(() => readFile('file.txt')),
  chain(log)
)
```

The actions:

```ts
pipe(readFile('file.txt'), chain(log))
```

is repeated more than once in the program, but given that referential transparency holds we can factor it and assign it to a constant:

```ts
const read = pipe(readFile('file.txt'), chain(log))
const modify = modifyFile('file.txt', (s) => s + '\n// eof')

const program2 = pipe(
  read,
  chain(() => modify),
  chain(() => read)
)
```

We can even define a combinator and leverage it to make the code more compact:

```ts
const interleave = <A, B>(action: IO<A>, middle: IO<B>): IO<A> =>
  pipe(
    action,
    chain(() => middle),
    chain(() => action)
  )

const program3 = interleave(read, modify)
```

Another example: implementing a function similar to Unix' `time` (the part related to the execution time) for `IO`.

```ts
import * as IO from 'fp-ts/IO'
import { now } from 'fp-ts/Date'
import { log } from 'fp-ts/Console'
import { pipe } from 'fp-ts/function'

// logs the computation lenght in milliseconds
export const time = <A>(ma: IO.IO<A>): IO.IO<A> =>
  pipe(
    now,
    IO.chain((startMillis) =>
      pipe(
        ma,
        IO.chain((a) =>
          pipe(
            now,
            IO.chain((endMillis) =>
              pipe(
                log(`Elapsed: ${endMillis - startMillis}`),
                IO.map(() => a)
              )
            )
          )
        )
      )
    )
  )
```

**Digression**. As you can notice, using `chain` when it is required to maintain a scope leads to verbose code.
In languages that support monadic style natively there is often syntax support that goes by the name of "do notation" which eases this kind of situations.

Let's see a Haskell example

```haskell
now :: IO Int
now = undefined -- `undefined` in Haskell is equivalent to TypeScript's declare

log :: String -> IO ()
log = undefined

time :: IO a -> IO a
time ma = do
  startMillis <- now
  a <- ma
  endMillis <- now
  log ("Elapsed:" ++ show (endMillis - startMillis))
  return a
```

TypeScript does not support such syntax, but it can be emulated with something similar:

```ts
import { log } from 'fp-ts/Console'
import { now } from 'fp-ts/Date'
import { pipe } from 'fp-ts/function'
import * as IO from 'fp-ts/IO'

// logs the computation lenght in milliseconds
export const time = <A>(ma: IO.IO<A>): IO.IO<A> =>
  pipe(
    IO.Do,
    IO.bind('startMillis', () => now),
    IO.bind('a', () => ma),
    IO.bind('endMillis', () => now),
    IO.chainFirst(({ endMillis, startMillis }) =>
      log(`Elapsed: ${endMillis - startMillis}`)
    ),
    IO.map(({ a }) => a)
  )
```

Let's see a usage example of the `time` combinator:

```ts
import { randomInt } from 'fp-ts/Random'
import { Monoid, concatAll } from 'fp-ts/Monoid'
import { replicate } from 'fp-ts/ReadonlyArray'

const fib = (n: number): number => (n <= 1 ? 1 : fib(n - 1) + fib(n - 2))

// launches `fib` with a random integer between 30 and 35
// logging both the input and output
const randomFib: IO.IO<void> = pipe(
  randomInt(30, 35),
  IO.chain((n) => log([n, fib(n)]))
)

// a monoid instance for `IO<void>`
const MonoidIO: Monoid<IO.IO<void>> = {
  concat: (first, second) => () => {
    first()
    second()
  },
  empty: IO.of(undefined)
}

// executes `n` times the `mv` computation
const replicateIO = (n: number, mv: IO.IO<void>): IO.IO<void> =>
  concatAll(MonoidIO)(replicate(n, mv))

// -------------------
// usage example
// -------------------

time(replicateIO(3, randomFib))()
/*
[ 31, 2178309 ]
[ 33, 5702887 ]
[ 30, 1346269 ]
Elapsed: 89
*/
```

Logs also the partial:

```ts
time(replicateIO(3, time(randomFib)))()
/*
[ 33, 5702887 ]
Elapsed: 54
[ 30, 1346269 ]
Elapsed: 13
[ 32, 3524578 ]
Elapsed: 39
Elapsed: 106
*/
```

One of the most interesting aspects of working with the monadic interface (`map`, `of`, `chain`) is the possibility to inject dependencies which the program needs, including the **way of concatenating different computations**.

To see that, let's refactor the small program that reads and writes a file:

```ts
import { IO } from 'fp-ts/IO'
import { pipe } from 'fp-ts/function'

// -----------------------------------------
// Deps interface, what we would call a "port" in the Hexagonal Architecture
// -----------------------------------------

interface Deps {
  readonly readFile: (filename: string) => IO<string>
  readonly writeFile: (filename: string, data: string) => IO<void>
  readonly log: <A>(a: A) => IO<void>
  readonly chain: <A, B>(f: (a: A) => IO<B>) => (ma: IO<A>) => IO<B>
}

// -----------------------------------------
// program
// -----------------------------------------

const program4 = (D: Deps) => {
  const modifyFile = (filename: string, f: (s: string) => string) =>
    pipe(
      D.readFile(filename),
      D.chain((s) => D.writeFile(filename, f(s)))
    )

  return pipe(
    D.readFile('file.txt'),
    D.chain(D.log),
    D.chain(() => modifyFile('file.txt', (s) => s + '\n// eof')),
    D.chain(() => D.readFile('file.txt')),
    D.chain(D.log)
  )
}

// -----------------------------------------
// a `Deps` instance, what we would call an "adapter" in the Hexagonal Architecture
// -----------------------------------------

import * as fs from 'fs'
import { log } from 'fp-ts/Console'
import { chain } from 'fp-ts/IO'

const DepsSync: Deps = {
  readFile: (filename) => () => fs.readFileSync(filename, 'utf-8'),
  writeFile: (filename: string, data: string) => () =>
    fs.writeFileSync(filename, data, { encoding: 'utf-8' }),
  log,
  chain
}

// dependency injection
program4(DepsSync)()
```

There's more, we can even abstract the effect in which the program runs. We can define our own `FileSystem` effect (the effect representing read-write operations over the file system):

```ts
import { IO } from 'fp-ts/IO'
import { pipe } from 'fp-ts/function'

// -----------------------------------------
// our program's effect
// -----------------------------------------

interface FileSystem<A> extends IO<A> {}

// -----------------------------------------
// dependencies
// -----------------------------------------

interface Deps {
  readonly readFile: (filename: string) => FileSystem<string>
  readonly writeFile: (filename: string, data: string) => FileSystem<void>
  readonly log: <A>(a: A) => FileSystem<void>
  readonly chain: <A, B>(
    f: (a: A) => FileSystem<B>
  ) => (ma: FileSystem<A>) => FileSystem<B>
}

// -----------------------------------------
// program
// -----------------------------------------

const program4 = (D: Deps) => {
  const modifyFile = (filename: string, f: (s: string) => string) =>
    pipe(
      D.readFile(filename),
      D.chain((s) => D.writeFile(filename, f(s)))
    )

  return pipe(
    D.readFile('file.txt'),
    D.chain(D.log),
    D.chain(() => modifyFile('file.txt', (s) => s + '\n// eof')),
    D.chain(() => D.readFile('file.txt')),
    D.chain(D.log)
  )
}
```

With a simple change in the definition of the `FileSystem` effect. we can modify the program to make it run asynchronously

```diff
// -----------------------------------------
// our program's effect
// -----------------------------------------

-interface FileSystem<A> extends IO<A> {}
+interface FileSystem<A> extends Task<A> {}
```

now all there's left is to modify the `Deps` instance to adapt to the new definition.

```ts
import { Task } from 'fp-ts/Task'
import { pipe } from 'fp-ts/function'

// -----------------------------------------
// our program's effect (modified)
// -----------------------------------------

interface FileSystem<A> extends Task<A> {}

// -----------------------------------------
// dependencies (NOT modified)
// -----------------------------------------

interface Deps {
  readonly readFile: (filename: string) => FileSystem<string>
  readonly writeFile: (filename: string, data: string) => FileSystem<void>
  readonly log: <A>(a: A) => FileSystem<void>
  readonly chain: <A, B>(
    f: (a: A) => FileSystem<B>
  ) => (ma: FileSystem<A>) => FileSystem<B>
}

// -----------------------------------------
// program (NOT modified)
// -----------------------------------------

const program5 = (D: Deps) => {
  const modifyFile = (filename: string, f: (s: string) => string) =>
    pipe(
      D.readFile(filename),
      D.chain((s) => D.writeFile(filename, f(s)))
    )

  return pipe(
    D.readFile('file.txt'),
    D.chain(D.log),
    D.chain(() => modifyFile('file.txt', (s) => s + '\n// eof')),
    D.chain(() => D.readFile('file.txt')),
    D.chain(D.log)
  )
}

// -----------------------------------------
// a `Deps` instance (modified)
// -----------------------------------------

import * as fs from 'fs'
import { log } from 'fp-ts/Console'
import { chain, fromIO } from 'fp-ts/Task'

const DepsAsync: Deps = {
  readFile: (filename) => () =>
    new Promise((resolve) =>
      fs.readFile(filename, { encoding: 'utf-8' }, (_, s) => resolve(s))
    ),
  writeFile: (filename: string, data: string) => () =>
    new Promise((resolve) => fs.writeFile(filename, data, () => resolve())),
  log: (a) => fromIO(log(a)),
  chain
}

// dependency injection
program5(DepsAsync)()
```

**Quiz**. The previous examples overlook, on purpose, possible errors. Example give: the file we're operating on may not exist at all. How could we modify the `FileSystem` effect to take this into account?

```ts
import { Task } from 'fp-ts/Task'
import { pipe } from 'fp-ts/function'
import * as E from 'fp-ts/Either'

// -----------------------------------------
// our program's effect (modified)
// -----------------------------------------

interface FileSystem<A> extends Task<E.Either<Error, A>> {}

// -----------------------------------------
// dependencies (NOT modified)
// -----------------------------------------

interface Deps {
  readonly readFile: (filename: string) => FileSystem<string>
  readonly writeFile: (filename: string, data: string) => FileSystem<void>
  readonly log: <A>(a: A) => FileSystem<void>
  readonly chain: <A, B>(
    f: (a: A) => FileSystem<B>
  ) => (ma: FileSystem<A>) => FileSystem<B>
}

// -----------------------------------------
// program (NOT modified)
// -----------------------------------------

const program5 = (D: Deps) => {
  const modifyFile = (filename: string, f: (s: string) => string) =>
    pipe(
      D.readFile(filename),
      D.chain((s) => D.writeFile(filename, f(s)))
    )

  return pipe(
    D.readFile('-.txt'),
    D.chain(D.log),
    D.chain(() => modifyFile('file.txt', (s) => s + '\n// eof')),
    D.chain(() => D.readFile('file.txt')),
    D.chain(D.log)
  )
}

// -----------------------------------------
// `Deps` instance (modified)
// -----------------------------------------

import * as fs from 'fs'
import { log } from 'fp-ts/Console'
import { chain, fromIO } from 'fp-ts/TaskEither'

const DepsAsync: Deps = {
  readFile: (filename) => () =>
    new Promise((resolve) =>
      fs.readFile(filename, { encoding: 'utf-8' }, (err, s) => {
        if (err !== null) {
          resolve(E.left(err))
        } else {
          resolve(E.right(s))
        }
      })
    ),
  writeFile: (filename: string, data: string) => () =>
    new Promise((resolve) =>
      fs.writeFile(filename, data, (err) => {
        if (err !== null) {
          resolve(E.left(err))
        } else {
          resolve(E.right(undefined))
        }
      })
    ),
  log: (a) => fromIO(log(a)),
  chain
}

// dependency injection
program5(DepsAsync)().then(console.log)
```

**Demo**

[`06_game.ts`](src/06_game.ts)
