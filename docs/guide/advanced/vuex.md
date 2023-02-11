# Testing Vuex

Vuex は単なる実装の詳細であり、Vuex を使用したコンポーネントのテストに特別な扱いは必要ありません。とはいえ、テストを読みやすく、書きやすくするためのテクニックはいくつかあります。ここでは、それらについて見ていきます。

このガイドは、あなたが Vuex に精通していることを前提にしています。Vuex 4 は、Vue.js 3 で動作するバージョンです。ドキュメントは [こちら](https://next.vuex.vuejs.org/) をご覧ください。

## 簡単な例 {#a-simple-example}

ここでは、単純な Vuex ストアと、Vuex ストアが存在することに依存するコンポーネントを紹介します:

```js
import { createStore } from 'vuex'

const store = createStore({
  state() {
    return {
      count: 0
    }
  },
  mutations: {
    increment(state: any) {
      state.count += 1
    }
  }
})
```

ストアは単純にカウントを保存し、`increment` 変異がコミットされたときにそれを増加させます。これは私たちがテストするコンポーネントです:

```js
const App = {
  template: `
    <div>
      <button @click="increment" />
      Count: {{ count }}
    </div>
  `,
  computed: {
    count() {
      return this.$store.state.count
    }
  },
  methods: {
    increment() {
      this.$store.commit('increment')
    }
  }
}
```

## 実際の Vuex ストアを使ったテスト {#testing-with-a-real-vuex-store}

このコンポーネントと Vuex ストアが動作していることを完全にテストするために、`<button>` をクリックしてカウントが増加することを確認します。Vue アプリケーションでは、通常 `main.js` で、このように Vuex をインストールします:

```js
const app = createApp(App)
app.use(store)
```

これは、Vuex がプラグインであるためです。プラグインは、`app.use` を呼び出してプラグインを渡すことで適用されます。

Vue Test Utils では、`global.plugins` のマウントオプションを使用して、プラグインもインストールすることができます。

```js
import { createStore } from 'vuex'

const store = createStore({
  state() {
    return {
      count: 0
    }
  },
  mutations: {
    increment(state: any) {
      state.count += 1
    }
  }
})

test('vuex', async () => {
  const wrapper = mount(App, {
    global: {
      plugins: [store]
    }
  })

  await wrapper.find('button').trigger('click')

  expect(wrapper.html()).toContain('Count: 1')
})
```

プラグインをインストールした後、ボタンをクリックする `trigger` を使って、`count` が増加したことをアサートします。この種のテストは、異なるシステム間 (この場合はコンポーネントとストア) の相互作用をカバーするもので、統合テストとして知られています。

## モックストアを使ったテスト {#testing-with-a-mock-store}

対照的に、ユニットテストはコンポーネントとストアを別々に分離してテストするかもしれません。これは、複雑なストアを持つ非常に大きなアプリケーションを持っている場合に便利です。このような場合、`global.mocks` を使ってストアの一部だけをモックします:

```js
test('vuex using a mock store', async () => {
  const $store = {
    state: {
      count: 25
    },
    commit: jest.fn()
  }

  const wrapper = mount(App, {
    global: {
      mocks: {
        $store
      }
    }
  })

  expect(wrapper.html()).toContain('Count: 25')
  await wrapper.find('button').trigger('click')
  expect($store.commit).toHaveBeenCalled()
})
```

本物の Vuex ストアを使用して `global.plugins` 経由でインストールするのではなく、独自のモックストアを作成し、コンポーネントで使用する Vuex の部分（この場合、`state` と `commit` の関数）だけを実装しました。

ストアを分離してテストするのは便利だと思われるかもしれませんが、Vuex ストアを壊しても何の警告も出ないことに注意してください。Vuex ストアをモックするか、本物のストアを使用するかをよく検討し、トレードオフを理解してください。

## Vuex を分離してテストする {#testing-vuex-in-isolation}

Vuex の変異やアクションを完全に分離してテストしたいと思うかもしれません。それらが複雑な場合は特に。Vuex のストアは通常の JavaScript なので、Vue Test Utils は必要ありません。以下は、Vue Test Utils を使わずに `increment` 変異をテストする方法です:

```js
test('increment mutation', () => {
  const store = createStore({
    state: {
      count: 0
    },
    mutations: {
      increment(state) {
        state.count += 1
      }
    }
  })

  store.commit('increment')

  expect(store.state.count).toBe(1)
})
```

## Vuex の状態をプリセットする {#presetting-the-vuex-state}

Vuex ストアを特定の状態にしておくと、テストに便利なことがあります。`global.mocks` 以外で使える便利なテクニックは、`createStore` をラップして、初期状態の種となる引数を取る関数を作成することです。この例では、`state.count` に追加される引数を取るために `increment`を拡張しています。この引数がない場合は、`state.count` を 1 だけ増加させます。

```js
const createVuexStore = (initialState) =>
  createStore({
    state: {
      count: 0,
      ...initialState
    },
    mutations: {
      increment(state, value = 1) {
        state.count += value
      }
    }
  })

test('increment mutation without passing a value', () => {
  const store = createVuexStore({ count: 20 })
  store.commit('increment')
  expect(store.state.count).toBe(21)
})

test('increment mutation with a value', () => {
  const store = createVuexStore({ count: -10 })
  store.commit('increment', 15)
  expect(store.state.count).toBe(5)
})
```

初期状態を受け取る `createVuexStore` 関数を作成することで、簡単に初期状態を設定することができます。これにより、テストを簡素化しつつ、すべてのエッジケースをテストすることができます。

[Vue Testing Handbook](https://lmiller1990.github.io/vue-testing-handbook/testing-vuex.html) には、Vuex をテストするためのより多くの例があります。注意: この例は Vue.js 2 と Vue Test Utils v1 に関係します。考え方やコンセプトは同じで、Vue Testing Handbook は近いうちに Vue.js 3 と Vue Test Utils 2 に更新される予定です。

## Composition API を使ったテスト {#testing-using-the-composition-api}

Vuex は Composition API を使用する場合、`useStore` 関数でアクセスします。[詳しくはこちらをご覧ください](https://next.vuex.vuejs.org/guide/composition-api.html) 。

`useStore` は、[Vuex のドキュメント](https://next.vuex.vuejs.org/guide/typescript-support.html#typing-usestore-composition-function) で説明されているように、オプションでユニークなインジェクションキーを使用することができます。

このような感じです:

```js
import { createStore } from 'vuex'
import { createApp } from 'vue'

// create a globally unique symbol for the injection key
const key = Symbol()

const App = {
  setup () {
    // use unique key to access store
    const store = useStore(key)
  }
}

const store = createStore({ /* ... */ })
const app = createApp({ /* ... */ })

// specify key as second argument when calling app.use(store)
app.use(store, key)
```

`useStore` を使うたびにキーパラメータの受け渡しを繰り返すことを避けるため、Vuex のドキュメントでは、そのロジックをヘルパー関数に抽出し、デフォルトの `useStore` 関数の代わりにその関数を再利用することを推奨しています。[詳しくはこちらをご覧ください](https://next.vuex.vuejs.org/guide/typescript-support.html#typing-usestore-composition-function) 。Vue Test Utils を使用してストアを提供するアプローチは、コンポーネントで `useStore` 関数を使用する方法によって異なります。

### インジェクションキーなしで `useStore` を利用するコンポーネントのテスト {#testing-components-that-utilize-use-store-without-an-injection-key}

インジェクションキーがなければ、グローバルなマウントオプションである `provide` を使ってストアデータをコンポーネントに注入することができます。注入されるストアの名前は、コンポーネント内の名前と同じである必要があります (例: "store")。

#### キーによらない `useStore` を提供する場合の例 {#example-for-providing-the-unkeyed-use-store}

```js
import { createStore } from 'vuex'

const store = createStore({
  // ...
})

const wrapper = mount(App, {
  global: {
    provide: {
      store: store
    },
  },
})
```

### インジェクションキーで `useStore` を利用するコンポーネントをテストする {#testing-components-that-utilize-use-store-with-an-injection-key}

インジェクションキーでストアを使用する場合、以前のアプローチは機能しません。`useStore` からはストアのインスタンスは返されません。正しいストアにアクセスするために、識別子を指定する必要があります。

これは、コンポーネントの `setup` 関数で `useStore` に渡されるキー、 あるいはカスタムヘルパー関数内で `useStore` に渡されるキーとまったく同じである必要があります。JavaScript のシンボルは一意であり、再作成することができないので、実際のストアからキーをエクスポートするのが最善です。

ストアを注入するには、正しいキーを持つ `global.provide` を使用するか、ストアをインストールしキーを指定する `global.plugins` を使用することができます:

#### `global.provide` を使用して Keyed `useStore` を提供する {#providing-the-keyed-use-store-using-global-provide}

```js
// store.js
export const key = Symbol()
```

```js
// app.spec.js
import { createStore } from 'vuex'
import { key } from './store'

const store = createStore({ /* ... */ })

const wrapper = mount(App, {
  global: {
    provide: {
      [key]: store
    },
  },
})
```

#### `global.plugins` を使用した Keyed `useStore` の提供 {#providing-the-keyed-use-store-using-global-plugins}

```js
// store.js
export const key = Symbol()
```

```js
// app.spec.js
import { createStore } from 'vuex'
import { key } from './store'

const store = createStore({ /* ... */ })

const wrapper = mount(App, {
  global: {
    // to pass options to plugins, use the array syntax.
    plugins: [[store, key]]
  },
})
```



## 結論 {#conclusion}

- Vuex をプラグインとしてインストールするには `global.plugins` を使用します。
- 高度なユースケースのために Vuex のようなグローバルオブジェクトをモックするために `global.mocks` を使用します。
- 複雑な Vuex の変異とアクションを分離してテストすることを検討します。
- 特定のテストシナリオを設定するために、引数を取る関数で `createStore` をラップします。
