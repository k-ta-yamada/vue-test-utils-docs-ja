# Transitions

一般に、transition の結果の DOM をテストしたい場合があります。そのため、Vue Test Utils はデフォルトで `<transition>` と `<transition-group>` をモックしています。

以下は、フェード遷移で包まれたコンテンツをトグルさせるシンプルなコンポーネントです:

```vue
<template>
  <button @click="show = !show">Toggle</button>

  <transition name="fade">
    <p v-if="show">hello</p>
  </transition>
</template>

<script>
import { ref } from 'vue'

export default {
  setup() {
    const show = ref(false)

    return {
      show
    }
  }
}
</script>

<style lang="css">
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.5s ease;
}

.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
</style>
```

Vue Test Utils は組み込みの transition をスタブ化するので、他のコンポーネントをテストするのと同じように、上記のコンポーネントをテストすることができます:

```js
import Component from './Component.vue'
import { mount } from '@vue/test-utils'

test('works with transitions', async () => {
  const wrapper = mount(Component)

  expect(wrapper.find('hello').exists()).toBe(false)

  await wrapper.find('button').trigger('click')

  // After clicking the button, the <p> element exists and is visible
  expect(wrapper.get('p').text()).toEqual('hello')
})
```
