
## v-if & v-else-if & v-else

`v-if`是vue的内部指令，作用于html标签上，用于判断是否加载html的DOM。

示例：

```
<body>
  <div id="app">
	<div v-if="isLogin">你好，openXu</div>
	<div v-else-if="pass">你好，游客</div>
	<div v-else>你好</div>
  </div>
  <script>
    var vm = new Vue({
      el: '#app',
      data: {
		isLogin:false,
		pass:false,
      },
    });
  </script>
</body>
```

## v-show

根据表达式的真假值，切换元素的 display CSS 属性，DOM已经加载，知识通过CSS控制是否显示。

示例：

```
<body>
  <div id="app">
	<div v-show="isLogin">你好, openXu</div>
  </div>
  <script>
    var vm = new Vue({
      el: '#app',
      data: {
		isLogin:true
      },
    });
  </script>
</body>
```

**v-if 和v-show的区别**

* v-if： 判断是否加载，可以减轻服务器的压力，在需要时加载。
* v-show：调整css dispaly属性，可以使客户端操作更加流畅。

## v-for

`v-for`指令循环渲染一组data中的数组





