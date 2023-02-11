# 条件付きレンダリング {#conditional-rendering}

Vue Test Utils には、コンポーネントのレンダリングと状態についてのアサーションを行うためのさまざまな機能があり、コンポーネントの動作が正しいかどうかを確認することを目的としています。この記事では、コンポーネントをレンダリングする方法と、コンポーネントがコンテンツを正しくレンダリングしているかどうかを検証する方法を探ります。

この記事は [short video](https://www.youtube.com/watch?v=T3CHtGgEFTs&list=PLC2LZCNWKL9ahK1IoODqYxKu5aA9T5IOA&index=15) でもご覧いただけます。

## 要素を探す {#finding-elements}

Vue の最も基本的な機能の 1 つは、`v-if` を使って動的に要素を挿入したり削除したりする機能です。ここでは、`v-if` を使ったコンポーネントのテスト方法について見ていきましょう。

```js
const Nav = {
  template: `
    <nav>
      <a id="profile" href="/profile">My Profile</a>
      <a v-if="admin" id="admin" href="/admin">Admin</a>
    </nav>
  `,
  data() {
    return {
      admin: false
    }
  }
}
```

`<Nav>` コンポーネントでは、ユーザーのプロファイルへのリンクを表示します。さらに、`admin` の値が `true` の場合、admin セクションへのリンクが表示されます。3 つのシナリオがありますが、これらは正しく動作していることを確認する必要があります:

1. `/profile` のリンクが表示されていること。
2. ユーザーが管理者である場合、`/admin` のリンクを表示すること。
3. ユーザーが管理者でない場合、`/admin`のリンクは表示されないこと。

## 　`get()` の使用法 {#using-get}

`wrapper` には、既存の要素を検索するための `get()` メソッドがあります。[`querySelector`](https://developer.mozilla.org/en-US/docs/Web/API/Document/querySelector) の構文が使用されます。

`get()` を使用することで、プロファイルリンクの内容をアサートすることができます:

```js
test('renders a profile link', () => {
  const wrapper = mount(Nav)

  // Here we are implicitly asserting that the
  // element #profile exists.
  const profileLink = wrapper.get('#profile')

  expect(profileLink.text()).toEqual('My Profile')
})
```

`get()` がセレクターに一致する要素を返さない場合、エラーが発生し、テストは失敗します。 `get()` は、要素が見つかった場合、`DOMWrapper` を返します。`DOMWrapper` は、[Wrapper API](/api/#wrapper-methods) を実装した、DOM 要素の薄いラッパーです - だからこそ、 `profileLink.text()` を実行して、テキストにアクセスすることができるのです。`element` プロパティを使えば、生のエレメントにアクセスすることができます。

[`getComponent`](/api/#getcomponent) から返されるラッパーには、同じように動作する別のタイプ、`VueWrapper` があります。

## `find()` と `exists()` の使用法 {#using-find-and-exists}

`get()` は、要素が存在することを前提に動作し、存在しない場合はエラーを投げます。存在を証明するために使用することはお勧め _しません_。

そのために、`find()` と `exists()` を使用します。次のテストでは、`admin` が `false` (デフォルト) の場合、`admin` リンクが存在しないことを保証します:

```js
test('does not render an admin link', () => {
  const wrapper = mount(Nav)

  // Using `wrapper.get` would throw and make the test fail.
  expect(wrapper.find('#admin').exists()).toBe(false)
})
```

`.find()` から返された値に対して `exists()` を呼び出していることに注意してください。`find()` も `mount()` と同様、ラッパーを返します。`mount()` は Vue コンポーネントをラップしているため、いくつかの追加メソッドがあり、`find()` は通常の DOM ノードを返すだけですが、多くのメソッドは両者で共有されています。その他にも、DOM ノードが持つクラスを取得する `classes()` や、ユーザーとの対話をシミュレートする `trigger()` などのメソッドがあります。サポートするメソッドの一覧は [こちら](../../api/#wrapper-methods) でご確認いただけます。

## `data` の使用法 {#using-data}

最後のテストは、`admin` が `true` のときに admin リンクがレンダリングされることを保証するものです。デフォルトでは `false` ですが、 `mount()` の二番目の引数である [マウントオプション](../../api/#mount-options) で上書きすることができます。

`data` には、その名も `data` オプションを使用します:

```js
test('renders an admin link', () => {
  const wrapper = mount(Nav, {
    data() {
      return {
        admin: true
      }
    }
  })

  // Again, by using `get()` we are implicitly asserting that
  // the element exists.
  expect(wrapper.get('#admin').text()).toEqual('Admin')
})
```

`data` に他のプロパティがある場合でも心配ありません。Vue Test Utils が 2 つをマージします。マウントオプションの `data` は、デフォルト値よりも優先されます。

その他のマウントオプションについては、[データの受け渡し](../essentials/passing-data.md) または [マウントオプション](../../api/#mount-options) をご覧ください。

## 要素の visibility を確認する {#checking-elements-visibility}

ある要素を DOM に残したまま、その要素だけを隠したり見せたりしたいことがあります。Vue はそのようなシナリオのために `v-show` を提供しています。(`v-if` と `v-show` の違いは [こちら](https://ja.vuejs.org/guide/essentials/conditional.html#v-if-vs-v-show) で確認できます）。

`v-show` を使ったコンポーネントはこのような感じです:

```js
const Nav = {
  template: `
    <nav>
      <a id="user" href="/profile">My Profile</a>
      <ul v-show="shouldShowDropdown" id="user-dropdown">
        <!-- dropdown content -->
      </ul>
    </nav>
  `,
  data() {
    return {
      shouldShowDropdown: false
    }
  }
}
```

このシナリオでは、要素は表示されていませんが、常にレンダリングされています。 `get()` や `find()` は常に `Wrapper` を返し、 `.exists()` と共に `find()` は常に `true` を返します - なぜなら要素はまだ DOM 内にあるからです。

## `isVisible()` の使用法 {#using-is-visible}

`isVisible()` は、隠された要素をチェックする機能を提供します。特に、`isVisible()` は次のような場合にチェックします:

- 要素またはその祖先が `display: none`, `visibility: hidden`, `opacity :0` のスタイルである
- 要素またはその祖先が、折りたたまれた `<details>` タグの中にある
- 要素またはその祖先が `hidden` 属性を持っている

これらの場合、`isVisible()` は `false` を返します。

`v-show` を使ったテストシナリオは、次のようになります:

```js
test('does not show the user dropdown', () => {
  const wrapper = mount(Nav)

  expect(wrapper.get('#user-dropdown').isVisible()).toBe(false)
})
```

## 結論 {#conclusion}

- ある要素が DOM 内にあるかどうかを調べるには、`find()` と `exists()` を使用します。
- 要素が DOM 内にあることが期待される場合は `get()` を使用します。
- `data` マウントオプションは、コンポーネントにデフォルト値を設定するために使用されます。
- DOM 内にある要素の可視性を確認するには、 `get()` と `isVisible()` を併用します。
