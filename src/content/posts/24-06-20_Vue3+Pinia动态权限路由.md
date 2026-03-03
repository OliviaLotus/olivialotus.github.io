---
title: Vue3+Pinia动态权限路由
published: 2024-06-20
weight: 1
description: Vue3+Pinia动态权限路由
tags: [javascript, typescript, vue]
category: 前端
---

## 大致代码
```ts
// src/store/modules/permission.ts
import type { RouteRecordRaw } from "vue-router";
import { constantRoutes } from "@/router";
import { defineStore } from "pinia";
import router from "@/router";
import MenuAPI from "@/api/system/menu";
import { RouteItem } from "@/types";

const modules = import.meta.glob("../../views/**/**.vue");
const Layout = () => import("../../layouts/index.vue");

/**
 * 解析组件路径，返回组件导入函数
 */
function resolveViewComponent(componentPath: string) {
  const normalized = componentPath
    .trim()
    .replace(/^\/+/, "")
    .replace(/\.vue$/i, "");
  return (
    modules[`../../views/${normalized}.vue`] ||
    modules[`../../views/${normalized}/index.vue`] ||
    modules[`../../views/error/404.vue`]
  );
}

/**
 * 转换后端路由为 Vue Router 格式
 */
const transformRoutes = (routes: RouteItem[], isTopLevel: boolean = true): RouteRecordRaw[] => {
  const processedIds = new Set(); // 防循环引用
  return routes
    .map((route) => {
      if (processedIds.has(route.id)) return null;
      processedIds.add(route.id);

      const { component, children, ...args } = route;
      const processedComponent = isTopLevel || component !== "Layout" ? component : undefined;
      const normalizedRoute = { ...args } as RouteRecordRaw;

      if (!processedComponent) {
        normalizedRoute.component = undefined;
      } else {
        normalizedRoute.component =
          processedComponent === "Layout" ? Layout : resolveViewComponent(processedComponent);
      }

      if (children?.length) {
        normalizedRoute.children = transformRoutes(children, false);
      }

      return normalizedRoute;
    })
    .filter(Boolean);
};

export const usePermissionStore = defineStore("permission", () => {
  const routes = ref<RouteRecordRaw[]>(constantRoutes);
  const isRouteGenerated = ref(false);

  /**
   * 生成动态路由
   */
  async function generateRoutes() {
    try {
      const data = await MenuAPI.getRoutes();
      const dynamicRoutes = transformRoutes(data);
      routes.value = [...constantRoutes, ...dynamicRoutes];
      isRouteGenerated.value = true;
      return dynamicRoutes;
    } catch (error) {
      isRouteGenerated.value = false;
      throw error;
    }
  }

  /**
   * 重置路由
   */
  function resetRouter() {
    const constantNames = new Set(constantRoutes.map(r => r.name).filter(Boolean));
    routes.value.forEach(route => {
      if (route.name && !constantNames.has(route.name)) {
        router.removeRoute(route.name);
      }
    });
    routes.value = [...constantRoutes];
    isRouteGenerated.value = false;
  }

  /**
   * 重载动态路由（单例模式，避免重复请求）
   */
  let reloadPromise: Promise<RouteRecordRaw[]> | null = null;
  async function reloadDynamicRoutesOnce() {
    if (reloadPromise) return reloadPromise;
    reloadPromise = (async () => {
      try {
        resetRouter();
        const dynamicRoutes = await generateRoutes();
        dynamicRoutes.forEach(route => router.addRoute(route));
        return dynamicRoutes;
      } finally {
        reloadPromise = null;
      }
    })();
    return reloadPromise;
  }

  return {
    routes,
    isRouteGenerated,
    generateRoutes,
    resetRouter,
    reloadDynamicRoutesOnce
  };
});

// 非组件环境使用
export function usePermissionStoreHook() {
  return usePermissionStore();
}
```

---

## 一、代码整体功能总结
这段代码是一个基于 Vue 3 生态的**前端权限路由核心模块**，主要负责：
1. 从后端接口拉取当前登录用户的菜单/路由数据；
2. 将后端返回的路由数据转换为 Vue Router 可识别的 `RouteRecordRaw` 格式；
3. 动态注册/移除路由，实现基于用户权限的路由控制；
4. 提供路由重载、权限快照刷新等工具方法，处理权限变更场景；
5. 维护路由相关的状态（如所有路由列表、左侧菜单路由、路由生成状态）。

## 二、核心模块逐段解析

### 1. 依赖导入与基础配置
```typescript
import type { RouteRecordRaw } from "vue-router"; // Vue Router 路由记录类型
import { constantRoutes } from "@/router"; // 静态路由（如登录页、404页，所有用户都能访问）
import { store } from "@/store"; // Pinia 根store
import router from "@/router"; // Vue Router 实例
import { useUserStoreHook } from "@/store/modules/user"; // 用户信息store

import MenuAPI from "@/api/system/menu"; // 菜单/路由接口请求
import { RouteItem } from "@/types"; // 后端返回的路由类型
const modules = import.meta.glob("../../views/**/**.vue"); // 批量导入views下所有.vue文件（Vite语法）
const Layout = () => import("../../layouts/index.vue"); // 布局组件懒加载
```
- `import.meta.glob`：Vite 提供的批量导入语法，提前收集所有页面组件，避免手动导入；
- `constantRoutes`：静态路由，不随用户权限变化（如登录、404、首页框架）。

### 2. Pinia Store 核心逻辑（usePermissionStore）
这是整个模块的核心，维护路由状态并提供操作方法：

#### （1）状态定义
```typescript
const routes = ref<RouteRecordRaw[]>([]); // 所有路由（静态+动态）
const mixLayoutSideMenus = ref<RouteRecordRaw[]>([]); // 混合布局的左侧菜单路由
const isRouteGenerated = ref(false); // 动态路由是否已生成（避免重复生成）
```

#### （2）生成动态路由（generateRoutes）
```typescript
async function generateRoutes(): Promise<RouteRecordRaw[]> {
  try {
    const data = await MenuAPI.getRoutes(); // 从后端获取当前用户的路由数据
    const dynamicRoutes = transformRoutes(data); // 转换为Vue Router格式

    routes.value = [...constantRoutes, ...dynamicRoutes]; // 合并静态+动态路由
    isRouteGenerated.value = true;

    return dynamicRoutes;
  } catch (error) {
    isRouteGenerated.value = false;
    throw error; // 抛出错误让上层处理（如提示用户）
  }
}
```
- 核心流程：拉取后端路由 → 格式转换 → 合并路由 → 更新状态。

#### （3）设置混合布局左侧菜单（setMixLayoutSideMenus）
```typescript
const setMixLayoutSideMenus = (parentPath: string) => {
  const parentMenu = routes.value.find((item) => item.path === parentPath);
  mixLayoutSideMenus.value = parentMenu?.children || [];
};
```
- 作用：针对混合布局场景（如顶部导航+左侧菜单），根据父路由路径筛选出对应的子路由作为左侧菜单。

#### （4）重置路由（resetRouter）
```typescript
const resetRouter = () => {
  // 1. 收集静态路由的name（避免误删静态路由）
  const constantRouteNames = new Set(constantRoutes.map((route) => route.name).filter(Boolean));
  // 2. 移除所有动态路由
  routes.value.forEach((route) => {
    if (route.name && !constantRouteNames.has(route.name)) {
      router.removeRoute(route.name);
    }
  });
  // 3. 重置状态
  routes.value = [...constantRoutes];
  mixLayoutSideMenus.value = [];
  isRouteGenerated.value = false;
};
```
- 关键逻辑：只移除动态路由，保留静态路由，避免登录页、404页等核心路由被删除。

#### （5）路由重载（reloadDynamicRoutesOnce）
```typescript
let reloadPromise: Promise<RouteRecordRaw[]> | null = null;
async function reloadDynamicRoutesOnce(): Promise<RouteRecordRaw[]> {
  if (reloadPromise) return reloadPromise; // 单飞模式：避免重复请求

  reloadPromise = (async () => {
    try {
      resetRouter(); // 先清空旧路由
      const dynamicRoutes = await generateRoutes(); // 重新拉取路由
      dynamicRoutes.forEach((route) => {
        router.addRoute(route); // 注册新路由
      });
      return dynamicRoutes;
    } finally {
      reloadPromise = null; // 请求完成后重置
    }
  })();

  return reloadPromise;
}
```
- **单飞模式**（防抖/防重复）：通过 `reloadPromise` 避免同一时间多次调用该方法，导致路由重复注册；
- 典型场景：后端权限变更后，前端刷新路由以同步最新权限。

#### （6）权限快照刷新（reloadPermissionSnapshotOnce）
```typescript
let snapshotPromise: Promise<void> | null = null;
async function reloadPermissionSnapshotOnce(): Promise<void> {
  if (snapshotPromise) return snapshotPromise;

  snapshotPromise = (async () => {
    try {
      const userStore = useUserStoreHook();
      await userStore.getUserInfo(); // 先刷新用户信息（角色/权限）
      await reloadDynamicRoutesOnce(); // 再刷新路由
    } finally {
      snapshotPromise = null;
    }
  })();

  return snapshotPromise;
}
```
- 作用：不仅刷新路由，还同步刷新用户的角色、权限标识等核心信息，是更完整的权限刷新逻辑。

### 3. 组件路径解析函数
```typescript
function resolveViewComponent(componentPath: string) {
  const normalized = componentPath
    .trim()
    .replace(/^\/+/, "") // 移除开头的/
    .replace(/\.vue$/i, ""); // 移除.vue后缀
  // 优先匹配精准路径 → 匹配index.vue → 匹配404页面
  return (
    modules[`../../views/${normalized}.vue`] ||
    modules[`../../views/${normalized}/index.vue`] ||
    modules[`../../views/error/404.vue`]
  );
}
```
- 作用：将后端返回的组件路径（如 `system/user`）转换为前端实际的组件导入函数；
- 容错处理：找不到对应组件时自动返回404页面，避免路由跳转报错。


### 4. 路由格式转换（transformRoutes）
`transformRoutes` 的唯一目的：
把后端返回的**自定义格式路由数据**（`RouteItem` 类型），转换成 Vue Router 要求的**标准路由格式**（`RouteRecordRaw` 类型），同时处理「布局组件（Layout）」的层级嵌套问题。

#### 先看两个关键数据格式（简化版）
##### 1. 后端返回的路由数据（RouteItem）
后端返回的结构通常是这样的（你可以理解为“纯数据描述”）：
```typescript
interface RouteItem {
  path: string; // 路由路径，如 "/system"
  name: string; // 路由名称，如 "System"
  component: string; // 组件路径/标识，如 "Layout" 或 "system/user/index"
  children?: RouteItem[]; // 子路由
  meta?: { title: string; icon: string }; // 菜单标题、图标等
}
```

##### 2. Vue Router 要求的路由格式（RouteRecordRaw）
Vue Router 能识别的结构（核心是 `component` 必须是**组件导入函数**，而非字符串）：
```typescript
interface RouteRecordRaw {
  path: string;
  name?: string;
  component?: Component; // 必须是 () => import("xxx.vue") 这种函数
  children?: RouteRecordRaw[];
  meta?: Record<string, any>;
}
```

#### transformRoutes 逐行拆解
先贴简化后的核心代码，再分步解释：
```typescript
const transformRoutes = (routes: RouteItem[], isTopLevel: boolean = true): RouteRecordRaw[] => {
  // 遍历后端路由数组，逐个转换
  return routes.map((route) => {
    // 1. 解构出后端路由的字段：把component和children单独拿出来，其余字段（path/name/meta等）放到args里
    const { component, children, ...args } = route;

    // 2. 核心：处理Layout组件的层级问题（最关键的逻辑）
    // isTopLevel：标记当前路由是否是“顶层路由”（默认true）
    const processedComponent = isTopLevel || component !== "Layout" ? component : undefined;

    // 3. 初始化标准路由对象：把path/name/meta等基础字段先赋值
    const normalizedRoute = { ...args } as RouteRecordRaw;

    // 4. 处理组件（把字符串 → Vue组件导入函数）
    if (!processedComponent) {
      // 多级菜单的父级：不需要组件（比如二级菜单的父节点）
      normalizedRoute.component = undefined;
    } else {
      // 区分两种情况：Layout组件 / 业务页面组件
      normalizedRoute.component =
        // 如果是Layout标识 → 用提前导入的Layout组件
        processedComponent === "Layout" ? Layout : 
        // 否则 → 用resolveViewComponent解析为页面组件导入函数
        resolveViewComponent(processedComponent);
    }

    // 5. 递归处理子路由：子路由一定不是顶层（isTopLevel=false）
    if (children && children.length > 0) {
      normalizedRoute.children = transformRoutes(children, false);
    }

    // 6. 返回转换后的单个标准路由
    return normalizedRoute;
  });
};
```

#### 实例场景
后端返回的路由数据（多级菜单）：
```javascript
const backendRoutes = [
  {
    path: "/system",
    name: "System",
    component: "Layout", // 顶层Layout
    meta: { title: "系统管理", icon: "setting" },
    children: [
      {
        path: "user",
        name: "User",
        component: "system/user/index", // 业务页面
        meta: { title: "用户管理" }
      },
      {
        path: "role",
        name: "Role",
        component: "system/role/index",
        meta: { title: "角色管理" }
      }
    ]
  }
];
```

##### 第一步：处理顶层路由（isTopLevel = true）
1. 解构出：`component = "Layout"`，`children = [用户/角色路由]`，`args = { path: "/system", name: "System", meta: {...} }`；
2. `processedComponent` 判断：`isTopLevel=true` → 保留 `component="Layout"`；
3. 组件赋值：`processedComponent === "Layout"` → `normalizedRoute.component = Layout`（布局组件导入函数）；
4. 子路由存在 → 调用 `transformRoutes(children, false)`（子路由标记为非顶层）。

##### 第二步：处理子路由（isTopLevel = false）
以“用户管理”子路由为例：
1. 解构出：`component = "system/user/index"`，`children = undefined`，`args = { path: "user", name: "User", meta: {...} }`；
2. `processedComponent` 判断：`isTopLevel=false`，但 `component !== "Layout"` → 保留 `component="system/user/index"`；
3. 组件赋值：不是Layout → 调用 `resolveViewComponent("system/user/index")`，最终转换为：
   ```javascript
   () => import("../../views/system/user/index.vue")
   ```
4. 无子路由 → 跳过递归。

##### 最终转换结果（Vue Router 可识别）
```javascript
const vueRouterRoutes = [
  {
    path: "/system",
    name: "System",
    component: () => import("../../layouts/index.vue"), // Layout组件
    meta: { title: "系统管理", icon: "setting" },
    children: [
      {
        path: "user",
        name: "User",
        component: () => import("../../views/system/user/index.vue"), // 页面组件
        meta: { title: "用户管理" }
      },
      {
        path: "role",
        name: "Role",
        component: () => import("../../views/system/role/index.vue"),
        meta: { title: "角色管理" }
      }
    ]
  }
];
```

#### 为什么要处理 `isTopLevel`？
这个设计是为了解决**多级Layout嵌套的问题**：
- 顶层路由（一级菜单）需要 Layout 组件：因为页面要先渲染整体布局（侧边栏、顶部导航）；
- 子路由（二级/三级菜单）如果后端也返回 `component: "Layout"`，则**强制设为undefined**：
  比如后端返回三级菜单时，二级菜单的 `component` 也写了 "Layout"，此时 `isTopLevel=false` + `component="Layout"` → `processedComponent=undefined` → 子路由的 `component` 设为undefined，避免页面重复渲染多个Layout。


### 5. 非组件环境使用Store
```typescript
export function usePermissionStoreHook() {
  return usePermissionStore(store);
}
```
- 作用：在非组件环境（如拦截器、工具函数）中使用 Pinia Store，避免直接调用 `usePermissionStore()` 报错（Pinia 需要在组件上下文或传入根 store）。
