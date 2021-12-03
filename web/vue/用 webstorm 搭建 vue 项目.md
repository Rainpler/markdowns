> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/thumbs_up_sign_ygj/article/details/104979272)

前提：已安装 webstorm，这里主要讲搭建，webstorm 的下载就不在这里说了。

**1、了解一下基本知识  
1.1、Node.js：**  
Node.js 是一个基于  
Chrome V8 引擎的 [JavaScript](https://so.csdn.net/so/search?from=pc_blog_highlight&q=JavaScript) 运行环境。  
Node.js 使用了一个事件驱动、非阻塞式 I/O 的模型，使其轻量又高效。

Node.js 的包管理器 npm，是全球最大的开源库生态系统。

nodejs 版本查询：node -v

**1.2、npm：**  
npm 全称为 Node Package Manager，是一个基于 Node.js 的包管理器，也是整个 Node.js 社区最流行、支持的第三方模块最多的包管理器 (类似于 java 中的 Maven)。

npm 的初衷：JavaScript 开发人员更容易分享和重用代码。

npm 的使用场景：  
允许用户获取第三方包并使用。  
允许用户将自己编写的包或命令行程序进行发布分享。

npm 版本查询：npm -v

**1.3、Webpack**  
WebPack 可以看做是模块打包机：它做的事情是，分析你的项目结构，找到 JavaScript 模块以及其它的一些浏览器不能直接运行的拓展语言（Scss，TypeScript 等），并将其转换和打包为合适的格式供浏览器使用。

**2、下载 Node.js（自带 npm）**  
打开官网下载链接: https://nodejs.org/en/download/

**3、在 nodejs 安装路径下，新建 node_global 和 node_cache 两个文件夹，如下图：**  
![](https://img-blog.csdnimg.cn/2020031922573415.png)  
**4、设置 nodejs prefix（全局）和 cache（缓存）路径**  
win+R，输入 cmd

设置缓存文件夹  
npm config set cache “D:\[vue](https://so.csdn.net/so/search?from=pc_blog_highlight&q=vue)Project\nodejs\node_cache”

设置全局模块存放路径  
npm config set prefix “D:\vueProject\nodejs\node_global”

**5、设置环境变量**  
5.1、系统变量  
![](https://img-blog.csdnimg.cn/20200319225924340.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RodW1ic191cF9zaWduX3lnag==,size_16,color_FFFFFF,t_70)  
path：  
![](https://img-blog.csdnimg.cn/20200319225943464.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RodW1ic191cF9zaWduX3lnag==,size_16,color_FFFFFF,t_70)  
5.2、用户变量  
![](https://img-blog.csdnimg.cn/20200319230011348.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RodW1ic191cF9zaWduX3lnag==,size_16,color_FFFFFF,t_70)  
**6、测试**  
![](https://img-blog.csdnimg.cn/20200319230050398.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RodW1ic191cF9zaWduX3lnag==,size_16,color_FFFFFF,t_70)

**7、安装淘宝镜像（类似于阿里云的 maven 中央仓库镜像）**

安装命令：npm install -g cnpm --registry=https://registry.npm.taobao.org

验证命令：cnpm -v

**8、安装 webpack**

命令行语句为 cnpm install webpack -g

**9、安装 Vue**  
cnpm install vue -g

**10、安装 vue 命令行工具，即 vue-cli 脚手架**  
cnpm install vue-cli -g

**11、检测是否安装成功**  
![](https://img-blog.csdnimg.cn/20200319230307231.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RodW1ic191cF9zaWduX3lnag==,size_16,color_FFFFFF,t_70)  
11.1、如果 webpack -v 出现

```
We will use "npm" to install the CLI via "npm install -D".
Do you want to install 'webpack-cli' (yes/no):
不用管 直接回车 然后执行
此命令npm install webpack-cli -g
```

11.2、vue -V(大写)

**12、下载安装 git（webStorm 创建 Vue 需要用的，不然会报错）**  
官网：https://git-scm.com/  
三方：https://pc.qq.com/detail/13/detail_22693.html  
没办法，我下载时，官网不知道咋回事就是下不了，三方也还行啦。

**13、使用 webstorm 创建 vue**

打开 webStorm （我用是版本是 2019.1）  
![](https://img-blog.csdnimg.cn/20200319230827187.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RodW1ic191cF9zaWduX3lnag==,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20200319230833587.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RodW1ic191cF9zaWduX3lnag==,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20200319230839137.png)  
![](https://img-blog.csdnimg.cn/2020031923085396.png)  
![](https://img-blog.csdnimg.cn/202003192309074.png)  
![](https://img-blog.csdnimg.cn/20200319230917120.png)  
![](https://img-blog.csdnimg.cn/20200319230923322.png)  
![](https://img-blog.csdnimg.cn/20200319230929992.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RodW1ic191cF9zaWduX3lnag==,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20200319230938693.png)  
![](https://img-blog.csdnimg.cn/20200319231106651.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RodW1ic191cF9zaWduX3lnag==,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20200319231117517.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RodW1ic191cF9zaWduX3lnag==,size_16,color_FFFFFF,t_70)  
OK，使用 webstorm 搭建 vue 项目到此就结束啦~~~