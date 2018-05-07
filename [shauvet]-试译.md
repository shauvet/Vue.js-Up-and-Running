## 第六章 

### 使用vuex进行状态管理



- 本书讲到现在为止，所有的数据已经被存到组件里了。我们构建一个 API ，然后把返回的数据存到 data 对象里。然后把表单绑定到一个对象，把这个对象存到 data 对象。所有组件之间的通信使用事件（从子组件到父组件）和 props（从父组件到子组件）。这对于简单的场景还好，但是对于更复杂的应用就应付不过来了。
- 我们来看一个社交应用-具体来说，比如短信。你想在顶部导航栏显示短信的数量，而且你还想在页面底部弹出信息也展示短信的数量。页面上的这两个组件差得很远，所以使用事件或 props 来连接它们绝对是场噩梦：与组件完全无关联的通知将不得不使用事件来传递它们。替代方案是，你可以为每个组件构建独立的 API 请求，而不是连接它们以共享数据。这还不如上一个方案。每个组件会在不同的时间更新，意味着它们会显示不同的内容，而且页面将会有比它所需要的多得多的API请求。
- vuex 是一个帮助开发者管理他们的 Vue 应用的状态的库。它提供了一个中心化储存机制，使你可以在整个应用中储存和维护全局状态，并且提供数据流入时提供校验数据的能力，以确保这些数据是可预测的和准确的。



### 安装

你可以通过 CDN 来使用 vuex ，只需要增加如下链接：

```javascript
<script src="https://unpkg.com/vuex"></script>
```

或者，你可以使用 npm ，你可以这样安装 vuex，通过 npm install —save vuex 。假如你正在使用打包工具例如webpack，那么就与使用 vue-router 类似，你将不得不使用 Vue.use() 语法：

```javascript
import Vue from 'vue';
import Vuex from 'vuex';

Vue.use(Vuex);
```

然后你需要启动你的 store 。我们来创建一个如下的文件并存为 store/index.js：

```javascript
import Vuex from 'vuex';

export default new Vuex.Store({
    state: {}
});
```

现在我们有了一个空的 store对象：我们将在这一章把它加进去。然后把它引入到你的主 app 文件并且把它当做新建 Vue 实例的一个属性：

```javascript
import Vue from 'vue'; 
import store from './store';

new Vue({
    el: '#app', 
    store, 
    components: {
      App
	}
});
```

现在你已经在你的 app 添加上 store 了，然后就可以通过 this.\$store 来连接它。我们来看一下 vuex 的概念，然后就会看到通过  this.\$store 可以做什么。



### 概念

就像我们在本章的介绍中说的一样，vuex 可以被用于复杂应用中需要超过一个组件来分享状态的场景。

我们用写好的一个简单的组件不用 vuex 来展示页面上用户信息的数量：

```javascript
const NotificationCount = {
	template: `<p>Messages: {{ messageCount }}</p>`, 
    data: () => ({
        messageCount: 'loading'
    }),
    mounted() {
		const ws = new WebSocket('/api/messages');
		ws.addEventListener('message', (e) => {
            const data = JSON.parse(e.data); this.messageCount = data.messages.length;
		});
    }
};
```

这很简单，以上代码打开一个 websocket 连接到 /api/messages ，然后当服务端像客户端发送数据时，当 socket 处于打开状态(初始化 messageCount )然后 count 被更新(在新的信息上面)，发出去的信息被计数并且展示到页面上。

> 在实践中，真实代码将会复杂很多：在本例中没有websocket连接的鉴权，而且总是以为返回的是带有 messages 属性的合法的 json 数组，在实际项目中很可能不是这样。不过对于这个案例来说，这么简单的代码足以演示我们要做的事情。

当我们试图在同一个页面使用超过一个 Notification Count 组件时遇到了问题。因为每个组件开了一个 websocket 连接，这会打开一个不必要的重复的连接，而且由于网络延迟，组件之间的更新或许会稍微不同步。为了解决这个问题，我们可以把 websocket 连接的逻辑放到 vuex 。



我们来做一个正确的示例。组件将会变成下面这样：

```javascript
const NotificationCount = {
	template: `<p>Messages: {{ messageCount }}</p>`, computed: {
		messageCount() {
			return this.$store.state.messages.length;
		} 
    }
	mounted() { 
    	this.$store.dispatch('getMessages');
	} 
};
```

而我们的 vuex store 将会变成下面这样：

```javascript
let ws;
export default new Vuex.Store({ 
    state: {
        messages: [],
      },
    mutations: {
        setMessages(state, messages) {
            state.messages = messages;
        } },
    actions: {
        getMessages({ commit }) {
            if (ws) { 
                return;
                    }
            ws = new WebSocket('/api/messages');
            ws.addEventListener('message', (e) => { 
                const data = JSON.parse(e.data); commit('setMessages', data.messages);
            }); 
        }
	} 
});
```

现在当我们触发 getMessages 这个 action 时，每个 Notification Count 组件的计数都会被更新，这个 action 会检查当前是否已经有 websocket 连接存在，仅当连接未打开时才会开启一个新的连接。然后它会监听这个 socket ，提交变化到state，因为 store 是响应式的，所以可以更新 Notification Count 组件将会被更新，就像 vue 内部的其他东西一样。当这个 socket 发送新的信息，全局的 store 就会更新，然后每个组件也会被同时更新。

在这个章节的剩余部分，我将介绍一些你在示例代码中见到的独立的概念包括 - state，mutations 和 actions - 以说明一种在大型应用中结构化我们的 vuex 模块的方法，以避免产生一个大的混乱的文件。



### State 和 State Helpers

首先，我们来看一下 state，State 显示了数据是如何存储到我们的 vuex store中的。它就像一个一个大的对象，我们应用中任何地方都可以连接到的 - 单一可信来源。

我们来看一个仅包含一个数字的简单 store：

```javascript
import Vuex from 'vuex';
export default new Vuex.Store({ 
    state: {
        messageCount: 10
    }
});
```
