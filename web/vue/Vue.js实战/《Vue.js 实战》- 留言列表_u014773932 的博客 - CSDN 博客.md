> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/u014773932/article/details/117773928?spm=1001.2014.3001.5502)

使用 Render 函数来完成一个留言列表的小功能

练习 1 : 给每条留言都增加 一个删除的功能。  
练习 2 : 将该示例的 render 写法改写为 template 写法, 加以对比, 总结出两者的差异性, 深刻理解其使用场景。

效果图：

![](https://img-blog.csdnimg.cn/20210610114523216.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ3NzM5MzI=,size_16,color_FFFFFF,t_70)

文件：index.html index.js input-render.js input-template.js list-render.js list-template.js  style.css

index.html

```
<!DOCTYPE html>
<html lang="en">
 
<head>
    <meta charset="UTF-8">
    <meta >
    <title>留言列表</title>
    <link rel="stylesheet" href="style.css">
</head>
 
<body>
 
    <div id="app" v-cloak style="width: 500px;margin:0 auto;">
        <div class="message">
            <v-input v-model="username"></v-input>
            <v-textarea v-model="message" ref="message"></v-textarea>
            <button @click="handleSend">发布</button>
        </div>
        <list :list="list" @reply="handleReply" @delete="deleteReply"></list>
    </div>
 
    <script src="../../vue.min.js"></script>
    <script src="input.js"></script>
    <script src="list.js"></script>
    <script src="index.js"></script>
</body>
 
</html>
```

index.js

```
let app = new Vue({
    el: "#app",
    data() {
        return {
            username: "",
            message: "",
            list: []
        }
    },
    methods: {
        handleSend: function(){
            if(this.username === ""){
                window.alert("请输入昵称");
                return;
            }
            if(this.message === ""){
                window.alert("请输入留言内容");
                return;
            }
            this.list.push({
                name: this.username,
                message: this.message
            });
            this.message = "";
        },
        handleReply: function(index){
            let name = this.list[index].name;
            this.message = "回复@" + name + ":";
            this.$refs.message.focus();
        },
        deleteReply: function(index){
            this.list.splice(index, 1);
        }
    },
})
```

input-render.js

```
Vue.component("vInput", {
    props: {
        value: {
            type: [String, Number],
            default: ""
        }
    },
    // render方法
    render: function (h) {
        let _this = this;
        return h("div", [
            h("span", "昵称:"),
            h("input", {
                attrs: {
                    type: "text",
                },
                domProps: {
                    value: this.value
                },
                on: {
                    input: function (event) {
                        _this.value = event.target.value;
                        _this.$emit("input", event.target.value);
                    }
                }
            })
        ])
    }
})
 
Vue.component("vTextarea", {
    props: {
        value: {
            type: String,
            default: ""
        }
    },
    render: function(h){
        let _this = this;
        return h("div", [
            h("span", "留言内容:"),
            h("textarea", {
                attrs: {
                    placeholder: "请输入留言内容",
                },
                domProps: {
                    value: this.value
                },
                ref: "message",
                on:{
                    input: function(event){
                        _this.value = event.target.value;
                        _this.$emit("input", event.target.value);
                    }
                }
            })
        ])
    },
    methods: {
        focus: function () {
            this.$refs.message.focus();
        }
    },
})
```

input-template.js

```
Vue.component("vInput", {
    template: "<div> \
        <span>昵称:</span> \
        <input type='text' :value='value' @input='inputName'> \
    </div>",
    props: {
        value: {
            type: [String, Number],
            default: ""
        }
    },
    data() {
        return {
            value: this.value
        }
    },
    methods: {
        inputName: function(event){
            this.value = event.target.value;
            this.$emit("input", event.target.value);
        }
    }
})
 
Vue.component("vTextarea", {
    template: "<div> \
        <span>留言内容:</span> \
        <textarea placeholder='请输入留言内容' :value='value' ref='message' @input='inputMsg'></textarea> \
    </div>",
    props: {
        value: {
            type: String,
            default: ""
        }
    },
    data() {
        return {
            value: this.value
        }
    },
    methods: {
        focus: function () {
            this.$refs.message.focus();
        },
        inputMsg: function(event){
            this.value = event.target.value;
            this.$emit("input", event.target.value);
        }
    }
})
```

list-render.js

```
Vue.component("list", {
    props:{
        list: {
            type: Array,
            default: function(){
                return [];
            }
        }
    },
    render: function(h){
        let _this = this;
        let list = [];
 
        this.list.forEach((msg, index) => {
            let node = h("div", {
                attrs: {
                    class: "list-item"
                }
            },[
                h("span", msg.name + ":"),
                h("div", {
                    attrs: {
                        class: "list-msg"
                    },
                },[
                    h("p", msg.message),
                    h("a", {
                        attrs: {
                            class: "list-reply"
                        },
                        on:{
                            click: function(){
                                _this.handleReply(index);
                            }
                        }
                    }, "回复"),
                    h("a", {
                        attrs: {
                            class: "list-del"
                        },
                        on:{
                            click: function(){
                                _this.deleteReply(index);
                            }
                        }
                    }, "删除")
                ])
            ])
            list.push(node);
        });
        
        if(this.list.length){
            return h("div", {
                attrs:{
                    class: "list"
                }
            }, list)
        }else{
            return h("div", {
                attrs:{
                    class: "list-nothing"
                }
            }, "留言列表为空");
        }
    },
    methods: {
        handleReply: function(index){
            this.$emit("reply", index);
        },
        deleteReply: function(index){
            this.$emit("delete", index);
        }
    },
})
```

list-template.js 

```
Vue.component("list", {
    template: "<div v-if='list.length'> \
    <div class='list-item' v-for='(item, index) in list'> \
        <span>{{item.name}}:</span> \
        <div class='list-msg'> \
            <p>{{item.message}}</p> \
            <a class='list-reply' @click='handleReply(index)'>回复</a> \
            <a class='list-del' @click='deleteReply(index)'>删除</a> \
        </div>\
    </div>\
    </div> \
    <div v-else class='list-nothing'>留言列表为空</div>",
    props:{
        list: {
            type: Array,
            default: function(){
                return [];
            }
        }
    },
    data() {
        return {
            list: this.list
        }
    },
    methods: {
        handleReply: function(index){
            this.$emit("reply", index);
        },
        deleteReply: function(index){
            this.$emit("delete", index);
        }
    },
})
```

style.css

```
[v-cloak]{
    display: none;
}
 
*{
    padding: 0;
    margin: 0;
}
 
.message{
    width: 450px;
    text-align: right;
    margin-top: 20px;
}
 
.message div{
    margin-bottom: 12px;
}
 
.message span{
    display: inline-block;
    width: 100px;
    vertical-align: top;
}
 
.message input, .message textarea{
    width: 300px;
    height: 32px;
    padding: 0 6px;
    color: #657180;
    border: 1px solid #d7dde4;
    border-radius: 4px;
    cursor: text;
    outline: none;
}
 
.message input:focus, .message textarea:focus{
    border: 1px solid #3399ff;
}
 
.message textarea{
    height: 60px;
    padding: 4px 6px;
}
 
.message button{
    display: inline-block;
    padding: 6px 15px;
    border: 1px solid #39f;
    border-radius: 4px;
    color: #fff;
    background-color: #3399ff;
    cursor: pointer;
    outline: none;
}
 
.list{
    margin-top: 50px;
}
 
.list-item{
    padding: 10px;
    border-bottom: 1px solid #e3e8ee;
    overflow: hidden;
}
 
.list-item span{
    display: block;
    width: 60px;
    float: left;
    color: #3399ff;
}
.list-msg{
    display: block;
    margin-left: 60px;
    text-align: justify;
}
 
.list-msg a{
    color: #9ea7b4;
    cursor: pointer;
    float: right;
}
 
.list-msg a:hover{
    color: #3399ff;
}
 
.list-nothing{
    text-align: center;
    color: #9ea7b4;
    padding: 20px;
}
```