# テストしやすいコンポーネントを書く {#write-components-that-are-easy-to-test}

Vue Test Utils は、Vue コンポーネントのテストを書くのに役立ちます。しかし、VTU ができることは限られています。

以下は、テストをしやすいコードを書き、意味のあるテストを書き、保守を簡単にするための提案のリストです。

以下のリストは一般的なガイダンスであり、一般的なシナリオで役に立つかもしれません。

## 実装の詳細をテストしない {#do-not-test-implementation-details}

ユーザーの視点からの入力と出力で考える。大雑把に言うと、Vue コンポーネントのテストを書くときに考慮すべきことは、このようなことです:

| **入力**      | 例                                                   |
| ------------ | ---------------------------------------------------- |
| Interactions | クリック、タイピング...あらゆる "ヒューマン "インタラクション |
| Props        | コンポーネントが受け取る引数                             |
| Data streams | APIコールやデータのサブスクリプションから入ってくるデータ... |

| **出力**      | 例                                                   |
| ------------ | ---------------------------------------------------- |
| DOM elements | ドキュメントにレンダリングされた任意の _observable_ なノード |
| Events       | 放出されるイベント（`$emit` を使用）                      |
| Side Effects | `console.log` や API コールなど                        |

**それ以外はすべて、実装の詳細です**。

このリストには、内部メソッド、中間状態、データなどの要素が含まれていないことに注意してください。

経験則では、**テストはリファクタリングで壊れるべきではない** とされています。つまり、動作を変えずに内部実装を変更する場合です。もしそうなった場合、テストは実装の詳細に依存することになるかもしれません。

例えば、カウンターをインクリメントするためのボタンを備えた基本的な Counter コンポーネントを想定してみましょう:

```vue
<template>
  <p class="paragraph">Times clicked: {{ count }}</p>
  <button @click="increment">increment</button>
</template>

<script>
export default {
  data() {
    return { count: 0 }
  },
  methods: {
    increment() {
      this.count++
    }
  }
}
</script>
```

次のようなテストが書けます:

```js
import { mount } from '@vue/test-utils'
import Counter from './Counter.vue'

test('counter text updates', async () => {
  const wrapper = mount(Counter)
  const paragraph = wrapper.find('.paragraph')

  expect(paragraph.text()).toBe('Times clicked: 0')

  await wrapper.setData({ count: 2 })

  expect(paragraph.text()).toBe('Times clicked: 2')
})
```

ここでは、内部データを更新しており、CSSクラスなどの詳細（ユーザーの視点）にも依存していることに注目してください。

:::tip
データまたは CSS クラス名のいずれかを変更すると、テストが失敗することに注意してください。しかし、コンポーネントは期待通りに動作します。これは **誤検出** と呼ばれるものです。
:::

代わりに、次のテストでは、上記の入力と出力にこだわるようにしています:

```js
import { mount } from '@vue/test-utils'

test('text updates on clicking', async () => {
  const wrapper = mount(Counter)

  expect(wrapper.text()).toContain('Times clicked: 0')

  const button = wrapper.find('button')
  await button.trigger('click')
  await button.trigger('click')

  expect(wrapper.text()).toContain('Times clicked: 2')
})
```

[Vue Testing Library](https://github.com/testing-library/vue-testing-library/) のようなライブラリは、この原則に基づいて構築されています。もし、このアプローチに興味があれば、ぜひチェックしてみてください。

## より小さく、よりシンプルな部品を作る {#build-smaller-simpler-components}

一般的な経験則では、あるコンポーネントの機能が小さいほど、テストがしやすいと言われています。

コンポーネントをより小さくすることで、よりコンポーザブルになり、理解しやすくなります。以下は、コンポーネントをよりシンプルにするための提案リストです。

### APIコールの抽出 {#extract-api-calls}

通常、アプリケーション全体を通して、いくつかの HTTP リクエストを実行することになります。テストの観点からは、HTTP リクエストはコンポーネントへの入力を提供し、コンポーネントは HTTP リクエストを送信することもできます。

:::tip
APIコールのテストに不慣れな方は、[Making HTTP requests](../advanced/http-requests.md) のガイドをご覧ください。
:::

### 複雑なメソッドを抽出する {#extract-complex-methods}

コンポーネントが複雑なメソッドを備えていたり、重い計算を実行したり、複数の依存関係を使用したりすることがあります。

ここで提案したいのは、**このメソッドを抽出してコンポーネントにインポートすることです**。そうすれば、Jest やその他のテストランナーを使って、このメソッドを単独でテストすることができます。

また、複雑なロジックが別のファイルにカプセル化されているため、最終的に理解しやすいコンポーネントになるという利点もあります。

また、複雑なメソッドを設定するのが難しい、あるいは遅い場合は、モックを作成してテストを単純化し、高速化するのもよいでしょう。[making HTTP requests](../advanced/http-requests.md) の例などは良い例です。axios はかなり複雑なライブラリですから!

## コンポーネントを書く前にテストを書く {#write-tests-before-writing-the-component}

あらかじめテストを書いておけば、テスト不能なコードは書けません!

この [Crash Course](../essentials/a-crash-course.md) では、コードの前にテストを書くことが、いかにテスト可能なコンポーネントにつながるかを例示しています。また、エッジケースを検出し、テストするのに役立ちます。
