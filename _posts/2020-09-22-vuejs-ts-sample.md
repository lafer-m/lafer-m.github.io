---
layout: post
title: vue3.0+typescript+ant-design-vue+beego搭建简单的devops页面自动部署工具（一）
tags: [web]
---
  vue3.0 + typescript + ant-design-vue@next 搭建简单的web开发环境, 由于vue3.0组件库还不完善，所以不推荐生产环境使用，仅测试用。   
  本文搭建的工具为前后端分离的，[github地址](https://github.com/lafer-m/devops-deploy), 后台beego，部署工具默认为ansible,通过将ansible的all.yml及
  inventory配置放到页面生成，达到简单定制部署的目的。当然不是很完善，希望这个初始化的项目对你有所帮助。ansible-tower是官方的部署页面，功能完善，推荐使用。
### 起手式
如果有安装1.x 2.x的vue-cli需要先卸载，本文基于vue cli来搭建 
 
```
    npm uninstall vue-cli -g 
#Vue CLI 4.x 需要 Node.js v8.9 或更高版本 (推荐 v10 以上), 可以用nvm管理多版本node  
    node --version
    v10.15.3
``` 

安装vue cli
```
    npm install -g @vue/cli
    @vue/cli 4.5.4
    #创建一个新的自动部署的页面项目  
    vue create auto-setup-web
    手动选择如下的配置创建项目, 然后一路回车下去。    
    Vue CLI v4.5.4
    ? Please pick a preset: Manually select features
    ? Check the features needed for your project: 
     ◉ Choose Vue version
     ◉ Babel
     ◉ TypeScript
    ❯◯ Progressive Web App (PWA) Support
     ◉ Router
     ◉ Vuex
     ◉ CSS Pre-processors
     ◉ Linter / Formatter
     ◉ Unit Testing
     ◯ E2E Testing

```

配置https://cli.vuejs.org/zh/config/#%E5%85%A8%E5%B1%80-cli-%E9%85%8D%E7%BD%AE  
  官方文档撸起，vue.config.js配置好之后，就可以开始撸码了，引入element ui组件,如果是用的最新的vue3.0版本，那么现在element ui还是不支持的。  
  本文用的就是vue3.0，目前pc端的ui库有ant-design-vue支持vue3.0的测试版本，所以本文就基于ant vue来做一些测试。  
  本文的示例代码全部来自devops-deploy仓库，详细请查看github，可随便使用。
  
### ant-design-vue
安装依赖
```
    yarn add ant-design-vue@next
```

经过初始化后，可以通过src目录下的main.js来引入ant,本文简单使用，只import了部分组件    
vue3.0果然是简单明了。  

```
  import {createApp} from 'vue'
  import {Select} from 'ant-design-vue'
  import App from './App.vue'
  import router from './router'
  import store from './store'
  // 引入style文件
  import 'ant-design-vue/dist/antd.css'
  
  createApp(App).use(Select).use(store).use(router).mount('#app')
```

### axios http client使用  

```
npm install -save axios
# src目录下新建api目录， 代码如下，添加请求拦截，封装request
import axios, {AxiosRequestConfig, Method} from 'axios'

const host = window.location.hostname
const baseURL = 'http://' + host + ':8082'
const axiosInstance = axios.create({
    baseURL: baseURL,
    timeout: 60000
})

// 添加请求拦截器, 例如添加一个 auth token到请求头部
axiosInstance.interceptors.request.use((config: AxiosRequestConfig) => {
    config.headers['Authorization'] = 'test'
    return config
}, err => {
    console.log(err)
    return Promise.reject(err)
})

// 添加响应拦截器
axiosInstance.interceptors.response.use(resp => {
    return resp
}, err => {
    console.log(err)
    return Promise.reject(err)
})

interface FetchConfig {
    path?: string;
    body?: object;
    query?: object;
    headers?: object;
}

/*
 * request 封装axios http请求
 * @Param  method  get/post/put/delete..
 * @Param  axios params
 */
export default function request(method: Method, config: FetchConfig = {path: '', body: {}, query: {}, headers: {}}) {
    const url = baseURL + config.path
    return axiosInstance({
        url,
        method,
        headers: {
            ...config.headers
        },
        data: config.body,
        params: {
            ...config.query
        }
    })
}
```

### vue-class-component@next使用,目前还在还是beta版本  

这个组件目前也有beta版本支持vue3，如下说明一下简单的使用

```
##先封装一个Watch及Prop的decorator装饰器，方便后续的component页面使用
import {createDecorator} from "vue-class-component";

export const Watch = (valueKey: string) => {
    return createDecorator((options, handler) => {
        if (!options.watch) {
            options.watch = {}
        }
        options.watch[valueKey] = {
            handler: handler,
            deep: true
        }
    })
}

export const Prop = (cops: {}) => {
    if (!cops) {
        cops = {}
    }
    return createDecorator((options, key) => {
        if (!options.props) {
            options.props = {}
        }
        options.props[key] = cops
    })
}

# topbar组件部署代码使用示例,
    @Options({})
    export default class Topbar extends Vue {
        @Prop({})
        ProductName!: string;

        // ProductName = '';
        projectChoosed = '';
        version = '';
        projectNames: string[] = [];
        projects: Products[] = [];
        test = 'abc'

        @Watch('version')
        versionChange(nv: string, ov: string) {
            this.$store.commit('products/setVersion', nv)
            console.log(nv, ov)
            console.log(this.$store.state.products.version)
        }
    }
```

### vuex store使用  

vuex store使用官方文档已经有4.0的，可以用来给各个组件间做数据
共享。

```
# products 实现modules的接口 state getters actions mutations
import {createStore} from 'vuex'
import products from "@/store/modules/products";

export default createStore({
    modules: {
        products
    }
})
```

### ant-design-vue 组件使用  

2.x版本支持vue3，文档https://2x.antdv.com/docs/vue/introduce-cn/  
引入ant的时候需要开启less loader,安装相应的依赖。vue.config.js中引入如下配置

```
css: {
        loaderOptions: {
            less: {
                lessOptions: {
                    javascriptEnabled: true
                }
            }
        }
    }
```


### style-resources-loader如何全局引入scss  

vue.config.js配置文件中引入如下配置
```
pluginOptions: {
        'style-resources-loader': {
            preProcessor: 'scss',
            patterns: [
                path.resolve(__dirname, './src/style/placeholders/*.scss'),
                path.resolve(__dirname, './src/style/mixins/*.scss'),
                path.resolve(__dirname, './src/style/common/var.scss'),
            ],
        },
    },
```

#### 全局mixin扩展引入  

mixin引入的话，也是通过style-resources-loader来全局引入

### yaml配置文件生成及下载
todo

### deploy部署触发。
todo


### sample pic  

![demo](http://www.mrzzjiy.cn/assets/demo.png)






    

   
     


    
    