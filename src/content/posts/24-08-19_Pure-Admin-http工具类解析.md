---
title: Pure-Admin-http工具类解析
published: 2024-08-19
description: Pure-Admin-http工具类解析
tags: [javascript, typescript, vue]
category: 前端
---

## types.d.ts类型定义
### 1. 引入类型
```ts
import type { Method, AxiosError, AxiosResponse, AxiosRequestConfig } from "axios";
```
从 `axios` 里引入了一些类型，后面会用到。比如：
- `Method`：HTTP 方法（get、post、put...）
- `AxiosError`：请求错误类型
- `AxiosResponse`：响应类型
- `AxiosRequestConfig`：请求配置类型

---

### 2. 定义响应数据结构
```ts
export type resultType = {
  accessToken?: string;
};
```
这只是一个示例类型，表示接口返回的数据结构里可能有一个 `accessToken` 字段，实际项目里你可以换成你自己的。(删了也行)

---

### 3. 限制 HTTP 方法
```ts
export type RequestMethods = Extract<
  Method,
  "get" | "post" | "put" | "delete" | "patch" | "option" | "head"
>;
```
`RequestMethods` 是 `axios` 中 `Method` 的子集，只保留了常用的几种 HTTP 方法(也就是第二个参数)。

---

### 4. 扩展错误类型
```ts
export interface PureHttpError extends AxiosError {
  isCancelRequest?: boolean;
}
```
给 `AxiosError` 加了一个字段 `isCancelRequest`，用来标记是否是主动取消的请求。

---

### 5. 扩展请求配置类型
```ts
export interface PureHttpRequestConfig extends AxiosRequestConfig {
  beforeRequestCallback?: (request: PureHttpRequestConfig) => void;
  beforeResponseCallback?: (response: PureHttpResponse) => void;
}
```
在 `axios` 的 `AxiosRequestConfig` 基础上加了可选两个钩子函数：
- `beforeRequestCallback`：发请求前调
- `beforeResponseCallback`：拿到响应后调

---

### 6. 扩展响应类型
```ts
export interface PureHttpResponse extends AxiosResponse {
  config: PureHttpRequestConfig;
}
```
把 `AxiosResponse` 里的 `config` 字段换成我们上面扩展的 `PureHttpRequestConfig`。

---

### 7. 类定义
```ts
export default class PureHttp {
  request<T>(
    method: RequestMethods,
    url: string,
    param?: AxiosRequestConfig,
    axiosConfig?: PureHttpRequestConfig,
  ): Promise<T>;

  post<T, P>(url: string, params?: P, config?: PureHttpRequestConfig): Promise<T>;
  get<T, P>(url: string, params?: P, config?: PureHttpRequestConfig): Promise<T>;
}
```
这是 `PureHttp` 类的**类型声明**，。它告诉 TypeScript：
- 有个类叫 `PureHttp`
- 它有这些方法
- 每个方法的参数和返回值类型是什么

例如：
```ts
const http = new PureHttp();
const data = await http.get<User>('/api/user', { id: 123 });
```
这行代码的意思是：用 GET 请求 `/api/user`，传参 `{ id: 123 }`，返回的数据是 `User` 类型。

## index.ts具体实现

### 1. 先把 axios 和周边工具请进来  
```ts
import Axios, {
  type AxiosInstance,
  type AxiosRequestConfig,
  type CustomParamsSerializer
} from "axios";
```

**技术注解**  
- `AxiosInstance`：axios.create() 出来的实例类型。  
- `CustomParamsSerializer`：axios 1.x 以后对 `paramsSerializer` 的严格类型要求。  

---

### 2. 引入自定义类型
```ts
import type {
  PureHttpError,
  RequestMethods,
  PureHttpResponse,
  PureHttpRequestConfig
} from "./types.d";
```

---

### 3. 引入工具
```ts
import { stringify } from "qs";
import NProgress from "../progress";
import { getToken, formatToken } from "@/utils/auth";
import { useUserStoreHook } from "@/store/modules/user";
```
**解析作用**  
- `qs`：把 { a:[1,2] } 变成 `a[0]=1&a[1]=2`。  
- `NProgress`：页面顶部小蓝条。  
- `getToken / formatToken`：从本地拿 token 并加上 `Bearer ` 前缀。  
- `useUserStoreHook`：Pinia 用户 store，用来刷新 token。

---

### 4. 默认配置“地基”
```ts
const defaultConfig: AxiosRequestConfig = {
  timeout: 10000,
  headers: {
    Accept: "application/json, text/plain, */*",
    "Content-Type": "application/json",
    "X-Requested-With": "XMLHttpRequest"
  },
  paramsSerializer: {
    serialize: stringify as unknown as CustomParamsSerializer
  }
};
```
**解析**  
“默认配置：10 秒超时、常见请求头、数组参数用 qs 序列化，防止 axios 5142 bug。”

**技术注解**  
`as unknown as …` 是“先抹掉类型再伪装成目标类型”，绕过 ts 严格检查。

---

### 5. 类本体与四个“静态私有财产”
```ts
class PureHttp {
  /** 暂存队列：token 刷新期间谁想发请求，先排队 */
  private static requests = [];

  /** 防止同时刷新多次 token 的锁 */
  private static isRefreshing = false;

  /** 全局级别的 beforeXXX 钩子 */
  private static initConfig: PureHttpRequestConfig = {};

  /** axios 实例本尊 */
  private static axiosInstance: AxiosInstance = Axios.create(defaultConfig);
```
**技术讲解**  
- `requests`：一个临时数组，存“等 token 刷新好以后再继续”的回调。  
- `isRefreshing`：布尔值，相当于“门闩”，防止并发刷新。  
- `initConfig`：如果项目初始化时想统一加钩子，塞这里。  
- `axiosInstance`：真正去发请求的 axios 实例。

---

### 6. 构造函数
```ts
constructor() {
  this.httpInterceptorsRequest();
  this.httpInterceptorsResponse();
}
```
装配

---

### 7. 工具函数：token 刷新好后，把排队的请求重新配置 Authorization
```ts
private static retryOriginalRequest(config: PureHttpRequestConfig) {
  return new Promise(resolve => {
    PureHttp.requests.push((token: string) => {
      config.headers["Authorization"] = formatToken(token);
      resolve(config);
    });
  });
}
```
“token 更新完以后，遍历队列，把新 token 塞到每个请求的头上，再放行。”

---

### 8. 请求拦截器
```ts
private httpInterceptorsRequest(): void {
  PureHttp.axiosInstance.interceptors.request.use(
    async (config: PureHttpRequestConfig): Promise<any> => {
      NProgress.start(); // 1. 小蓝条开始

      // 2. 优先执行单次回调，再执行全局回调
      if (typeof config.beforeRequestCallback === "function") {
        config.beforeRequestCallback(config);
        return config;
      }
      if (PureHttp.initConfig.beforeRequestCallback) {
        PureHttp.initConfig.beforeRequestCallback(config);
        return config;
      }

      // 3. 白名单直接放行
      const whiteList = ["/refresh-token", "/login"];
      if (whiteList.some(url => config.url.endsWith(url))) return config;

      // 4. 拿 token & 判断是否过期
      return new Promise(resolve => {
        const data = getToken();
        if (!data) { resolve(config); return; }

        const now = Date.now();
        const expired = parseInt(data.expires) - now <= 0;

        if (!expired) {
          // 4-a 没过期：直接带 token
          config.headers["Authorization"] = formatToken(data.accessToken);
          resolve(config);
        } else {
          // 4-b 已过期：刷新 token
          if (!PureHttp.isRefreshing) {
            PureHttp.isRefreshing = true;
            useUserStoreHook()
              .handRefreshToken({ refreshToken: data.refreshToken })
              .then(res => {
                const token = res.data.accessToken;
                PureHttp.requests.forEach(cb => cb(token));
                PureHttp.requests = [];
              })
              .finally(() => (PureHttp.isRefreshing = false));
          }
          // 把当前请求塞进队列
          resolve(PureHttp.retryOriginalRequest(config));
        }
      });
    },
    error => Promise.reject(error)
  );
}
```
**流程**  
发请求前要做的事情：  
1. 小蓝条动画；
2. 走钩子函数；
3. 白名单直接过；
4. 拿 token，没过期直接带，过期就刷新，刷新期间把请求排队，刷新完再统一放行。

---

### 9. 响应拦截器
```ts
private httpInterceptorsResponse(): void {
  const instance = PureHttp.axiosInstance;
  instance.interceptors.response.use(
    (response: PureHttpResponse) => {
      NProgress.done(); // 1. 小蓝条结束
      const $config = response.config;

      // 2. 走回调
      if (typeof $config.beforeResponseCallback === "function") {
        $config.beforeResponseCallback(response);
        return response.data;
      }
      if (PureHttp.initConfig.beforeResponseCallback) {
        PureHttp.initConfig.beforeResponseCallback(response);
        return response.data;
      }
      return response.data; // 3. 默认只返回 data
    },
    (error: PureHttpError) => {
      error.isCancelRequest = Axios.isCancel(error); // 标记是否主动取消
      NProgress.done();
      return Promise.reject(error);
    }
  );
}
```
**流程**  
收到后端回复后：  
1. 小蓝条结束；
2. 走回调；
3. 正常时只把 `data` 抛出去；出错时加个 `isCancelRequest` 标记，再抛出去。”

---

### 10. 通用 request 方法
```ts
public request<T>(
  method: RequestMethods,
  url: string,
  param?: AxiosRequestConfig,
  axiosConfig?: PureHttpRequestConfig
): Promise<T> {
  const config = { method, url, ...param, ...axiosConfig } as PureHttpRequestConfig;
  return new Promise((resolve, reject) => {
    PureHttp.axiosInstance
      .request(config)
      .then(resolve)
      .catch(reject);
  });
}
```
所有动词（get/post/…）最后都走到这里，合成一个 config 交给 axios 实例。

---

### 11. 语法糖：post & get
```ts
public post<T, P>(
  url: string,
  params?: AxiosRequestConfig<P>,
  config?: PureHttpRequestConfig
): Promise<T> {
  return this.request<T>("post", url, params, config);
}

public get<T, P>(
  url: string,
  params?: AxiosRequestConfig<P>,
  config?: PureHttpRequestConfig
): Promise<T> {
  return this.request<T>("get", url, params, config);
}
```

给最常用的两个请求方式直接定义，省得每次手写 method。

---

### 12. 导出单例
```ts
export const http = new PureHttp();
```

---

### 13. 基本使用
```ts
import { http } from '@/utils/http';

// 发一个带类型的 GET
const user = await http.get<User>('/api/user', { params: { id: 1 } });

// 发一个 POST，并且在发请求前做点事
await http.post<Res, Req>('/api/order', params, {
  beforeRequestCallback(cfg) {
    console.log('马上发请求了', cfg);
  }
});
```
