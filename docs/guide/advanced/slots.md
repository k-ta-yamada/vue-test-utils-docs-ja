# Slots

Vue Test Utils は、`slots` を使ったコンポーネントのテストに便利な機能をいくつか提供しています。

## A Simple Example

一般的な `<layout>` コンポーネントがあり、デフォルトのスロットを使用してコンテンツをレンダリングする場合があります。例えば:

```js
const Layout = {
  template: `
    <div>
      <h1>Welcome!</h1>
      <main>
        <slot />
      </main>
      <footer>
        Thanks for visiting.
      </footer>
    </div>
  `
}
```

デフォルトのスロットコンテンツがレンダリングされることを確認するためのテストを書きたいと思うかもしれません。VTU はこの目的のために、`slots` のマウントオプションを提供しています:

```js
test('layout default slot', () => {
  const wrapper = mount(Layout, {
    slots: {
      default: 'Main Content'
    }
  })

  expect(wrapper.html()).toContain('Main Content')
})
```

合格です! この例では、デフォルトスロットにテキストコンテンツを渡しています。もっと具体的に、デフォルトスロットのコンテンツが `<main>` の内部でレンダリングされることを確認したい場合は、 アサーションを変更します。

```js
test('layout default slot', () => {
  const wrapper = mount(Layout, {
    slots: {
      default: 'Main Content'
    }
  })

  expect(wrapper.find('main').text()).toContain('Main Content')
})
```

## Named Slots

より複雑な `<layout>` コンポーネントで、いくつかの名前付きスロットを使用することができます。例えば:

```js
const Layout = {
  template: `
    <div>
      <header>
        <slot name="header" />
      </header>

      <main>
        <slot name="main" />
      </main>
      <footer>
        <slot name="footer" />
      </footer>
    </div>
  `
}
```

VTU も対応しています。以下のようにテストを書くことができます。この例では、スロットにテキストコンテンツではなく、HTML を渡していることに注意してください。

```js
test('layout full page layout', () => {
  const wrapper = mount(Layout, {
    slots: {
      header: '<div>Header</div>',
      main: '<div>Main Content</div>',
      footer: '<div>Footer</div>'
    }
  })

  expect(wrapper.html()).toContain('<div>Header</div>')
  expect(wrapper.html()).toContain('<div>Main Content</div>')
  expect(wrapper.html()).toContain('<div>Footer</div>')
})
```

## Multiple Slots

スロットの配列も渡すことができます:

```js
test('layout full page layout', () => {
  const wrapper = mount(Layout, {
    slots: {
      default: [
        '<div id="one">One</div>',
        '<div id="two">Two</div>'
      ]
    }
  })

  expect(wrapper.find('#one').exists()).toBe(true)
  expect(wrapper.find('#two').exists()).toBe(true)
})
```

## Advanced Usage

また、レンダー関数、テンプレート付きオブジェクト、あるいは `vue` ファイルからインポートした SFC をスロット取り付けオプションに渡すことができます。

```js
import { h } from 'vue'
import Header from './Header.vue'

test('layout full page layout', () => {
  const wrapper = mount(Layout, {
    slots: {
      header: Header
      main: h('div', 'Main Content'),
      sidebar: { template: '<div>Sidebar</div>' },
      footer: '<div>Footer</div>',
    }
  })

  expect(wrapper.html()).toContain('<div>Header</div>')
  expect(wrapper.html()).toContain('<div>Main Content</div>')
  expect(wrapper.html()).toContain('<div>Footer</div>')
})
```

その他の例や使用例については、[テストを参照](https://github.com/vuejs/test-utils/blob/9d3c2a6526f3d8751d29b2f9112ad2a3332bbf52/tests/mountingOptions/slots.spec.ts#L124-L167) してください。

## Scoped Slots

[スコープ付きスロット](https://v3.vuejs.org/guide/component-slots.html#scoped-slots) とバインディングもサポートされています。

```js
const ComponentWithSlots = {
  template: `
    <div class="scoped">
      <slot name="scoped" v-bind="{ msg }" />
    </div>
  `,
  data() {
    return {
      msg: 'world'
    }
  }
}

test('scoped slots', () => {
  const wrapper = mount(ComponentWithSlots, {
    slots: {
      scoped: `<template #scoped="scope">
        Hello {{ scope.msg }}
        </template>
      `
    }
  })

  expect(wrapper.html()).toContain('Hello world')
})
```

スロットの内容に文字列テンプレートを使用する場合、**`<template #scoped="scopeVar">` タグを使用して明示的に定義されていなければ**、スロットが評価されたときにスロットスコープは `params` オブジェクトとして利用可能になります。

```js
test('scoped slots', () => {
  const wrapper = mount(ComponentWithSlots, {
    slots: {
      scoped: `Hello {{ params.msg }}` // no wrapping template tag provided, slot scope exposed as "params"
    }
  })

  expect(wrapper.html()).toContain('Hello world')
})
```

## 結論 {#conclusion}

- `slots` マウントオプションを使用して、`<slot>` を使用するコンポーネントがコンテンツを正しくレンダリングしているかどうかをテストします。
- コンテンツには、文字列、レンダー関数、インポートされた SFC のいずれかを指定できます。
- デフォルトのスロットは `default`、名前付きスロットは正しい名前を使います。
- scopedスロットと `#` の省略形もサポートされています。
