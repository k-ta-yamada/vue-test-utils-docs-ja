# `v-model` のテスト {#testing-v-model}

`v-model` の相互作用（`update:modelValue` イベント）に依存するコンポーネントを記述する場合、`event` と `props` を処理する必要があります。

["vmodel integration" ディスカッション](https://github.com/vuejs/test-utils/discussions/279) で、コミュニティによる解決策を確認してください。

[VueJS の VModelイベントのドキュメント](https://vuejs.org/guide/components/events.html#usage-with-v-model) を確認してください。

## 簡単な例 {#a-simple-example}

ここでは、シンプルなEditorコンポーネントを紹介します:

```js
const Editor = {
  props: {
    label: String,
    modelValue: String
  },
  emits: ['update:modelValue'],
  template: `<div>
    <label>{{label}}</label>
    <input :value="modelValue" @input="$emit('update:modelValue', $event.target.value)">
  </div>`
}
```

このコンポーネントは、単に input コンポーネントとして動作します:

```js
const App {
  components: {
    Editor
  },
  template: `<editor v-model="text" label="test" />`,
  data(){
    return {
      text: 'test'
    }
  }
}
```

これで、input にタイプすると、コンポーネント上の `text` が更新されます。

この動作をテストするために:

```js
test('modelValue should be updated', async () => {
  const wrapper = mount(Editor, {
    props: {
      modelValue: 'initialText',
      'onUpdate:modelValue': (e) => wrapper.setProps({ modelValue: e })
    }
  })

  await wrapper.find('input').setValue('test')
  expect(wrapper.props('modelValue')).toBe('test')
})
```

## 複数の `v-model` {#multiple-v-model}

状況によっては、特定のプロパティをターゲットとする複数の `v-model` を持つことができます。

Money Editor の例では、`currency` と `modelValueの` プロパティを持つことができます。

```js
const MoneyEditor = {
  template: `<div> 
    <input :value="currency" @input="$emit('update:currency', $event.target.value)"/>
    <input :value="modelValue" type="number" @input="$emit('update:modelValue', $event.target.value)"/>
  </div>`,
  props: ['currency', 'modelValue'],
  emits: ['update:currency', 'update:modelValue']
}
```

両方をテストすることができます:

```js
test('modelValue and currency should be updated', async () => {
  const wrapper = mount(MoneyEditor, {
    props: {
      modelValue: 'initialText',
      'onUpdate:modelValue': (e) => wrapper.setProps({ modelValue: e }),
      currency: '$',
      'onUpdate:currency': (e) => wrapper.setProps({ currency: e })
    }
  })

  const [currencyInput, modelValueInput] = wrapper.findAll('input')
  await modelValueInput.setValue('test')
  await currencyInput.setValue('£')

  expect(wrapper.props('modelValue')).toBe('test')
  expect(wrapper.props('currency')).toBe('£')
})
```
