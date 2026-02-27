# WayneVoyager's Blog

每次抛出硬币，记录在此。

基于 [Astro](https://astro.build) 框架和 [astro-whono](https://github.com/cxro/astro-whono) 开源主题搭建的个人博客，部署在 [GitHub Pages](https://pages.github.com) 上。

## 关于

我是 WayneVoyager，关注产品、AI 和生活中值得记下的事。每一次尝试都留下记录，不管正反。

## 项目结构

```text
├── public/               # 静态资源（图片、字体、favicon）
├── src/
│   ├── components/       # UI 组件
│   ├── content/          # 内容集合
│   │   ├── essay/        # 航行日志（长文）
│   │   ├── bits/         # 瓶中信（短想法）
│   │   └── memo/         # 锚地（回忆）
│   ├── layouts/          # 页面布局
│   ├── pages/            # 路由与页面
│   ├── scripts/          # 客户端脚本
│   └── styles/           # 样式
├── astro.config.mjs      # Astro 配置
├── site.config.mjs       # 站点元信息
└── package.json          # 依赖管理
```

## 部署

通过 [GitHub Actions](.github/workflows/deploy.yml) 自动部署，每次 push 到 `main` 分支即触发。

访问地址：https://waynevoyager.github.io/whoami/

## 主题声明

本博客使用 [astro-whono](https://github.com/cxro/astro-whono) 主题（MIT 协议）

## 许可证

MIT
