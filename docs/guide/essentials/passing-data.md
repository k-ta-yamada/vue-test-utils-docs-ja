# コンポーネントへのデータの受け渡し {#passing-data-to-components}

Vue Test Utils は、コンポーネントに data や props を設定するいくつかの方法を提供し、さまざまなシナリオでコンポーネントの動作を完全にテストできるようにします。

このセクションでは、`data` と `props` のマウントオプションと、コンポーネントが受け取る props を動的に更新する `VueWrapper.setProps()` について説明します。

## パスワードコンポーネント {#the-password-component}

ここでは、`<Password>` コンポーネントを作成することで、上記の機能を実証します。このコンポーネントは、パスワードが長さや複雑さなどの特定の基準を満たしているかどうかを検証します。以下から始めて、機能を追加し、機能が正しく動作していることを確認するためのテストも行います:

```js
const Password = {
  template: `
    <div>
      <input v-model="password">
    </div>
  `,
  data() {
    return {
      password: ''
    }
  }
}
```

最初に追加する要件は、最小の長さです。

## `props` を使い最小限の長さを設定する {#using-props-to-set-a-minimum-length}

このコンポーネントをすべてのプロジェクトで再利用したいのですが、それぞれのプロジェクトで異なる要件があるかもしれません。このため、`minLength` を `<Password>` に渡す **prop** にすることにします:

パスワードが `minLength` より小さい場合は、エラーを表示するようにします。これは、`error` computed プロパティを作成し、`v-if` を使用して条件付きでレンダリングすることで実現できます:

```js
const Password = {
  template: `
    <div>
      <input v-model="password">
      <div v-if="error">{{ error }}</div>
    </div>
  `,
  props: {
    minLength: {
      type: Number
    }
  },
  computed: {
    error() {
      if (this.password.length < this.minLength) {
        return `Password must be at least ${this.minLength} characters.`
      }
      return
    }
  }
}
```

これをテストするには、`minLength` を設定し、さらにその数値よりも小さい `password` を設定する必要があります。これは、`data` と `props` のマウントオプションを使用して行うことができます。最後に、正しいエラーメッセージがレンダリングされることを保証します:

```js
test('renders an error if length is too short', () => {
  const wrapper = mount(Password, {
    props: {
      minLength: 10
    },
    data() {
      return {
        password: 'short'
      }
    }
  })

  expect(wrapper.html()).toContain('Password must be at least 10 characters')
})
```

`maxLength` ルールのテストを書くことは、読者のための練習として残されています! 他の書き方としては、`setValue` を使って短すぎるパスワードの入力を更新することもできます。詳しくは [Forms](./forms) で確認できます。

## `setProps` の使用 {#using-set-props}

時には、prop が変更されることによる副作用のテストを書く必要があるかもしれません。この単純な `<Show>` コンポーネントは、`show` プロパティが `true` である場合に挨拶を表示します。

```vue
<template>
  <div v-if="show">{{ greeting }}</div>
</template>

<script>

export default {
  props: {
    show: {
      type: Boolean,
      default: true
    }
  },
  data() {
    return {
      greeting: 'Hello'
    }
  }
}
</script>
```

これを完全にテストするには、`greeting` がデフォルトでレンダリングされることを確認したいと思うかもしれません。`setProps()` を使用して `show` prop を更新することができ、これにより `greeting` が隠されるようになります:

```js
import { mount } from '@vue/test-utils'
import Show from './Show.vue'

test('renders a greeting when show is true', async () => {
  const wrapper = mount(Show)
  expect(wrapper.html()).toContain('Hello')

  await wrapper.setProps({ show: false })

  expect(wrapper.html()).not.toContain('Hello')
})
```

また、`setProps()` を呼び出す際に `await` キーワードを使用して、アサーションが実行される前に DOM が更新されていることを確認します。

## 結論 {#conclusion}

- コンポーネントの状態を事前に設定するには、 `props` および `data` マウントオプションを使用します。
- `setProps()` を使用して、テスト中に prop を更新します。
- `setProps()` の前に `await` キーワードを使用して、テストが継続される前に Vue が DOM を更新することを確認します。
- コンポーネントと直接対話することで、より大きなカバレッジを得ることができます。すべてが正しく動作することを確認するために、`data` と組み合わせて `setValue` または `trigger` を使用することを検討してください。
