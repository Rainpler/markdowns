> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/u014773932/article/details/117444373?spm=1001.2014.3001.5502)

练习 1 : 在当前示例基础上扩展商品列表 , 新增一项是否选中该商品的功能, 总价变为只计算选中商品的总价 , 同时提供一个全选的按钮。  
练习 2 : 将商品列表 list 改为一个[二维数组](https://so.csdn.net/so/search?from=pc_blog_highlight&q=%E4%BA%8C%E7%BB%B4%E6%95%B0%E7%BB%84)来实现商品的分类 , 比如可分为 “电子产品”“生活用品” 和“果蔬” , 同类商品聚合在一起。提示, 你可能会用到两次 v-for 。

效果图：

![](https://img-blog.csdnimg.cn/20210601153941751.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ3NzM5MzI=,size_16,color_FFFFFF,t_70)

代码：

```
<!DOCTYPE html>
<html lang="en">
 
<head>
    <meta charset="UTF-8">
    <meta >
    <title>购物车示例</title>
    <style>
        [v-cloak] {
            display: none;
        }
 
        table {
            border: 1px solid #e9e9e9;
            border-collapse: collapse;
            border-spacing: 0;
            empty-cells: show;
        }
 
        th,
        td {
            padding: 8px 16px;
            border: 1px solid #e9e9e9;
            text-align: left;
        }
 
        th {
            background-color: #f7f7f7;
            color: #5c6b77;
            font-weight: 600;
            white-space: nowrap;
        }
    </style>
</head>
 
<body>
    <div id="app" v-cloak>
        <template v-if="list.length">
            <table>
                <thead>
                    <tr>
                        <!--根据checkAll是否为真，动态添加属性checked-->
                        <th><input type="checkbox" @click="checkAllItem" :checked="checkAll">全选</th>
                        <th>类型</th>
                        <th>序号</th>
                        <th>商品名称</th>
                        <th>商品单价</th>
                        <th>购买数量</th>
                        <th>操作</th>
                    </tr>
                </thead>
                <tbody>
                    <template v-for="(item, index) in list">
                        <tr v-for="(sub, p) in item">
                            <td><input type="checkbox" class="check-input" @click="checkItem(index, p)" :checked="sub.check"></td>
                            <!-- 第一行向下合并同类的数据 -->
                            <td v-if="p === 0" :rowspan="item.length">{{ sub.class | formatterClass}}</td>
                            <td>{{ p + 1 }}</td>
                            <td>{{ sub.name }}</td>
                            <td>{{ sub.price }}</td>
                            <td>
                                <button @click="handleReduce(index, p)" :disable="sub.count === 1">-</button>
                                {{ sub.count }}
                                <button @click="handleAdd(index, p)">+</button>
                            </td>
                            <td><button @click="handleRemove(index, p)">移除</button></td>
                        </tr>
                    </template>
                </tbody>
            </table>
            <div>总价: ￥ {{ totalPrice }} </div>
        </template>
        <div v-else>购物车为空</div>
    </div>
 
    <script src="../vue.js"></script>
    <script>
        let app = new Vue({
            el: "#app",
            data() {
                return {
                    checkAll: true,
                    list: [
                        [{
                            id: 1,
                            name: "iphone 7",
                            price: 6188,
                            count: 1,
                            class: "electronic",
                            check: true
                        }, {
                            id: 2,
                            name: "ipad pro",
                            price: 5188,
                            count: 1,
                            class: "electronic",
                            check: true
                        }, {
                            id: 3,
                            name: "macbook pro",
                            price: 21488,
                            count: 1,
                            class: "electronic",
                            check: true
                        }],
                        [{
                            id: 4,
                            name: "牙刷",
                            price: 5,
                            count: 2,
                            class: "dailyUse",
                            check: true
                        }, {
                            id: 5,
                            name: "毛巾",
                            price: 15,
                            count: 1,
                            class: "dailyUse",
                            check: true
                        }, {
                            id: 6,
                            name: "勺子",
                            price: 10,
                            count: 1,
                            class: "dailyUse",
                            check: true
                        }, {
                            id: 7,
                            name: "杯子",
                            price: 52,
                            count: 1,
                            class: "dailyUse",
                            check: true
                        }],
                        [{
                            id: 8,
                            name: "荔枝",
                            price: 5,
                            count: 1,
                            class: "FruitsAndVegetables",
                            check: true
                        }, {
                            id: 9,
                            name: "李果",
                            price: 2.5,
                            count: 1,
                            class: "FruitsAndVegetables",
                            check: true
                        }]
                    ]
                }
            },
            filters: {
                formatterClass(value) {
                    if (value == "FruitsAndVegetables") return "果蔬";
                    if (value == "dailyUse") return "生活用品";
                    if (value == "electronic") return "电子产品";
                }
            },
            computed: {
                totalPrice: function () {
                    let total = 0;
                    for (let i = 0; i < this.list.length; i++) {
                        let item = this.list[i];
                        for (let j = 0; j < item.length; j++) {
                            if (item[j].check) {
                                total += item[j].price * item[j].count;
                            }
                        }
                    }
                    // “千位分隔符”
                    return total.toString().replace(/\B(?=(\d{3})+$)/g, ",");
                }
            },
            methods: {
                handleReduce: function (index, p) {
                    if (this.list[index][p].count === 1) return;
                    this.list[index][p].count--;
                },
                handleAdd: function (index, p) {
                    this.list[index][p].count++;
                },
                handleRemove: function (index, p) {
                    this.list[index].splice(p, 1)
                },
                checkAllItem: function () {
                    this.checkAll = !this.checkAll;
                    for (let i = 0; i < this.list.length; i++) {
                        let item = this.list[i];
                        for (let j = 0; j < item.length; j++) {
                            item[j].check = this.checkAll;
                        }
                    }
                },
                checkItem: function (index, p) {
                    this.list[index][p].check = !this.list[index][p].check;
                }
            },
        })
    </script>
</body>
 
</html>
```