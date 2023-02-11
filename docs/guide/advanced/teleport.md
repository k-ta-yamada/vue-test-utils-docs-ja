# Teleport のテスト {#testing-teleport}

Vue 3 には、新しい組み込みコンポーネントが搭載されています。`<Teleport>` は、コンポーネントが自身の `<template>` のはるか外側にそのコンテンツを「テレポート」することを可能にします。Vue Test Utils で書かれたテストのほとんどは、`mount` に渡されたコンポーネントにスコープされています。これは、最初にレンダリングされたコンポーネントの外側にテレポートされたコンポーネントをテストする際に、いくつかの複雑な問題を引き起こします。

ここでは、`<Teleport>` を使用したコンポーネントのテストに関する戦略やテクニックを紹介します。

::: tip
もし、テレポートを無視してコンポーネントの残りの部分をテストしたい場合は、[global stubs option](../../api/#global-stubs) で `teleport: true` を渡すことで、`teleport` をスタブ化できます。
:::

## 例 {#example}

この例では、`<Navbar>` コンポーネントをテストしています。これは、`<Teleport>` の内部に `<Signup>` コンポーネントをレンダリングします。`<Teleport>` のターゲットプロップは、`<Navbar>` コンポーネントの外側にある要素です。

これは、`Navbar.vue` コンポーネントです:

```vue
<template>
  <Teleport to="#modal">
    <Signup />
  </Teleport>
</template>

<script lang="ts">
import { defineComponent } from 'vue'
import Signup from './Signup.vue'

export default defineComponent({
  components: {
    Signup
  }
})
</script>
```

これは単に `<Signup>` をどこかにテレポートさせるだけです。この例の目的では単純なことです。

`Signup.vue` は、`username` が 8 文字以上であるかどうかを検証するフォームです。もし 8 文字以上であれば、送信時に `username` をペイロードとする `signup` イベントを発行します。これをテストすることが私たちの目標です。

```vue
<template>
  <div>
    <form @submit.prevent="submit">
      <input v-model="username" />
    </form>
  </div>
</template>

<script>
export default {
  emits: ['signup'],
  data() {
    return {
      username: ''
    }
  },
  computed: {
    error() {
      return this.username.length < 8
    }
  },
  methods: {
    submit() {
      if (!this.error) {
        this.$emit('signup', this.username)
      }
    }
  }
}
</script>
```

## コンポーネントの実装 {#mounting-the-component}

最小限のテストから始めます:

```ts
import { mount } from '@vue/test-utils'
import Navbar from './Navbar.vue'
import Signup from './Signup.vue'

test('emits a signup event when valid', async () => {
  const wrapper = mount(Navbar)
})
```

このテストを実行すると、次のような警告が表示されます: `[Vue warn]: Failed to locate Teleport target with selector "#modal"` 。では、作ってみましょう:

```ts {5-15}
import { mount } from '@vue/test-utils'
import Navbar from './Navbar.vue'
import Signup from './Signup.vue'

beforeEach(() => {
  // create teleport target
  const el = document.createElement('div')
  el.id = 'modal'
  document.body.appendChild(el)
})

afterEach(() => {
  // clean up
  document.body.outerHTML = ''
})

test('teleport', async () => {
  const wrapper = mount(Navbar)
})
```

この例では Jest を使用していますが、Jest はテストごとに DOM をリセットするわけではありません。このため、テストごとに `afterEach` でクリーンアップするのがよいでしょう。

## テレポートコンポーネントとの相互作用 {#interacting-with-the-teleported-component}

次にやるべきことは、ユーザー名 input の入力です。残念なことに、`wrapper.find('input')` は使えません。どうしてでしょうか? `console.log(wrapper.html())` で簡単に確認できます。

```html
<!--teleport start-->
<!--teleport end-->
```

Vueが `<Teleport>` を処理するために使用するいくつかのコメントが表示されます - しかし `<input>` はありません。これは、`<Signup>` コンポーネント（とその HTML）が`<Navbar>` の内部でレンダリングされなくなったからです - それは外にテレポートされました。

実際の HTML は外部にテレポートされますが、`<Navbar>` に関連付けられた Virtual DOM は元のコンポーネントへの参照を保持していることがわかります。つまり、通常の DOM ではなく、Virtual DOM に対して操作する `getComponent` と `findComponent` を使用することができます。

```ts {12}
beforeEach(() => {
  // ...
})

afterEach(() => {
  // ...
})

test('teleport', async () => {
  const wrapper = mount(Navbar)

  wrapper.getComponent(Signup) // got it!
})
```

`getComponent` は `VueWrapper` を返します。これで、`get`、`find`、`trigger` などのメソッドが使えるようになりました。

テストを終わらせましょう:

```ts {4-8}
test('teleport', async () => {
  const wrapper = mount(Navbar)

  const signup = wrapper.getComponent(Signup)
  await signup.get('input').setValue('valid_username')
  await signup.get('form').trigger('submit.prevent')

  expect(signup.emitted().signup[0]).toEqual(['valid_username'])
})
```

パスしました!

テスト全体:

```ts
import { mount } from '@vue/test-utils'
import Navbar from './Navbar.vue'
import Signup from './Signup.vue'

beforeEach(() => {
  // create teleport target
  const el = document.createElement('div')
  el.id = 'modal'
  document.body.appendChild(el)
})

afterEach(() => {
  // clean up
  document.body.outerHTML = ''
})

test('teleport', async () => {
  const wrapper = mount(Navbar)

  const signup = wrapper.getComponent(Signup)
  await signup.get('input').setValue('valid_username')
  await signup.get('form').trigger('submit.prevent')

  expect(signup.emitted().signup[0]).toEqual(['valid_username'])
})
```

## 結論 {#conclusion}

- `document.createElement` でテレポートターゲットを作成します。
- Virtual DOM レベルで動作する `getComponent` または `findComponent` を使用して、テレポートされたコンポーネントを見つけます。
