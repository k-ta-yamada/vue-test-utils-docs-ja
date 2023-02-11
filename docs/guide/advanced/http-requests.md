# HTTPリクエストの作成 {#making-http-requests}

最近のテストランナーは、HTTP リクエストのテストに関して、すでに多くの素晴らしい機能を提供しています。したがって、Vue Test Utils は、そのためのユニークなツールを備えていません。

しかし、テストする上で重要な機能であり、強調したいいくつかの落とし穴があります。

このセクションでは、HTTP リクエストの実行、モック、およびアサートのためのいくつかのパターンを探ります。

## ブログの記事一覧 {#a-list-of-blog-posts}

まず、基本的な使用例から見ていきましょう。次の `PostList` コンポーネントは、外部 API から取得したブログ投稿のリストをレンダリングします。これらの記事を取得するために、このコンポーネントはリクエストをトリガーする `button` 要素を備えています:

```vue
<template>
  <button @click="getPosts">Get posts</button>
  <ul>
    <li v-for="post in posts" :key="post.id" data-test="post">
      {{ post.title }}
    </li>
  </ul>
</template>

<script>
import axios from 'axios'

export default {
  data() {
    return {
      posts: null
    }
  },
  methods: {
    async getPosts() {
      this.posts = await axios.get('/api/posts')
    }
  }
}
</script>
```

このコンポーネントを正しくテストするために必要なことがいくつかあります。

私たちの最初の目標は、**実際に API に到達することなく**、このコンポーネントをテストすることです。そうすると、壊れやすく、遅い可能性のあるテストが作成されます。

次に、コンポーネントが適切なパラメータで正しい呼び出しを行ったことを保証する必要があります。私たちはその API から結果を得ることはありませんが、それでも私たちが正しいリソースを要求したことを確認する必要があります。

また、DOM がそれに応じて更新され、データが表示されていることを確認する必要があります。これは `@vue/test-utils` の `flushPromises()` 関数で行います。

```js
import { mount, flushPromises } from '@vue/test-utils'
import axios from 'axios'
import PostList from './PostList.vue'

const mockPostList = [
  { id: 1, title: 'title1' },
  { id: 2, title: 'title2' }
]

// Following lines tell Jest to mock any call to `axios.get`
// and to return `mockPostList` instead
jest.spyOn(axios, 'get').mockResolvedValue(mockPostList)

test('loads posts on button click', async () => {
  const wrapper = mount(PostList)

  await wrapper.get('button').trigger('click')

  // Let's assert that we've called axios.get the right amount of times and
  // with the right parameters.
  expect(axios.get).toHaveBeenCalledTimes(1)
  expect(axios.get).toHaveBeenCalledWith('/api/posts')

  // Wait until the DOM updates.
  await flushPromises()

  // Finally, we make sure we've rendered the content from the API.
  const posts = wrapper.findAll('[data-test="post"]')

  expect(posts).toHaveLength(2)
  expect(posts[0].text()).toContain('title1')
  expect(posts[1].text()).toContain('title2')
})
```

変数 `mockPostList` に接頭辞 `mock` を付けていることに注意してください。そうでないと、"The module factory of jest.mock() is not allowed to reference any out-of-scope variables." というエラーが発生します。これは jest 固有のもので、この動作については jest の [ドキュメント](https://jestjs.io/docs/es6-class-mocks#calling-jestmock-with-the-module-factory-parameter) を参照してください。

また、`flushPromises` を待ってから Component と対話したことにも注目してください。これは、アサーションが実行される前に DOM が更新されたことを確認するためです。

:::tip Alternatives to jest.mock()
Jest でモックを設定するには、いくつかの方法があります。上の例で使っているのは最もシンプルなものです。より強力な代用品として、[axios-mock-adapter](https://github.com/ctimmerm/axios-mock-adapter) や [msw](https://github.com/mswjs/msw) などをチェックしてみてください。
:::

### ローディング状態のアサーション {#asserting-loading-state}

さて、この `PostList` コンポーネントはかなり便利なのですが、他の素晴らしい機能が欠けています。これを拡張して、投稿を読み込むときに派手なメッセージを表示するようにしてみましょう!

また、ローディング中は `<button>` 要素も無効にしておきましょう。フェッチ中にユーザーがリクエストを送り続けるようなことは避けたいのです!

```vue {2,4,19,24,28}
<template>
  <button :disabled="loading" @click="getPosts">Get posts</button>

  <p v-if="loading" role="alert">Loading your posts…</p>
  <ul v-else>
    <li v-for="post in posts" :key="post.id" data-test="post">
      {{ post.title }}
    </li>
  </ul>
</template>

<script>
import axios from 'axios'

export default {
  data() {
    return {
      posts: null,
      loading: null
    }
  },
  methods: {
    async getPosts() {
      this.loading = true

      this.posts = await axios.get('/api/posts')

      this.loading = null
    }
  }
}
</script>
```

ローディングに関連するすべての要素が時間通りにレンダリングされることを保証するためのテストを書いてみましょう。

```js
test('displays loading state on button click', async () => {
  const wrapper = mount(PostList)

  // ボタンをクリックする前に、以下のアサーションを実行していることに注意してください。
  // ここでは、コンポーネントは "not loading" の状態でなければなりません。
  expect(wrapper.find('[role="alert"]').exists()).toBe(false)
  expect(wrapper.get('button').attributes()).not.toHaveProperty('disabled')

  // では、いつものようにトリガーしてみましょう。
  await wrapper.get('button').trigger('click')

  // すべての Promise をフラッシュする前に、「ローディング状態」をアサートします。
  expect(wrapper.find('[role="alert"]').exists()).toBe(true)
  expect(wrapper.get('button').attributes()).toHaveProperty('disabled')

  // 前と同じように、DOM が更新されるまで待ちます。
  await flushPromises()

  // その後、"not loading" の状態に戻ります。
  expect(wrapper.find('[role="alert"]').exists()).toBe(false)
  expect(wrapper.get('button').attributes()).not.toHaveProperty('disabled')
})
```

## Vuex からの HTTP リクエスト {#http-requests-from-vuex}

より複雑なアプリケーションの典型的なシナリオは、HTTP リクエストを実行する Vuex アクションをトリガーすることです。

これは、上で説明した例と変わりません。ストアをそのままロードして、`axios` などのサービスをモックするのもよいでしょう。こうすることで、システムの境界をモックすることができ、より高い信頼性を得ることができます。

Vue Test Utils を使った Vuexのテストの詳細については、[Testing Vuex](vuex.md) ドキュメントをご覧ください。

## 結論 {#conclusion}

- Vue Test Utils は、HTTP リクエストをテストするために特別なツールを必要としません。考慮すべきは、非同期の動作をテストしていることくらいです。
- テストは外部サービスに依存してはいけません。それを避けるために、`jest.mock` などのモッキングツールを使いましょう。
- `flushPromises()` は、非同期操作の後に DOM が更新されることを確認するための便利なツールです。
- コンポーネントと対話し、HTTP リクエストを直接トリガーすることで、テストはより弾力的になります。
