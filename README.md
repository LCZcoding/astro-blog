# 个人博客

欢迎访问：**https://lichangzhuo.xyz**

一个用于记录学习过程的个人博客，内容主要是后端开发、Agent开发、学习心得、工具使用技巧，以及一些生活随想。

基于 [fuwari](https://github.com/saicaca/fuwari) 模板搭建，感谢原作者 [@saicaca](https://github.com/saicaca)。

## 技术栈

- **框架**：[Astro](https://astro.build/) + [Svelte](https://svelte.dev/)
- **样式**：Tailwind CSS + Stylus
- **站内搜索**：[Pagefind](https://pagefind.app/)（构建时生成索引）
- **代码高亮**：[Expressive Code](https://expressive-code.com/)
- **CI/CD**：GitHub Actions

## 项目结构

```
astro-blog/
├── fuwari/                # 博客主项目（基于 fuwari 模板）
│   ├── src/content/posts/ # 文章目录（Markdown）
│   ├── src/config.ts      # 站点配置
│   └── scripts/new-post.js # 新建文章脚本
└── .github/workflows/     # 自动部署流水线
```

## 本地开发

需要 Node.js 22+ 和 pnpm 9：

```bash
cd fuwari
pnpm install

# 启动开发服务器
pnpm dev

# 新建一篇文章
pnpm new-post

# 构建并生成搜索索引
pnpm build
```

## 部署

推送到 `main` 分支后，GitHub Actions 会自动执行：

1. `pnpm build` 构建静态站点
2. 通过 SSH + rsync 将 `dist/` 同步到服务器
