> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/u014773932/article/details/117707306?spm=1001.2014.3001.5502)

 描述: 商品列表按照价格、销量排序：商品列表按照品牌、价格过滤：动态的购物车；使用优惠码等。

练习 1 ：将品牌和颜色的筛选扩展为支持多选，比如支持同时选择自色和红色  
练习 2 ：购物车数据支持持久化（本地保存， 重新打开页面仍然有记录〉。

效果图:

![](https://img-blog.csdnimg.cn/20210629161008256.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ3NzM5MzI=,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20210629161019561.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ3NzM5MzI=,size_16,color_FFFFFF,t_70)

目录结构：

![](https://img-blog.csdnimg.cn/20210629153758642.png)

compontents/product.[vue](https://so.csdn.net/so/search?from=pc_blog_highlight&q=vue)

```
<template>
    <div class="product">
        <router-link :to="'/product/' + info.id" class="product-main">
            <img :src="info.image">
            <h4>{{ info.name }}</h4>
            <div class="product-color" :style="{ background: colors[info.color]}"></div>
            <div class="product-cost">¥ {{ info.cost }}</div>
            <div class="product-add-cart" @click.prevent="handleCart">加入购物车</div>
        </router-link>
    </div>
</template>
<script>
    export default {
        props: {
            info: Object
        },
        data () {
            return {
                colors: {
                    '白色': '#ffffff',
                    '金色': '#dac272',
                    '蓝色': '#233472',
                    '红色': '#f2352e'
                }
            }
        },
        methods: {
            handleCart () {
                this.$store.commit('addCart', this.info.id);
            }
        }
    };
</script>
<style scoped>
    .product{
        width: 25%;
        float: left;
    }
    .product-main{
        display: block;
        margin: 16px;
        padding: 16px;
        border: 1px solid #dddee1;
        border-radius: 6px;
        overflow: hidden;
        background: #fff;
        text-align: center;
        position: relative;
    }
    .product-main img{
        width: 100%;
    }
    h4{
        color: #222;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
    .product-main:hover h4{
        color: #0070c9;
    }
    .product-color{
        display: block;
        width: 16px;
        height: 16px;
        border: 1px solid #dddee1;
        border-radius: 50%;
        margin: 6px auto;
    }
    .product-cost{
        color: #de4037;
        margin-top: 6px;
    }
    .product-add-cart{
        display: none;
        padding: 4px 8px;
        background: #2d8cf0;
        color: #fff;
        font-size: 12px;
        border-radius: 3px;
        cursor: pointer;
        position: absolute;
        top: 5px;
        right: 5px;
    }
    .product-main:hover .product-add-cart{
        display: inline-block;
    }
</style>
```

views/cart.vue

```
<template>
    <div class="cart">
        
        <div class="cart-header">
            <div class="cart-header-title">购物清单</div>
            <div class="cart-header-main">
                <div class="cart-info">商品信息</div>
                <div class="cart-price">单价</div>
                <div class="cart-count">数量</div>
                <div class="cart-cost">小计</div>
                <div class="cart-delete">删除</div>
            </div>
        </div>
        <div class="cart-content">
            <div class="cart-content-main" v-for="(item, index) in cartList">
                <div class="cart-info">
                    <img :src="productDictList[item.id].image">
                    <span>{{ productDictList[item.id].name }}</span>
                </div>
                <div class="cart-price">￥{{ productDictList[item.id].cost }}</div>
                <div class="cart-count">
                    <span class="cart-control-minus" @click="handleCount(index, -1)">-</span>
                    {{ item.count }}
                    <span class="cart-control-add" @click="handleCount(index, 1)">+</span>
                </div>
                <div class="cart-cost">
                    ￥{{ productDictList[item.id].cost * item.count }}
                </div>
                <div class="cart-delete">
                    <span class="cart-control-delete" @click="handleDelete(index)">删除</span>
                </div>
            </div>
            <div class="cart-empty" v-if="!cartList.length">购物车为空</div>
        </div>
        <div class="cart-promotion" v-show="cartList.length">
            <span>使用优惠券码：</span>
            <input type="text" v-model="promotionCode">
            <span class="cart-control-promotion" @click="handleCheckedCode">验证</span>
        </div>
        <div class="cart-footer" v-show="cartList.length">
            <div class="cart-footer-desc">
                共计 <span>{{ countAll }}</span>件商品
            </div>
            <div class="cart-footer-desc">
                应付总额<span>￥{{ costAll - promotion }}</span>
                <br>
                <template v-if="promotion">
                    (优惠 <span>￥{{ promotion }}</span>)
                </template>
            </div>
            <div class="cart-footer-desc">
                <div class="cart-control-order" @click="handleOrder">现在结算</div>
            </div>
        </div>
    </div>
</template>
<script>
import product_data from '../product.js';
export default {
     data(){
        return{
            productList: product_data,
            promotionCode: '',
            promotion: 0
        }
    },
    methods:{
        handleCount(index, count){
            if(count < 0 && this.cartList[index].count === 1) return;
            this.$store.commit('editCartCount', {
                id: this.cartList[index].id,
                count: count
            });
        },
        handleDelete(index){
            this.$store.commit('deleteCart', this.cartList[index].id);
        },
        //使用优惠券
        handleCheckedCode(){
            if(this.promotionCode === ''){
                window.alert('请输入优惠码');
                return;
            }
            if(this.promotionCode !== 'Vue.js'){
                window.alert('优惠券验证失败');
            }else{
                this.promotion = 500;
            }
        },
        handleOrder(){
            this.$store.dispatch('buy').then(() => {
                window.alert('购买成功');
            })
        }
    },
    computed:{
        cartList () {
            // return this.$store.state.cartList;
            return this.$store.getters.cartList;
        },
        productDictList(){
            const dict = {};
            this.productList.forEach(element => {
                dict[element.id] = element;
            });
            return dict;
        },
        countAll(){
            let count = 0;
            this.cartList.forEach(item => {
                count += item.count;
            })
            return count;
        },
        costAll(){
            let cost = 0;
            this.cartList.forEach(item => {
                cost += this.productDictList[item.id].cost * item.count;
            });
            return cost;
 
        }
    }
}
</script>
<style scoped>
    .cart{
        margin: 32px;
        background: #fff;
        border: 1px solid #dddee1;
        border-radius: 10px;
    }
    .cart-header-title{
        padding: 16px 32px;
        border-bottom: 1px solid #dddee1;
        border-radius: 10px 10px 0 0;
        background: #f8f8f9;
    }
    .cart-header-main{
        padding: 8px 32px;
        overflow: hidden;
        border-bottom:  1px solid #dddee1;
        background: #eee;
        overflow: hidden;
    }
    .cart-empty{
        text-align: center;
        padding: 32px;
    }
    .cart-header-main div{
        text-align: center;
        float: left;
        font-size: 14px;
    }
    div.cart-info{
        width: 60%;
        text-align: left;
    }
    .cart-price, .cart-count, .cart-cost, .cart-delete{
        width: 10%;
    }
    .cart-content-main{
        padding: 0 32px;
        height: 60px;
        line-height: 60px;
        text-align: center;
        border-bottom: 1px dashed #e9eaec;
        overflow: hidden;
    }
    .cart-content-main div{
        float: left;
    }
    .cart-content-main img{
        width: 40px;
        height: 40px;
        position: relative;
        top: 10px;
    }
    .cart-control-minus,
    .cart-control-add{
        display: inline-block;
        margin: 0 4px;
        width: 24px;
        height: 24px;
        line-height: 22px;
        text-align: center;
        background: #f8f8f9;
        border-radius: 50%;
        box-shadow: 0 1px 1px rgba(0, 0, 0, .2);
        cursor: pointer;
    }
    .cart-control-delete{
        cursor: pointer;
        color: #2d8cf0;
    }
    .cart-promotion{
        padding: 16px 32px;
    }
    .cart-control-promotion,
    .cart-control-order{
        display: inline-block;
        padding: 8px 32px;
        border-radius: 6px;
        background: #2d8cf0;
        color: #fff;
        cursor: pointer;
    }
    .cart-control-promotion{
        padding: 2px 6px;
        font-size: 12px;
        border-radius: 3px;
    }
    .cart-footer{
        padding: 32px;
        text-align: right;
    }
    .cart-footer-desc{
        display: inline-block;
        padding: 0 16px;
    }
    .cart-footer-desc span{
        color: #f2352e;
        font-size: 20px;
    }
</style>
```

views/list.vue

```
<template>
  <div v-show="list.length">
    <div class="list-control">
      <div class="list-control-filter">
        <span>品牌:</span>
        <span
          class="list-control-filter-item"
          :class="{ on: filterBrand.indexOf(item) > -1 }"
          v-for="item in brands"
          @click="handleFilterBrand(item)"
          >{{ item }}</span
        >
      </div>
      <div class="list-control-filter">
        <span>颜色:</span>
        <span
          class="list-control-filter-item"
          :class="{ on: filterColor.indexOf(item) > -1 }"
          v-for="item in colors"
          @click="handleFilterColor(item)"
          >{{ item }}</span
        >
      </div>
      <div class="list-control-order">
        <span>排序:</span>
        <span
          class="list-control-order-item"
          :class="{ on: order === '' }"
          @click="handleOrderDefault"
          >默认</span
        >
        <span
          class="list-control-order-item"
          :class="{ on: order === 'sales' }"
          @click="handleOrderSales"
          >销量
          <template v-if="order === 'sales'">↓</template>
        </span>
        <span
          class="list-control-order-item"
          :class="{ on: order.indexOf('cost') > -1 }"
          @click="handleOrderCost"
          >价格
          <template v-if="order === 'cost-asc'">↑</template>
          <template v-if="order === 'cost-desc'">↓</template>
        </span>
      </div>
    </div>
    <Product v-for="item in filteredAndOrderedList" :info="item" :key="item.id"></Product>
    <div class="product-not-found" v-show="!filteredAndOrderedList.length">
      暂无相关产品
    </div>
  </div>
</template>
<script>
//导入商品简介组件
import Product from "../compontents/product.vue";
export default {
  components: { Product },
  data() {
    return {
      //排序依据，可选值为：
      //sales(销量)
      //cost-desc(价格降序)
      //cost-asc(价格升序)
      order: "",
      filterBrand: [],
      filterColor: [],
    };
  },
  methods: {
    handleOrderDefault() {
      this.order = "";
    },
    handleOrderSales() {
      this.order = "sales";
    },
    handleOrderCost() {
      if (this.order === "cost-desc") {
        this.order = "cost-asc";
      } else {
        this.order = "cost-desc";
      }
    },
    //筛选品牌
    handleFilterBrand(brand) {
      //单次点击选中，再次点击取消选中
      let index = this.filterBrand.indexOf(brand);
      if (index > -1) {
        this.filterBrand.splice(index, 1);
      } else {
        this.filterBrand.push(brand);
      }
    },
    //筛选颜色
    handleFilterColor(color) {
      let index = this.filterColor.indexOf(color);
      if (index > -1) {
        this.filterColor.splice(index, 1);
      } else {
        this.filterColor.push(color);
      }
    }
  },
  computed: {
    list() {
      //从Vuex获取商品列表数据
      return this.$store.state.productList;
    },
    brands() {
      return this.$store.getters.brands;
    },
    colors() {
      return this.$store.getters.colors;
    },
    filteredAndOrderedList() {
      //复制原始数据
      let list = [...this.list];
      //todo按品牌过滤
      if (this.filterBrand.length > 0) {
        list = list.filter((item) => this.filterBrand.indexOf(item.brand) > -1);
      }
      //todo按颜色过滤
      if (this.filterColor.length > 0) {
        list = list.filter((item) => this.filterColor.indexOf(item.color) > -1);
      }
      //排序
      if (this.order !== "") {
        if (this.order === "sales") {
          list = list.sort((a, b) => b.sales - a.sales);
        } else if (this.order === "cost-desc") {
          list = list.sort((a, b) => b.cost - a.cost);
        } else if (this.order === "cost-asc") {
          list = list.sort((a, b) => a.cost - b.cost);
        }
      }
      return list;
    },
  },
  mounted() {
    //初始化时，通过Vuex的action请求数据
    this.$store.dispatch("getProductList");
  },
};
</script>
<style scoped>
.product-not-found {
  text-align: center;
  padding: 32px;
}
.list-control {
  background: #fff;
  border-radius: 6px;
  margin: 16px;
  padding: 16px;
  box-shadow: 0 1px 1px rgba(0, 0, 0, 0.2);
}
.list-control-filter {
  margin-bottom: 16px;
}
.list-control-filter-item,
.list-control-order-item {
  cursor: pointer;
  display: inline-block;
  border: 1px solid #e9eaec;
  border-radius: 4px;
  margin-right: 6px;
  padding: 2px 6px;
}
.list-control-filter-item.on,
.list-control-order-item.on {
  background: #f2352e;
  border: 1px solid #ff2233;
  color: #fff;
}
</style>
```

views/product.vue

```
<template>
    <div v-if="product">
        <div class="product">
            <div class="product-image">
                <img :src="product.image.replace('.', '..')">
            </div>
            <div class="product-info">
                <h1 class="product-name">{{ product.name }}</h1>
                <div class="product-cost">￥{{ product.cost }}</div>
                <div class="product-add-cart" @click="handleAddToCart">加入购物车</div>
            </div>
        </div>
        <div class="product-desc">
            <h2>产品介绍</h2>
            <img v-for="n in 10" :src="'../images/'+ n +'.jpeg'">
        </div>
    </div>
</template>
<script>
import product_data from '../product.js';
export default {
    data(){
        return {
            //获取路由中的参数
            id: parseInt(this.$route.params.id),
            product: null
        }
    },
    methods:{
        getProduct(){
            setTimeout(() => {
                this.product = product_data.find(item => item.id === this.id);
            }, 550)
        },
        handleAddToCart(){
            this.$store.commit('addCart', this.id);
        }
    },
    mounted(){
        //初始化时，请求数据
        this.getProduct();
    }
}
</script>
<style scoped>
    .product{
        margin: 32px;
        padding: 32px;
        background: #fff;
        border: 1px solid #dddee1;
        border-radius: 10px;
        overflow: hidden;
    }
    .product-image{
        width: 50%;
        height: 550px;
        float: left;
        text-align: center;
    }
    .product-image img{
        height: 100%;
    }
    .product-info{
        width: 50%;
        padding: 150px 0 250px;
        height: 150px;
        float: left;
        text-align: center;
    }
    .product-cost{
        color: #f2352e;
        margin: 8px 0;
    }
    .product-add-cart{
        display: inline-block;
        padding: 8px 64px;
        margin: 8px 0;
        background: #2d8cf0;
        color: #fff;
        font-size: 12px;
        border-radius: 4px;
        cursor: pointer;
    }
    .product-desc{
        background: #fff;
        margin: 32px;
        padding: 32px;
        border: 1px solid #dddee1;
        border-radius: 10px;
        text-align: center;
    }
    .product-desc img{
        display: block;
        width: 50%;
        margin: 32px auto;
        padding: 32px;
        border-bottom: 1px solid #dddee1;
    }
</style>
```

app.vue

```
<template>
    <div>
        <div class="header">
            <router-link to="/list" class="header-title">电商网站示例</router-link>
            <div class="header-menu">
                <router-link to="/cart" class="header-menu-cart">
                    购物车
                    <span v-if="cartList.length">{{ cartList.length }}</span>
                </router-link>
            </div>
        </div>
        <router-view></router-view>
    </div>
</template>
<script>
export default {
    computed: {
        cartList(){
            // return this.$store.state.cartList;
            return this.$store.getters.cartList;
        }
    },
    mounted() {
        //初始化时，请求数据
        this.$store.dispatch("getProductList");
    }
}
</script>
```

main.js

```
import Vue from 'vue';
import VueRouter from 'vue-router';
import Routers from './router';
import Vuex from 'vuex';
import App from './app.vue';
import './style.css';
 
import product_data from './product';
 
Vue.use(VueRouter);
Vue.use(Vuex);
 
const RouterConfig = {
    mode: 'history',
    routes: Routers
}
const router = new VueRouter(RouterConfig);
 
router.beforeEach((to, from, next) => {
    window.document.title = to.meta.title;
    next();
})
 
router.afterEach((to, from, next) => {
    window.scrollTo(0, 0);
})
 
 
const store = new Vuex.Store({
    //vuex的配置
    state: {
        //商品列表数据
        productList: [],
        //购物车数据
        cartList: localStorage['cart'] ? JSON.parse(localStorage['cart']) : []
    },
    getters:{
        brands: state => {
            const brands = state.productList.map(item => item.brand);
            return getFilterArray(brands);
        },
        colors: state => {
            const colors = state.productList.map(item => item.color);
            return getFilterArray(colors);
        },
        //实时监听state值的变化(最新状态)
        cartList: state => state.cartList
    },
    mutations:{
        //添加商品列表
        setProductList(state, data){
            state.productList = data;
        },
        //添加到购物车
        addCart(state, id){
 
            //先判断购物车是否已有，如果有，数量+1
            const isAdded = state.cartList.find(item => item.id === id);
 
            if(isAdded){
                isAdded.count ++;
            }else{
                state.cartList.push({
                    id: id,
                    count: 1
                })
            }
            localStorage.setItem('cart', JSON.stringify(state.cartList));
        },
        //修改商品数量
        editCartCount(state, payload){
            const product = state.cartList.find(item => item.id === payload.id);
            product.count += payload.count;
            localStorage.setItem('cart', JSON.stringify(state.cartList));
        },
        //删除商品
        deleteCart(state, id){
            const index = state.cartList.findIndex(item => item.id === id);
            state.cartList.splice(index, 1);
            localStorage.setItem('cart', JSON.stringify(state.cartList));
        },
        emptyCart(state){
            state.cartList = [];
            localStorage.setItem('cart', JSON.stringify(state.cartList));
        }
    },
    actions: {
        //请求商品列表
        getProductList(context){
            //真实环境通过ajax获取，这里用异步模拟
            setTimeout(() => {
                context.commit('setProductList', product_data);
            }, 500)
        },
        buy(context){
            return new Promise(resolve => {
                setTimeout(() => {
                    context.commit('emptyCart');
                    resolve();
                }, 500)
            })
        }
    }
});
 
new Vue({
    el: '#app',
    router: router,
    store: store,
    render: h => {
        return h(App)
    }
});
 
//数组排重
function getFilterArray(array){
    const res = [];
    const json = {};
    for(let i = 0; i < array.length; i++){
        const _self = array[i];
        if(!json[_self]){
            res.push(_self);
            json[_self] = 1;
        }
    }
    return res;
}
```

product.js

```
export default[
    {
        id: 1,
        name: 'AirPods',
        brand: 'Apple',
        image: './images/1.jpeg',
        sales: 10000,
        cost: 1288,
        color: '白色'
    },
    {
        id: 2,
        name: 'BeatsX入耳式耳机',
        brand: 'Beats',
        image: './images/2.jpeg',
        sales: 11000,
        cost: 1188,
        color: '白色'
    },
    {
        id: 3,
        name: 'Beats Solos Wireless 头戴式耳机',
        brand: 'Beats',
        image: './images/3.jpeg',
        sales: 5000,
        cost: 2288,
        color: '金色'
    },
    {
        id: 4,
        name: 'Beats Pill+ 便捷式扬声器',
        brand: 'Beats',
        image: './images/4.jpeg',
        sales: 3000,
        cost: 1888,
        color: '红色'
    },
    {
        id: 5,
        name: 'Sonos PLAY:1 无线扬声器',
        brand: 'Sonos',
        image: './images/5.jpeg',
        sales: 8000,
        cost: 1578,
        color: '白色'
    },
    {
        id: 6,
        name: 'Powerbeats3 by Dr. Dre Wireless 入耳式耳机',
        brand: 'Beats',
        image: './images/6.jpeg',
        sales: 12000,
        cost: 1488,
        color: '金色'
    },
    {
        id: 7,
        name: 'Beats EP 头戴式耳机',
        brand: 'Beats',
        image: './images/7.jpeg',
        sales: 25000,
        cost: 788,
        color: '蓝色'
    },
    {
        id: 8,
        name: 'B&O PLAY BeoPlay Al 便携式蓝牙扬声器',
        brand: 'B&O',
        image: './images/8.jpeg',
        sales: 15000,
        cost: 1898,
        color: '金色'
    },
    {
        id: 9,
        name: 'Bose® QuietComfort® 35 无线耳机',
        brand: 'Bose',
        image: './images/9.jpeg',
        sales: 14000,
        cost: 2878,
        color: '蓝色'
    },
    {
        id: 10,
        name: 'B&O PLAY Beoplay H4 无线头戴式耳机',
        brand: 'B&O',
        image: './images/10.jpeg',
        sales: 9000,
        cost: 2289,
        color: '金色'
    }
]
```

router.js

```
const routers = [
    {
        path: '/product/:id',
        meta:{
            title: '商品详情'
        },
        component: (resolve) => require(['./views/product.vue'], resolve)
    },
    {
        path: '/list',
        meta:{
            title: '商品列表'
        },
        component: (resolve) => require(['./views/list.vue'], resolve)
    },
    {
        path: '/cart',
        meta:{
            title: '购物车'
        },
        component: (resolve) => require(['./views/cart.vue'], resolve)
    },
    {
        path: '*',
        redirect: '/list'
    }
];
export default routers;
```

style.css

```
*{
    margin: 0;
    padding: 0;
}
a{
    text-decoration: none;
}
body{
    background-color: #f8f8f9;
}
.header{
    height: 48px;
    line-height: 48px;
    background: rgba(0, 0, 0, .8);
    color: #fff;
}
.header-title{
    padding: 0 32px;
    float: left;
    color: #fff;
}
.header-menu{
    float: right;
    margin-right: 32px;
}
.header-menu-cart{
    color: #f8f8f9;
}
.header-menu-cart span{
    display: inline-block;
    width: 16px;
    height: 16px;
    line-height: 16px;
    text-align: center;
    border-radius: 50%;
    background-color: #ff5500;
    color: #fff;
    font-size: 12px;
}
```