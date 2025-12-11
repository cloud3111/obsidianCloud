---
title: 前后端联调
date: 2024-10-30 15:52:02
tags:
  - Web
  - Servlet
  - CORS
categories: 编程
---

> 🥲假如我们分手的话，绝不是出于我的意思，要知道，树是不愿离开花的，是花离开树          —— 大仲马

# 解决联调发生的各种问题

## 联调前的分析
- 前端给后端的请求数据一般常用的有两种方式: 
	- form表单: application/x-www-form-urlencoded
	- json字符串: application/json
- 而后端给前端的响应数据, 根据RestFul API风格只有一种: json字符串
```java
// 前端实体数据
code: 200,
data: {
    tableData: [
        {
            date: "1962-06-22",
            name: "周星驰",
            action: "演员",
            address: "大话西游",
        },
        {
            date: "1963-06-23",
            name: "刘慈欣",
            action: "作家",
            address: "三体",
        },
        { 
            date: "1881-09-25",
            name: "鲁迅",
            action: "作家",
            address: "孔乙己",
        },
        {
            date: "1960-04-03",
            name: "余华",
            action: "作家",
            address: "活着",
        },
    ],
},
```

### 分析数据组成:

1.是否符合统一响应格式**Result**

```java
public class Result {
    private Integer code;
    private String message;
    private Object data;
}
```

2.判断是否嵌套:如果有多层嵌套建议先剥橘子(从里到外)

3.判断字段数量和类型(一 一对应,用封装对象来包裹)

```java
class tableData {
    String name;
    String data;
    String action;
    String address;
}
```

4.设置响应体类型参数,将对象转化成 Json 字符串

```java
response.setContentType("application/json;charset=UTF-8");
list.add(new tableData("周润发", "2024-10-29", "演员", "赌场"));
list.add(new tableData("周星驰", "2024-10-29", "演员", "大话西游"));

方式一: 谷歌Gson
Gson gson = new Gson();
gson.toJson(list);

方式二: jackJson(spring默认)
ObjectMapper jackJson = new ObjectMapper;
jackJson.writeValueAsString(list);

方式三: fastJson
JSONObject.toJSONString(list);
```

## HTTP请求的两种方式

### 1.原始Servlet

```xml
<!--xml配置文件--> 
<servlet>
    <servlet-name>testServlet</servlet-name>
    <servlet-class>com.example.demo.testServlet</servlet-class> <!--类路径--> 
</servlet>

<servlet-mapping>
    <servlet-name>testServlet</servlet-name>
    <url-pattern>/home/getTableData</url-pattern>		<!--请求参数--> 
</servlet-mapping>
```

```java
// 通过继承HttpServlet重写GenericServlet的两方法,最后由Servlet接口实例调用service方法交由Tomcat发起HTTP请求响应
public class testServlet extends HttpServlet {
    @Override
    void doGet(HttpServletRequest req, HttpServletResponse resp) {
        // 处理Get请求的逻辑
    }
    @Override
    void doPost(HttpServletRequest req, HttpServletResponse resp) {
        // 处理Post请求的逻辑
    }
}
```

### 2.springBoot Web容器

- Spring-Boot: 内嵌Tomcat容器 + Servlet

```java
@CrossOrigin
@RequestMapping
@RestController
public class testController {

    @GetMapping("/home/getTableData")
    public Result getTableData() throws Exception {
        System.out.println("testController");

        List<tableData> list = new ArrayList<>();
        list.add(new tableData("周润发", "2024-10-29", "演员", "赌场"));
        list.add(new tableData("周星驰", "2024-10-29", "演员", "大话西游"));

        Gson gson = new Gson();
        return Result.success(gson.toJson(list));
    }
}	
```

### 3.解决跨域问题
- 什么是跨域
	- 通俗来说就是端对端访问了不同的 **协议 地址 端口 路径** 的其中一种
- 解决方式: 
	- ​对于spring-Boot项目来说,只需要加一个注解 **@CrossOrigin**
	- 或者全局加一个过滤器添加请求头参数
	- ​如果是部署在Nginx上的话可以添加请求头参数
  ```nginx
  add_header Access-Control-Allow-Origin *;  # CORS 设置 跨域问题
  add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
  add_header Access-Control-Allow-Headers 'Origin, Content-Type, Accept';
  ```
## 分析Json

```java
// 原始的Json格式: 都是字符串 最外层data用的是一个数组   
{
    "code": 200,
    "message": "success",
    "data": "[{\"name\":\"周润发\",\"data\":\"2024-10-29\",\"action\":\"演员\",\"address\":\"赌场\"},{\"name\":\"周星驰\",\"data\":\"2024-10-29\",\"action\":\"演员\",\"address\":\"大话西游\"}]"
```

- 前端拿到数据应该对数据进行Json解析

- 但是像 Axios 这样的库时，它会自动将返回的 JSON 数据解析为 JavaScript 对象

- 前端内置的fetch就不行,需要手动parse, 但若是axios就不需要手动JSON.parse转换

## 前后端联调会测试1.0

#### 前端

- 创建请求实例:设置默认url和headers

```javascript
// 创建 axios 实例
const axiosInstance = axios.create({
  baseURL: "https://localhost:5173", // 请求的基础URL:不写baseURL默认请求本地
  timeout: 1000, // 请求超时时间
  headers: {
    "Content-Type": "application/json", // 全局设置请求数据格式JSON
  },
});
```

单独设置实例调用Api方法(区分请求方式) 

```javascript
getTableData(params) {
    return request({
      url: "/api/home/getTableData/${params.id}",
      method: "get",
      params: params,
    });
  },
```



- 在原本axios实例的基础上, 对受到的数据进行拦截 并进行JSON格式的解析

```javascript
// 添加响应拦截器
axiosInstance.interceptors.response.use((response) => {
  // 像 Axios 这样的库时，它会自动将返回的 JSON 数据解析为 JavaScript 对象
  const { code, data, msg } = response.data.data; 
  if (response.data.code === 200) {
    return response.data.data; // 只返回data
  } else {
    ElMessage.error(msg || NETWORK_ERROR);
    return Promise.reject(msg || NETWORK_ERROR);
  }
});
```

- 对拿到的数据进行渲染 

```vue
<el-card shadow="hover" class="userTable">
    <el-table :data="tableData" size="small" border>
        <el-table-column prop="date" label="生日" />
        <el-table-column prop="name" label="名字" />
        <el-table-column prop="action" label="身份" />
        <el-table-column prop="address" label="代表作" />
    </el-table>
</el-card>
```

#### Nginx

```nginx
// 对前端端口的代理
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/json;

    server {
        listen 5173;
        server_name localhost;

        location /api/home/getTableData {
            proxy_pass http://localhost:8080/home/getTableData;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # 添加响应头
            add_header Content-Type application/json;
            add_header Access-Control-Allow-Origin *;  # CORS 设置 跨域问题
            add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
            add_header Access-Control-Allow-Headers 'Origin, Content-Type, Accept';
        }
    }
}



```

#### 后端

```java
@RequestMapping
@RestController
@CrossOrigin(cros = "前端地址")
public class testController {
    @GetMapping("/home/getTableData")
    public Result getTableData() throws Exception {
        System.out.println("testController");

        List<tableData> list = new ArrayList<>();
        list.add(new tableData("周星驰", "1962-06-22", "演员", "大话西游"));
        list.add(new tableData("刘慈欣", "1963-06-23", "作家", "三体"));
        list.add(new tableData("鲁迅", "1881-09-25", "作家", "孔乙己"));
        list.add(new tableData("余华", "1960-04-03", "作家", "活着"));
        Gson gson = new Gson();
        return Result.success(gson.toJson(list));

    }
```

## 前后端联调会测试2.0

- 每次要发起请求之前先测试  👍

![好习惯](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17312441657891731244165185.png)



#### 前端

- **前端会传page为1的原因: 这个1为视图层面上的第一页**
- **而在数据库层面, 要通过计算得到跳过的条数: (page-1) * pageSize**

```javascript
// 调用axios方法发起请求
const data = await instance.proxy.$api.getUserData(config);
tableData.value = data.list || [];
config.total = data.count; //总条数

// 请求数据的参数 用对象封装
const config = reactive({
    total: 0, // 总条数  因为不知道会不会走条件查询,所有每次请求都要带上
    page: 1,  // 当前页码
    name: "", // 查询字段...
    age ...
});

const tableLabel = reactive([	// 指向性实体类
    {
        prop: "name",
        label: "姓名",
    },
    {
        prop: "age",
        label: "年龄",
    },
    {
        prop: "sexLabel",
        label: "性别",
    },
    {
        prop: "birth",
        label: "出生日期",
        width: 200,
    },
    {
        prop: "addr",
        label: "地址",
        width: 200,
    },
]);
<template>
    <el-table
:data="tableDataObject"	// 对象绑定
>
    <el-table-column
v-for="item in tableLabel"	// 高明手段: 不用逐个 prop="name" 
:key="item.prop"	// 嵌套循环
:prop="item.prop"
:label="item.label"	 // label表头
/>
    ...
```

```javascript
// 1.为什么要挂载: 不挂载的函数不会执行
// 2.钩子中执行的代码可以确保组件的 DOM 元素已经可用，从而允许进行数据加载和其他操作。
// 3.自动挂载：当你在 Vue 组件中定义 setup 函数时，Vue 会自动处理组件的挂载过程。当组件被渲染并添加到 DOM 中时，Vue会自动调用onMounted钩子。
// 4.在组件挂载后调用 fetchData
onMounted(() => {
  getUserDataMethod();  // 每次页面渲染执行第一页数据的查询 默认page=1 pageSize=10
});
```



#### 后端

- Mybatis(pageHelp+xml映射文件)和MybatisPlus(自带插件+warpper)

```java
@GetMapping("/user/getUserData")
    public Result getUserData(@RequestParam(defaultValue = "1") int page,	
                              @RequestParam(defaultValue = "5") int limit,	// 每页展示数,前端没传,默认10
                              @RequestParam String name...		// 模糊匹配,范围查询,排序字段
    )
```

```java
// 返回VO 
public class pageVO<T> {
    private Long Total; // 总条数
    private Long pages; // 总页数,看前端要求
    private List<T> list; // 结果列表
}
```

```java
// DTO: 中间数据传递	这个Page是mp自带的 有排序器 也可作为链式编程的容器 lambdaQuery().Page(page); 
public class Page<T> implements IPage<T> {
    private static final long serialVersionUID = 8545996863226528798L;
    protected List<T> records;
    protected long total;
    protected long size;
    protected long current;
    protected List<OrderItem> orders;
    protected boolean optimizeCountSql;
    protected boolean searchCount;
    protected boolean optimizeJoinOfCountSql;
    protected String countId;
    protected Long maxLimit;
}
```

