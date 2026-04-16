# 自动部署网站预览规则

当用户的任务涉及创建或修改 Web 项目时，自动使用 `/deploy-website` skill 部署网站以供预览。

## 触发条件

满足以下任一条件时，执行网站部署：

1. **静态 HTML 文件被创建或修改**
   - 在工作空间中新建了 `.html` 文件
   - 已有的 `.html` 文件发生了较大改动

2. **Node.js Web 项目**
   - 项目包含 `package.json`，且定义了 Web 相关脚本（如 `dev`、`start`、`serve`）
   - 检测到前端框架：Next.js、Nuxt.js、Vite、Create React App、Vue CLI、Angular 等

3. **用户试图拉起前端或后端服务**
   - 用户要求启动、运行开发服务器
   - 用户要求在本地预览或测试应用
   - 涉及的命令如 `npm run dev`、`npm start`、`yarn dev`、`python manage.py runserver`、`go run` 等

## 使用方式

完成 Web 开发任务后，或者用户尝试拉起项目中的前端或后端服务时，调用以下 skill：

```
/deploy-website
```

该 skill 会自动检测项目类型并启动对应的服务。
