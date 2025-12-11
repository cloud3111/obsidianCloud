---
banner: "[[pixel-banner-image.png]]"
title: web开发的踩坑避雷
tags:
  - Web
categories: 编程
date: 2025-03-22T15:48:00
---

	we've got you covered there, too.
# Web开发总结
## `axios` 详解
### `axiox` 实例的创建
```javascript
import axios from 'axios'; 
/* 创建 axios 实例的好处：  
 可以配置默认 baseURL，避免重复写 API 地址
 可以添加请求/响应拦截器，统一处理 Token、错误消息等
 便于维护和管理多个 API 端点
*/
export function createAxiosInstance() {
    const axiosInstance = axios.create({
        baseURL: process.env.VUE_APP_SERVER, // 替换为你的后端 API 地址
        timeout: 5000, // 请求超时时间
        withCredentials: true,  //允许跨域
        headers: {
            'Content-Type': 'application/json',
        },
    });
    return axiosInstance;
}
// 或者直接修改全局默认配置, 否则需要调用实例 注意: 要定义在mian.js中才生效
axios.defaults.baseURL = process.env.VUE_APP_SERVER;
axios.defaults.timeout = 5000;
axios.defaults.withCredentials = true;
axios.defaults.headers['Content-Type'] = 'application/json';
```
### `axios` 拦截器
```javascript
// 请求拦截器
axios.interceptors.request.use(
    (config) => {
        // 打印请求参数
        console.log("请求参数: " ,config)
        // 从本地存储或者商店中拿
        const token = localStorage.getItem('Authorization');
        config.headers["Authorization"] = `${token}`;
        config.headers['Content-Encoding'] = "utf-8"
        return config;
    }
);
// 响应拦截器
axios.interceptors.response.use(
    (response) => {
        // 打印响应参数
        console.log("响应参数: " ,response)
        // HTTP响应状态码正常 200 OK
        if (response.status === 200) {
            const data = response.data;
            // 自定义统一响应结果的code值: 由后端定义
            if (data.code === HttpCode.SUCCESS) {
                message.success("响应一切正常:" , response.statusText);
                return Promise.resolve(response)
            } else if (data.code === HttpCode.NOT_FOUND) {
                    message.error("404 Not Fount")
                    return Promise.reject(response.data.message);
                } else if (data.code === HttpCode.UNAUTHORIZED) {
                        message.error("401 未授权 token过期")
                        return Promise.reject(response.data.message);
                    } else if (data.code === HttpCode.SERVER_ERROR) {
                            message.error("500 服务器错误")
                            return Promise.reject(response.data.message);
                        } else {
                            message.warn("未知错误: " + data.message)
                            return Promise.reject(response.data.message);
                        }
        }
        // 接口404的情况
        else if (response.status === 404) {
            message.warning("404访问页面不存在");
            return Promise.reject(response.data.message);
        }
        // 其他情况
        else {
            message.error("response.status: " + response.status)
            return Promise.reject(response.data.message);
        }
    }
);
```
### `axios` 发送请求
```javascript
// 发送验证码请求: baseUrl由axios的实例定义了
export const sendCodeRequest = async (data) => {
    const response = await axios.post("/member/sendCode", data);
    if (response.data.data) {
        // Axios 默认已经解析 JSON，不需要 `JSON.parse(response.data)`
        return {
            mobile: response.data.data.mobile || "",
            verificationCode: response.data.data.verificationCode
        };
    }
    throw new Error("返回数据为空");
}
```
- 一个简单地axios由3部分组成: (url, data, config) 以及一个Promise
### `Router`守卫
```javascript
// 添加一个路由的全局前置守卫
router.beforeEach(async function (to, from, next) {
  // 判断是否是登录页面
  if (to.name === "login" || to.name === "NotFound" || to.name === "Forbidden" || to.name === "Error") {
    next();
    return false;
  }
  // 判断本地是否记录token值
  const memberStore = useMemberStore();
  const token = memberStore.$state.Authorization;
  // 如果没有token
  if (!token) {
    // next({ name: "Login" });
    notification.error({description: "在未登录时，禁止访问其他页面！"});
    router.push("/login")
    return false;
  }
  next();
  return true;
});
```
## `Promise` 详解
### 什么是`Promise`
- `Promise` 是 **JavaScript 中的一种异步处理机制**，用于解决 **回调地狱（Callback Hell）** 和 **异步操作的可读性问题**。它代表 **一个未来可能会完成或失败的操作**，并允许在操作完成后执行相应的代码
### `Promise`的三种状态
1. **`pending`（进行中）：** 初始状态，表示异步操作尚未完成。
2. **`fulfilled`（已完成）：** 操作成功，`resolve()` 被调用。
3. **`rejected`（已失败）：** 操作失败，`reject()` 被调用。
- 一旦 `Promise` 从 `pending` 变为 `fulfilled` 或 `rejected`，就不能再改变状态（不可逆)
### `Promise` 的基本用法
```javascript
const myPromise = new Promise((resolve, reject) => {
  setTimeout(() => {
    let success = true;
    if (success) {
      resolve("操作成功！"); // 成功调用 resolve
    } else {
      reject("操作失败！"); // 失败调用 reject
    }
  }, 2000);
});

```
在 `Promise` 内部：
- 创建了一个promise, 函数体是执行内容
- 根据内容决定  -> 执行哪一个内部函数resolve还是reject
- `resolve(value)` **表示成功**，会触发 `.then()`
- `reject(error)` **表示失败**，会触发 `.catch()`
### `Promise`异步回调处理
```javascript
myPromise
  .then((result) => {
    console.log("成功:", result);
  })
  .catch((error) => {
    console.error("失败:", error);
  })
  .finally(() => {
    console.log("无论成功还是失败都会执行！");
  });

```
执行流程：
- 根据内部函数的执行, 从而决定调用then还是catch, 也就是会被捕捉
1. 如果 `resolve()` 被调用，执行 `.then(result)` ,then()处理异步操作成功后的结果，返回一个新的 `Promise`，可以链式调用。
2. 如果 `reject()` 被调用, 执行 `.catch(error)`
3. `.finally()` 一定会执行，无论成功还是失败
### `async/await`（Promise 的语法糖）
```javascript
const fetchData = async () => {
  try {
    const result = await myPromise;
    console.log(result);
  } catch (error) {
    console.error("错误:", error);
  } finally {
    console.log("执行完成");
  }
};
fetchData();

```
`await` 关键字的作用：
- 暂停代码执行，直到 `Promise` 解决
- 可以直接获取 `Promise` 结果
- 比 `.then()` 语法更直观
### 用async还是then
- 注意这里的两个return
	- 其实没必要用两个异步回调
```javascript
export const batchJobAddOrResRequest = async (url, data) => {
    return await axios.post("/batch/admin/job/" + url, data).then((response) => {
        try {
            const Result = response.data;
            if (Result.code === HttpStatusCode.Ok) {
                return true;
            }
        } catch (error) {
            console.error("定时任务添加错误");
        }
    });
}
```
## 组件中间值的传递
- **对参数赋值, 值通过子组件传递进来**
```javascript
// 父组件: 定义了一个Switch, 通过子组件双向数据绑定的值进行设置打开状态
<template>
  <div>
    <a-switch v-model:checked="switchValue" />
  <childComponent v-model="switchValue" messageText="我对子盒子说话" @messageMethod="getMessageDetail"/>
  </div>
  <div>
    {{ detailMessage }}
  </div>
</template>

<script setup>
import { ref } from "vue";
import childComponent from "./childComponent.vue";
const switchValue = ref(false);
let detailMessage = ref();
const getMessageDetail = (detail) => {
  detailMessage.value = detail;
}
</script>
```

```javascript
watch(
  () => watchValue,
  (newVal, oldVal) => {}
```

```javascript
//  自定义子组件的属性和方法, 通过父组件传递参数
<template>
  <a-button
    @click="handleClick"
    :style="'messageText: ' + '父组件可以显示设置子组件的style'"
  >
    开关
  </a-button>
  <div>
    {{ props.messageText }}
  </div>
</template>

<script setup>
import { defineProps, defineEmits, ref, watch, onMounted } from "vue";
onMounted(() => {
  emits("messageMethod", "我是通过方法传递给父盒子的信息")
})
// 定义了传递字段
const props = defineProps({
  modelValue: Boolean,
  messageText: String,
});
// 定义了传递函数
const emits = defineEmits(["update:modelValue", "messageMethod"]);

let switchValue = ref(false);
const handleClick = () => {
  switchValue.value = !switchValue.value;
  emits("update:modelValue", switchValue.value ? false : true);
};
// 也可以通过对props.modelValue值的监听(vue3的watch), 来收到父组件的信息
watch(
  () => props.modelValue,
  (modelValue) => {
    console.log(modelValue, "的值变化了");
  }
);
</script>

<style scoped>
</style>
```
## 组件小部件
- 表格数据渲染, row与col 通过v-for进行表格数据渲染, 作用与a-table和el-table一样
	- 数据源 - 表头行 - 间隔
	- 用v-for渲染: 在每一行数据中可以灵活运用v-if和v-model和差值语法进行处理
```javascript
<a-row class="order-tickets-header" v-if="tickets.length > 0">
      <a-col :span="2">乘客</a-col>
      <a-col :span="6">身份证</a-col>
      <a-col :span="4">票种</a-col>
      <a-col :span="4">座位类型</a-col>
      <a-col :span="6">乘客详细信息</a-col>
    </a-row>
    <a-row
      class="order-tickets-row"
      v-for="(ticket, i) in tickets"
      :key="ticket.passengerId"
    >
      <a-col :span="2">{{ ticket.passengerName }}</a-col>
      <a-col :span="6">{{ ticket.passengerIdCard }}</a-col>
      <a-col :span="4">
        <a-select v-model:value="ticket.passengerType" style="width: 100%">
          <a-select-option
            v-for="item in PASSENGER_TYPE_ARRAY"
            :key="item.code"
            :value="item.code"
          >
            {{ item.desc }}
          </a-select-option>
        </a-select>
      </a-col>
	  </a-row>
```
## Ref变量和reactive变量
- reactive是响应式的(能自动解包), 就不需要通过 `.value`获取值, 对于对象和数组能快速取值
- 在 Vue 3 的 Composition API 里，`ref()` 返回的是一个对象，这个对象有一个 `value` 属性，它才是真正存储数据的地方。因此，访问 `ref()` 变量时必须使用 `.value`，否则你拿到的是 `Ref` 对象，而不是它的值, 所以ref还是比较适合基础类型
```javascript
const form = ref({ id: "", name: "" });
console.log(form); // ->  { form: { id: "", name: "" } }
```
- **`reactive()` 直接返回代理对象，而 `ref()` 返回的是一个 `{ value: 数据 }` 的对象**。
- `const`和`let`用来定义变量，`{}`是对象，`[]`是数组，`({})`是对象赋值，`([])`是数组赋值, 本质上来说是一样的, 只不过为了区分

## 注意事项
- vue的if判断不单单能判断boolean类型, 甚至对空对象, 空值, 0 , 正数…都能判断
- 对象的打印不是用加号而是用逗号, 否则打印出来的是"Object"这个字符
```javascript
let obj = { name: "Alice" };
console.log("obj: " + obj); 
// 打印结果：obj: [object Object]
console.log("obj:", obj);    
// 打印结果：obj: { name: "Alice" }
```
- computed属性函数: 有使用内置缓存的特性 
- vue3新语法: `<script setup>`, 不用手动return, 让代码更简洁
- vue3新功能: watch监听器
- history: createWebHistory(),  启用 vue3的History 路由模式: 
	- `http://example.com/about` -> vue3新模式: 真实路径
	- `http://example.com/#/about`  -> 而不是 Vue 2 时代 `hash` 模式的
- 状态码code和统一响应结果code不一样
- vue的打包dist要指定mode(dev, prod)
- 一个按钮内有点击事件, 内容是一个路由跳转函数, 那么只会执行一个
- 前端对JSON的处理: JSON.stringify(object)和JSON.parse(jsonObject)
- Axios 默认已经解析 JSON，不需要 JSON.parse(response.data.data)
- localStorage永久保存在本地, 而sessionStorage会话存储, 值的有效期只在当前浏览器会话
- 对如何类的调用都应该创建实例, 只有实例才能取属性调用函数
- `Uncaught (in promise)` 错误通常意味着你没有处理 `Promise` 被拒绝的情况。需要进行请求的捕获tryCatch
- 设置为一个 URL 字符串时，浏览器会自动发出一个 HTTP 请求去获取这个 URL 对应的资源
![17452385641011745238563053.png|700x196](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17452385641011745238563053.png)
- 响应拦截器获得的响应数据其实是被3层包裹 
![17426307478001742630746874.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17426307478001742630746874.png)
- 后端定义的首字母大写字段有时候会被Lombok变成小写, 例如: Authorization -> authorization, 可以添加@JsonProperty("Authorization") 或者全局修改规则
- 解决前后端交互Long类型精度丢失的问题: 1.加一个jackson序列化器 2.每一个响应类加一个注解:@JsonSerialize(using = ToStringSerializer.class)
- 在使用 `map` 前，务必确保你操作的变量是 **数组**, 因为JS的赋值类型是多变得,所有必须多加一层判断语句
- 其实anDesignVue的东西都是在原始html的基础上添加了一些属性和方法, 过程跟我们使用父子组件的方法一样, 所以我们也可以自定义自己的组件库