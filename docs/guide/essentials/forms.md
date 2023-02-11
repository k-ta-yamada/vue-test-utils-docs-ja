# Form Handling

Vue のフォームは、プレーンな HTML フォームのような単純なものから、カスタム Vue コンポーネントのフォーム要素の複雑なネストツリーになるものまであります。
ここでは、フォーム要素と対話し、値を設定し、イベントをトリガーする方法について、徐々に説明します。

最も多く使用するメソッドは、`setValue()` と `trigger()` です。

## フォーム要素との連動 {#interacting-with-form-elements}

ごく基本的なフォームを見てみましょう:

```vue
<template>
  <div>
    <input type="email" v-model="email" />

    <button @click="submit">Submit</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      email: ''
    }
  },
  methods: {
    submit() {
      this.$emit('submit', this.email)
    }
  }
}
</script>
```

### 要素値の設定 {#setting-element-values}

Vue で入力をデータにバインドする最も一般的な方法は、`v-model` を使用することです。もうご存知かと思いますが、各フォーム要素がどのようなイベントを emit するか、
そしてどのような props を受け付けるかを管理してくれるので、フォーム要素の扱いが簡単になります。

VTU で入力の値を変更するには、`setValue()` メソッドを使用します。これはパラメータ（多くの場合 `String` または `Boolean`）を受け取り、`Promise` を返し、Vue が DOM を更新した後に解決されます。

```js
test('sets the value', async () => {
  const wrapper = mount(Component)
  const input = wrapper.find('input')

  await input.setValue('my@mail.com')

  expect(input.element.value).toBe('my@mail.com')
})
```

ご覧のように、`setValue` は input 要素の `value` プロパティを渡した値に設定します。

`await` を使って、Vue の更新が完了し、DOM に変更が反映されたことを確認してから、アサーションを行っています。

### イベントのトリガー {#triggering-events}

イベントのトリガーは、フォームやアクションエレメントを扱う上で、2 番目に重要な動作です。前の例で作った `button` を見てみましょう。

```html
<button @click="submit">Submit</button>
```

クリックイベントを発生させるには、`trigger` メソッドを使用します。

```js
test('trigger', async () => {
  const wrapper = mount(Component)

  // trigger the element
  await wrapper.find('button').trigger('click')

  // assert some action has been performed, like an emitted event.
  expect(wrapper.emitted()).toHaveProperty('submit')
})
```

> `emitted()` を見たことがなくても、心配は要りません。これは、Component の emitted イベントをアサートするために使用されます。詳しくは [Event Handling](./event-handling) で説明します。

`click` イベントリスナーをトリガーして、Component が `submit` メソッドを実行するようにします。`setValue` で行ったように、`await` を使用してアクションが Vue に反映されていることを確認します。

そして、何らかのアクションが起こったことをアサートすることができます。この場合、私たちは正しいイベントを emit したということです。

この 2 つを組み合わせて、シンプルなフォームがユーザーの入力を emit しているかどうかをテストしてみましょう。

```js
test('emits the input to its parent', async () => {
  const wrapper = mount(Component)

  // set the value
  await wrapper.find('input').setValue('my@mail.com')

  // trigger the element
  await wrapper.find('button').trigger('click')

  // assert the `submit` event is emitted,
  expect(wrapper.emitted('submit')[0][0]).toBe('my@mail.com')
})
```

## Advanced workflows

さて、基本がわかったところで、より複雑な例題に飛び込んでみましょう。

### さまざまなフォーム要素を扱う {#working-with-various-form-elements}

`setValue` は input 要素で動作することを確認しましたが、様々なタイプの input 要素に値を設定できるため、より汎用的です。

ここでは、入力の種類が多い、より複雑なフォームを見てみましょう。

```vue
<template>
  <form @submit.prevent="submit">
    <input type="email" v-model="form.email" />

    <textarea v-model="form.description" />

    <select v-model="form.city">
      <option value="new-york">New York</option>
      <option value="moscow">Moscow</option>
    </select>

    <input type="checkbox" v-model="form.subscribe" />

    <input type="radio" value="weekly" v-model="form.interval" />
    <input type="radio" value="monthly" v-model="form.interval" />

    <button type="submit">Submit</button>
  </form>
</template>

<script>
export default {
  data() {
    return {
      form: {
        email: '',
        description: '',
        city: '',
        subscribe: false,
        interval: ''
      }
    }
  },
  methods: {
    async submit() {
      this.$emit('submit', this.form)
    }
  }
}
</script>
```

拡張された Vue コンポーネントは少し長くなり、入力タイプがいくつか増え、`submit` ハンドラは `<form/>` 要素に移動されました。

`input` に値を設定したのと同じ方法で、フォーム内の他のすべての入力に値を設定することができます。

```js
import { mount } from '@vue/test-utils'
import FormComponent from './FormComponent.vue'

test('submits a form', async () => {
  const wrapper = mount(FormComponent)

  await wrapper.find('input[type=email]').setValue('name@mail.com')
  await wrapper.find('textarea').setValue('Lorem ipsum dolor sit amet')
  await wrapper.find('select').setValue('moscow')
  await wrapper.find('input[type=checkbox]').setValue()
  await wrapper.find('input[type=radio][value=monthly]').setValue()
})
```

ご覧の通り、`setValue` は非常に汎用性の高いメソッドです。あらゆるタイプのフォーム要素で動作させることができます。

`await` を随所で使用しているのは、各変更が適用されたことを確認してから次の変更をトリガーするためです。これは、DOM が更新されたときにアサーションを行うことを確認するために推奨されます。

::: tip
`OPTION`、`CHECKBOX`、`RADIO` の入力に対して `setValue` にパラメータを渡さない場合、それらは `checked` として設定されます。
:::

フォームに値を設定したので、次はフォームを送信してアサーションを行う番です。

### 複雑なイベントリスナーのトリガー {#triggering-complex-event-listeners}

イベントリスナーは、必ずしも単純な `click` イベントとは限りません。Vue では、あらゆる種類の DOM イベントをリスニングしたり、`.prevent` などの特別な修飾子を追加したりすることができます。それでは、それらをどのようにテストするか見てみましょう。

上のフォームでは、イベントを `button` から `form` 要素に移動しました。これにより、`enter` キーを押すことでフォームを送信できるようになり、よりネイティブなアプローチとなります。

`submit` ハンドラを起動するために、再び `trigger` メソッドを使用します。

```js {14,16-22}
test('submits the form', async () => {
  const wrapper = mount(FormComponent)

  const email = 'name@mail.com'
  const description = 'Lorem ipsum dolor sit amet'
  const city = 'moscow'

  await wrapper.find('input[type=email]').setValue(email)
  await wrapper.find('textarea').setValue(description)
  await wrapper.find('select').setValue(city)
  await wrapper.find('input[type=checkbox]').setValue()
  await wrapper.find('input[type=radio][value=monthly]').setValue()

  await wrapper.find('form').trigger('submit.prevent')

  expect(wrapper.emitted('submit')[0][0]).toStrictEqual({
    email,
    description,
    city,
    subscribe: true,
    interval: 'monthly'
  })
})
```

イベント修飾子をテストするために、イベント文字列 `submit.prevent` を直接 `trigger` にコピーペーストしました。 `trigger` は渡されたイベントとそのすべての修飾子を読み、必要なものを選択的に適用することができます。

::: tip
`.prevent` や `.stop` などのネイティブイベント修飾子は Vue 固有のものであり、そのため、私たちはそれらをテストする必要はありません。
:::

そして、フォームが正しいイベントとペイロードを emit したかどうか、簡単なアサーションを行います。

#### ネイティブフォームの送信 {#native-form-submission}

`<form>` 要素で `submit` イベントを発生させることは、フォーム送信時のブラウザの動作を模倣することになります。もっと自然にフォーム送信のきっかけを作りたいのであれば、代わりに送信ボタンの `click` イベントをトリガーすればよいでしょう。[HTML の仕様](https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#form-submission-algorithm) として、`document` に接続されていないフォーム要素は、送信できないので、 ラッパーの要素を接続するために、[`attachTo`](../../api/#attachto) を使う必要があります。

#### 同一イベントへの複数の修飾子 {#multiple-modifiers-on-the-same-event}

非常に細かく複雑で、特殊なインタラクション処理を行うフォームがあるとします。それをテストするにはどうしたらいいでしょうか？

```html
<input @keydown.meta.c.exact.prevent="captureCopy" v-model="input" />
```

ユーザーが `cmd` + `c` をクリックしたときに処理する入力があり、それを阻止してコピーさせないようにしたいとします。このテストは、Component から `trigger()` メソッドにイベントをコピー＆ペーストすることで簡単に行うことができます。

```js
test('handles complex events', async () => {
  const wrapper = mount(Component)

  await wrapper.find(input).trigger('keydown.meta.c.exact.prevent')

  // run your assertions
})
```

Vue Test Utils は、イベントを読み込んで、イベントオブジェクトに適切なプロパティを適用します。この場合、以下のようなマッチングになります:

```js
{
  // ... other properties
  "key": "c",
  "metaKey": true
}
```

#### イベントへのデータ追加 {#adding-extra-data-to-an-event}

例えば、あなたのコードが `event` オブジェクトの内部から何かを必要とするとしましょう。このようなシナリオをテストするには、2 番目のパラメータとして余分なデータを渡します。

```vue
<template>
  <form>
    <input type="text" v-model="value" @blur="handleBlur" />
    <button>Submit</button>
  </form>
</template>

<script>
export default {
  data() {
    return {
      value: ''
    }
  },
  methods: {
    handleBlur(event) {
      if (event.relatedTarget.tagName === 'BUTTON') {
        this.$emit('focus-lost')
      }
    }
  }
}
</script>
```

```js
import Form from './Form.vue'

test('emits an event only if you lose focus to a button', () => {
  const wrapper = mount(Form)

  const componentToGetFocus = wrapper.find('button')

  wrapper.find('input').trigger('blur', {
    relatedTarget: componentToGetFocus.element
  })

  expect(wrapper.emitted('focus-lost')).toBeTruthy()
})
```

ここでは、`relatedTarget` がボタンであるかどうかを `event` オブジェクトの内部でチェックするコードを想定しています。このような要素への参照を渡すだけで、ユーザが `input` に何かを入力した後に `button` をクリックした場合に起こることを真似ることができます。

## Vue Component の入力とのインタラクション {#interacting-with-vue-component-inputs}

Input はプレーンな要素だけではありません。私たちはしばしば、input のように動作する Vue コンポーネントを使用します。これらは、マークアップやスタイリング、多くの機能を使いやすい形で追加することができます。

このような input を使用するフォームのテストは、最初は大変かもしれませんが、いくつかの簡単なルールを使えば、すぐに公園を散歩するようにできるようになります。

以下は、`label` と `input` 要素をラップした Component です:

```vue
<template>
  <label>
    {{ label }}
    <input
      type="text"
      :value="modelValue"
      @input="$emit('update:modelValue', $event.target.value)"
    />
  </label>
</template>

<script>
export default {
  name: 'CustomInput',

  props: ['modelValue', 'label']
}
</script>
```

また、この Vue コンポーネントは、あなたが入力したものをそのまま返します。使うには、次のようにします:

```html
<custom-input v-model="input" label="Text Input" class="text-input" />
```

上記のように、これらの Vue で動く input のほとんどは、その中に実際の `button` や `input` を持っています。その要素を見つけて、それを操作することも同様に簡単にできます:

```js
test('fills in the form', async () => {
  const wrapper = mount(CustomInput)

  await wrapper.find('.text-input input').setValue('text')

  // continue with assertions or actions like submit the form, assert the DOM…
})
```

### 複雑な input コンポーネントのテスト {#testing-complex-input-components}

Input コンポーネントがそれほど単純でない場合はどうなるのでしょうか? Vuetify のような UI ライブラリを使用しているかもしれません。正しい要素を見つけるためにマークアップの内部を調べることに依存している場合、外部ライブラリがその内部を変更することを決定した場合、テストは壊れるかもしれません。

そのような場合は、コンポーネントのインスタンスと `setValue` を使用して、直接値を設定することができます。

Vuetify の textarea を使用するフォームがあると仮定します:

```vue
<template>
  <form @submit.prevent="handleSubmit">
    <v-textarea v-model="description" ref="description" />
    <button type="submit">Send</button>
  </form>
</template>

<script>
export default {
  name: 'CustomTextarea',
  data() {
    return {
      description: ''
    }
  },
  methods: {
    handleSubmit() {
      this.$emit('submitted', this.description)
    }
  }
}
</script>
```

`findComponent` でコンポーネントのインスタンスを見つけ、その値を設定することができます。

```js
test('emits textarea value on submit', async () => {
  const wrapper = mount(CustomTextarea)
  const description = 'Some very long text...'

  await wrapper.findComponent({ ref: 'description' }).setValue(description)

  wrapper.find('form').trigger('submit')

  expect(wrapper.emitted('submitted')[0][0]).toEqual(description)
})
```

## 結論 {#conclusion}

- `setValue` を使用して、DOM input と Vue コンポーネントの両方で値を設定します。
- `trigger` を使用して、修飾子付きまたは修飾子なしの DOM イベントをトリガーします。
- `trigger` の 2 番目のパラメータを使用して、 イベントデータを追加します。
- DOM が変更され、正しいイベントが emit されたことを確認します。Component インスタンスにデータをアサートしないようにします。
