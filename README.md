# Monorepo 架构学习案例

> deepseek 生成的答案，执行测试可行，记录学习

使用 Vite 搭建 Monorepo  workspace 协议和 pnpm 的**硬链接**机制，让代码共享和依赖管理变得非常顺畅 。下面，我将带你从零开始，搭建一个包含一个应用 (`web`) 和一个共享 UI 组件库 (`ui`) 的 Monorepo 项目。

### 🗺️ 项目蓝图与核心工具

在开始之前，我们先明确一下要使用的工具和最终的项目结构。

+ **包管理器**: 我们将使用 **pnpm**，它对 Monorepo 有着极佳的支持，能通过 `workspace` 协议高效地管理本地包 。
+ **构建工具**: 所有的应用和包都基于 **Vite** 进行构建，享受其快速的开发体验 。
+ **可选增强**: 对于更复杂的场景，可以引入 **Turborepo** 来优化任务的并行执行和缓存 。

项目最终的目录结构如下：

```plain
my-vite-monorepo/
├── packages/                 # 存放所有内部包
│   └── ui/                   # 共享 UI 组件库
│       ├── src/
│       │   ├── Button.tsx
│       │   └── index.ts
│       ├── package.json      # name: @myorg/ui
│       └── vite.config.ts    # UI 库的 Vite 配置
├── apps/                     # 存放各种应用
│   └── web/                  # 主 Web 应用
│       ├── src/
│       │   └── App.tsx
│       ├── package.json      # name: @myorg/web, 依赖 @myorg/ui
│       └── vite.config.ts    # 应用的 Vite 配置
├── package.json              # 根目录的配置文件
├── pnpm-workspace.yaml       # pnpm 工作区定义
├── pnpm-lock.yaml            # 统一的依赖锁文件
└── tsconfig.json             # 可选的根目录 TypeScript 配置
```

### 👣 分步搭建指南

#### 步骤 1: 初始化项目根目录

首先，创建项目文件夹并使用 pnpm 初始化。

```bash
# 创建项目目录并进入
mkdir my-vite-monorepo && cd my-vite-monorepo

# 初始化 package.json
pnpm init
```

#### 步骤 2: 定义 pnpm 工作区

在根目录创建 `pnpm-workspace.yaml` 文件，告诉 pnpm 哪些目录是工作区的一部分 。

```yaml
# pnpm-workspace.yaml
packages:
  # 所有在 apps/ 和 packages/ 目录下的包
  - 'apps/*'
  - 'packages/*'
  # 如果某些子目录不属于工作区，可以排除
  - '!**/test/**'
```

#### 步骤 3: 创建共享 UI 组件库 (`packages/ui`)

接下来，我们创建一个供应用使用的共享 UI 库。

```bash
# 进入 packages 目录并创建 ui 包
mkdir packages && cd packages
mkdir ui && cd ui

# 初始化包，name 使用 scope 命名，方便区分 
pnpm init
```

编辑 `packages/ui/package.json`，修改 `name` 和 `main` 字段，并设置 `type: "module"` 。

```json
{
  "name": "@myorg/ui",
  "version": "1.0.0",
  "type": "module",
  "main": "src/index.ts",
  "scripts": {
    "build": "vite build"
  },
  "devDependencies": {
    "vite": "^4.0.0"
  }
}
```

> 如果你的 UI 库使用 React，需要安装 `@vitejs/plugin-react`；如果是 Vue，则安装 `@vitejs/plugin-vue` 。
>

```bash
# React
pnpm install @vitejs/plugin-react

# Vue
pnpm intall @vitejs/plugin-vue
```

现在，创建 `packages/ui/src/Button.tsx` 组件文件：

```tsx
// packages/ui/src/Button.tsx
import React from 'react';

interface ButtonProps {
  children: React.ReactNode;
  onClick?: () => void;
}

export const Button = ({ children, onClick }: ButtonProps) => {
  return <button onClick={onClick}>{children}</button>;
};
```

⚠️如果是 Vue，如下

```vue
<template>
  <button @click="onClick">{{ btnText }}</button>
</template>

<script setup>
const props = defineProps([
  {
    btnText: {
      type: String,
      default: `按钮默认文本`,
      required: true,
    },
  },
])

const onClick = () => {
  alert(`clicking`)
}
</script>

```

创建 `packages/ui/src/index.ts` 作为库的入口文件，导出所有公共组件：

```typescript
// packages/ui/src/index.ts
export { Button } from './Button';
```

#### 步骤 4: 创建 Web 应用 (`apps/web`)

现在，我们来创建一个基于 Vite 的 Web 应用，它将使用我们刚刚创建的 UI 库。

```bash
# 回到项目根目录，然后进入 apps 目录
cd ../../
mkdir apps && cd apps

# 使用 Vite 创建一个新项目，这里以 React + TypeScript 模板为例 
pnpm create vite web --template react-ts

# 进入应用目录
cd web
```

⚠️使用 Vue 的话，如下，自己选择库时候选择 Vue 即可

```bash
pnpm create vite
```

`pnpm create vite` 命令会自动生成 `package.json` 和基本的 Vite 配置。

#### 步骤 5: 建立本地依赖关系

最关键的一步，让 Web 应用能够引用本地的 UI 库。编辑 `apps/web/package.json`，在 `dependencies` 中添加对 `@myorg/ui` 的引用，版本号使用特殊的 `workspace:*` 协议 。

```json
// apps/web/package.json
{
  "name": "@myorg/web",
  "version": "0.0.0",
  "private": true,
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@myorg/ui": "workspace:*"   // 关键配置
  },
  "devDependencies": {
    // ... Vite 相关的 devDependencies
  }
}
```

然后，在**项目根目录**下执行安装命令。pnpm 会自动识别 `workspace:*` 协议，并将本地包链接过来 。

```bash
# 回到项目根目录
cd ../../

# 安装所有依赖，并自动链接本地包
pnpm install
```

安装完成后，你会在 `node_modules` 中看到 `@myorg/ui` 指向了本地 `packages/ui` 的软链接。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/555576/1772678640637-567d3e93-82e0-4f47-8ddc-e989e9baea6d.png)

#### 步骤 6: 在应用中使用本地 UI 库

现在可以在应用里导入并使用 `@myorg/ui` 了。打开 `apps/web/src/App.tsx`：

```tsx
// apps/web/src/App.tsx
import React, { useState } from 'react';
import { Button } from '@myorg/ui'; // 直接从本地包导入

function App() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <h1>Vite + React Monorepo</h1>
      <Button onClick={() => setCount((count) => count + 1)}>
        count is {count}
      </Button>
    </div>
  );
}

export default App;
```

#### 步骤 7: 配置根目录启动脚本

为了方便，我们可以在根目录的 `package.json` 中添加脚本来启动 Web 应用的开发服务器 。

```json
// package.json (根目录)
{
  "name": "my-vite-monorepo",
  "version": "1.0.0",
  "scripts": {
    "dev": "pnpm --filter @myorg/web dev",
    "build": "pnpm --filter @myorg/web build"
  }
}
```

这里的 `--filter` 选项告诉 pnpm 在 `@myorg/web` 包下执行 `dev` 命令。

最后，在根目录运行以下命令，就可以看到应用成功运行，并且它已经使用了本地的 UI 组件库 。

```bash
pnpm run dev
```

### 💡 进一步优化与最佳实践

搭建好基础结构后，你还可以通过以下方式让项目更加健壮和高效。

+ **共享 Vite/TypeScript 配置**：创建一个 `packages/config` 包，在里面存放公共的 `vite.config.base.ts` 或 `tsconfig.base.json`。其他包可以通过依赖这个配置包，并使用 `mergeConfig` 函数来继承和扩展基础配置 。
+ **引入 Turborepo 加速**：如果项目包含多个应用和包，构建速度会变慢。可以安装 Turborepo，在根目录创建 `turbo.json` 配置文件，通过其**智能缓存**和**并行执行**能力，显著提升构建效率 。
+ **使用 Changesets 管理版本**：对于需要发布到 npm 的包（如 UI 库），可以使用 Changesets 来管理版本号和生成变更日志，实现规范化的版本发布流程 。
+ **统一代码规范**：在根目录配置 ESLint、Prettier 和 Husky，确保所有包遵循统一的代码风格和 Git 提交规范 。

完成以上步骤，你就拥有了一个可扩展、易维护的 Monorepo 项目基础。你可以根据实际需求，在 `apps` 和 `packages` 下继续添加更多应用和共享模块。如果在配置共享 Vite 配置时遇到问题，或者想进一步了解 Turborepo 的集成，随时可以再问我。
