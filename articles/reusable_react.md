---
title: "Reducer と Hooks による React 再利用"
emoji: "⚛"
type: "tech"
topics: ["React", "TypeScript"]
published: true
---

## はじめに

React は UI を**コンポーネント**と呼ばれる部品を利用して構築するためのライブラリです.
コンポーネントはそれ自体を組み合わせて他のコンポーネントを作り上げたり, UI 全体の統一感を保ったりするために, **再利用**することが重要な要素としてちょこちょこ取り上げられます.

本記事では React のコンポーネントについて, Reducer と Hooks でいい感じに再利用できない? ということを語ります.

## 始まりのコンポーネント

> React コンポーネントとは、マークアップを添えることができる JavaScript 関数です。[^1]

いつの間にか Class に関する記述は消え去り, 現在ではもう「関数です」と明言されています.
個人の視点で述べるならコンポーネントは「状態」と「処理」と「View」を合成したオブジェクトです.

React を始めて, 初めて目にするコンポーネントといえばカウンターでしょう.
`create-vite`などで React を選択した場合にも, 最初に生成されるコンポーネントもカウンターです.

この原初のコンポーネントから再利用性を考えていきます.

```tsx
export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div className="counter">
      <p>{count}</p>
      <button onClick={() => setCount((count) => count + 1)}>+</button>
      <button onClick={() => setCount((count) => Math.max(count - 1, 0))}>
        -
      </button>
    </div>
  );
}
```

## 「コンポーネント」を再利用する

**_Mission 1_: 🐥 と 🐔 を数えるカウンターを設置したページを作成せよ!!!**

とても容易なミッションです.
`Counter`コンポーネントは独立 / カプセル化されたコンポーネントとしてそのまま利用できます.

```tsx
export function App() {
  return (
    <div>
      <h3>🐥Hiyoko🐥 Counter</h3>
      <Counter />

      <h3>🐔Chicken🐔 Counter</h3>
      <Counter />
    </div>
  );
}
```

### **カプセル化されたコンポーネント**の再利用性

コンポーネントは**特定のコンテキストに閉じている状態にすべき**という考え方があります. `Counter`コンポーネントの場合, 「数を増減する操作が行う」みたいなコンテキストに閉じています.
状態はコンポーネント内に完全に隠蔽され, そのコンポーネント自体が状態を管理し, 外部からの制御の介入を完全に防いでいます.

#### 利点

このようなコンポーネントはレイアウト的な部分を除けば**外部への影響力を持ちません**.
そのため, 表示上問題が発生しないのであれば任意の箇所に設置でき, 他の UI への移動なども容易です.

#### 欠点

こういったコンポーネントは**外部から制御 / 参照できません.** 例えばカウンターの値を外部から参照して他の表示に利用するようなことは行えません.
これは, 複数のコンポーネントを組み合わせて UI を作る React では, **他のコンポーネントとの連携を行えない**ということを意味します.

単独で独立したコンポーネントとしての利用しか無いコンポーネントであれば, 特に欠点はありません.

## 「View」を再利用する

**_Mission 2_: 🐥 と 🐔 の合計値を表示せよ!!!**

最初の`Counter`コンポーネントはそれ単体での用途には向いていますが, カプセル化されており, 他のコンポーネントとの連携には向いていません.

そこで外部から「状態」と「状態を変更する関数」を受け取るように`Counter`を修正します.

```tsx
export function Counter({ count, increment, decrement }: Props) {
  return (
    <div>
      <p>{count}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  );
}
```

このコンポーネントは, 引数が同一であれば同一の物を常に返却する, 見た目部分のみを制御する純粋関数となります.

この形態に到達すると, 同様の表示が必要になるあらゆる箇所でコンポーネントを再利用できるようになります.

```tsx
export function App() {
  const [hiyokoCount, setHiyokoCount] = useState(0);
  const [chickenCount, setChickenCount] = useState(0);

  return (
    <div>
      <h3>Total</h3>
      <p>{hiyokoCount + chickenCount}</p>

      <h3>🐥Hiyoko🐥 Counter</h3>
      <Counter
        count={hiyokoCount}
        increment={() => setHiyokoCount(hiyokoCount + 1)}
        decrement={() => setHiyokoCount(Math.max(hiyokoCount - 1, 0))}
      />

      <h3>🐔Chicken🐔 Counter</h3>
      <Counter
        count={chickenCount}
        increment={() => setChickenCount(chickenCount + 1)}
        decrement={() => setChickenCount(Math.max(chickenCount - 1, 0))}
      />
    </div>
  );
}
```

### **純粋関数コンポーネント**の再利用性

純粋関数となったコンポーネントは, **純然たる View としての役割のみを持ち, それ以外のその一切の役割を持ちません**.

React の公式にもある通り, この手のコンポーネントはバグの発生も少なければメンテナンス性も高くなります.[^2]

> コンポーネントを常に厳密に純関数として書くことで、コードベースが成長するにつれて起きがちな、あらゆる種類の不可解なバグ、予測不可能な挙動を回避することができます。

#### 利点

View のみを制御する形態のため, **同一の View を利用したい任意の箇所で再利用できます**.
表示を変更する場合以外の修正が行われることはなく, SOLID 原則 [^3] の単一責任の原則にも適合しています.

また外部から制御できるため, どのような処理が実行されるのかは親コンポーネントでの指定内容に依存します.
例えば`Counter`コンポーネントにカウントを+2 する関数を`increment`として渡してやれば,カウントは 2 ずつ増加するよう挙動を変化させられます. (それが妥当かどうかはさておき)

#### 欠点

コンポーネントを利用するためのデータや関数を親コンポーネントで宣言し, 管理する必要性が出てきます.
コンポーネント単体で見れば再利用性, メンテナンス性共に上がり上々ですが, カプセル化されたコンポーネントと比べ, **そのコンポーネントを利用する側のコードは複雑化させます**.

これは多くの操作を取りまとめる, そこそこ大きめのコンポーネントでのみ欠点となります.
Atomic Design の Atom に相当するコンポーネントなんかではまぁ問題にはならないでしょう.

## 「ロジック」の再利用

**_Mission 3_: Counter にリセット機能を追加せよ!!!**

純粋関数となった`Counter`コンポーネントの拡張は容易です. reset ボタンを追加し, それを押した際に`reset`関数を実行するようにすれば良いだけです.

```tsx
export function Counter({ count, increment, decrement, reset }: Props) {
  return (
    <div>
      <p>{count}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>⟳</button>
    </div>
  );
}
```

さて今回の Mission では 🐥 と 🐔 それぞれに対して`reset`処理を実行する関数を書かなければなりません. `Counter`コンポーネントの機能が増えれば増えるほど, それを利用する側のコードは複雑化し, 繰り返しが増えていくことになります.

```tsx
export function App() {
  const [hiyokoCount, setHiyokoCount] = useState(0);
  const [chickenCount, setChickenCount] = useState(0);

  return (
    <div>
      <h3>Total</h3>
      <p>{hiyokoCount + chickenCount}</p>

      <h3>🐥Hiyoko🐥 Counter</h3>
      <Counter
        count={hiyokoCount}
        increment={() => setHiyokoCount(hiyokoCount + 1)}
        decrement={() => setHiyokoCount(Math.max(hiyokoCount - 1, 0))}
        reset={() => setHiyokoCount(0)}
      />

      <h3>🐔Chicken🐔 Counter</h3>
      <Counter
        count={chickenCount}
        increment={() => setChickenCount(chickenCount + 1)}
        decrement={() => setChickenCount(Math.max(chickenCount - 1, 0))}
        reset={() => setChickenCount(0)}
      />
    </div>
  );
}
```

`Counter`コンポーネントに必要なのは, 状態とその状態を変更する関数だけです. これらの処理は毎回ほとんど同様の制御となるはずです.
不要に同一の処理を記述するのは, 後々バグを生み, コードを長くし, メンテナンス性を下げる原因となり得ます.

これらの処理を**カスタム Hooks** としてまとめて共通化できます.

```ts
const initialCount = 0;

export function useCounter() {
  const [count, setCount] = useState(initialCount);

  const increment = () => setCount(count + 1);
  const decrement = () => setCount(Math.max(count - 1, 0));
  const reset = () => setCount(0);

  return { count, increment, decrement, reset };
}
```

```tsx
export function App() {
  const { count: hiyokoCount, ...hiyokoMethods } = useCounter();
  const { count: chickenCount, ...chickenMethods } = useCounter();

  return (
    <div>
      <h3>Total</h3>
      <p>{hiyokoCount + chickenCount}</p>

      <h3>🐥Hiyoko🐥 Counter</h3>
      <Counter count={hiyokoCount} {...hiyokoMethods} />

      <h3>🐔Chicken🐔 Counter</h3>
      <Counter count={chickenCount} {...chickenMethods} />
    </div>
  );
}
```

すっきりとまとまり, 同じ計算処理や繰り返される setCount の記述がなくなりました.

### カスタム Hooks の再利用性

カスタム Hooks は `useState`などの React の Hooks を利用したロジックを記述し, それをコンポーネントに提供するためのものです.
ロジックはコンポーネントと分離して記述され, そのロジックは単体で存在できます.[^4]
通常は複数のコンポーネントに共通するロジックを分離するものですが, 別に用途はそれだけに限りません.

#### 利点

コンポーネントではなく, その**ロジックだけを再利用**できます.

今回作成した`useCounter`は単に`Counter`コンポーネントで実行されるロジックを集約したものですが, 他のコンポーネントへの移植は自由に行えます.　このため, 事前に集約しておけば, 子コンポーネントから親コンポーネントへとロジックを移動するのはとても容易になります.
他にも全く異なるコンポーネントでもユースケースが合致すれば利用できます. 例えばひたすらに連打回数を記録するようなコンポーネントとか.

#### 欠点

**View とロジックが乖離し**, コンポーネントの実行している処理内容の分かりづらくなるケースがあります. 凝集度が下がるとも表現できます.
大抵の場合は Hooks の名称と受け取れる返却値で処理が推測可能なものとして回避可能でしょう. 幸い TypeScript 使いなら, 返却値は変数名も含みエディタ上で確認ができます.

なおコンポーネントのロジックを切り出すのみであれば, 特に別のファイルに切り出す必要性はありません. コンポーネント定義の上にでもカスタム Hooks を定義してしまえば, 凝集度の低下は回避できます.
共通化すべきカスタム Hooks に切り出される部分がある場合は, そもそもそのコンポーネント特有のロジックではなく, むしろ無駄に結合度が高いだけであるため分離すべきでしょう.

## 「状態管理」の再利用

**_Mission4_: 🐥 が 🐔 になったときに押すボタンを用意せよ!!!**

ヒヨコは成長するものです. 🐥 は 🐔 になり, 🐔 から 🥚 が生まれ, 🥚 は 🐥 になり, 再び 🐥 は 🐔 になります. まれに 🐔 は 🍗 になります.
無駄なことを考えましたが, 状態管理が複雑化してきました. 愚直に実装するのであれば次のようになるでしょう.

```tsx
export function App() {
  const { count: hiyokoCount, ...hiyokoMethods } = useCounter();
  const { count: chickenCount, ...chickenMethods } = useCounter();

  const growUp = () => {
    if (hiyokoCount === 0) return;

    hiyokoMethods.decrement();
    chickenMethods.increment();
  };

  return (
    <div>
      <h3>Total</h3>
      <p>{hiyokoCount + chickenCount}</p>

      <h3>🐥Hiyoko🐥 Counter</h3>
      <Counter count={hiyokoCount} {...hiyokoMethods} />

      <p>
        <button onClick={growUp} disabled={hiyokoCount === 0}>
          Grow up!
        </button>
      </p>

      <h3>🐔Chicken🐔 Counter</h3>
      <Counter count={chickenCount} {...chickenMethods} />
    </div>
  );
}
```

`useCounter`は単一の`Counter`のロジックをまとめたものであるため, 複数のカウンターにまたがるロジックには対応できません. その結果コンポーネント内に再びロジックが記述されることとなりました.

まだ次に 🥚 や 🍗 についてのロジックも書かなくてはならなくなる可能性もあります.
このままではそのたびにコンポーネントが膨れ上がり, メンテナンス性の低下が予期されます.

さて`useCounter`は内部で`useState`を利用し, 1 つの状態を管理していました. しかし今回は複数の状態を同時に管理する必要があります. こういった**複数の状態にまたがる管理**を担うために`useReducer`があります.

まず `useCounter`を`useReducer`を利用するように書き換えてみましょう.

```ts
type State = number;
const initialState = 0;

type Action = { type: "increment" } | { type: "decrement" } | { type: "reset" };
export function counterReducer(state: State, action: Action): State {
  switch (action.type) {
    case "increment":
      return state + 1;
    case "decrement":
      return Math.max(state - 1, 0);
    case "reset":
      return 0;
  }
}

export function counterDispatcher(dispatch: React.Dispatch<Action>) {
  return {
    increment: () => dispatch({ type: "increment" }),
    decrement: () => dispatch({ type: "decrement" }),
    reset: () => dispatch({ type: "reset" }),
  };
}

export function useCounter() {
  const [count, dispatch] = useReducer(counterReducer, initialState);

  return {
    count,
    ...counterDispatcher(dispatch),
  };
}
```

状態管理の方法が変わりましたが, 外から見たこのカスタム Hook の用途や実行できる処理に違いはありません.
変わった点は**状態の遷移が純粋関数で記述された**という点だけです.

これを利用して, 新たな状態管理を担うカスタム Hook を作成してみましょう.

```ts
type State = Record<Kind, number>;
const initialState: State = { hiyoko: 0, chicken: 0 };

type Kind = "hiyoko" | "chicken";
type Action =
  | { type: "increment"; payloads: { kind: Kind } }
  | { type: "decrement"; payloads: { kind: Kind } }
  | { type: "reset"; payloads: { kind: Kind } }
  | { type: "growUp" };

function appCounterReducer(state: State, action: Action): State {
  switch (action.type) {
    case "growUp":
      if (state.hiyoko === 0) return state;
      return {
        hiyoko: state.hiyoko - 1,
        chicken: state.chicken + 1,
      };
    default:
      return {
        ...state,
        [action.payloads.kind]: counterReducer(state[action.payloads.kind], {
          type: action.type,
        }),
      };
  }
}

function appCounterDispatcher(dispatch: React.Dispatch<Action>) {
  return {
    growUp: () => dispatch({ type: "growUp" }),
    hiyokoMethods: counterDispatcher((action) =>
      dispatch({ ...action, payloads: { kind: "hiyoko" } }),
    ),
    chickenMethods: counterDispatcher((action) =>
      dispatch({ ...action, payloads: { kind: "chicken" } }),
    ),
  };
}

export function useAppCounter() {
  const [counts, dispatch] = useReducer(appCounterReducer, initialState);

  return { counts, ...appCounterDispatcher(dispatch) };
}
```

```tsx
export function App() {
  const { counts, hiyokoMethods, chickenMethods, growUp } = useAppCounter();

  return (
    <div>
      <h3>Total</h3>
      <p>{counts.hiyoko + counts.chicken}</p>

      <h3>🐥Hiyoko🐥 Counter</h3>
      <Counter count={counts.hiyoko} {...hiyokoMethods} />

      <p>
        <button onClick={growUp} disabled={counts.hiyoko === 0}>
          Grow up!
        </button>
      </p>

      <h3>🐔Chicken🐔 Counter</h3>
      <Counter count={counts.chicken} {...chickenMethods} />
    </div>
  );
}
```

これで新たに複数のカウント管理に対応した, `useAppCounter`が爆誕しました.
カウンターのロジックを再び書く必要はありません. もう書いてありますから.

### Reducer の再利用性

Reducer は複数の値を管理するもの, と言う見方が広まりつつありますが, Reducer を利用する上で最も重要なポイントは**状態を変更する方法を純粋関数で記述すること**です.
状態管理を純粋関数化することで, 他の Reducer からそれを呼び出し, Reducer の挙動はそのままに, **拡張のみを加えた新たな Reducer** を構築できます.

もともとの`useCounter`は単一のカスタム Hook として, それ自体が再利用されることを意識して作成されました. しかしその内部で行われている状態管理の方法も, 再利用可能な処理であり, カスタム Hook 内に留める必要性はありません.

#### 利点

純粋関数にて状態遷移が記述されるため, **状態管理の方法自体を再利用できます**.
`appCounterReducer`は`counterReducer`のカウント管理方法をそのまま再利用し, それに加えて新たな状態遷移を追加する拡張を完備しました.
このような形態が取れることで, 複雑な状態管理をしなければならない場合であっても, 単純な状態管理を別の Reducer に移譲するなどして, 自身の行わなければならない処理に注力できます.

今回は単純な拡張でしたが, 「ページ全体の状態を一元管理しなくてはならない」といった場合にも, ページの各要素の Reducer を合成する形で対応できます.
また, 合成や拡張を施した Reducer を構築しても, 元々の Reducer は細かい部品の存在し続けるため, 細やかな状態管理と全体の状態管理とのどちらにも対応が効きます.

また純粋関数であるが故に, **状態管理自体をテストすることも容易です**. 入力は状態とアクション, 出力は更新後の状態です.

#### 欠点

デメリットは初期時点での記述量がどうしても増えてしまうことにあります.
ライブラリ依存を不要に増やすメリットをあまり感じないので, 自力で書くことを推奨しますが, React Tool Kit などのライブラリを利用することで, 低減する手法もあります.

## 理想論

再利用性が高く, ロジックの重複がなく, メンテナンス時に修正しなければならないコード量を最小にするには次の構成を取ると良いでしょう.

- 状態を保持するコンポーネントは, 管理する状態に閉じた状態を意識する
- 状態を保持しないコンポーネントは, 純粋関数として記述する.
- 表示処理以外の任意のロジックはカスタム Hook を利用して分離する
- 状態の保持は Reducer を用いて行う

原初に立ち返りましょう.

これを...

```tsx
export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div className="counter">
      <p>{count}</p>
      <button onClick={() => setCount((count) => count + 1)}>+</button>
      <button onClick={() => setCount((count) => Math.max(count - 1, 0))}>
        -
      </button>
    </div>
  );
}
```

こんな感じにできると, 色々合成とかしやすいよね!!!

```tsx
type State = number;
const initialState = 0;

type Action = { type: "increment" } | { type: "decrement" };

export function counterReducer(state: State, action: Action): State {
  switch (action.type) {
    case "increment":
      return state + 1;
    case "decrement":
      return Math.max(state - 1, 0);
  }
}

export function counterDispatcher(dispatch: React.Dispatch<Action>) {
  return {
    increment: () => dispatch({ type: "increment" }),
    decrement: () => dispatch({ type: "decrement" }),
  };
}

export function useCounter() {
  const [count, dispatch] = useReducer(counterReducer, initialState);

  return {
    count,
    ...counterDispatcher(dispatch),
  };
}

export function Counter({ count, increment, decrement }: Props) {
  return (
    <div>
      <p>{count}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  );
}
```

#### 現実的に

まず YAGNI [^5] という原則があります.

> You ain't gonna need it、縮めて YAGNI とは、機能は実際に必要となるまでは追加しないのがよいとする、エクストリーム・プログラミングにおける原則である。

必要がないことはしない. 実際とりあえずコンポーネント作りました!って理想論のコードをレビュー依頼されたらナニコレってなります.

問題となるのは状態管理が複雑化してきても, 依然として`useState`を使い続けたり, 完全に別のコンポーネントを書き始めたりしてしまうケースです.
同じロジックが横行し, 状態管理は錯綜し, `useEffect`で謎の値同期を行いだし, UI は崩壊を迎えていきます.

実際には次のような流れで対応するのがベストでしょう.

- 普通にコンポーネントを構築する
- 状態を別のコンポーネントに管理させたくなったら, カスタム Hooks を切り出す
- 複数の状態を管理する必要に駆られたら, Reducer で書き直す

View に徹するのか, 状態管理をしつつカプセル化するのか等は考慮しつつ, なんら分離せず初めはコンポーネントを構築してしまっていいでしょう. 再利用されることがないのに分離して使えるようにしておく必要はありません.

状態管理をもともとしていて, その管理を別のコンポーネントに移譲する際にはカスタム Hooks を当ててやりましょう.
Hooks はどこからでも呼び出せるので, 移譲する際には元のコンポーネント内部で呼び出していた Hooks を, 親コンポーネントで呼び出すようにするだけで済ませられます.

状態管理が複雑化してきたら, Reducer で管理するようにしましょう. 既存の状態管理に影響を与えることなく, 拡張したり合成したりできるようになります.

## おわりに

(このコンポーネント状態管理内部でゴリゴリにやってるから使いまわしづらいんだよな...)とか思ったときに少しでも使える, 「再利用性の情報のかけら」を心の片隅にでも残してもらえたら幸いです.

React のコンポーネントは, 分離ができることを見ればわかる通り, 「状態管理とロジックと View を混ぜ合わせたオブジェクト」であり, 癒着している状態ではあまり再利用性は高くなりません.
必要に応じて分離し, 拡張や合成に備えておくのが吉です.

[^1]: [初めてのコンポーネント - React#コンポーネントの定義](https://ja.react.dev/learn/your-first-component#defining-a-component)
[^2]: [コンポーネントを純粋に保つ - React](https://ja.react.dev/learn/keeping-components-pure)
[^3]: [イラストで理解する SOLID 原則](https://qiita.com/baby-degu/items/d058a62f145235a0f007) この記事めっちゃわかりやすいです.
[^4]: [カスタムフックでロジックを再利用する - React](https://ja.react.dev/learn/reusing-logic-with-custom-hooks)
[^5]: [YAGNI - wikipedia](https://ja.wikipedia.org/wiki/YAGNI)
