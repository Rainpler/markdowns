> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/u014773932/article/details/117526494?spm=1001.2014.3001.5502)

数字输入框只能输入数字, 而且有两个快捷按钮, 可以直接减 l 或加 1 。除此之外, 还可以  
设置初始值、最大值、最小值, 在数值改变时, 触发一个自定义事件来通知父组件。

练习 1 : 在输入框聚焦时, 增加对键盘上下按键的支持, 相当于加 1 和减 lo  
练习 2 : 增加一个控制步伐的 prop-step , 比如设置为 10 , 点击加号按钮, 一 次增加 10 。

效果图：

![](https://img-blog.csdnimg.cn/2021060316115962.png)

代码：

index.html

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta >
    <title>两个常用组件的开发</title>
</head>
<body>
    <div id="app">
        <input-number v-model="value" :step="10" :max="10" :min="0"></input-number>
    </div>
 
    <script src="../vue.min.js"></script>
    <script src="./input-number.js"></script>
    <script src="./index.js"></script>
</body>
</html>
```

index.js

```
let app = new Vue({
    el: "#app",
    data() {
        return {
            value: 0
        }
    },
})
```

input-number.js

```
Vue.component("input-number", {
    template: "\<div class='input-number'>\
        <input \ type='text' \ :value='currentValue' \ @change='handleChange' \ @keyup.38='handleUp' @keyup.40='handleDown' >\
        <button \ @click='handleDown' \ :disable='currentValue <= min'>-</button> \
        <button \ @click='handleUp' \ :disable='currentValue >= max'>+</button> \
        </div>",
    props: {
        max: {
            type: Number,
            default: Infinity
        },
        min:{
            type: Number,
            default: -Infinity
        },
        value: {
            type: Number,
            default: 0
        },
        step:{
            type: Number
        }
    },
    data() {
        return {
            currentValue: this.value
        }
    },
    watch: {
        currentValue: function(val){
            this.$emit("input", val);
            this.$emit("on-change", val);
        },
        value: function(val){
            this.updateValue(val);
        }
    },
    methods: {
        handleDown: function(){
            if(this.currentValue <= this.min) return;
            this.currentValue -= this.step;
        },
        handleUp: function(){
            if(this.currentValue >= this.max) return;
            this.currentValue += this.step;
        },
        handleChange: function(event){
            let val = event.target.value.trim();
            let max = this.max;
            let min = this.min;
 
            if(isValueNumber(val)){
                val = Number(val);
                this.currentValue = val;
 
                if(val > max){
                    this.currentValue = max;
                }else if(val < min){
                    this.currentValue = min;
                }
            }else{
                event.target.value = this.currentValue;
            }
        },
        updateValue: function(val){
            if(val >  this.max) val = this.max;
            if(val < this.min) val = this.min;
            this.currentValue = val;
        }
    },
    mounted() {
        this.updateValue(this.value);
    },
})
 
function isValueNumber(value){
    return (/(^-?[0-9]+\.{1}\d+$) | (^-?[1-9][0-9]*$) | (^-?0{1}$)/).test(value + "");
}
```