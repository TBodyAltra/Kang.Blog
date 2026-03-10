# Debug Notes: Hexo Blog (Kang.Blog)

## 1) 空白页面（public/index.html 0 字节）
- 现象：页面空白，`public/index.html` 为 0 字节。
- 根因：执行了 `hexo clean` 但未生成，或生成失败/中断，导致部署了空的 `public/index.html`。
- 处理：
  - 本地：`npx hexo clean && npx hexo generate`
  - 部署：`npx hexo deploy -g` 或让 CI 自动生成。

## 2) 文章图片不显示（post_asset_folder）
### 关键规则
- `post_asset_folder: true` 时，**文章资源必须放在“与文章同名的目录”里**。
- 如果文章路径是：
  - `source/_posts/triton_fa_opt/index.md`
  - 实际 slug 为 `triton_fa_opt/index`
  - **正确的资源目录**应为：`source/_posts/triton_fa_opt/index/`

### 现象与原因
- 文章里图片用相对路径或 `{% asset_img %}`：
  - HTML 里生成了 `<img src="...">`
  - 但 `public/` 中没有对应图片文件
- 原因是图片不在正确的 post asset 目录，Hexo 不会拷贝。

### 解决方案
- 把图片移动到正确目录：
  - `source/_posts/triton_fa_opt/index/large_idle.jpg`
  - `source/_posts/triton_fa_opt/index/small_idle.jpg`
- 对应 Markdown 使用：
  - `{% asset_img large_idle.jpg large idle %}`
  - `{% asset_img small_idle.jpg small idle %}`

### 批量修复过的文章
- `source/_posts/triton_fa_opt/index.md`
- `source/_posts/amd_page_fault/index.md`
- `source/_posts/企业AI Agent安全部署与任务调度系统/企业AI Agent安全部署与任务调度系统.md`

## 3) 公式不显示（MathJax 未加载）
### 现象
- HTML 内有 `<script type="math/tex; mode=display">...</script>`
- 但页面不渲染公式

### 根因
- Butterfly 主题未启用 `math.use`，只引入了 KaTeX CSS，没有 MathJax JS。
- CI 会重新 clone 主题，直接改 `themes/butterfly/_config.yml` 会被覆盖。

### 修复
- 新增根目录覆盖配置：`_config.butterfly.yml`
- 内容：
  ```yaml
  math:
    use: mathjax
    per_page: true
    hide_scrollbar: false
    mathjax:
      enableMenu: true
      tags: none
  ```

## 4) GitHub Actions 部署规则
- `.github/workflows/deploy.yml` 只在 **push 到 master** 触发。
- Actions 会：
  - `hexo clean && hexo generate`
  - 将 `public/` 部署到 `gh-pages` 分支。
- GitHub Pages 设置为 `gh-pages` + `/(root)` 正常。

## 5) 如何查看 Actions 状态
### 推荐（网页）
- 仓库 → Actions → 选择 `Deploy to GitHub Pages` → 查看最新 run

### 使用 GitHub API（需要网络 + 可能需要 Token）
- API：`https://api.github.com/repos/TBodyAltra/Kang.Blog/actions/runs?per_page=5`
- 若遇到 `API rate limit exceeded`：
  - 需要提供 GitHub Token（只需 `public_repo` 权限）
  - 或改用网页查看

## 6) 常用检查命令
- 查看文章生成后的 HTML：
  - `rg -n "math/tex|katex|mathjax" public/2024/06/15/RethinkBayes/index.html`
- 检查图片是否被生成：
  - `find public -name "large_idle.jpg" -o -name "small_idle.jpg"`
- 检查 post asset 目录结构：
  - `ls -la source/_posts/<slug>/` 或 `source/_posts/<slug>/index/`

