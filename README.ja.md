# ヴェラ

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)

説明

> "version":  "0.1.3"

[ギブスソース](https://github.com/topics/vuera)

[中文版](./README.zh.md)

* * *

反応して同時に動いている. 

-   [vueに反応する](#ReactInVue)

-   [反応時の意見](#VueInReact)

* * *

-   続きを見る

    -   [isreactcomponent](#isreactcomponent)

* * *

2つの状況

最初に言った: 

# 反応する

```js
import Vue from 'vue'
import {VuePlugin} from 'vuera'

Vue.use (VuePlugin)
/* ... */
```

通常はあなたのvueコンポーネントを使用するようにㄡあなたの反応コンポーネントを使用してください!

```vue
<template>
  <div>
    <h1> I'm a Vue component </h1>
    <my-react-component: message = "message" @ reset = "reset" />
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
    components: {'my-react-component': MyReactComponent},
  }
</script>
```

* * *

vueプラグインとして反応する

vuera / src / vueplugin.js

```js
// Determine if React's component
import isReactComponent from './utils/isReactComponent'
// React Component -> Vue Component
import VueResolver from './resolvers/Vue'

/**
 * vue plugin
 */
export default {
  install (Vue, options) {
    /**
     * Custom merge strategy, this strategy is really just
     * Wraps all React components, while retaining the Vue components.
     */
    const originalComponentsMergeStrategy = Vue.config.optionMergeStrategies.components

    Vue.config.optionMergeStrategies.components = function (parent, ... args) {
      // value set before <- return Object.assign (parent, wrappedComponents)
      const mergedValue = originalComponentsMergeStrategy (parent, ... args)

      //
      const wrappedComponents = mergedValue
        Object.entries (mergedValue) .reduce (
            (acc, [k, v]) => ({
              ... acc,
              [k]: isReactComponent (v)? VueResolver (v): v,
            }),
            {}
          )
        : mergedValue
        //merge
      return Object.assign (parent, wrappedComponents)
    }
  },
}
```

登場した

-   [install()](https://cn.vuejs.org/v2/guide/plugins.html#%E5%BC%80%E5%8F%91%E6%8F%92%E4%BB%B6)

-   [vue.configはㄡvueのグローバルコンフィグレーションを含むオブジェクトです. ](https://cn.vuejs.org/v2/api/#optionMergeStrategies)

-   [アプリケーションを起動する前に以下のプロパティを変更することができます: ](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/entries)

```js
const obj = {foo: 'bar', baz: 42};
console.log (Object.entries (obj)); // [['foo', 'bar'], ['baz', 42]]
```

-   [object.entries](https://developer.mozilla.org/wiki/Reduce)

```js
var flattened = [[0, 1], [2, 3], [4, 5]]. reduce (function (a, b) {
  return a.concat (b);
}, []);
// flattened is [0, 1, 2, 3, 4, 5]
```

-   array.prototype.reduce

vueresolver

> vuera / src / resolvers / vue.js

反応コンポーネント→vueコンポーネント`あなたはㄡあなたの側を横切ることの使用が`ビュー`コンポーネントが見つかった場合ㄡ`反応する`コンポーネントが変わったように見える`ビュー

. 

# 第二に言う: 

vueinreact`追加する`ヴェラ/ベルベル`あなたの` `.babelrc`プラグイン

オプション

```json
{
  "presets": "react",
  "plugins": ["vuera/babel"]
}
```

.babelrc

```jsx
import React from 'react'
import MyVueComponent from './MyVueComponent.vue'

export default () => (
  <div>
    <h1> I'm a react component </h1>
    <div>
      <MyVueComponent message = 'Hello from Vue!' />
    </div>
  </div>
)
```

つかいます`追加の方法から`.babelrc

* * *

ㄡそれはバベルを使用することは可能ですか?

```js
/* eslint-env node */

function processCreateElement (maybeReactComponent, args, file, path, types) {
  // If the first argument is a string (built-in React component), return
  if (maybeReactComponent.type === 'StringLiteral') return

  if (! file.insertedVueraImport) {
    file.path.node.body.unshift (
      types.importDeclaration (
        [
          types.importSpecifier (
            types.identifier ('__ vueraReactResolver'),
            types.identifier ('__ vueraReactResolver')
          ),
        ],
        types.stringLiteral ('vuera')
      )
    )
  }
  // Prevent duplicate imports
  file.insertedVueraImport = true

  // Replace React.createElement (component, props) with our helper function
  path.replaceWith (
    types.callExpression (types.identifier ('__ vueraReactResolver'), [maybeReactComponent, ... args])
  )
}

// types is the babel plugin API? One of the modules
module.exports = function ({types}) {
  return {
    visitor: {
      CallExpression (path, {file}) {
        const callee = path.node.callee
        const [maybeReactComponent, ... args] = path.node.arguments

        // If there is a react module, reactImport
        const reactImport = file.path.node.body.find (
          x => x.type === 'ImportDeclaration' && x.source.value === 'react'
        )
        if (! reactImport) return

        // if CallExpression is react.createElement
        if (callee.type === 'MemberExpression') {
          /**
           * Get the default import name Examples:
           * import React from 'react' => "React"
           * import hahaLOL from 'react' => "hahaLOL"
           */
          const defaultImport = reactImport.specifiers.find (
            x => x.type === 'ImportDefaultSpecifier'
          )
          if (! defaultImport) return
          const reactName = defaultImport.local.name

          const {object, property} = callee
          if (! (object.name === reactName && property.name === 'createElement')) {
            return
          }
          
          processCreateElement (maybeReactComponent, args, file, path, types)
        }
        // Check CallExpression is react's 'createElement'
        if (callee.type === 'Identifier' && callee.name! == '__vueraReactResolver') {
          // Return unless createElement was imported
          const createElementImport = reactImport.specifiers.find (
            x => x.type === 'ImportSpecifier' && x.imported.name === 'createElement'
          )
          if (! createElementImport) return

          processCreateElement (maybeReactComponent, args, file, path, types)
        }
      },
    },
  }
}
```

-   [vuera / babel.js](https://github.com/babel/babel/tree/master/packages/babel-types)

-   [バベルタイプ](http://web.jobbole.com/88236/)

-   [バベル](https://github.com/thejameskyle/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md#toc-api)

-   [バーベル構文プラグイン中国語](http://astexplorer.net/)

オンラインast構文ツリー`上記のリンクはあなたがシンプルで明確になるのを助けることができます` `バベル`プラグイン`プラグイン`importspecifier

> メンバーの問題?`一般的にbabel.jsはㄡ`react.createelement``\`\_\_vuerareactresolver

```js
export function babelReactResolver (component, props, children) {
  return isReactComponent (component)
    ? React.createElement (component, props, children)
    : React.createElement (VueWrapper, Object.assign ({component}, props), children)
}

// babelReactResolver as __vueraReactResolver
```

組み込み関数

* * *

上記はバベルがjsをダウングレードするために行う最後のものです

vuewrapper上記のコードが最も重要です

```js
import React from 'react'
import Vue from 'vue'
import ReactWrapper from './React'

const VUE_COMPONENT_NAME = 'vuera-internal-component-name'

const wrapReactChildren = (createElement, children) =>
  createElement ('vuera-internal-react-wrapper', {
    props: {
      component: () => <div> {children} </div>,
    },
  })

export default class VueContainer extends React.Component {
  constructor (props) {
    super (props)

    /**
     * Incoming and redefinition of real components
     * `component` prop.
     */
    this.currentVueComponent = props.component

    /**
     * Modify the createVueInstance function to pass this binding correctly. Doing this
     * Constructor Avoid rendering functions in the rendering.
     *// I feel a bit difficult to understand: translator said
     */
    const createVueInstance = this.createVueInstance
    const self = this
    this.createVueInstance = function (element, component, prevComponent) {
      createVueInstance (element, self, component, prevComponent)
    }
  }

  componentWillReceiveProps (nextProps) {
    const {component, ... props} = nextProps

    if (this.currentVueComponent! == component) {
      this.updateVueComponent (this.props.component, component)
    }
    /**
     * NOTE: Did not compare props and nextprops, because I did not want to write a
     function for deep object comparison. I do not know if this hurts performance a lot, maybe
     * we do need to compare those objects.
     */
    Object.assign (this.vueInstance. $ Data, props)
  }

  componentWillUnmount () {
    this.vueInstance. $ destroy ()
  }

  /**
   * Create and load VueInstance interface
   * NOTE: VueInstance inside VueContainer
   * We can not bind createVueInstance to this VueContainer object and need to be explicit
   Pass the binding object
   * @param {HTMLElement} targetElement - element to attact the Vue instance to
   * @param {ReactInstance} reactThisBinding - current instance of VueContainer
   */
  createVueInstance (targetElement, reactThisBinding) {
    const {component, ... props} = reactThisBinding.props

    // `this` refers to Vue instance in the constructor
    reactThisBinding.vueInstance = new Vue ({
      // targetElement is VueContainer render () element
      el: targetElement,
      data: props,
      render (createElement) {
        return createElement (
          VUE_COMPONENT_NAME,
          {
            props: this. $ data,
          },
          [wrapReactChildren (createElement, this.children)]
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
    this.vueInstance. $ options.components [VUE_COMPONENT_NAME] = nextComponent
    this.vueInstance. $ forceUpdate ()
  }

  render () {
    return <div ref = {this.createVueInstance} />
  }
}
```

vuera / src / wrapper / vue.js`その`props.component`コンポーネントが`react.component`ㄡ次に`vue instance reactthisbinding.vueinstance

-   内部で作成されます. `これと同じことはㄡ`&lt;div ref = {this.createvueinstance} />

* * *

## vue react.componentコンポーネントのイベントハンドラ呼び出しの一部に渡します. 

### 続きを見るisreactcomponent

```js
export default function isReactComponent (component) {
  if (typeof component === 'object') {
    return false // no object == no react
  } else if (
    typeof component === 'function' &&
    component.prototype.constructor.super &&
    component.prototype.constructor.super.name.startsWith('Vue')
  ) {
    return false // is vue
  } else {
    return true // is react
  }
}
```