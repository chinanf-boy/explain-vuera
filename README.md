# vuera

[![explain](./minilogo.svg)](https://github.com/chinanf-boy/Source-Explain)

Explanation
> "version": "0.1.3"

[中文版](./README.zh.md)

Run in react and vue at the same time.

- [react on vue](#ReactInVue)

- [vue at react](#VueInReact)

Two situations

First said first:

# ReactInVue

```js
import Vue from 'vue'
import {VuePlugin} from 'vuera'

Vue.use (VuePlugin)
/* ... */
```

Now, use your React components like you will normally use your Vue components!

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

---

Use React as a Vue plugin

vuera/src/VuePlugin.js
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

Which appeared

- [install ()](https://cn.vuejs.org/v2/guide/plugins.html#%E5%BC%80%E5%8F%91%E6%8F%92%E4%BB%B6)

- [vue.config is an object that contains the global configuration of Vue. You can modify the following properties before starting the application:](https://cn.vuejs.org/v2/api/#optionMergeStrategies)

- [Object.entries](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/entries)

```js
const obj = {foo: 'bar', baz: 42};
console.log (Object.entries (obj)); // [['foo', 'bar'], ['baz', 42]]
```
- [Array.prototype.reduce](https://developer.mozilla.org/wiki/Reduce)

```js
var flattened = [[0, 1], [2, 3], [4, 5]]. reduce (function (a, b) {
  return a.concat (b);
}, []);
// flattened is [0, 1, 2, 3, 4, 5]
```

- VueResolver

vuera/src/resolvers/Vue.js
> React Component -> Vue Component

So you can see that the use of traversing the side of the `` Vue`` component, if found, `` React`` component looks like it is transformed into `` Vue``.

Say the second:

# VueInReact

Add `vuera/babel` to your` .babelrc` `plugins` option

.babelrc
```json
{
  "presets": "react",
  "plugins": ["vuera/babel"]
}
```

use

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

From the way of adding `` .babelrc``, is it possible to use babel?

---

vuera/babel.js
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

- [babel-types](https://github.com/babel/babel/tree/master/packages/babel-types)

- [babel-AST](http://web.jobbole.com/88236/)

- [babel-syntax plug-in Chinese](https://github.com/thejameskyle/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md#toc-api)

- [Online AST Syntax Tree](http://astexplorer.net/)

This part of the need to understand the AST syntax tree, the above link can help you to be simple and clear `babel` `` plugins`` plug-in `` ImportSpecifier` ```` ```` ```` ```` ```` ```member problems?

> babel.js in general, is to turn `` react.createElement`` into a `__vueraReactResolver`` built-in function

```js
export function babelReactResolver (component, props, children) {
  return isReactComponent (component)
    ? React.createElement (component, props, children)
    : React.createElement (VueWrapper, Object.assign ({component}, props), children)
}

// babelReactResolver as __vueraReactResolver
```

The above is only the last thing babel does to downgrade js

---

VueWrapper The above code is the most important

vuera/src/wrapper/vue.js
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

The `` props.component`` component is wrapped with `` React.Component``, and then the `` Vue instance reactThisBinding.vueInstance`` is created internally.

- Equivalent to say that this part of the `` <div ref = {this.createVueInstance} /> `` to the Vue processing, and then part of Vue react.component component event handler call.