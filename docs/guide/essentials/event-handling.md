# イベントハンドリング {#event-handling}

Vue のコンポーネントは、props を介して、また `$emit` を呼び出してイベントを発行することによって、互いに作用します。このガイドでは、 `emitted()` 関数を使用して、イベントが正しく emit されたことを確認する方法について見ていきます。

この記事は [short video](https://www.youtube.com/watch?v=U_j-nDur4oU&list=PLC2LZCNWKL9ahK1IoODqYxKu5aA9T5IOA&index=14) でもご覧いただけます。

## カウンターコンポーネント {#the-counter-component}

ここでは、シンプルな `<Counter>` コンポーネントを紹介します。これは、クリックされると内部のカウント変数を増加させ、その値を emit するボタンを備えています:

```js
const Counter = {
  template: '<button @click="handleClick">Increment</button>',
  data() {
    return {
      count: 0
    }
  },
  methods: {
    handleClick() {
      this.count += 1
      this.$emit('increment', this.count)
    }
  }
}
```

このコンポーネントを完全にテストするには、最新の `count` 値を持つ `increment` イベントが emit されることを確認する必要があります。

## emit されたイベントのアサート {#asserting-the-emitted-events}

そのためには、`emitted()` メソッドに依存します。このメソッドは、**コンポーネントが emit したすべてのイベント** と、その引数を配列にしてオブジェクトを返します。どのように動作するか見てみましょう:

```js
test('emits an event when clicked', () => {
  const wrapper = mount(Counter)

  wrapper.find('button').trigger('click')
  wrapper.find('button').trigger('click')

  expect(wrapper.emitted()).toHaveProperty('increment')
})
```

> `trigger()` を見たことがなくても、心配はいりません。これはユーザーとの対話をシミュレートするために使用されます。詳しくは [Forms](./forms) をご覧ください。

まず、`emitted()` はオブジェクトを返し、各キーが emit したイベントにマッチすることに注意してください。この場合、`increment` です。

このテストはパスするはずです。適切な名前のイベントを emit していることを確認しました。

## イベントの引数のアサート {#asserting-the-arguments-of-the-event}

しかし、もっとうまくいくはずです。`this.$emit('increment', this.count)` が呼ばれたときに、正しい引数を emit しているかどうかをチェックする必要があります。

次のステップは、イベントに `count` 値が含まれていることを確認することです。そのためには、`emitted()` に引数を渡します。

```js {9}
test('emits an event with count when clicked', () => {
  const wrapper = mount(Counter)

  wrapper.find('button').trigger('click')
  wrapper.find('button').trigger('click')

  // `emitted()` は引数を受け取ります。
  // これは `this.$emit('increment')` の出現回数をすべて含む配列を返します。
  const incrementEvent = wrapper.emitted('increment')

  // 2 回「クリック」しているので、`increment`の配列には
  // 2 つの値があるはずです。
  expect(incrementEvent).toHaveLength(2)

  // 1 回目のクリックの結果をアサートします。
  // 値が配列であることに注意してください。
  expect(incrementEvent[0]).toEqual([1])

  // 続いて、2 回目の結果です。
  expect(incrementEvent[1]).toEqual([2])
})
```

`emitted()` の出力を要約して分解してみましょう。これらのキーのそれぞれには、テスト中に emit されたさまざまな値が含まれています:

```js
// console.log(wrapper.emitted('increment'))
;[
  [1], // first time it is called, `count` is 1
  [2] // second time it is called, `count` is 2
]
```

## 複雑なイベントのアサート {#asserting-complex-events}

今、私たちの `<Counter>` コンポーネントは、追加情報を持つオブジェクトを emit する必要があると想像してください。例えば、`@increment` イベントをリッスンしている親コンポーネントに、`count` が偶数か奇数かを伝える必要があります:

```js {12-15}
const Counter = {
  template: `<button @click="handleClick">Increment</button>`,
  data() {
    return {
      count: 0
    }
  },
  methods: {
    handleClick() {
      this.count += 1

      this.$emit('increment', {
        count: this.count,
        isEven: this.count % 2 === 0
      })
    }
  }
}
```

先ほどと同様に、`<button>` 要素で `click` イベントを trigger する必要があります。そして、`emitted('increment')` を使って、正しい値が emit されるようにします。

```js
test('emits an event with count when clicked', () => {
  const wrapper = mount(Counter)

  wrapper.find('button').trigger('click')
  wrapper.find('button').trigger('click')

  // We have "clicked" twice, so the array of `increment` should
  // have two values.
  expect(wrapper.emitted('increment')).toHaveLength(2)

  // Then, we can make sure each element of `wrapper.emitted('increment')`
  // contains an array with the expected object.
  expect(wrapper.emitted('increment')[0]).toEqual([
    {
      count: 1,
      isEven: false
    }
  ])

  expect(wrapper.emitted('increment')[1]).toEqual([
    {
      count: 2,
      isEven: true
    }
  ])
})
```

オブジェクトのような複雑なイベントペイロードのテストは、数値や文字列のような単純な値のテストと何ら変わりはありません。

## Composition API {#composition-api}

Composition API を使用している場合は、`this.$emit()` の代わりに `context.emit()` を呼び出します。`emitted()` は両方からイベントを取得するので、ここで説明したのと同じ手法でコンポーネントをテストすることができます。

## 結論 {#conclusion}

- Vue コンポーネントから emit されるイベントにアクセスするには、`emitted()` を使用します。
- `emitted(eventName)` は配列を返し、各要素は放出されたイベント 1 つを表します。
- 引数は、`emitted(eventName)[index]` に、emit されたのと同じ順番で配列に格納されます。
