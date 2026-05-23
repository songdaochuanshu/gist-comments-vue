# 🎨 艺术留言板 | Minimalist Gist Comments

![Gist Comments Vue Screenshot](https://raw.githubusercontent.com/songdaochuanshu/gist-comments-vue/main/screenshot.png) <!-- 占位符，待后续添加实际截图 -->

## ✨ 项目简介

**艺术留言板 | Minimalist Gist Comments** 是一个基于 Vue 3、Vite 和 GitHub Gist API 构建的轻量级、高颜值留言板应用。它旨在为个人博客、静态网站或任何需要简单评论功能的场景提供一个无需后端数据库的解决方案。通过利用 GitHub Gist 作为数据存储，本应用实现了评论的发布、展示和分页功能，同时兼顾了现代 Web 应用的性能与用户体验。

近期，我们对项目进行了全面的 UI/UX 升级，融入了**毛玻璃（Glassmorphism）设计风格**，并优化了整体视觉效果，使其更具**高级感和艺术感**。无论是在桌面还是移动设备上，都能提供流畅且美观的交互体验。

## 🚀 主要特性

*   **无需后端数据库**：所有留言数据存储在 GitHub Gist 中，极大简化了部署和维护。
*   **现代 UI 设计**：采用毛玻璃效果、柔和渐变和精致动画，打造独特的视觉体验。
*   **响应式布局**：完美适应不同屏幕尺寸，提供一致的用户体验。
*   **深色模式支持**：自动适配系统主题，提供舒适的阅读环境。
*   **留言发布与展示**：用户可提交留言，并以分页形式展示。
*   **冷却时间机制**：防止恶意刷屏，保障留言质量。
*   **简洁高效**：基于 Vue 3 和 Vite，打包体积小，加载速度快。

## 🛠️ 技术栈

*   **前端框架**：Vue 3
*   **构建工具**：Vite
*   **HTTP 客户端**：Axios
*   **数据存储**：GitHub Gist API
*   **样式**：原生 CSS (Scoped CSS)

## ⚙️ 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/songdaochuanshu/gist-comments-vue.git
cd gist-comments-vue
```

### 2. 安装依赖

```bash
pnpm install
# 或者使用 npm:
npm install
# 或者使用 yarn:
yarn install
```

### 3. 配置环境变量

本应用需要两个 GitHub 相关的环境变量来读写 Gist。请在项目根目录创建 `.env` 文件，并添加以下内容：

```
VITE_GIST_ID=你的Gist ID
VITE_GITHUB_TOKEN=你的GitHub Personal Access Token
```

*   **如何获取 Gist ID**：
    1.  前往 [GitHub Gist](https://gist.github.com/) 创建一个新的 Gist。
    2.  文件名为 `comments.json`，内容为一个空的 JSON 数组 `[]`。
    3.  创建后，Gist 的 URL 类似于 `https://gist.github.com/你的用户名/你的Gist ID`，复制 `你的Gist ID` 部分。

*   **如何获取 GitHub Personal Access Token (PAT)**：
    1.  前往 [GitHub Settings -> Developer settings -> Personal access tokens](https://github.com/settings/tokens)。
    2.  点击 `Generate new token (classic)`。
    3.  为 Token 命名，例如 `gist-comments-vue-token`。
    4.  **重要**：在 `Select scopes` 中，勾选 `gist` 权限。
    5.  生成 Token 后，**务必立即复制并保存**，它只会显示一次。

### 4. 运行项目

#### 开发模式

```bash
pnpm run dev
```

项目将在 `http://localhost:5173` 启动（默认端口）。

#### 生产构建

```bash
pnpm run build
```

构建后的文件将输出到 `dist` 目录，可部署到任何静态文件托管服务。

## 🎨 UI/UX 亮点

*   **毛玻璃卡片**：留言板主体和模态框均采用半透明毛玻璃效果，与背景深度融合。
*   **动态交互**：按钮、输入框等元素在交互时伴有流畅的动画和视觉反馈。
*   **色彩美学**：选用柔和且现代的色彩搭配，强调高级感。
*   **信息可视化**：加载状态和冷却时间通过简洁的动画和提示清晰传达。

## 🤝 贡献

欢迎所有形式的贡献！如果您有任何建议、功能请求或 Bug 报告，请随时提交 Issue 或 Pull Request。

## 📄 许可证

本项目采用 MIT 许可证。详情请参阅 `LICENSE` 文件。
