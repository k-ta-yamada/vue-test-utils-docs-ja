# Stubs and Shallow Mount

Vue Test Utils は、コンポーネントやディレクティブをスタブ化するための高度な機能を提供します。_スタブ_ は、カスタムコンポーネントやディレクティブの既存の実装を、全く何もしないダミーのものに置き換えることで、複雑なテストを簡素化することができます。例を見てみましょう。

## 単一の子コンポーネントのスタブ化 {#stubbing-a-single-child-component}

よくある例としては、コンポーネント階層の非常に高い位置にあるコンポーネントで何かをテストしたい場合です。

この例では、メッセージを表示する `<App>` と、API を呼び出してその結果を表示する `FetchDataFromApi` コンポーネントを用意しています。

```js
const FetchDataFromApi = {
  name: 'FetchDataFromApi',
  template: `
    <div>{{ result }}</div>
  `,
  async mounted() {
    const res = await axios.get('/api/info')
    this.result = res.data
  },
  data() {
    return {
      result: ''
    }
  }
}

const App = {
  components: {
    FetchDataFromApi
  },
  template: `
    <h1>Welcome to Vue.js 3</h1>
    <fetch-data-from-api />
  `
}
```

このテストでは API 呼び出しを行わず、メッセージがレンダリングされたことを確認したいだけです。この場合、`global` マウントオプションに表示される `stubs` を使用することができます。

```js
test('stubs component with custom template', () => {
  const wrapper = mount(App, {
    global: {
      stubs: {
        FetchDataFromApi: {
          template: '<span />'
        }
      }
    }
  })

  console.log(wrapper.html())
  // <h1>Welcome to Vue.js 3</h1><span></span>

  expect(wrapper.html()).toContain('Welcome to Vue.js 3')
})
```

テンプレートが `<fetch-data-from-api />` があった場所に `<span></span>` を表示していることに注意してください。これをスタブに置き換えたのです。この場合、`template` を渡すことで独自の実装を提供しました。

また、自分でスタブを用意するのではなく、デフォルトのスタブを取得することも可能です。

```js
test('stubs component', () => {
  const wrapper = mount(App, {
    global: {
      stubs: {
        FetchDataFromApi: true
      }
    }
  })

  console.log(wrapper.html())
  /*
    <h1>Welcome to Vue.js 3</h1>
    <fetch-data-from-api-stub></fetch-data-from-api-stub>
  */

  expect(wrapper.html()).toContain('Welcome to Vue.js 3')
})
```

これにより、レンダーツリー全体のすべての `<FetchDataFromApi />` コンポーネントが、どのレベルに表示されるかに関係なく、スタブ出力されます。そのため、`global` マウントオプションにあるのです。

::: tip
スタブアウトするには、`components` のキーを使うか、コンポーネントの名前を使うかのどちらかです。`global.stubs` で両方が指定された場合は、キーが先に使用されます。
:::

## すべての子コンポーネントをスタブする {#stubbing-all-children-components}

時には、_すべて_ のカスタムコンポーネントをスタブアウトしたい場合があります。例えば、以下のようなコンポーネントがあるとします:

```js
const ComplexComponent = {
  components: { ComplexA, ComplexB, ComplexC },
  template: `
    <h1>Welcome to Vue.js 3</h1>
    <ComplexA />
    <ComplexB />
    <ComplexC />
  `
}
```

各 `<Complex>` が何か複雑なことをしていて、`<h1>` が正しい挨拶をレンダリングしているかどうかだけをテストすることに興味があると想像してください。次のようなことができます:

```js
const wrapper = mount(ComplexComponent, {
  global: {
    stubs: {
      ComplexA: true,
      ComplexB: true,
      ComplexC: true
    }
  }
})
```

しかし、これでは定型文が多くなってしまいます。VTU には `shallow` マウントオプションがあり、自動的に子コンポーネントをすべてスタブアウトしてくれます:

```js {3}
test('shallow stubs out all child components', () => {
  const wrapper = mount(ComplexComponent, {
    shallow: true
  })

  console.log(wrapper.html())
  /*
    <h1>Welcome to Vue.js 3</h1>
    <complex-a-stub></complex-a-stub>
    <complex-b-stub></complex-b-stub>
    <complex-c-stub></complex-c-stub>
  */
})
```

::: tip
VTU V1 を使っていた人は、これを `shallowMount` と覚えているかもしれません。その方法もまだ利用できます。`shallow: true` と書くのと同じです。
:::

## 例外を含むすべての子コンポーネントをスタブする {#stubbing-all-children-components-with-exceptions}

特定のコンポーネントを _除き_、_すべて_ のカスタムコンポーネントをスタブアウトしたい場合があります。その例を考えてみましょう:

```js
const ComplexA = {
  template: '<h2>Hello from real component!</h2>'
}

const ComplexComponent = {
  components: { ComplexA, ComplexB, ComplexC },
  template: `
    <h1>Welcome to Vue.js 3</h1>
    <ComplexA />
    <ComplexB />
    <ComplexC />
  `
}
```

`shallow` マウントオプションを使用すると、すべての子コンポーネントが自動的にスタブアウトされます。特定のコンポーネントのスタブを明示的に無効にしたい場合は、そのコンポーネントの名前を `stubs` で指定して `false` を設定します。

```js {3}
test('shallow allows opt-out of stubbing specific component', () => {
  const wrapper = mount(ComplexComponent, {
    shallow: true,
    global: {
      stubs: { ComplexA: false }
    }
  })

  console.log(wrapper.html())
  /*
    <h1>Welcome to Vue.js 3</h1>
    <h2>Hello from real component!</h2>
    <complex-b-stub></complex-b-stub>
    <complex-c-stub></complex-c-stub>
  */
})
```

## 非同期コンポーネントのスタブ化 {#stubbing-an-async-component}

非同期コンポーネントをスタブアウトさせたい場合、2 つの動作があります。例えば、以下のようなコンポーネントがあるとします:

```js
// AsyncComponent.js
export default defineComponent({
  name: 'AsyncComponent',
  template: '<span>AsyncComponent</span>'
})

// App.js
const App = defineComponent({
  components: {
    MyComponent: defineAsyncComponent(() => import('./AsyncComponent'))
  },
  template: '<MyComponent/>'
})
```

最初の動作は、非同期コンポーネントをロードするコンポーネントで定義されたキーを使用します。この例では、"MyComponent "というキーを使用しています。
コンポーネントは解決する前にスタブ化されているので、テストケースで `async/await` を使用する必要はありません。

```js
test('stubs async component without resolving', () => {
  const wrapper = mount(App, {
    global: {
      stubs: {
        MyComponent: true
      }
    }
  })

  expect(wrapper.html()).toBe('<my-component-stub></my-component-stub>')
})
```

2 番目の動作は、非同期コンポーネントの名前を使用することです。 この例では、"AsyncComponent" という名前を使用しています。
非同期コンポーネントを解決する必要があり、非同期コンポーネントで定義された名前でスタブ化できるため、`async/await` を使用する必要があります。

**非同期コンポーネントで名前を定義していることを確認してください!**

```js
test('stubs async component with resolving', async () => {
  const wrapper = mount(App, {
    global: {
      stubs: {
        AsyncComponent: true
      }
    }
  })

  await flushPromises()

  expect(wrapper.html()).toBe('<async-component-stub></async-component-stub>')
})
```

## ディレクティブをスタブする {#stubbing-a-directive}

時には、ディレクティブは非常に複雑なことをすることがあります。例えば、多くの DOM 操作を行い、テストでは（JSDOM が DOM 全体の挙動に似ていないために）エラーになる可能性があります。一般的な例としては、様々なライブラリのツールチップディレクティブがあり、これらは通常、DOM ノードの位置やサイズの測定に大きく依存しています。

この例では、ツールチップ付きのメッセージを表示する別の `<App>` を用意しています。

```js
// tooltip directive declared somewhere, named `Tooltip`

const App = {
  directives: {
    Tooltip
  },
  template: '<h1 v-tooltip title="Welcome tooltip">Welcome to Vue.js 3</h1>'
}
```

このテストでは、`Tooltip` ディレクティブのコードを実行させたくないので、メッセージがレンダリングされることを保証したいだけです。この場合、`vTooltip` を渡す `global` マウントオプションに表示される `stubs` を使用することができます。

```js
test('stubs component with custom template', () => {
  const wrapper = mount(App, {
    global: {
      stubs: {
        vTooltip: true
      }
    }
  })

  console.log(wrapper.html())
  // <h1>Welcome to Vue.js 3</h1>

  expect(wrapper.html()).toContain('Welcome to Vue.js 3')
})
```

::: tip
コンポーネントとディレクティブを区別するための `vCustomDirective` 命名スキームの使用は、`<script setup>` で使用されているのと [同じアプローチ](https://vuejs.org/api/sfc-script-setup.html#using-custom-directives) に触発されたものです。
:::

時々、ディレクティブの機能の一部が必要になることがあります（通常は、いくつかのコードがそれに依存しているため）。例えば、ディレクティブが実行されたときに `with-tooltip` CSS クラスを追加し、それが私たちのコードにとって重要な動作であるとしましょう。この場合、モックディレクティブの実装で `true` を交換することができます。

```js
test('stubs component with custom template', () => {
  const wrapper = mount(App, {
    global: {
      stubs: {
        vTooltip: {
          beforeMount(el: Element) {
            console.log('directive called')
            el.classList.add('with-tooltip')
          }
        }
      }
    }
  })

  // 'directive called' logged to console

  console.log(wrapper.html())
  // <h1 class="with-tooltip">Welcome to Vue.js 3</h1>

  expect(wrapper.classes('with-tooltip')).toBe(true)
})
```

ディレクティブの実装を自分たちのものと入れ替えただけです!

::: warning
スタブディレクティブは、[withDirectives](https://ja.vuejs.org/api/render-function.html#withdirectives) 関数内にディレクティブ名がないため、 機能コンポーネントや `<script setup>` では動作しません。機能的なコンポーネントで使用されるディレクティブをモックする必要がある場合は、 テストフレームワークでディレクティブモジュールをモックすることを検討してください。このような機能を解除するための提案については、 https://github.com/vuejs/core/issues/6887 を参照してください。
:::

## デフォルトスロットと `shallow` {#default-slots-and-shallow}

`shallow` はコンポーネントのすべての内容をスタブ化するので、`shallow` を使用すると `<slot>` はすべてレンダリングされません。ほとんどの場合、これは問題ではありませんが、これが理想的でないシナリオもあります。

```js
const CustomButton = {
  template: `
    <button>
      <slot />
    </button>
  `
}
```

そして、こんな風に使うかもしれません:

```js
const App = {
  props: ['authenticated'],
  components: { CustomButton },
  template: `
    <custom-button>
      <div v-if="authenticated">Log out</div>
      <div v-else>Log in</div>
    </custom-button>
  `
}
```

`shallow` を使用している場合、`<custom-button />` の render 関数がスタブ化されるため、スロットはレンダリングされません。つまり、正しいテキストがレンダリングされたかどうかを確認することができないのです!

この使用例では、`shallow` を使用している場合でも、デフォルトのスロットコンテンツをレンダリングする `config.renderStubDefaultSlot` を使用することができます。

```js {1,4,8}
import { config, mount } from '@vue/test-utils'

beforeAll(() => {
  config.global.renderStubDefaultSlot = true
})

afterAll(() => {
  config.global.renderStubDefaultSlot = false
})

test('shallow with stubs', () => {
  const wrapper = mount(AnotherApp, {
    props: {
      authenticated: true
    },
    shallow: true
  })

  expect(wrapper.html()).toContain('Log out')
})
```

この動作は `mount` 毎ではなくグローバルに行われるため、各テストの前後に有効/無効にすることを覚えておく必要があります。

::: tip
また、テストのセットアップファイルで `config` をインポートし、 `renderStubDefaultSlot` を `true` に設定することで、グローバルに有効にすることができます。残念ながら、技術的な制限により、この動作はデフォルトスロット以外のスロットには拡張されません。
:::

## `mount`、`shallow`、`stubs`: いつ、どれを使う? {#mount-shallow-and-stubs-which-one-and-when}

経験則では、**テストがソフトウェアの使用方法に似ていればいるほど**、より高い信頼性を得ることができます。

`mount` を使用するテストでは、コンポーネント階層全体をレンダリングするため、実際のブラウザでユーザーが体験するものに近くなります。

一方、`shallow` を使用したテストは特定のコンポーネントに焦点を当てたものになります。 `shallow` は、高度なコンポーネントを完全に分離してテストするのに便利です。テストに関係のないコンポーネントがひとつかふたつあるだけなら、`shallow` ではなく `stubs` と組み合わせて `mount` を使うことを考えましょう。スタブを増やせば増やすほど、テストがプロダクションライクでなくなります。

フルマウントでもシャローレンダリングでも、良いテストは実装の詳細ではなく、入力（`props` や `trigger` などのユーザーインタラクション）と出力（レンダリングされる DOM 要素やイベント）に焦点を当てることを心に留めておいてください。

そのため、どのような実装方法を選択する場合でも、以下のガイドラインを念頭に置くことをお勧めします。

## 結論 {#conclusion}

- `global.stubs` を使って、コンポーネントやディレクティブをダミーに置き換えることで、テストを簡略化することができます。
- `shallow: true` （または `shallowMount`）を使用すると、すべての子コンポーネントをスタブ化することができます。
- `global.renderStubDefaultSlot` を使用して、スタブされたコンポーネントのデフォルトの `<slot>` をレンダリングします
