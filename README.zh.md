# vuera

[![explain](./minilogo.svg)](https://github.com/chinanf-boy/Source-Explain)

解释   
>"version": "0.1.3"


同时在 react 与 vue 中 运行。

- [react 在 vue](#ReactInVue)

- [vue 在 react](#VueInReact)

两种情况

先说第一种：

# ReactInVue

``` js
import Vue from 'vue'
import { VuePlugin } from 'vuera'

Vue.use(VuePlugin)
/* ... */
```

Now, use your React components like you would normally use your Vue components!

``` vue
<template>
  <div>
    <h1>I'm a Vue component</h1>
    <my-react-component :message="message" @reset="reset" />
  </div>
</template>

<script>
  import MyReactComponent from './MyReactComponent'

  export default {
    data () {
      message: 'Hello from React!',
    },
    methods: {
      reset () {
        this.message = ''
      }
    },
    components: { 'my-react-component': MyReactComponent },
  }
</script>
```

以 Vue插件的形式 用 React 

vuera/src/VuePlugin.js
``` js
// 判断是否 React的组件
import isReactComponent from './utils/isReactComponent'
// React Component -> Vue Component
import VueResolver from './resolvers/Vue'

/**
 * vue 插件
 */
export default {
  install (Vue, options) {
    /**
     * 自定义合并策略，这个策略真的只是
     * 包装所有React组件，同时保留Vue组件。
     */
    const originalComponentsMergeStrategy = Vue.config.optionMergeStrategies.components

    Vue.config.optionMergeStrategies.components = function (parent, ...args) {
      // 之前设置的 值  <-- return Object.assign(parent, wrappedComponents)
      const mergedValue = originalComponentsMergeStrategy(parent, ...args)

      // 
      const wrappedComponents = mergedValue
        ? Object.entries(mergedValue).reduce(
            (acc, [k, v]) => ({
              ...acc,
              [k]: isReactComponent(v) ? VueResolver(v) : v,
            }),
            {}
          )
        : mergedValue
        //合并
      return Object.assign(parent, wrappedComponents)
    }
  },
}

```

其中出现 

- [install()](https://cn.vuejs.org/v2/guide/plugins.html#%E5%BC%80%E5%8F%91%E6%8F%92%E4%BB%B6)

- [vue.config 是一个对象，包含 Vue 的全局配置。可以在启动应用之前修改下列属性：](https://cn.vuejs.org/v2/api/#optionMergeStrategies)

- [Object.entries](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/entries)

``` js
const obj = { foo: 'bar', baz: 42 };
console.log(Object.entries(obj)); // [ ['foo', 'bar'], ['baz', 42] ]
```
- [Array.prototype.reduce](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce)

``` js
var flattened = [[0, 1], [2, 3], [4, 5]].reduce(function(a, b) {
  return a.concat(b);
}, []);
// flattened is [0, 1, 2, 3, 4, 5]
```

- VueResolver 

vuera/src/resolvers/Vue.js
> React Component -> Vue Component

那么可以看出，使用遍历了一边 ``Vue``组件，如果发现 ``React``组件的样子 就改造成``Vue``。

再说第二种：

# VueInReact

加 `vuera/babel` 到 你的`.babelrc` `plugins` 选项

.babelrc
``` json
{
  "presets": "react",
  "plugins": ["vuera/babel"]
}
```

使用

``` jsx
import React from 'react'
import MyVueComponent from './MyVueComponent.vue'

export default () => (
  <div>
    <h1>I'm a react component</h1>
    <div>
      <MyVueComponent message='Hello from Vue!' />
    </div>
  </div>
)
```

从 添加 ``.babelrc`` 的方式来看，难道是用 babel 

vuera/babel.js
``` js
/* eslint-env node */

function processCreateElement (maybeReactComponent, args, file, path, types) {
  // If the first argument is a string (built-in React component), return
  if (maybeReactComponent.type === 'StringLiteral') return

  if (!file.insertedVueraImport) {
    file.path.node.body.unshift(
      types.importDeclaration(
        [
          types.importSpecifier(
            types.identifier('__vueraReactResolver'),
            types.identifier('__vueraReactResolver')
          ),
        ],
        types.stringLiteral('vuera')
      )
    )
  }
  // Prevent duplicate imports
  file.insertedVueraImport = true

  // Replace React.createElement(component, props) with our helper function
  path.replaceWith(
    types.callExpression(types.identifier('__vueraReactResolver'), [maybeReactComponent, ...args])
  )
}

// types 是 babel 插件 API 其中一个模块
module.exports = function ({ types }) {
  return {
    visitor: {
      CallExpression (path, { file }) {
        const callee = path.node.callee
        const [maybeReactComponent, ...args] = path.node.arguments

        // 如果有 react 模块 ，reactImport
        const reactImport = file.path.node.body.find(
          x => x.type === 'ImportDeclaration' && x.source.value === 'react'
        )
        if (!reactImport) return

        // 如果 CallExpression 是 react.createElement
        if (callee.type === 'MemberExpression') {
          /**
           * 获取默认导入名称. Examples:
           * import React from 'react' => "React"
           * import hahaLOL from 'react' => "hahaLOL"
           */
          const defaultImport = reactImport.specifiers.find(
            x => x.type === 'ImportDefaultSpecifier'
          )
          if (!defaultImport) return
          const reactName = defaultImport.local.name

          const { object, property } = callee
          if (!(object.name === reactName && property.name === 'createElement')) {
            return
          }

          processCreateElement(maybeReactComponent, args, file, path, types)
        }
        // 检查 CallExpression 是 react's 'createElement'
        if (callee.type === 'Identifier' && callee.name !== '__vueraReactResolver') {
          // Return unless createElement was imported
          const createElementImport = reactImport.specifiers.find(
            x => x.type === 'ImportSpecifier' && x.imported.name === 'createElement'
          )
          if (!createElementImport) return

          processCreateElement(maybeReactComponent, args, file, path, types)
        }
      },
    },
  }
}

```

- [babel-types](https://github.com/babel/babel/tree/master/packages/babel-types)

- [babel-AST](http://web.jobbole.com/88236/)

- [babel-语法插件中文](https://github.com/thejameskyle/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md#toc-api)

- [在线AST语法树](http://astexplorer.net/)

这部分需要理解 AST语法树的 问题 , 上面的链接能帮助你简单明确 ``babel`` 的 ``plugins`` 插件中 ``ImportSpecifier`` ``MemberExpression`` 之类 问题

>babel.js中总得来说，就是把 ``react.createElement`` 变成 ``__vueraReactResolver`` 内置函数

```js
export function babelReactResolver (component, props, children) {
  return isReactComponent(component)
    ? React.createElement(component, props, children)
    : React.createElement(VueWrapper, Object.assign({ component }, props), children)
}

// babelReactResolver as __vueraReactResolver
```

上面的只是再 ``babel`` 将 ``js`` 降级时所作的事情

---

VueWrapper 上面代码中 👄最重要的

vuera/src/wrapper/vue.js
``` js
import React from 'react'
import Vue from 'vue'
import ReactWrapper from './React'

const VUE_COMPONENT_NAME = 'vuera-internal-component-name'

const wrapReactChildren = (createElement, children) =>
  createElement('vuera-internal-react-wrapper', {
    props: {
      component: () => <div>{children}</div>,
    },
  })

export default class VueContainer extends React.Component {
  constructor (props) {
    super(props)

    /**
     * 传入并重新定义真正的 组件
     * `component` prop.
     */
    this.currentVueComponent = props.component

    /**
     * 修改createVueInstance函数以正确传递此绑定。 在做这个
     *       构造函数避免在渲染中实例化函数。
     *  //我觉得有点难理解 ：译者曰
     */
    const createVueInstance = this.createVueInstance
    const self = this
    this.createVueInstance = function (element, component, prevComponent) {
      createVueInstance(element, self, component, prevComponent)
    }
  }

  componentWillReceiveProps (nextProps) {
    const { component, ...props } = nextProps

    if (this.currentVueComponent !== component) {
      this.updateVueComponent(this.props.component, component)
    }
    /**
     * NOTE: 没有去比较 props 和 nextprops, because I didn't want to write a
     * function for deep object comparison. I don't know if this hurts performance a lot, maybe
     * we do need to compare those objects.
     */
    Object.assign(this.vueInstance.$data, props)
  }

  componentWillUnmount () {
    this.vueInstance.$destroy()
  }

  /**
   * 创建和加载 VueInstance 接口
   * NOTE:  VueInstance inside VueContainer
   * 我们不能绑定 createVueInstance 到 这个 VueContainer 对象, 需要明确
   * 传递 绑定对象
   * @param {HTMLElement} targetElement - element to attact the Vue instance to
   * @param {ReactInstance} reactThisBinding - current instance of VueContainer
   */
  createVueInstance (targetElement, reactThisBinding) {
    const { component, ...props } = reactThisBinding.props

    // `this` refers to Vue instance in the constructor
    reactThisBinding.vueInstance = new Vue({
      // targetElement 就是 VueContainer render() element
      el: targetElement,
      data: props,
      render (createElement) {
        return createElement(
          VUE_COMPONENT_NAME,
          {
            props: this.$data,
          },
          [wrapReactChildren(createElement, this.children)]
        )
      },
      components: {
        [VUE_COMPONENT_NAME]: component,
        'vuera-internal-react-wrapper': ReactWrapper,
      },
    })
  }

  updateVueComponent (prevComponent, nextComponent) {
    this.currentVueComponent = nextComponent

    /**
     * Replace the component in the Vue instance and update it.
     */
    this.vueInstance.$options.components[VUE_COMPONENT_NAME] = nextComponent
    this.vueInstance.$forceUpdate()
  }

  render () {
    return <div ref={this.createVueInstance} />
  }
}
```

用 ``React.Component`` 包裹传入的 ``props.component`` 组件，然后内部新建 ``Vue实例 reactThisBinding.vueInstance``，

- 相当于说 这部分 ``<div ref={this.createVueInstance} />`` 交给 Vue 处理, 然后把 Vue的部分事件 给予 react.component 组件事件处理调用。