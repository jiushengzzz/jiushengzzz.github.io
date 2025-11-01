+++
date = '2025-10-31T16:16:48+08:00'
draft = false
title = 'Monorepo策略下的前端项目结构'
categories = ["前端"]
+++


最近正在寻找一套新的前端框架来满足不断扩充的业务，目标是组件库和公共逻辑库能够优雅的穿插在不同的项目中，打包速度要快，在翻阅[coze](https://github.com/coze-dev/coze-studio)的源码时发现其项目结构清晰合理，本着对字节前端团队的信任，便研究了一下，其Monorepo+Rsbuild的架构确实能满足我的要求。

## 概念
Monorepo是一种项目开发与管理的策略模式，它允许**多个工程项目**集中在一个文件夹中进行**源码层面**的开发和管理，但是模块之间又是独立的代码仓库。

## 背景
传统的前端项目中，代码放在一个文件夹中作为一个仓库，随着代码量的增加，公共模块和业务模块会按照文件夹进行拆分，但此时公共模块还是耦合在这个项目中无法被复用，因此后面逐渐把这些公共的组件/逻辑变成一个个模块包，放在公共或者私有的npm仓库，以dist的形式运行在各个项目中，这也是当下最流行的项目结构，但是这样会有几个缺点：
- 模块包的版本更新比较繁琐，一旦升级了版本，所有项目都需要跟着改
- 调试比较麻烦
- 企业环境下，模块包无法放在公共的npm仓库，需要搭建私有的npm仓库或者软链接的方式进行管理

于是，Monorepo的出现解决了这个问题，既能够集中开发，又能保持各个模块的相对独立。

## 基本使用
在我的项目中，最终使用的是Monorepo+Rsbuild+Turbo的组合架构。其中：
- Monorepo+pnpm负责串联项目结构，天生支持workspace的概念
- Rsbuild是构建工具，基于Rspack，负责代码的编译、转换、打包、优化等，类似于Webpack或者Vite
- Turbo是一个构建系统和增量构建工具，自主调度和加速多个项目中的任务（如 build, test, lint），不直接处理代码转换和打包

## 项目结构
```text
my-monorepo/
├── apps/
│   ├── web-app/（一个 React 应用）
│   └── admin-app/（另一个 React 应用）
├── packages/
│   ├── ui/（共享组件库）
│   └── core/（共享逻辑库）
├── pnpm-workspace.yaml
└── turbo.json
```
其中 pnpm-workspace.yaml 的内容如下：
```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```
使用packages字段声明哪些目录下的项目是工作区的子项目。这些目录必须包含package.json文件，并且package.json文件中必须包含name字段，一般为@{根目录名称}/{具体包名}，如：@my-monorepo/ui。

有了上述配置后，如果在apps/web-app应用上想引用@my-monorepo/ui，则只需要在apps/web-app的package.json中做如下配置：
```json
{
     "dependencies": {
        "@my-monorepo/ui": "workspace:*",
     }
}
```
即可实现源码引用和调试开发了。

## 关于Turbo
关于Turbo的用法和配置这里就不多讲了，可以直接参考[官方文档](https://turborepo.com/docs)，Turbo能够理解项目中的依赖关系，比如构建apps/web-app时会先构建@my-monorepo/ui，如果没有依赖关系就直接并行构建，功能非常强大，非常适合追求打包速度的开发者。

