## Vue2中，父子组件通信
Vue中父子组件通信规则是：父组件要给子组件传递数据，子组件需要将它内部发生的事情
告知给父组件。可以总结为：props down, events up.父组件向下传递数据给子组件，子组件通过events
给父组件发送消息。  
如下图所示：  
![父子组件通信示意图](http://cn.vuejs.org/images/props-events.png)

### 使用Prop传递数据(props down)

#### 动态Prop
类似于用 v-bind 绑定 HTML 特性到一个表达式，也可以用 v-bind 动态绑定 props 的值到父组件的数据中。
每当父组件的数据变化时，该变化也会传导给子组件：
```HTML
<div>
  <input v-model="parentMsg">
  <br>
  <child :my-message="parentMsg"></child> <!-- v-bind:my-message="parentMsg"的缩写 -->
</div>
```

#### 单向数据流
prop 是**单向绑定**的：当父组件的属性变化时，将传导给子组件，但是不会反过来。
这是为了防止子组件无意修改了父组件的状态——这会让应用的数据流难以理解。   
另外，每次父组件更新时，子组件的所有 prop 都会更新为最新值。
这意味着你**不应该**在子组件内部改变 prop 。   
通常有两种改变 prop 的情况：
1. prop 作为初始值传入，子组件之后只是将它的初始值作为本地数据的初始值使用；
2. prop 作为需要被转变的原始值传入。

#### Prop验证
组件可以为 props 指定验证要求。如果未指定验证要求，Vue 会发出警告。当组件给其他人使用时这很有用。
如下：
```js
Vue.component('example', {
  props: {
    propA: {
      type: Object, // 验证类型
      default: function () { // 数组／对象的默认值应当由一个工厂函数返回
        return {message: 'hello'}
      }
    },
    propB: {
      validator: function (value) {
        return value > 10 // 自定义验证规则
      }
    }
  }
})
```
type 也可以是一个自定义构造器，使用 instanceof 检测。  
当 prop 验证失败了，如果使用的是开发版本会抛出一条警告。

### 自定义事件(events up)
父组件使用props传递数据给子组件，而子组件用**自定义事件**把数据传递回去  

#### 使用`v-on`绑定自定义事件
每个Vue实例都实现了[事件接口](http://cn.vuejs.org/v2/api/#Instance-Methods-Events),即：
- 使用 `$on(eventName)` 监听事件
- 使用 `$emit(eventName)` 触发事件
父组件可以在使用子组件的地方直接用 v-on 来监听子组件触发的事件。  
下面是一个例子：
```html
<div id="counter-event-example">
  <p>{{ total }}</p>
  <button-counter v-on:increment="incrementTotal"></button-counter>
  <button-counter v-on:increment="incrementTotal"></button-counter>
</div>
```
```js
Vue.component('button-counter', {
  template: '<button v-on:click="increment">{{ counter }}</button>',
  data: function () {
    return {
      counter: 0
    }
  },
  methods: {
    increment: function () {
      this.counter += 1
      this.$emit('increment')
    }
  },
})
new Vue({
  el: '#counter-event-example',
  data: {
    total: 0
  },
  methods: {
    incrementTotal: function () {
      this.total += 1
    }
  }
})
```

#### 子组件索引
尽管有 props 和 events ，但是有时仍然需要在 JavaScript 中直接访问子组件。
为此可以使用 ref 为子组件指定一个**索引 ID** 。例如：
```html
<div id="parent">
  <user-profile ref="profile"></user-profile>
</div>
```
```js
var parent = new Vue({
  el: '#parent'
})
// 访问子组件
var child = parent.$ref.profile
```
当 ref 和 v-for 一起使用时， ref 是一个数组或对象，包含相应的子组件。
> $refs 只在组件渲染完成后才填充，并且它是非响应式的。
它仅仅作为一个直接访问子组件的**应急方案**——应当避免在模版或计算属性中使用 $refs 。

### 非父子组件通信
有时候非父子关系的组件也需要通信。在简单的场景下，使用一个空的 Vue 实例作为中央事件总线：
```js
var bus = new Vue()
// 触发组件 A 中的事件
bus.$emit('id-selected', 1)
// 在组件 B 创建的钩子中监听事件
bus.$on('id-selected', function (id) {
  // ...
})
```
更为复杂的情况就应该考虑使用[状态管理模式](http://cn.vuejs.org/v2/guide/state-management.html)

