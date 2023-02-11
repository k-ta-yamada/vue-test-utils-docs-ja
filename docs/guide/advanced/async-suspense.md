# Asynchronous Behavior

このガイドの他の部分で、`wrapper` や `setValue` など、 `wrapper`上のいくつかのメソッドを呼び出す際に、 `await` を使っていることにお気づきかもしれません。それはいったい何なのでしょうか?

Vue は、値を変更すると、最新の値を反映するために DOM が自動的に更新される、リアクティブな更新を行うことはご存知かと思います。[Vue はこれらの更新を非同期で行います](https://v3.vuejs.org/guide/change-detection.html#async-update-queue) 。これに対して、Jest のようなテストランナーは、_同期的_ に実行されます。これは、テストにおいて驚くべき結果を引き起こす可能性があります。

テストの実行時に、Vue が期待通りに DOM を更新していることを確認するための戦略をいくつか見てみましょう。

## 簡単な例 - `trigger` で更新する {#a-simple-example-updating-with-trigger}

[イベントハンドリング](../essentials/event-handling) で使用した `<Counter>` コンポーネントを、1つ変更して再利用してみましょう。`template` で `count` をレンダリングします。

```js
const Counter = {
  template: `
    <p>Count: {{ count }}</p>
    <button @click="handleClick">Increment</button>
  `,
  data() {
    return {
      count: 0
    }
  },
  methods: {
    handleClick() {
      this.count += 1
    }
  }
}
```

`count` が増加していることを確認するためのテストを書いてみましょう:

```js
test('increments by 1', () => {
  const wrapper = mount(Counter)

  wrapper.find('button').trigger('click')

  expect(wrapper.html()).toContain('Count: 1')
})
```

意外なことに、これは失敗します! `count` が増加しても、Vue は次のイベントループの tick まで DOM を更新しないためです。このため、アサーション（`expect()...`）は、Vue が DOM を更新する前に呼び出されます。

:::tip
この JavaScript のコアとなる動作についてもっと知りたい方は、[イベントループとそのマクロタスク、マイクロタスク](https://javascript.info/event-loop#macrotasks-and-microtasks) についてをお読みください。
:::

実装の詳細はさておき、どうすればこれを解決できるでしょうか。Vue には、DOM が更新されるまで待つ方法があります: `nextTick`

```js {1,7}
import { nextTick } from 'vue'

test('increments by 1', async () => {
  const wrapper = mount(Counter)

  wrapper.find('button').trigger('click')
  await nextTick()

  expect(wrapper.html()).toContain('Count: 1')
})
```

次の "tick" が実行され、アサーションが実行される前に DOM が更新されていることが確認できたので、テストは合格となります。

`await nextTick()` は一般的なので、Vue Test Utils はショートカットを提供しています。`trigger` や `setValue` など、DOM を更新させるメソッドは `nextTick` を返すので、それらを直接 `await` すればよいのです:

```js {4}
test('increments by 1', async () => {
  const wrapper = mount(Counter)

  await wrapper.find('button').trigger('click')

  expect(wrapper.html()).toContain('Count: 1')
})
```

## その他の非同期動作の解決 {#resolving-other-asynchronous-behavior}

`nextTick` は、テストを継続する前に、反応データの何らかの変更が DOM に反映されることを確認するのに便利です。しかし、時には、Vue に関連しない他の非同期動作も完了することを確認したい場合があります。

一般的な例としては、`Promise` を返す関数があります。おそらく、`jest.mock` を使って `axios` HTTP クライアントをモックしたことでしょう:

```js
jest.spyOn(axios, 'get').mockResolvedValue({ data: 'some mocked data!' })
```

この場合、Vue は未解決の Promise について何も知らないので、`nextTick` を呼び出しても動作しません - 解決する前にアサーションが実行される可能性があります。このようなシナリオのために、Vue Test Utils は [`flushPromises`](../../api/#flushPromises) を公開し、すべての未解決の Promise を直ちに解決するようにします。

例を見てみましょう:

```js{1,12}
import { flushPromises } from '@vue/test-utils'
import axios from 'axios'

jest.spyOn(axios, 'get').mockResolvedValue({ data: 'some mocked data!' })

test('uses a mocked axios HTTP client and flushPromises', async () => {
  // some component that makes a HTTP called in `created` using `axios`
  const wrapper = mount(AxiosComponent)

  await flushPromises() // axios promise is resolved immediately

  // after the line above, axios request has resolved with the mocked data.
})
```

:::tip
Component でのリクエストのテストについてもっと知りたい場合は、[Making HTTP Requests](http-requests.md) ガイドを確認してください。
:::

## 非同期 `setup` のテスト {#testing-asynchronous-setup}

テストしたいコンポーネントが非同期 `setup` を使用している場合、
そのコンポーネントを `Suspense` コンポーネント内にマウントする必要があります
（アプリケーションで使用するときと同様に）。

例えば、この `Async` コンポーネント:

```js
const Async = defineComponent({
  async setup() {
    // await something
  }
})
```

次のようにテストする必要があります:

```js
test('Async component', () => {
  const TestComponent = defineComponent({
    components: { Async },
    template: '<Suspense><Async/></Suspense>'
  })

  const wrapper = mount(TestComponent)
  // ...
})
```

## 結論 {#conclusion}

- Vue は DOM を非同期で更新しますが、テストランナーは代わりに同期的にコードを実行します。
- `await nextTick()` を使用して、テストが継続される前に DOM が更新されることを確認します。
- DOM を更新する可能性のある関数（`trigger` や `setValue` など）は `nextTick` を返すので、それらを `await` する必要があります。
- Vue Test Utils の `flushPromises` を使用して、Vue 以外の依存関係（APIリクエストなど）から未解決の Promise を解決します。
- `Suspense` を使用して、非同期 `setup` のコンポーネントをテストします。
