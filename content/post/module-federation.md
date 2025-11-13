+++
date = '2025-11-09T22:40:08+08:00'
draft = false
title = '模块联邦的使用场景'
categories = ['前端']
+++

产品层面有 A，B，C，D 若干个 web 系统，但是被吐槽想要用一个功能模块要跳转来跳转去，很不方便，于是决定把这些若干个系统组装起来搭建一个平台级应用，但是因为 toB 的性质又想保留各个系统的独立售卖，时间紧迫，想到了组件共享的模式，复用现有代码（但不是单纯的拷贝），也就是本文的主角：**模块联邦**。

## 背景介绍

### 传统前端架构的痛点

随着前端应用的复杂度不断提升，传统的单体应用架构逐渐暴露出以下问题：

- **应用臃肿**：随着业务功能增加，单个应用变得异常庞大
- **技术栈升级困难**：升级某个依赖可能影响整个应用
- **团队协作效率低**：多个团队维护同一个代码库，容易产生冲突
- **独立部署困难**：任何小的改动都需要重新构建和部署整个应用

### Module Federation 的诞生

为了解决上述问题，微前端架构应运而生，其中的典型代表就是模块联邦（Module Federation），这是 Webpack 5 推出的革命性功能，它允许不同的 JavaScript 应用在运行时动态导入和共享依赖，无需预先打包。这种方案既保持了应用的独立性，又实现了高效的资源共享，能解决当前项目上的问题。

## Module Federation 核心概念

### 基本原理

Module Federation 基于**依赖运行时加载**的理念，将应用分为以下角色：

- **Host（消费者）**：消费其他应用暴露的模块
- **Remote（供应商）**：暴露模块供其他应用使用
- **双向应用**：既可以暴露模块，也可以消费其他应用的模块

### 关键特性

- **运行时加载**：模块在运行时动态加载，而非构建时静态打包
- **依赖共享**：可以共享依赖，避免重复加载
- **版本协商**：自动处理依赖版本兼容性
- **独立部署**：各个应用可以独立开发和部署

## 基本用法

### 安装和配置

首先需要安装 Webpack 5 或更高版本，当然用新一代的 Rsbuild 也可以：

```bash
npm install webpack@5 webpack-cli@5
```

### Webpack 配置

#### Remote 应用配置

```javascript
// webpack.config.js (供应商)
const { ModuleFederationPlugin } = require("webpack").container;
const deps = require("./package.json").dependencies;

module.exports = {
  mode: "development",
  entry: "./src/index.js",
  plugins: [
    new ModuleFederationPlugin({
      name: "remoteApp",
      filename: "remoteEntry.js",
      exposes: {
        "./Button": "./src/components/Button",
        "./App": "./src/App",
        "./utils": "./src/utils/index",
      },
      shared: {
        ...deps,
        react: {
          singleton: true,
          requiredVersion: deps.react,
        },
        "react-dom": {
          singleton: true,
          requiredVersion: deps["react-dom"],
        },
      },
    }),
  ],
};
```

#### Host 应用配置

```javascript
// webpack.config.js (消费者)
const { ModuleFederationPlugin } = require("webpack").container;
const deps = require("./package.json").dependencies;

module.exports = {
  mode: "development",
  entry: "./src/index.js",
  plugins: [
    new ModuleFederationPlugin({
      name: "hostApp",
      remotes: {
        remoteApp: "remoteApp@http://localhost:3001/remoteEntry.js",
      },
      shared: {
        ...deps,
        react: {
          singleton: true,
          requiredVersion: deps.react,
        },
        "react-dom": {
          singleton: true,
          requiredVersion: deps["react-dom"],
        },
      },
    }),
  ],
};
```

### 在应用中使用

#### Host 应用消费模块

```tsx
// src/App.js (Host应用)
import React, { useState } from "react";

// 动态导入远程模块
const RemoteButton = React.lazy(() => import("remoteApp/Button"));
const RemoteApp = React.lazy(() => import("remoteApp/App"));

const App = () => {
  const [showRemoteApp, setShowRemoteApp] = useState(false);

  return (
    <div>
      <h1>Host Application</h1>
      <React.Suspense fallback="Loading Button...">
        <RemoteButton onClick={() => alert("Hello from Host!")}>
          Remote Button in Host
        </RemoteButton>
      </React.Suspense>

      <br />
      <br />

      <button onClick={() => setShowRemoteApp(!showRemoteApp)}>
        {showRemoteApp ? "Hide" : "Show"} Remote App
      </button>

      {showRemoteApp && (
        <React.Suspense fallback="Loading App...">
          <RemoteApp />
        </React.Suspense>
      )}
    </div>
  );
};

export default App;
```

## 常见参数详解

### ModuleFederationPlugin 配置参数

```javascript
new ModuleFederationPlugin({
  name: "myApp", // 应用名称，必须唯一
  filename: "remoteEntry.js", // 远程入口文件名
  exposes: {
    // 暴露模块的路径映射
    "./Button": "./src/components/Button",
    "./utils": "./src/utils/index",
    // 支持相对路径
    "./components/Button": "./src/components/Button",
  },
  remotes: {
    // 基本语法：应用名@URL/入口文件
    remoteApp: "remoteApp@http://localhost:3001/remoteEntry.js",
  },
  shared: {
    ...deps,
    react: {
      singleton: true, // 确保只加载一个版本的React
      requiredVersion: "^18.0.0", // 指定版本要求
      eager: true, // 是否立即加载，如果配置成true，则立即加载，反之则懒加载，该参数会影响包的体积大小
      strictVersion: true, // 严格版本检查
    },
    "react-dom": {
      singleton: true,
      requiredVersion: deps["react-dom"],
    },
  },
});
```
