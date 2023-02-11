# Reusability & Composition

Mostly:

- `global.mixins`.
- `global.directives`.

## コンポーザブルのテスト {#testing-composables}

コンポジション API を使用してコンポーザブルを作成する場合、コンポーザブルだけをテストしたいことがよくあります。
まずは簡単な例から見てみましょう:

```typescript
export function useCounter() {
  const counter = ref(0)

  function increase() {
    counter.value += 1
  }

  return { counter, increase }
}
```

この場合、実際には `@vue/test-utils` は必要ありません。以下は、対応するテストです:

```typescript
test('increase counter on call', () => {
  const { counter, increase } = useCounter()

  expect(counter.value).toBe(0)

  increase()

  expect(counter.value).toBe(1)
})
```

`onMounted` や `provide`/`inject` 処理などのライフサイクルフックを使うような、 より複雑なコンポーザブルの場合は、
シンプルなテストヘルパーコンポーネントを作成することができます。次のコンポーザブルは、`onMounted` フックの中でユーザーデータを取得します。

```typescript
export function useUser(userId) {
  const user = ref()
  
  function fetchUser(id) {
    axios.get(`users/${id}`)
      .then(response => (user.value = response.data))
  }

  onMounted(() => fetchUser(userId))

  return { user }
}
```

このコンポーザブルをテストするには、テストの中に簡単な `TestComponent` を作成します。`TestComponent` は、実際のコンポーネントが使用するのとまったく同じように、
このコンポーザブルを使用しなければなりません。

```typescript
// Mock API request
jest.spyOn(axios, 'get').mockResolvedValue({ data: { id: 1, name: 'User' } })

test('fetch user on mount', async () => {
  const TestComponent = defineComponent({
    props: {
      // 異なる入力引数で composable をテストするための props を定義する。
      userId: {
        type: Number,
        required: true
      }
    },
    setup (props) {
      return {
        // コンポーザブルを呼び出して、全ての戻り値をコンポーネントインスタンスに公開し、
        // wrapper.vmでアクセスできるようにします。
        ...useUser(props.userId)
      }
    }
  })

  const wrapper = mount(TestComponent, {
    props: {
      userId: 1
    }
  })

  expect(wrapper.vm.user).toBeUndefined()

  await flushPromises()

  expect(wrapper.vm.user).toEqual({ id: 1, name: 'User' })
})
```

## Provide / inject

Vue は `provide` と `inject` ですべての子コンポーネントに props を渡す方法を提供しています。この動作をテストする最良の方法は、
ツリー全体（親 + 子）をテストすることです。しかし、ツリーが複雑すぎたり、
単一のコンポーザブルだけをテストしたい場合など、これが不可能な場合もあります。

### `provide` のテスト {#testing-provide}

テストしたい次のようなコンポーネントを想定してみましょう:
```vue
<template>
  <div>
    <slot />
  </div>
</template>

<script setup>
provide('my-key', 'some-data')
</script>
```

この場合、実際の子コンポーネントをレンダリングして `provide` の正しい使い方をテストするか、
あるいは単純なテスト用ヘルパーコンポーネントを作成してそれをデフォルトのスロットに渡すことができます。

```typescript
test('provides correct data', () => {
  const TestComponent = defineComponent({
    template: '<span id="provide-test">{{value}}</span>',
    setup () {
      const value = inject('my-key')
      return { value }
    }
  })

  const wrapper = mount(ParentComponent, {
    slots: {
      default: () => h(TestComponent)
    }
  })

  expect(wrapper.find('#provide-test').text()).toBe('some-data')
})
```

コンポーネントにスロットがない場合は、[`stub`](./stubs-shallow-mount.md#stubbing-a-single-child-component) を使用して
子コンポーネントをテストヘルパーに置き換えることができます。

```vue
<template>
  <div>
    <SomeChild />
  </div>
</template>

<script setup>
import SomeChild from './SomeChild.vue'

provide('my-key', 'some-data')
</script>
```

そしてテスト:

```typescript
test('provides correct data', () => {
  const TestComponent = defineComponent({
    template: '<span id="provide-test">{{value}}</span>',
    setup () {
      const value = inject('my-key')
      return { value }
    }
  })

  const wrapper = mount(ParentComponent, {
    global: {
      stubs: {
        SomeChild: TestComponent
      }
    }
  })

  expect(wrapper.find('#provide-test').text()).toBe('some-data')
})
```

### `inject` のテスト {#testing-inject}

Component が `inject` を使用していて、`provide` でデータを渡す必要がある場合、`global.provide` オプションを使用することができます。

```vue
<template>
  <div>
    {{ value }}
  </div>
</template>

<script setup>
const value = inject('my-key')
</script>
```

ユニットテストは単純に次のようになります:

```typescript
test('renders correct data', () => {
  const wrapper = mount(MyComponent, {
    global: {
      provide: {
        'my-key': 'some-data'
      }
    }
  })

  expect(wrapper.text()).toBe('some-data')
})
```

## 結論 {#conclusion}

- 簡単なコンポーザブルは、コンポーネントと `@vue/test-utils` を使わずにテストします。
- より複雑なコンポーザブルのテストは、テストヘルパーコンポーネントを作成します
- `provide` で正しいデータを提供するためのテストヘルパーコンポーネントを作成します。
- `global.provide` を使って、`inject` を使うコンポーネントにデータを渡します。
