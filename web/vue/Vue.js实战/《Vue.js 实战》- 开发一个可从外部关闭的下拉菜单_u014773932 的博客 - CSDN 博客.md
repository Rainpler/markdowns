> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/u014773932/article/details/117671846?spm=1001.2014.3001.5502)

练习 1:  在 update 钩子中支持表达式的更新。
练习 2: 扩展 clickoutside. , 实现在点击按钮显示下拉菜单后, 通过按下键盘的 ESC 键也可
以关闭下拉菜单。
练习 3: 将练习 2 的 ESC 按键关闭功能作为可选项 。 提示, 可以用修饰符, 比如 v-clickoutside.esc

效果图：

![](https://img-blog.csdnimg.cn/20210607185701630.png)

共有四个文件：index.html index.js clickoutside.js style.css

### index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta >
    <title>自定义指令-下拉菜单</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div id="app" v-cloak>
        <!-- 一个包含修饰符的对象 。 例如v-my-directive.foo.bar,修饰符对象 modifiers
             的值是{foo: true, bar: true }。
             以下为： esc: true 如果去掉esc则为undefined
        -->
        <div class="main" v-clickoutside.esc="handleClose" data-id="0" data-first v-test="1*0">
            <button @click="show = !show">点击显示下拉菜单</button>
            <div class="dropdown" v-show="show">
                <p>下拉框的内容，点击外面的区域可以关闭</p>
            </div>
        </div>
    </div>

    <script src="../../vue.min.js"></script>
    <script src="clickoutside.js"></script>
    <script src="index.js"></script>
</body>
</html>
```

### index.js

```js
let app = new Vue({
    el: "#app",
    data() {
        return {
            show: false
        }
    },
    methods: {
        handleClose: function () {
            this.show = false;
        }
    },
    directives: {
        test: {
            bind(el, binding, vnode) {
                el.dataset.first = binding.expression;
            },
            update(el, binding, vnode) {
                if (el.dataset.id++ == 0) {
                    var prevExp = el.dataset.first;
                    el.dataset.first = binding.expression;
                    var currentExp = binding.expression;
                    console.log(prevExp);
                    console.log(currentExp);
                } else {
                    prevExp = el.dataset.first;

                    currentExp = binding.expression;
                    el.dataset.first = binding.expression;
                    console.log(prevExp);
                    console.log(currentExp);
                }
            }
        }
    }
})
```

### clickoutside.js

```js
Vue.directive("clickoutside", {
    /**
     * bind 中：
     * 1. 首先在定义了点击函数，内部逻辑为：如果点击区域在所在指令元素内部，则直接返回；如果定义了表达式，则执行表达式中的函数（示例中是 close）。
     * 2. 这里用到了 contains 函数， A.contains(B) 是判断元素 A 是否包含了元素 B。
     * 3. 接着在 el 中定义了一个变量，用于存放刚才定义的点击函数。bind() 与 unbind() 通过 el 变量进行参数传递。
     * 4. 然后绑定到 document 的点击事件。
     *
     * unbind 中：
     * 1. 解绑在 bind 中绑定的点击事件。
     * 2. 销毁该变量。
     *
     * 在bind函数中，强化了判断，如果点击区域在所在指令元素内部并且没有按下 ESC 键时，才直接返回。即按下  ESC 键时，会执行后续操作（执行表达式中的函数）。
     * 在unbind函数中，也解绑了keyup事件。
     */
    bind: function(el, binding, vnode){
        function documentHanlder(e){

            var escSwitch = (binding.modifiers && binding.modifiers.esc);

            // contains 函数， A.contains(B) 是判断元素 A 是否包含了元素 B
            // 带有了 esc 修饰符，则让程序往下执行
            if(el.contains(e.target)){ //如果点击区域在所在指令元素内部，则直接返回

                if(!(escSwitch && e.keyCode === 27)){ //带有了 esc 修饰符，则让程序往下执行
                    return false;
                }
            }
            if(binding.expression){ //如果定义了表达式，则执行表达式中的函数
                console.log(binding.expression);
                console.log(e);
                binding.value(e); //指令的绑定值,例如v-my-directive=1+1,value的值是2
            }
        }
        el._vueClickOutside_ = documentHanlder;
        document.addEventListener("click", documentHanlder);//绑定到 document 的点击事件
        document.addEventListener("keyup", documentHanlder, false)
    },
    unbind: function(el, binding){
        document.removeEventListener("click", el._vueClickOutside_);
        document.removeEventListener("keyup", el._vueClickOutside_);
        delete el._vueClickOutside_;
    },
})
```

### style.css

```css
[v-cloak] {
    display: none;
}

.main {
    width: 125px;
}
button {
    display: block;
    width: 100%;
    color: #fff;
    background-color: #39f;
    border: 0;
    padding: 6px;
    text-align: center;
    font-size: 12px;
    border-radius: 4px;
    cursor: pointer;
    outline: none;
    position: relative;
}

button:active {
    top: 1px;
    left: 1px;
}
.dropdown {
    width: 100%;
    height: 150px;
    margin: 5px 0;
    font-size: 12px;
    background-color: #fff;
    border-radius: 4px;
    box-shadow: 0 1px 6px rgba(0, 0, 0, .2);
}
.dropdown p {
    display: inline-block;
    padding: 6px;
}
```
