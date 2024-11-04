---
title: 整合现有项目，搭建monorepo
date: 2024-11-04 22:13:12
tags:
---

## 前言

由于我在开发项目或者练习的时候，不喜欢新建仓库，一个个 clone，所以我把所有的项目丢到一个仓库里，这样我可以在不同的项目中反复横跳，方便的修改，并且通过一个 git 请求，上传到 git。但这造成一个问题，也是多包仓库一个共同问题，依赖重复度高，偶然的机会让我接触到 `monorepo` ，于是打算深入一下，重构一下我的仓库。

## 为什么使用 Monorepo

- **代码库的可见性**：使用 Monorepo 可以让你轻松查看公司所有的代码库，而不需要去查找和克隆多个不同的仓库。
- **一致性**：Monorepo 能保持代码风格和配置的一致性。你可以在多个项目间共享 ESLint 配置、UI 组件库以及设计系统中的 Web 组件、实用工具库和文档。
- **依赖管理的优势**：Monorepo 的真正优势在于依赖管理，它可以帮助你直观地查看整个项目的依赖关系图。此外，对于第三方依赖库，它能够在多个应用中去重使用，从而减少重复安装的包。
- **持续集成和自动化**：Monorepo 非常适合用于持续集成和自动化流程，因为它可以轻松地一起构建和测试所有项目。

> 相比之下，Polyrepo 是一种传统的多仓库模式。Polyrepo 需要在不同的项目之间共享代码，通常流程较为复杂。


## Monorepo 的常见问题

- **构建时间较长**：随着项目规模增大，构建整个 Monorepo 会消耗更多时间。
- **编辑器性能问题**：在大型 Monorepo 中，VSCode 可能会因为处理大量的 Git 历史记录而出现卡顿。
- **CI/CD 速度缓慢**：在持续集成和部署流程中，Monorepo 会占用更多资源，导致速度变慢。

正因为如此，我们需要合适的工具来优化 Monorepo 的性能。Monorepo 官网定义了一系列构建工具需要遵守的约定，并且比较了市面上流行的 monorepo 构建工具，我们可以选择自己需要的工具。

## Turborepo

我们这里直接使用工具来搭建第一个 monorepo，而不用原始的手段，这可以很好避免上述的常见问题。
Turborepo 是一个优化 Monorepo 工作流的工具。在官网上介绍了多种可以用于 Monorepo 的工具，接下来我会挑选 Turborepo 进行讲解，展示它如何提升 Monorepo 的效率。

### 优势

Turborepo 遵循了官网的 monorepo 最佳实践的相关约定，并且将构建速度做了很大提升。

### Workspace 定义

在 Monorepo 的根目录的 `package.json` 中定义 `workspaces`，它会在第一层目录下查找带有 `package.json` 的文件夹，并将其识别为一个 workspace。


### 共享配置

他可以在 package 文件夹中创建 `tailwind.config.js` 配置文件包，在别的项目引入。

For example we want to share a tailwind config. We can generate a tailwind config and a package. Json in file.

If we need to support both commonJs and ESM ,we need to provide two js files.

Cjs. Js

```js
// packages/tailwind-config/tailwind.config.cjs
module.exports = {
  content: [
    // 其他项目可以在各自配置中重写 content 部分
  ],
  theme: {
    extend: {
      colors: {
        primary: '#3490dc',
        secondary: '#ffed4a',
        accent: '#38c172'
      },
      spacing: {
        '72': '18rem',
        '84': '21rem',
        '96': '24rem'
      }
    },
  },
  plugins: [],
};

```

Esm. Js

```js
// packages/tailwind-config/tailwind.config.js
export default {
  content: [
    // 其他项目可以在各自配置中重写 content 部分
  ],
  theme: {
    extend: {
      colors: {
        primary: '#3490dc',
        secondary: '#ffed4a',
        accent: '#38c172'
      },
      spacing: {
        '72': '18rem',
        '84': '21rem',
        '96': '24rem'
      }
    },
  },
  plugins: [],
};

```

Package. Json

```json
{
  "name": "@my-monorepo/tailwind-config",
  "version": "1.0.0",
  "main": "./tailwind.config.js",
  "exports": {
    "import": "./tailwind.config.js",
    "require": "./tailwind.config.cjs"
  },
  "type": "module",
  "license": "MIT",
  "files": [
    "tailwind.config.js",
    "tailwind.config.cjs"
  ],
  "peerDependencies": {
    "tailwindcss": "^3.0.0"
  }
}

```

Then we can install and use the config in other items. Its pnpm package. Json
Below 

```json
{
  "dependencies": {
    "@my-monorepo/tailwind-config": "workspace:*",
    "tailwindcss": "^3.0.0"
  }
}
```

In the tailwind. Config. Js we can write like . Show a commonjs way to import.


```js
// apps/app1/tailwind.config.js
const sharedConfig = require('@my-monorepo/tailwind-config');

module.exports = {
  presets: [sharedConfig],
  content: [
    "./src/**/*.{js,jsx,ts,tsx}",
    "./public/index.html"
  ]
};
```


### 步骤

全程跟着官网来没什么毛病。


## 参考文献

-  [turbo官网](https://turbo.build/repo/docs/getting-started)
- [monorepo 官网](https://monorepo.tools/#local-computation-caching)
- [Monorepos - How the Pros Scale Huge Software Projects // Turborepo vs Nx](https://www.youtube.com/watch?v=9iU_IE6vnJ8)
