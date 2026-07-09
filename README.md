<div align="center">

# svelte-skills

**Svelte framework development skills**

[![GitHub](https://img.shields.io/badge/github-full--stack--skills%2Fsvelte-skills-green.svg)](https://github.com/full-stack-skills/svelte-skills)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-Compatible-purple.svg)](https://agentskills.io)

English | [简体中文](./README.zh-CN.md)

[Introduction](#-introduction) ·
[Install](#-install) ·
[Skills](#-skills) ·
[Supported Agents](#-supported-agents) ·
[Ecosystem](#-ecosystem)

</div>

---

## 📖 Introduction

**Svelte Skills** is a curated collection of Agent Skills for AI coding agents, part of the [Full Stack Skills](https://github.com/partme-ai/full-stack-skills) ecosystem maintained by [PartMe.AI](https://github.com/partme-ai).

This package includes **16 skills**. Each skill is a self-contained `SKILL.md` file that AI agents load on-demand.

## 🔗 Official Documentation

We strongly recommend reading the official Svelte documentation alongside these skills:

- [Svelte Overview](https://svelte.dev/docs/svelte/overview) — Svelte 5 introduction and key concepts
- [Svelte LLMs.txt](https://svelte.dev/llms.txt) — Complete Svelte 5 documentation for LLMs (17.5k+ lines, covers Svelte + SvelteKit + CLI + AI)

Also see the package-specific docs:
- [Svelte 5 documentation](https://svelte.dev/docs/svelte/llms.txt)
- [SvelteKit documentation](https://svelte.dev/docs/kit/llms.txt)
- [Svelte CLI documentation](https://svelte.dev/docs/cli/llms.txt)
- [Svelte AI documentation](https://svelte.dev/docs/ai/llms.txt)

## 📦 Install

```bash
npx skills add full-stack-skills/svelte-skills
```

Or install specific skills:

```bash
npx skills add full-stack-skills/svelte-skills --skill <skill-name>
```

## 🎯 Skills (16)

| Skill | Description |
|-------|-------------|
| `svelte-awesome` | Svelte 5 导航入口，覆盖 8 个子技能（runes/template syntax/styling/special elements/runtime/misc/reference/legacy APIs）以及 SvelteKit、CLI、AI 工具 |
| `svelte-runes` | Svelte 5 Runes 响应式系统：$state / $derived / $effect / $props / $bindable / $inspect / $host 完整参考 |
| `svelte-template-syntax` | 块语法 (`#if` / `#each` / `#await` / `#snippet` / `#key`)、`{@render}` / `{@html}` / `{@attach}` / `{@debug}`、`bind:` 指令、events、use:、transition:、animate:、style:、class |
| `svelte-styling` | Scoped Styles、`:global`、`@keyframes`、`style:`、class 指令、`:where()`、Tailwind 集成 |
| `svelte-special-elements` | `<svelte:boundary>` / `<svelte:window>` / `<svelte:document>` / `<svelte:body>` / `<svelte:head>` / `<svelte:element>` / `<svelte:options>` |
| `svelte-runtime` | imperative API（mount/unmount/render/hydrate）、服务端渲染、CSP、生命周期与 mount/hydrate 协作 |
| `svelte-misc` | TypeScript（泛型、Component 类型、DOM 类型增强）、Custom Elements、Best Practices、Browser Support、Svelte 4 迁移 |
| `svelte-reference` | `use:` Action、tweened/spring 运动、`transition:` / `in:` / `out:` / `animate:`、完整 store 工具集 |
| `svelte-legacy-apis` | Svelte 4 → 5 迁移、Legacy Mode（`<svelte:options runes={false}>`）、保留可用的旧 API（export let / $: / on:click / slot / createEventDispatcher） |
| `svelte-lifecycle` | 生命周期钩子（onMount/onDestroy/tick）、Stores（writable/readable/derived）、Context API、Testing（Vitest/Storybook/Playwright）、Browser support |
| `sveltekit-overview` | SvelteKit 概览、创建项目、项目类型决策矩阵、项目结构、Web 标准（Fetch/FormData/Stream/URL/Web Crypto）、Routing（+page/+layout/+server/+error/$types） |
| `sveltekit-data` | 数据加载（universal/server load）、page.data、URL data、Cookies/Headers、Errors/Redirects、Streaming、Form actions、use:enhance、Page options（prerender/ssr/csr） |
| `sveltekit-advanced` | State management（SSR 安全）、Remote functions（query/form/command）、Env vars（$env/*）、Hooks（handle/sequence）、Errors、Link options、Service workers、$app/* 模块、$lib、shallow routing |
| `sveltekit-config` | 构建（vite build/preview）、Adapters（node/static/cloudflare/netlify/vercel）、Advanced routing/layouts、Auth、Performance、Images（@sveltejs/enhanced-img）、A11y、SEO、Migration（v1→v2、Sapper→SvelteKit） |
| `svelte-cli` | `sv` CLI：`sv create` 创建项目、`sv add` 添加 13 个官方集成、`sv check` 类型与编译检查、`sv migrate` 迁移脚本、`sv-utils` 低级 API |
| `svelte-ai` | Svelte AI 工具：AGENTS.md 规范、MCP server（list-sections/get-documentation/svelte-autofixer/playground-link）、Prompts（svelte-task）、CLI、Subagents（svelte-code-writer/svelte-core-bestpractices）、11+ AI 客户端安装 |

## 🤖 Supported Agents

Works with [Claude Code](https://code.claude.com), [Codex](https://developers.openai.com/codex), [Cursor](https://cursor.com), [OpenCode](https://opencode.ai), [Gemini CLI](https://geminicli.com), [GitHub Copilot](https://github.com/features/copilot), [Windsurf](https://codeium.com/windsurf), and [70+ others](https://agentskills.io/clients).

### Claude Code Installation

**Option 1: npx skills CLI (Recommended)**

```bash
npx skills add full-stack-skills/svelte-skills
```

**Option 2: Manual Installation**

```bash
git clone https://github.com/full-stack-skills/svelte-skills.git
cp -r svelte-skills/skills/* .claude/skills/
```

For more details, see the [Claude Code Skills Guide](https://code.claude.com/docs/en/skills) and [Agent Skills Spec](https://agentskills.io/).

## 🌐 Ecosystem

| Resource | Link |
|----------|------|
| **Full Stack Skills** | [github.com/partme-ai/full-stack-skills](https://github.com/partme-ai/full-stack-skills) |
| **All Skill Groups** | [github.com/full-stack-skills](https://github.com/full-stack-skills) |
| **Agent Skills Spec** | [agentskills.io](https://agentskills.io) |
| **Skills CLI** | [github.com/vercel-labs/skills](https://github.com/vercel-labs/skills) |

## 📄 License

Apache 2.0 — see [LICENSE](LICENSE).
