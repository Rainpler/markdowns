> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/u014773932/article/details/117736981?spm=1001.2014.3001.5502)

练习 1 : 查阅资料, 了解表格的 <colgroup> 和 < col > 元素用法后, 给 v-table 的 columns 增加一个可以设置列宽的 width 宇段, 并实现该功能。  
练习 2 : 将该示例的 render 写法改写为 template 写法, 加以对比 , 总 结出两者的差异性, 深刻理解其使用场景。

效果图:

![](https://img-blog.csdnimg.cn/20210609115040411.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ3NzM5MzI=,size_16,color_FFFFFF,t_70)

项目文件（render 和 template 实现）： index.htm index.js table.js(分 render.js 或 template.js) style.css

index.html

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta >
    <title>可排序的表格组件</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div id="app" v-cloak>
        <button @click="handleAddData">添加数据</button>
        <v-table :data="data" :columns="columns"></v-table>
    </div>
 
    <script src="../../../vue.min.js"></script>
    <script src="table.js"></script>
    <script src="index.js"></script>
</body>
</html>
```

index.js

```
let app = new  Vue({
    el: "#app",
    data() {
        return {
            columns: [{
                title: "姓名",
                key: "name",
                width: "25%"
            },{
                title: "年龄",
                key: "age",
                sortable: true,
                width: "25%"
            },{
                title: "出生日期",
                key: "birthday",
                sortable: true,
                width: "25%"
            },{
                title: "地址",
                key: "address",
                width: "25%"
            }],
            data: [{
                name: "王小明",
                age: 18,
                birthday: "2010-02-21",
                address: "北京市朝阳区芍药居"
            },{
                name: "张小刚",
                age: 25,
                birthday: "1992-01-23",
                address: "北京市海淀区西二旗"
            },{
                name: "李晓红",
                age: 30,
                birthday: "1987-11-10",
                address: "上海市浦东新区世纪大道"
            },{
                name: "周小伟",
                age: 20,
                birthday: "1991-10-10",
                address: "深圳市南山区深南大道"
            }]
        }
    },
    methods: {
        handleAddData: function(){
            this.data.push({
                
                    name: "刘晓天",
                    age: 19,
                    birthday: "2021-10-10",
                    address: "北京市东城区东直门"
                
            })
        }
    },
})
```

render.js

```
Vue.component("vTable", {
    props: {
        columns: {
            type: Array,
            default: function () {
                return [];
            }
        },
        data: {
            type: Array,
            default: function () {
                return [];
            }
        }
    },
    data: function () {
        return {
            currentColumns: [],
            currentData: []
        }
    },
    methods: {
        makeColumns: function () {
            this.currentColumns = this.columns.map((col, index) => {
                //添加一个字段标识当前列排序的状态,后续使用
                col._sortType = "normal";
                //添加一个字段标识当前列在数组中的索引,后续使用
                col._index = index;
 
                return col;
            })
        },
        makeData: function () {
            this.currentData = this.data.map((row, index) => {
                row._index = index;
                return row;
            })
        },
        handleSortByAsc: function (index) {
            let key = this.currentColumns[index].key;
            this.currentColumns.forEach(col => {
                col._sortType = "normal";
            })
            this.currentColumns[index]._sortType = "asc";
 
            this.currentData.sort((a, b) => {
                return a[key] > b[key] ? 1 : -1;
            })
        },
        handleSortByDesc: function (index) {
            let key = this.currentColumns[index].key;
            this.currentColumns.forEach(col => {
                col._sortType = "normal";
            })
            this.currentColumns[index]._sortType = "desc";
            this.currentData.sort((a, b) => {
                return a[key] < b[key] ? 1 : -1;
            })
        }
    },
    mounted() {
        // v -table 初始化时调用
        this.makeColumns();
        this.makeData();
    },
    render(h) {
        let _this = this;
 
        let trs = [];
        this.currentData.forEach(element => {
            let tds = [];
            _this.currentColumns.forEach(arg => {
                tds.push(h("td", element[arg.key]));
            })
            trs.push(h("tr", tds));
        });
 
        let ths = [];
        let colArray = []
        this.currentColumns.forEach((col, index) => {
            if (col.sortable) {
                ths.push(h("th", [
                    h("span", col.title),
                    h("a", {
                        class: {
                            on: col._sortType === "asc"
                        },
                        on: {
                            click: function () {
                                _this.handleSortByAsc(index)
                            }
                        }
                    }, "↑"),
                    h("a", {
                        class: {
                            on: col._sortType === "desc"
                        },
                        on: {
                            click: function () {
                                _this.handleSortByDesc(index)
                            }
                        }
                    }, "↓"),
                ]));
            } else {
                ths.push(h("th", col.title))
            }
            colArray.push(h("col",{
                attrs: {
                    width: col.width
                }
            }))
        })
        
        //h 就是 createElement
        return h("table", [
            h("colgroup", colArray),
            h("thead", [
                h("tr", ths)
            ]),
            h("tbody", trs)
        ])
    },
    watch: {
        data: function () {
            this.makeData();
            let sortedColumn = this.currentColumns.filter(col => {
                return col._sortType !== "normal";
            });
 
            if (sortedColumn.length > 0) {
                if (sortedColumn[0]._sortType === "asc") {
                    this.handleSortByAsc(sortedColumn[0]._index);
                } else {
                    this.handleSortByDesc(sortedColumn[0]._index);
                }
            }
        }
    }
})
```

template.js

```
Vue.component("vTable", {
    template: "<table>\
        <colgroup> \
            <col v-for='item in currentColumns' :width='item.width'></col> \
        </colgroup> \
        <thead> \
            <th v-for='(item, index) in currentColumns'>{{ item.title }} \
                <a v-if='item.sortable' :class='{on: item._sortType == \"asc\" }' @click='handleSortByAsc(index)'>↑</a> \
                <a v-if='item.sortable' :class='{on: item._sortType == \"desc\"}' @click='handleSortByDesc(index)'>↓</a> \
            </th> \
        </thead> \
        <tbody> \
            <tr v-for='item in currentData'> \
                <td v-for='(arg, key) in item' v-if='key != \"_index\"'>{{ arg }}</td> \
            </tr> \
        </tbody> \
    </table>",
    props: {
        columns: {
            type: Array,
            default: function () {
                return [];
            }
        },
        data: {
            type: Array,
            default: function () {
                return [];
            },
        }
    },
    data: function () {
        return {
            currentColumns: [],
            currentData: []
        }
    },
 
    methods: {
        makeColumns: function () {
            this.currentColumns = this.columns.map((col, index) => {
                //添加一个字段标识当前列排序的状态,后续使用
                col._sortType = "normal";
                //添加一个字段标识当前列在数组中的索引,后续使用
                col._index = index;
                return col;
            })
        },
        makeData: function () {
            this.currentData = this.data.map((row, index) => {
                row._index = index;
                return row;
            })
        },
        handleSortByAsc: function (index) {
            let key = this.currentColumns[index].key;
            this.currentColumns.forEach(col => {
                col._sortType = "normal";
            })
            this.currentColumns[index]._sortType = "asc";
 
            this.currentData.sort((a, b) => {
                return a[key] > b[key] ? 1 : -1;
            })
        },
        handleSortByDesc: function (index) {
            let key = this.currentColumns[index].key;
            this.currentColumns.forEach(col => {
                col._sortType = "normal";
            })
            this.currentColumns[index]._sortType = "desc";
            this.currentData.sort((a, b) => {
                return a[key] < b[key] ? 1 : -1;
            })
        }
    },
    mounted() {
        // v -table 初始化时调用
        this.makeColumns();
        this.makeData();
    },
    watch: {
        data: function () {
            this.makeData();
            let sortedColumn = this.currentColumns.filter(col => {
                return col._sortType !== "normal";
            });
 
            if (sortedColumn.length > 0) {
                if (sortedColumn[0]._sortType === "asc") {
                    this.handleSortByAsc(sortedColumn[0]._index);
                } else {
                    this.handleSortByDesc(sortedColumn[0]._index);
                }
            }
        }
    }
})
```

style.css

```
[v-cloak]{
    display: none;
}
 
table{
    width: 100%;
    margin-bottom: 24px;
    border-collapse: collapse;
    border-spacing: 0;
    empty-cells: show;
    border: 1px solid #e9e9e9;
}
 
table th{
    background: #f7f7f7;
    color: #5c6b77;
    font-weight: 600;
    white-space: normal;
}
 
table td, table th{
    padding: 8px 16px;
    border: 1px solid #e9e9e9;
    text-align: left;
}
 
table th a{
    display: inline-block;
    margin: 0 4px;
    cursor: pointer;
}
 
table th a.on{
    color: #3399ff;
}
 
table th a:hover{
    color: #3399ff;
}
```