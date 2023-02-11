# Component Instance

[`mount`](/api/#mount) は、Vue コンポーネントをテストするための便利なメソッドをたくさん持つ `VueWrapper` を返します。時には、基礎となる Vue のインスタンスにアクセスしたい場合があります。その場合は、`vm` プロパティでアクセスできます。

## 簡単な例 {#a-simple-example}

ここでは、props と data を組み合わせて挨拶をレンダリングするシンプルなコンポーネントを紹介します:

```ts
test('renders a greeting', () => {
  const Comp = {
    template: `<div>{{ msg1 }} {{ msg2 }}</div>`,
    props: ['msg1'],
    data() {
      return {
        msg2: 'world'
      }
    }
  }

  const wrapper = mount(Comp, {
    props: {
      msg1: 'hello'
    }
  })

  expect(wrapper.html()).toContain('hello world')
})
```

`console.log(wrapper.vm)` で、`vm` 上で何が利用できるのか見てみましょう:

```js
{
  msg1: [Getter/Setter],
  msg2: [Getter/Setter],
  hasOwnProperty: [Function]
}
```

`msg1` と `msg2` の両方が表示されます! `methods` や `computed` プロパティのようなものも、それらが定義されていれば表示されます。テストを書くとき、一般的には DOM に対してアサーションすることをお勧めしますが（`wrapper.html()` のようなものを使う）、稀に、基礎となる Vue インスタンスにアクセスする必要があるかもしれません。

## `getComponent` と `findComponent` を用いた使用法 {#usage-with-get-component-and-find-component}

`getComponent` と `findComponent` は、`mount` から取得したものと同じように `VueWrapper` を返します。これは、`getComponent` や `findComponent` の結果に対して、`vm` を含むすべての同じプロパティにアクセスできることを意味します。

ここで簡単な例を挙げてみましょう:

```js
test('asserts correct props are passed', () => {
  const Foo = {
    props: ['msg'],
    template: `<div>{{ msg }}</div>`
  }

  const Comp = {
    components: { Foo },
    template: `<div><foo msg="hello world" /></div>`
  }

  const wrapper = mount(Comp)

  expect(wrapper.getComponent(Foo).vm.msg).toBe('hello world')
  expect(wrapper.getComponent(Foo).props()).toEqual({ msg: 'hello world' })
})
```

これをより徹底的にテストするには、レンダリングされたコンテンツに対してアサーションするのがよいでしょう。こうすることで、正しい prop が渡され、*かつ* レンダリングされることを保証することになります。

::: tip

注意: `<script setup>` コンポーネントを使用している場合、`vm` は使用できません。これは、`<script setup>` コンポーネントが[デフォルトで閉じられている](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0040-script-setup.md#exposing-components-public-interface) からです。これらのコンポーネントや一般的には、`vm` を避け、レンダリングされたマークアップに対してアサートすることを検討してください。
:::

:::warning CSS セレクタ使用時の WrapperLike タイプ
例えば、 `wrapper.findComponent('.foo')` を使うとき、VTU は `WrapperLike` タイプを返します。これは、機能的なコンポーネントが
`DOMWrapper` を必要とし、そうでなければ `VueWrapper` を必要とするからです。正しいコンポーネントタイプを提供することで、`VueWrapper` を返すように強制することができます。

```typescript
wrapper.findComponent('.foo') // returns WrapperLike
wrapper.findComponent<typeof FooComponent>('.foo') // returns VueWrapper
wrapper.findComponent<DefineComponent>('.foo') // returns VueWrapper
```
:::

## 結論 {#conclusion}

- Vue の内部インスタンスにアクセスするために `vm` を使用します。
- `getComponent` と `findComponent` は、Vue のラッパーを返します。これらの Vue インスタンスは、`vm` を介して利用することもできます。
