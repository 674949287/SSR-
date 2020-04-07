# SSR-
同时需要一个客户端的渲染文件和服务端的渲染文件：entry-client.js（生成bundle.client.js,为了单页面操作）和entry-server.js（生成bundle.server.js，为了做SEO）
服务器渲染的核心：通过vue-server-renderer插件的renderToString()方法，将vue实例转换成为字符串插入到html文件（使用服务器渲染是为了弥补单页面应用SEO能力不足的问题）

1.安装vue-server-renderer插件和express服务器
   执行：npm i  express vue-server-renderer -S

客户端渲染做单页面操作 混合 服务端渲染做SEO优化

2.在package.json中新增server和client
    "server": "webpack --config build/webpack.server.conf.js",
    "client": "webpack --config build/webpack.client.conf.js"

3.在build文件夹下创建对应的webpack.server.conf.js和webpack.client.conf.js
（1）webpack.server.conf.js：

	const webpack = require('webpack')
	const merge = require('webpack-merge')
	//引入的是webpack的主要配置文件
	const base = require('./webpack.base.conf')
	module.exports = merge(base,{
 	   target:'node',  //让后端支持require语法
  	   entry:"./src/entry-server.js",
  	   output:{
    	   filename:'bundle.server.js',
   	    libraryTarget:'commonjs2'
  	   },
 	   plugins:[]
	})

（2）webpack.client.conf.js：
	const webpack = require('webpack')
	const path = require('path')
	function resolve (dir) {
	  return path.join(__dirname, '..', dir)
	}
	module.exports = {
	  entry:'./src/entry-client.js',
	  output:{
	    path:path.resolve(__dirname,'../dist'),
	    publicPath:'/dist/',
	    filename:'bundle.client.js'
	  },
	  plugins:[
	  ],
	  resolve:{
	    extensions:['.js','.vue','.json'],
	    alias:{
	      'vue$':'vue/dist/vue.esm.js',
	      '@':resolve('src')
	    }
	  },
	  module:{
	    rules:[
	      {
 	        test:/\.vue$/,
	        loader:'vue-loader',
	        options:{
	          compilerOptions:{
	            preserveWhiteSpace:false
	          }
	        }
	      },
	      {
	        test:'/\.js$/',
	        loader:'babel-loader',
	        include:[resolve('src'),resolve('test'),resolve('node_modules/webpack-dev-server/client')]
	      }
	    ]
	  }
	}

4.对应在src下新建entry-server.js和entry-client.js
（1）entry-server.js：

	import {createApp} from "./main";
	export default context =>{
	  return new Promise((resolve,reject)=>{
	    const {app} = createApp() //创建app
	    const router = app.$router  //创建路由
	    const {url} = context
	    const {fullPath} = router.resolve(url).route    
	    if (fullPath !==url ){
	      return reject({url:fullPath})
	    }
	    //更改路由
	    router.push(url)
	    router.onReady(()=>{
	      const matchedComponents = router.getMatchedComponents() //获取到组件
	      //如果组件没有长度
	      if(!matchedComponents.length){
	        return reject({code:404})
	      }
	    //遍历由下所有的组件，如果有需要服务器渲染的请求，则进行请求
	      Promise.all(matchedComponents.map(component =>{
	        if (component.serverRequest){
	          //未来各组件如果有serverRequest对象，判断是否需要服务器请求数据，并传入store参数
	          return component.serverRequest(app.$store);
	        }
	      })).then(()=>{
	        //将状态存储起来
	        context.state = app.$store.state;
	        resolve(app);
	      }).catch(reject)
	    },reject)
	  })
	}

（2）entry-client.js：
	//编写客户端入口
	import {createApp} from "./main";
	const {app} =createApp();
	const router = app.$router;
	//同步服务端信息
	if (window.__INITIAL_STATE__){
	  app.$store.replaceState(window.__INITIAL_STATE__); //状态中对象，替换全部组件
	}
	window.onload = function () {
	  app.$mount('#app');
	}

5.在SSR下新建store文件夹，和store文件夹下的index.js文件
   index.js:

	import Vue from 'vue'
	import Vuex from 'vuex'
	import axios from 'axios'
	Vue.use(Vuex);
	export function createStore() {
	  let store = new Vuex.Store({
	    state:{
	      homeInfo:""
	    },
	    mutations:{
	      setHomeInfo(state,res){
	        state.homeInfo = res;
	      }
	    },
	    actions:{
	      getHomeInfo({commit}){
	        return axios.get('http://localhost:8881/api/getHomeInfo').then((res)=>{
	          commit('setHomeInfo',res.data);
	        })
	      }
	    },
	  })
	  return store
	}

6.更改main.js和router下的index.js（更换原本输出的方式为函数，每次都会去创建一个新的）
（1）main.js:

	import Vue from 'vue'
	import App from './App'
	import {createRouter} from './router'
	import {createStore} from './store'
	//每一次执行创建一个新的组件
	export function createApp() {
	  const router = createRouter();
	  const store = createStore();
	  const app = new Vue({
	    store,
	    router,
	    components:{App},
	    template:'<App/>'
	  })
	  return {app,router,store}
	}
（2）router下的index.js
	//每次创建一个新的路由
	export function createRouter() {
	  return new Router({
	    mode:'history',
	    routes: [
	      {
	        path: '/',
	        name: 'Home',
	        component: Home
	      },
	      {
	        path: '/about',
	        name: 'About',
	        component: About
	      },
	      {
	        path: '/test',
	        name: 'Test',
	        component: Test
	      },
	    ]
	  })
	}

6.从后端发起的数据，通过serverRequest方法，去dispatch数据，然后在computed中拿到
   Home.vue:

      serverRequest(store){
        return store.dispatch('getHomeInfo');
      },
      computed:{
        homeInfo(){
            return this.$store.state.homeInfo;
          }
      }

7.执行npm run server 和 npm run client
   对应生成dist文件和dist文件下的bundle.server.js和bundle.client.js

8.在文件夹根目录下SSR新建一个server.js(node服务器)

   const Vue = require('vue')
   const exp = require('express')
   const express = exp()
   //创建服务端的渲染器
   const renderer = require('vue-server-renderer').createRenderer()
   //服务端渲染bundle文件
   const createApp = require('./dist/bundle.server')['default']

   //设置静态资源目录
   express.use('/',exp.static(__dirname+'/dist'));

   //客户端bundle
   const clientBundleFileUrl = '/bundle.client.js';

   express.get('/api/getHomeInfo',(req,res)=>{
     res.send('SSR发送请求了'); //服务端数据
   })
   // 服务器渲染的核心：通过vue-server-renderer插件的renderToString()方法，将vue实例转换成为字符串插入到html文件（使用服务器渲染事为了弥补单页面应用SEO能力不足的问题）
   express.get('*',(req,res)=>{
     const context = {url:req.url};
     createApp(context).then(app=>{
       console.log(context.state); //服务端的数据已经到服务端来了，接下来需要把数据共享给客户端
       let state = JSON.stringify(context.state);	//转换成JSON格式，通过scrpt标签window.__INITIAL_STATE__=${state}中间桥梁，把后端发送的数据渲染到前端html上
       renderer.renderToString(app,  (err,html)=>{
         // console.log(html);
         if (err){return res.state(500).end('运行错误')}
         res.send(`
         <!DOCTYPE html>
         <html lang="en">
           <head>
               <title>Vue2.0 SSR渲染页面</title>
               <script>window.__INITIAL_STATE__=${state}</script>
               <script src="${clientBundleFileUrl}"></script>
           </head>
           <body>${html}</body>
         </html>
       `)
       })
     })
       }).catch(()=>{
    
     })
   })
   //服务器监听地址
   express.listen(8881,()=>{
     console.log('服务器已启动！')
   })

9.通过node server.js 启动服务器，在对应的http://localhost:8881 端口下，看到渲染后的结果
