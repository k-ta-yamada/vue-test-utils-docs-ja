# Getting Started

Vue.js の公式テストユーティリティライブラリ、Vue Test Utils へようこそ!

これは、Vue 3 をターゲットとする Vue Test Utils v2 のドキュメントです。

簡単に言うと:

- [Vue Test Utils 1](https://github.com/vuejs/vue-test-utils/) のターゲットは [Vue 2](https://github.com/vuejs/vue/) です。
- [Vue Test Utils 2](https://github.com/vuejs/test-utils/) のターゲットは [Vue 3](https://github.com/vuejs/vue-next/) です。

## Vue Test Utilsとは? {#what-is-vue-test-utils}

Vue Test Utils (VTU) は、Vue.js コンポーネントのテストを簡素化することを目的としたユーティリティ関数のセットです。Vue コンポーネントをマウントし、分離された方法で対話するためのいくつかのメソッドを提供します。

例を見てみましょう:

```js
import { mount } from '@vue/test-utils'

// The component to test
const MessageComponent = {
  template: '<p>{{ msg }}</p>',
  props: ['msg']
}

test('displays message', () => {
  const wrapper = mount(MessageComponent, {
    props: {
      msg: 'Hello world'
    }
  })

  // Assert the rendered text of the component
  expect(wrapper.text()).toContain('Hello world')
})
```

## What Next?

Vue Test Utils の動作を確認するには、[クラッシュコースを受講してください](../guide/essentials/a-crash-course.md)。ここでは、テストファーストのアプローチでシンプルな Todo アプリを構築します。

ドキュメントは、主に2つのセクションに分かれています:

- **Essentials**, Vueコンポーネントをテストする際に直面する一般的なユースケースをカバーします。
- **Vue Test Utils in Depth**, ライブラリの他の高度な機能を探ります。

また、完全な [API](../api/) を探索することもできます。

また、ビデオで学習したい場合は、[ここで多くの講義を受けることができます](https://www.youtube.com/playlist?list=PLC2LZCNWKL9ahK1IoODqYxKu5aA9T5IOA)。
