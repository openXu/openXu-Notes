
## 简介

[Vue.js](https://cn.vuejs.org/)是在美中国学生[尤雨溪](https://baike.baidu.com/item/%E5%B0%A4%E9%9B%A8%E6%BA%AA/2281470?fr=aladdin)于2014年2月
开源的一个前端开发库，它是一个构建Web界面的JavaScrupt库，可以通过简洁的API提供高效的数据绑定和灵活的组件系统。2016年9月3日，在
南京的JSConf上，尤雨溪宣布以技术顾问的身份加盟阿里巴巴Weex团队来做Vue和Weex的JavaScript runtime整合，目的是让大家能用Vue的语
法跨三端。

Vue是一套用于构建用户界面的**渐进式框架**。与其它大型框架不同的是，Vue 被设计为可以自底向上逐层应用。Vue 的核心库只关注视图层，
不仅易于上手，还便于与第三方库或既有项目整合。另一方面，当与现代化的工具链以及各种支持类库结合使用时，Vue 也完全能够为复杂的单页应用提供驱动。

官方指南假设你已了解关于 HTML、CSS 和 JavaScript 的中级知识。如果你刚开始学习前端开发，将框架作为你的第一步可能不是最好的主意——掌握好基础知识再来吧！
之前有其它框架的使用经验会有帮助，但这不是必需的。

## 安装

直接下载[Vue.js开发版本](https://cn.vuejs.org/js/vue.js)并用 <script> 标签引入，Vue 会被注册为一个全局变量
```
<!-- 开发环境版本，包含了有帮助的命令行警告 -->
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
```

## Vue实例

```
<!DOCTYPE html>
<html>
	<head>
	  <meta charset="UTF-8">
	  <meta name="viewport" content="width=device-width, initial-scale=1.0">
	  <meta http-equiv="X-UA-Compatible" content="ie=edge">
	  <title>Document</title>
	  <script src="./vue.js"></script>
	</head>
		
	<body>
		
		<style type="text/css">
			.red{color:red; font-size:16px}
		</style>
		
		<div id="app">
			<!-- 6、“Mustache”标签({{}}插值表达式)将会被替代为对应数据对象上 msg 属性的值。数据对象上 msg 属性发生了改变，插值处的内容都会更新 -->
			<p>Message：{{msg}}</p>
			<!-- 7、使用 v-once 指令，你也能执行一次性地插值，当数据改变时，插值处的内容不会更新 -->
			<span v-once>这个将不会改变: {{ msg }}</span>
			<!-- 8、[v-指令](https://cn.vuejs.org/v2/api/#%E6%8C%87%E4%BB%A4)
				双大括号会将数据解释为普通文本。为了输出真正的 HTML，你需要使用 v-html 指令-->
			<p>普通文本: {{ rawHtml }}</p>
			<p>使用v-html输出html: <span v-html="rawHtml"></span></p>
			<p v-if="see">v-if指令控制是否可见</p>
			<!-- 使用 v-bind 指令为html标签动态绑定属性值 -->
			<div v-bind:class="bindColor">v-bind绑定html属性值</div>
			
			<!-- 9、{{}}中用 JavaScript 表达式 -->
			{{ number + 1 }}
			{{ ok ? 'YES' : 'NO' }}
			{{ message.split('').reverse().join('') }}
			<div v-bind:id="'list-' + id"></div>
			
			<!-- 10、Class与Style绑定 -->
			<!-- v-bind:class 指令也可以与普通的 class 属性共存。是否具有active须看isActive的真假 -->
			<div 
				class="static"
				v-bind:class="{active: isActive, 'text-danger': hasError}">Class与Style绑定</div>
				
			<!-- 11、v-for列表渲染 -->
			<ul id="example-1">
			  <li v-for="(item, index) in items">
			    {{index}}  {{ item.message }}
			  </li>
			</ul>
			
			<ul id="v-for-object" class="demo">
			  <li v-for="(value, name) in object">
			     {{ name }}: {{ value }}
			  </li>
			</ul>
			
			<!-- 12、v-on事件处理 -->
			<button v-on:click="click1">v-on绑定点击</button>
			<div @click="click2">
				<!-- .stop修饰符停止事件传播 -->
				<button @click.stop="click3">@click绑定点击</button>
			</div>
			
			<!-- 13、v-model表单输入绑定；
			 可以用 v-model 指令在表单 <input>、<textarea> 及 <select> 元素上创建双向数据绑定。
			 它会根据控件类型自动选取正确的方法来更新元素。通过 JavaScript 在组件的 data 选项中声明初始值。-->
			<input v-model="input1" placeholder="edit me">
			<p>input1 is: {{ input1 }}</p>
			<!-- 多行 -->
			<textarea v-model="input2" placeholder="add multiple lines"></textarea>
			<p style="white-space: pre-line;">Multiline ：{{ input2 }}</p>
			<!-- 单个复选框 -->
			<input type="checkbox" id="checkbox" v-model="checked">
			<label for="checkbox">{{ checked }}</label>
			<!-- 多个复选框 -->
			<div id='example-3'>
			  <input type="checkbox" id="jack" value="Jack" v-model="checkedNames">
			  <label for="jack">Jack</label>
			  <input type="checkbox" id="john" value="John" v-model="checkedNames">
			  <label for="john">John</label>
			  <input type="checkbox" id="mike" value="Mike" v-model="checkedNames">
			  <label for="mike">Mike</label>
			  <br>
			  <span>Checked names: {{ checkedNames }}</span>
			</div>
			 <br>
			<!-- 单选按钮 -->
			<div id="example-4">
			  <input type="radio" id="one" value="One" v-model="picked">
			  <label for="one">One</label>
			  <br>
			  <input type="radio" id="two" value="Two" v-model="picked">
			  <label for="two">Two</label>
			  <br>
			  <span>Picked: {{ picked }}</span>
			</div>
			 <br>
			<!-- 选择框 -->
			<div id="example-5">
			  <select v-model="selected">
			    <option disabled value="">请选择</option>
			    <option>A</option>
			    <option>B</option>
			    <option>C</option>
			  </select>
			  <span>Selected: {{ selected }}</span>
			</div>
			 <br>
			<!-- 多选框 -->
			<div id="example-6">
			  <select v-model="selected1" multiple style="width: 50px;">
			    <option>A</option>
			    <option>B</option>
			    <option>C</option>
			  </select>
			 
			  <span>Selected: {{ selected1 }}</span>
			</div>
			 <br>
			<!-- 14、组件 -->
			<button-counter name="组件1" @clicknow="clickNow"></button-counter>
			<button-counter name="组件2"></button-counter>
			
		</div>
		
		
		<script>
		// 定义一个名为 button-counter 的新组件
		Vue.component('button-counter', {
		  props: ['name'],    //Prop 是你可以在组件上注册的一些自定义属性
		  data: function () {
		    return {
		      count: 0
		    }
		  },
		  //每个组件必须只有一个根元素
		  template: '<div><button v-on:click="clickFun">{{name}} clicked {{ count }} times.</button><span>尾巴</span></div>',
		  methods:{
			  clickFun:function(){
				  this.count ++;
				  //子组件可以通过调用内建的 $emit 方法 并传入事件名称来触发一个事件
				  this.$emit('clicknow', this.count);
			  }
		  }
		  
		})
		/**
		 * 1、每个 Vue 应用都是通过用 Vue 函数创建一个新的 Vue 实例开始的。
		 * 虽然没有完全遵循 MVVM 模型，但是 Vue 的设计也受到了它的启发。
		 * 因此在文档中经常会使用 vm (ViewModel 的缩写) 这个变量名表示 Vue 实例
		 */
		var vm = new Vue({
			el: '#app',   //2、将vm实例作用于id为app的元素（element）
			//3、data实例中的属性会被插值到模板中的{{}}中，并响应变化，可通过Object.freeze()阻止修改现有属性，响应系统无法再追踪变化
		    data: {   
				msg:1,
				rawHtml:"<span>我是html文本</span>",
				bindColor:"red",
				number:1,
				see:true,
				ok:true,
				message:'vue倒写',
				id:5,
				isActive: true,
				hasError: false,
				items: [
				  { message: 'Foo' },
				  { message: 'Bar' }
				],
				object: {
				  title: 'How to do lists in Vue',
				  author: 'Jane Doe',
				  publishedAt: '2016-04-10'
				},
				input1:"",
				input2:"",
				checked:1,
				checkedNames: [],
				picked: '',
				 selected: '',
				 selected1:[]
				
			},
			methods:{
				click1:function(){
					alert("on-click绑定点击")
				},
				click2:function(){
					alert("穿透点击了")
				},
				click3:function(){
					alert("on-click简写绑定点击")
				},
				clickNow:function(e){
					console.log(e);
				}
			},
			//5、[生命周期钩子]（https://cn.vuejs.org/v2/api/#%E9%80%89%E9%A1%B9-%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E9%92%A9%E5%AD%90）
			 created: function () {
			    // `this` 指向 vm 实例
			    console.log('msg is: ' + this.msg)
			} 
		})
		
		//4、Vue 实例还暴露了一些有用的实例属性与方法，使用$做前缀，以便与用户定义的属性区分
		//vm.$data === data // => true
		vm.$el === document.getElementById('example') // => true
		// $watch 是一个实例方法，用于监听属性变化
		vm.$watch('msg', function (newValue, oldValue) {
		  // 这个回调将在 `vm.msg` 改变后调用
		})
		
		</script>
		
	</body>
</html>
```























