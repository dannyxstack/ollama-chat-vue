## Language

- [中文](README.md)
- [English](README_en.md)

---

## ollama-chat-ui-vue

使用`vue3 + vite + elementUi` 搭建的前端 chat,通过 ollama 可与模型对话,目前已经支持`deepseek`的独立思考标签和切换模型(联网查询后续支持)

## 前置工作

安装`ollama`,[ollama 官网地址](https://ollama.com/)
安装完`ollama`后，打开`cmd`，下载模型(我选了个最小的模型，[模型地址](https://ollama.com/library/deepseek-r1:1.5b))

```bash
ollama run deepseek-r1:1.5b
```

等待下载完成

## 启动 ollama

cmd 打开输入

```bash
ollama serve
```

等待服务启动

## 启动前端 ollama-chat-ui-vue

注意：node 版本选择 18.10.0

打开项目`cmd`安装依赖

```bash
npm install
```

启动项目

```bash
npm run dev
```

## 配置文件

src-api-aiSystem-ollama.js
vite.config


## 欢迎 Issues 交流

## 版权信息

开发不易如果内容帮助到您，请给我们点上星星。

遵循 MIT 开源协议发布，并提供免费使用。
