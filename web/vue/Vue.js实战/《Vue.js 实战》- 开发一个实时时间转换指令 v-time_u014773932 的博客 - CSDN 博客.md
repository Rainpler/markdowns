> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/u014773932/article/details/117706046?spm=1001.2014.3001.5502)

一 般在服务端的存储时间格式是 Unix 时间 戳, 比如 2017-01-01 00:00 : 00 的时间 戳是 1483200000 。前端在拿到数据后 , 将它转换为可读的时间格式再显示出来 。 为了显 示 出实时性, 在一些社交类产品中, 甚至会实时转换为几秒钟前、几分钟前、几小时前等不同的格式, 这样比直接转换为年、月、日、时、分、秒更友好。本示例就来实现这样一个自定义指令 v-time , 将表达式传入的时间戳实时转换为相对时间。

练习 1 : 开发一个自定义指令 v-birthday , 接收一个出生日期的时间戳, 将它转换为己经出生了 xxx 天 。  
练习 2 : 扩展练习 1 的 自定义指令 v-birthday , 将出生了 xxx 天转换为具体年龄, 比如 25 岁 8 个月 10 天 。

效果图:

![](https://img-blog.csdnimg.cn/20210608160212574.png)

文件：index.html index.js time.js birthday.js

index.html

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta >
    <title>时间转换指令</title>
</head>
<body>
    <!--在一般情况下, v-cloak 是一个解决初始化慢导致页面闪动的最佳实践-->
    <div id="app" v-cloak>
        <div v-time="timeNow"></div>
        <div v-time="timeBefore"></div>
        <hr>
        <div>出生日期： 2020-05-06</div>
        <div v-birthday="birthdayVal"></div>
    </div>
    <script src="../../vue.min.js"></script>
    <script src="time.js"></script>
    <script src="birthday.js"></script>
    <script src="index.js"></script>
</body>
</html>
```

index.js

```
let app = new Vue({
    el: "#app",
    //毫秒 服务端返回秒级时问戳需要乘以1000后再使用
    data() {
        return {
            timeNow: (new Date()).getTime(),
            timeBefore: 1488930695721,
            birthdayVal: "2020-05-06"
        }
    },
})
```

time.js

```
/* 
 * 1分钟以前,显示“刚刚”。
 * 1 分钟~ 1 小时之间,显示“xx 分钟前”.
 * 1 小时～ 1天之间,显示“xx 小时前”.
 * 1 天~ 1 个月 01 天)之间,显示“xx 天前 ” .
 * 大于 1 个月,显示“xx 年 xx 月 xx 曰’\ 
*/
let time = {
    //获取当前时间戳
    getUnix: function(){
        let date = new Date();
        return date.getTime();
    },
    //获取今天0点0分0秒的时间戳
    getTodayUnix: function(){
        let date = new Date();
        date.setHours(0);
        date.setMinutes(0);
        date.setSeconds(0);
        date.setMilliseconds(0);
        return date.getTime();
    },
    //获取今年1月1日0点0分0秒的时间戳
    getYearUnix: function(){
        let date = new Date();
        date.setMonth(0);
        date.setDate(1);
        date.setHours(0);
        date.setMinutes(0);
        date.setSeconds(0);
        date.setMilliseconds(0);
        return date.getTime();
    },
    //获取标准年月日
    getLastDate: function(time){
        let date = new Date(time);
        let month = date.getMonth() + 1 < 10 ? "0" + (date.getMonth() + 1): date.getMonth() + 1;
        let day = date.getDate() < 10 ? "0" + date.getDate() : date.getDate();
        return date.getFullYear() + "-" + month + "-" + day;
    },
    // 转换时间
    getFormatTime: function(timestamp){
        let now = this.getUnix(); //当前时间戳
        let today = this.getTodayUnix(); //今天 0 点时间戳
        let year = this.getYearUnix(); //今年 0 点时间戳
 
        let timer = (now-timestamp) / 1000; //转换为秒级时 间戳
        let tips = "";
        if(timer <= 0 ){
            tips = "刚刚";
        }else if(Math.floor(timer/60) <= 0){
            tips = "刚刚";
        }else if(timer < 3600){
            tips = Math.floor(timer/60) + "分钟前";
        }else if(timer >= 3600 && timestamp - today >= 0){
            tips = Math.floor(timer/3600) + "小时前";
        }else if(timer/86400 <= 31){
            tips = Math.ceil(timer/84600) + "天前";
        }else{
            tips = this.getLastDate(timestamp);
        }
        return tips;
    }
}
 
Vue.directive("time", {
    bind: function(el, binding){
        el.innerHTML = time.getFormatTime(binding.value);
        el._timeout_  = setInterval(function(){
            el.innerHTML = time.getFormatTime(binding.value);
        }, 60000);
    },
    unbind: function(el){
        clearInterval(el._timeout_);
        delete el._timeout_;
    }
})
```

birthday.js

```
let data = {
    getFormatTime: function (dateTemp) {
        let flag = [1, 3, 5, 7, 8, 10, 12, 4, 6, 9, 11, 2];
        
        let start = new Date(dateTemp);
        let end = new Date();
 
        let year = end.getFullYear() - start.getFullYear();
        let month = end.getMonth() - start.getMonth();
        let day = end.getDate() - start.getDate();
        if (month < 0) {
            year--;
            month = end.getMonth() + (12 - start.getMonth());
        }
        if (day < 0) {
            month--;
            let index = flag.findIndex((temp) => {
                return temp === start.getMonth() + 1
            });
            let monthLength;
            if (index <= 6) {
                monthLength = 31;
            } else if (index > 6 && index <= 10) {
                monthLength = 30;
            } else {
                monthLength = 28;
            }
            day = end.getDate() + (monthLength - start.getDate());
        }
        return year + "岁" + month + "个月"+ day + "天"
    }
}
 
Vue.directive("birthday", {
    bind: function (el, binding) {
        el.innerHTML = data.getFormatTime(binding.value);
        el._timeout_ = setInterval(function () {
            el.innerHTML = data.getFormatTime(binding.value);
        }, 1000 * 60 * 60 * 24);
    },
    unbind: function (el) {
        clearInterval(el._timeout_);
        delete el._timeout_;
    }
})
```