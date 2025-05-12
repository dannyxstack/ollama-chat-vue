## Language

- [中文](README.md)
- [English](README_en.md)

---

## ollama-chat-ui-vue

The front-end chat built with `vue3 + vite + elementUi` can talk to the model through ollama, and now supports `deepseek` independent thinking tags and switching models (follow-up support for online queries)

## Front-loading work

Installation `ollama`,[ollama](https://ollama.com/)
After installing `ollama`, open `cmd` and download the model(I chose the smallest model,[The address of the model](https://ollama.com/library/deepseek-r1:1.5b))

```bash
ollama run deepseek-r1:1.5b
```

Wait for the download to complete

## Start ollama

cmd to open the input

```bash
ollama serve
```

Wait for the service to start

## Launch the program ollama-chat-ui-vue

Note: Node version is 18.10.0

Open the project 'cmd' to install the dependencies

```bash
npm install
```

Start the project

```bash
npm run dev
```

## Profiles

src-api-aiSystem-ollama.js
vite.config


## Feel free to exchange issues

## Copyright Information

It's not easy to develop; if the content helps you, please give us a star.

Released under the MIT open source licence and free to use.
