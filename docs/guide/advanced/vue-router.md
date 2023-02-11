# Vue Router のテスト {#testing-vue-router}

この記事では、Vue Router を使用してアプリケーションをテストする 2 つの方法を紹介します:

1. 本物の Vue Router を使用することで、よりプロダクションライクになりますが、大規模なアプリケーションのテストでは複雑化する可能性があります。
2. モック化されたルーターを使用することで、テスト環境をより細かく制御することができます。

Vue Test Utils は、Vue Router に依存するコンポーネントのテストを支援する特別な関数を提供しないことに注意してください。

## モックルーターの使用 {#using-a-mocked-router}

モック化されたルーターを使用すると、ユニットテストで Vue Router の実装の詳細を気にする必要がなくなります。

本物の Vue Router のインスタンスを使用する代わりに、興味のある機能だけを実装したモックバージョンを作成することができます。これは、`jest.mock`（Jest を使用している場合）と `global.components` の組み合わせを使用して行うことができます。

依存関係をモックにするのは、たいていの場合、**その動作をテストすることに興味がないから** です。`<router-link>` が正しいページにナビゲートすることをテストしたいわけではありません - もちろんそうです! しかし、`<a>` が正しい `to` 属性を持っていることを確認することには興味があるかもしれません。

もっと現実的な例を見てみましょう! このコンポーネントは、認証されたユーザーを投稿の編集ページにリダイレクトするボタンを表示します（現在のルートパラメータに基づいて）。未認証のユーザーは `/404` ルートにリダイレクトされるはずです。

```js
const Component = {
  template: `<button @click="redirect">Click to Edit</button>`,
  props: ['isAuthenticated'],
  methods: {
    redirect() {
      if (this.isAuthenticated) {
        this.$router.push(`/posts/${this.$route.params.id}/edit`)
      } else {
        this.$router.push('/404')
      }
    }
  }
}
```

実際のルーターを使用して、このコンポーネントの正しいルートに移動し、ボタンをクリックした後に正しいページがレンダリングされたことを確認することができます... しかし、これは比較的単純なテストのための多くの設定です。私たちが書きたいテストの核心は、「認証されたら X にリダイレクトし、そうでなければ Y にリダイレクトする」ことです。`global.mocks` プロパティを使用してルーティングをモックすることで、これをどのように実現するかを見てみましょう:

```js
import { mount } from '@vue/test-utils';

test('allows authenticated user to edit a post', async () => {
  const mockRoute = {
    params: {
      id: 1
    }
  }
  const mockRouter = {
    push: jest.fn()
  }

  const wrapper = mount(Component, {
    props: {
      isAuthenticated: true
    },
    global: {
      mocks: {
        $route: mockRoute,
        $router: mockRouter
      }
    }
  })

  await wrapper.find('button').trigger('click')

  expect(mockRouter.push).toHaveBeenCalledTimes(1)
  expect(mockRouter.push).toHaveBeenCalledWith('/posts/1/edit')
})

test('redirect an unauthenticated user to 404', async () => {
  const mockRoute = {
    params: {
      id: 1
    }
  }
  const mockRouter = {
    push: jest.fn()
  }

  const wrapper = mount(Component, {
    props: {
      isAuthenticated: false
    },
    global: {
      mocks: {
        $route: mockRoute,
        $router: mockRouter
      }
    }
  })

  await wrapper.find('button').trigger('click')

  expect(mockRouter.push).toHaveBeenCalledTimes(1)
  expect(mockRouter.push).toHaveBeenCalledWith('/404')
})
```

各テストで理想的な状態を設定するために必要な依存関係（`this.$route` と `this.$router`）を提供するために `global.mocks` を使用しました。

そして、`jest.fn()` を使って、`this.$router.push` が何回、どの引数で呼ばれたかを監視できるようになりました。そして何より、このテストでは Vue Router の複雑さや注意点に対処する必要がありません。私たちは、アプリのロジックをテストすることだけに関心を持ちました。

:::tip
システム全体をエンドツーエンドでテストしたいと思うかもしれません。実際のブラウザを使用したシステム全体のテストには、[Cypress](https://www.cypress.io/) のようなフレームワークを検討することができます。
:::

## 本物のルーターを使用する {#using-a-real-router}

モック化されたルーターの使い方を見たので、本物の Vue Router の使い方を見てみましょう。

Vue Router を使用した基本的なブログアプリケーションを作成してみましょう。投稿は `/posts` ルートに一覧表示されます。

```js
const App = {
  template: `
    <router-link to="/posts">Go to posts</router-link>
    <router-view />
  `
}

const Posts = {
  template: `
    <h1>Posts</h1>
    <ul>
      <li v-for="post in posts" :key="post.id">
        {{ post.name }}
      </li>
    </ul>
  `,
  data() {
    return {
      posts: [{ id: 1, name: 'Testing Vue Router' }]
    }
  }
}
```

アプリのルートには、`/posts` につながる `<router-link>` が表示され、そこに投稿が一覧表示されます。

本物のルーターはこのようになります。後で個別のテストごとに新しいルーターをインスタンス化できるように、ルートとは別にエクスポートしていることに注目してください。

```js
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  {
    path: '/',
    component: {
      template: 'Welcome to the blogging app'
    }
  },
  {
    path: '/posts',
    component: Posts
  }
];

const router = createRouter({
  history: createWebHistory(),
  routes: routes,
})

export { routes };

export default router;
```

Vue Router を使用したアプリのテスト方法を説明するには、警告に導かれるのが一番良い方法です。以下の最小限のテストは、私たちを前進させるのに十分です:

```js
import { mount } from '@vue/test-utils'

test('routing', () => {
  const wrapper = mount(App)
  expect(wrapper.html()).toContain('Welcome to the blogging app')
})
```

テストは失敗します。また、2 つの警告が表示されます:

```bash
console.warn node_modules/@vue/runtime-core/dist/runtime-core.cjs.js:39
  [Vue warn]: Failed to resolve component: router-link

console.warn node_modules/@vue/runtime-core/dist/runtime-core.cjs.js:39
  [Vue warn]: Failed to resolve component: router-view
```

`<router-link>` と `<router-view>` コンポーネントが見つかりません。Vue Router をインストールする必要があります! Vue Router はプラグインなので、`global.plugins` のマウントオプションでインストールします。

```js {10,11,12}
import { mount } from '@vue/test-utils'
import { createRouter, createWebHistory } from 'vue-router'
import { routes } from "@/router" // This import should point to your routes file declared above

const router = createRouter({
  history: createWebHistory(),
  routes: routes,
})

test('routing', () => {
  const wrapper = mount(App, {
    global: {
      plugins: [router]
    }
  })
  expect(wrapper.html()).toContain('Welcome to the blogging app')
})
```

2 つの警告は消えましたが、今度は別の警告があります:

```js
console.warn node_modules/vue-router/dist/vue-router.cjs.js:225
  [Vue Router warn]: Unexpected error when starting the router: TypeError: Cannot read property '_history' of null
```

警告からは完全にはわかりませんが、**Vue Router 4 がルーティングを非同期で処理する** ことに関連しています。

Vue Router は、ルーターの準備ができたことを教えてくれる `isReady` 関数を提供しています。そして、最初のナビゲーションが行われたことを確認するために、この関数を `await` することができます。

```js {11,12}
import { mount } from '@vue/test-utils'
import { createRouter, createWebHistory } from 'vue-router'
import { routes } from "@/router"

const router = createRouter({
  history: createWebHistory(),
  routes: routes,
})

test('routing', async () => {
  router.push('/')

  // After this line, router is ready
  await router.isReady()

  const wrapper = mount(App, {
    global: {
      plugins: [router]
    }
  })
  expect(wrapper.html()).toContain('Welcome to the blogging app')
})
```

これでテストは合格です! かなり大変でしたが、これでアプリケーションが初期ルートに正しくナビゲートされることを確認できました。

では、`/posts` に移動して、ルーティングが期待通りに動作していることを確認しましょう:

```js {19,20}
import { mount } from '@vue/test-utils'
import { createRouter, createWebHistory } from 'vue-router'
import { routes } from "@/router"

const router = createRouter({
  history: createWebHistory(),
  routes: routes,
})

test('routing', async () => {
  router.push('/')
  await router.isReady()

  const wrapper = mount(App, {
    global: {
      plugins: [router]
    }
  })
  expect(wrapper.html()).toContain('Welcome to the blogging app')

  await wrapper.find('a').trigger('click')
  expect(wrapper.html()).toContain('Testing Vue Router')
})
```

またもや、やや不可解なエラーです:

```js
console.warn node_modules/@vue/runtime-core/dist/runtime-core.cjs.js:39
  [Vue warn]: Unhandled error during execution of native event handler
    at <RouterLink to="/posts" >

console.error node_modules/@vue/runtime-core/dist/runtime-core.cjs.js:211
  TypeError: Cannot read property '_history' of null
```

ここでも、Vue Router 4 の新しい非同期の性質により、ルーティングが完了するのを `await` してからアサーションを行う必要があります。

しかし、この場合、待ち受けることができる _hasNavigated_ フックがありません。代替案としては、Vue Test Utils からエクスポートされた `flushPromises` 関数を使用することです:

```js {1,20}
import { mount, flushPromises } from '@vue/test-utils'
import { createRouter, createWebHistory } from 'vue-router'
import { routes } from "@/router"

const router = createRouter({
  history: createWebHistory(),
  routes: routes,
})

test('routing', async () => {
  router.push('/')
  await router.isReady()

  const wrapper = mount(App, {
    global: {
      plugins: [router]
    }
  })
  expect(wrapper.html()).toContain('Welcome to the blogging app')

  await wrapper.find('a').trigger('click')
  await flushPromises()
  expect(wrapper.html()).toContain('Testing Vue Router')
})
```

_やっと_ 通過です。 素晴らしい! ただし、これはすべて非常に手動です。これは、小さくて簡単なアプリ用です。 これが、Vue Test Utils を使用して Vue コンポーネントをテストするときに、モックルーターを使用することが一般的なアプローチである理由です。 実際のルーターを引き続き使用したい場合は、次のように、各テストでルーターの独自のインスタンスを使用する必要があることに注意してください:

```js {1,20}
import { mount, flushPromises } from '@vue/test-utils'
import { createRouter, createWebHistory } from 'vue-router'
import { routes } from "@/router"

let router;
beforeEach(async () => {
  router = createRouter({
    history: createWebHistory(),
    routes: routes,
  })
});

test('routing', async () => {
  router.push('/')
  await router.isReady()

  const wrapper = mount(App, {
    global: {
      plugins: [router]
    }
  })
  expect(wrapper.html()).toContain('Welcome to the blogging app')

  await wrapper.find('a').trigger('click')
  await flushPromises()
  expect(wrapper.html()).toContain('Testing Vue Router')
})
```

## モック化したルーターを Composition API で使用する {#using-a-mocked-router-with-composition-api}

Vue router 4 では、`setup` 関数内部でルーターやルートを Composition API で操作することができます。

同じデモのコンポーネントを Composition API で書き換えたものを考えてみましょう。

```js
import { useRouter, useRoute } from 'vue-router'

const Component = {
  template: `<button @click="redirect">Click to Edit</button>`,
  props: ['isAuthenticated'],
  setup (props) {
    const router = useRouter()
    const route = useRoute()

    const redirect = () => {
      if (props.isAuthenticated) {
        router.push(`/posts/${route.params.id}/edit`)
      } else {
        router.push('/404')
      }
    }

    return {
      redirect
    }
  }
}
```

今回はコンポーネントをテストするために、jestの 機能であるインポートリソース、`vue-router` のモックを使い、ルーターとルートの両方を直接モックします。

```js
import { useRouter, useRoute } from 'vue-router'

jest.mock('vue-router', () => ({
  useRoute: jest.fn(),
  useRouter: jest.fn(() => ({
    push: () => {}
  }))
}))

test('allows authenticated user to edit a post', () => {
  useRoute.mockImplementationOnce(() => ({
    params: {
      id: 1
    }
  }))

  const push = jest.fn()
  useRouter.mockImplementationOnce(() => ({
    push
  }))

  const wrapper = mount(Component, {
    props: {
      isAuthenticated: true
    },
    global: {
      stubs: ["router-link", "router-view"], // Stubs for router-link and router-view in case they're rendered in your template
    }
  })

  await wrapper.find('button').trigger('click')

  expect(push).toHaveBeenCalledTimes(1)
  expect(push).toHaveBeenCalledWith('/posts/1/edit')
})

test('redirect an unauthenticated user to 404', () => {
  useRoute.mockImplementationOnce(() => ({
    params: {
      id: 1
    }
  }))

  const push = jest.fn()
  useRouter.mockImplementationOnce(() => ({
    push
  }))

  const wrapper = mount(Component, {
    props: {
      isAuthenticated: false
    }
    global: {
      stubs: ["router-link", "router-view"], // Stubs for router-link and router-view in case they're rendered in your template
    }
  })

  await wrapper.find('button').trigger('click')

  expect(push).toHaveBeenCalledTimes(1)
  expect(push).toHaveBeenCalledWith('/404')
})
```

## Composition API で本物のルーターを使う {#using-a-real-router-with-composition-api}

Composition API で本物のルーターを使うのは、Options API で本物のルーターを使うのと同じです。Options API の場合と同様、
ルーターをアプリから直接インポートするのではなく、 テストごとに新しいルーターオブジェクトをインスタンス化するのがよい方法だと考えていることに注意しましょう。

```js
import { mount } from '@vue/test-utils'
import { createRouter, createWebHistory } from 'vue-router'
import { routes } from "@/router"

let router;

beforeEach(async () => {
  router = createRouter({
    history: createWebHistory(),
    routes: routes,
  })

  router.push('/')
  await router.isReady()
});

test('allows authenticated user to edit a post', async () => {
  const wrapper = mount(Component, {
    props: {
      isAuthenticated: true
    },
    global: {
      plugins: [router],
    }
  })

  const push = jest.spyOn(router, 'push')
  await wrapper.find('button').trigger('click')

  expect(push).toHaveBeenCalledTimes(1)
  expect(push).toHaveBeenCalledWith('/posts/1/edit')
})
```

また、手動でないアプローチを好む人のために、Posva によって作成された [vue-router-mock](https://github.com/posva/vue-router-mock) というライブラリも代替手段として用意されています。

## 結論 {#conclusion}

- 本物のルーターインスタンスをテストに使用することができます。
- ただし、いくつか注意点があります。Vue Router 4 は非同期なので、テストを書くときにそれを考慮する必要があります。
- より複雑なアプリケーションでは、ルーターの依存関係をモックし、根本的なロジックのテストに集中することを検討してください。
- 可能であれば、テストランナーのスタブ/モッキング機能を利用しましょう。
- `global.mocks` を使って、`this.$route` や `this.$router` のようなグローバルな依存性をモックします。
