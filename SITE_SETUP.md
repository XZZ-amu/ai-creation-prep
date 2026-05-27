# MkDocs 站点搭建说明

## 本地预览

```bash
# 1. 装依赖（首次需要）
pip install mkdocs-material pymdown-extensions

# 2. 在项目根目录运行
cd ~/Projects/ai-creation-prep
mkdocs serve

# 3. 浏览器打开 http://127.0.0.1:8000
# 改 markdown 会自动热重载
```

## 部署到 GitHub Pages

### 第一次部署（一次性配置）

1. **打开 GitHub 仓库设置**：https://github.com/XZZ-amu/ai-creation-prep/settings/pages
2. **Source** 选 `GitHub Actions`（不是 `Deploy from a branch`）
3. **保存**

### 推送触发自动部署

```bash
cd ~/Projects/ai-creation-prep

git add .
git commit -m "Setup MkDocs Material site for AI creation docs"
git push origin main
```

推送后到 https://github.com/XZZ-amu/ai-creation-prep/actions 看部署进度，约 1-2 分钟完成。

### 部署完成后访问

https://xzz-amu.github.io/ai-creation-prep/

## 后续怎么更新

每次写完新的 `01-mechanism.md` 或修改任何 markdown：

1. 本地 `mkdocs serve` 预览（可选）
2. 如果新增了文档要让它出现在左侧导航，编辑 `mkdocs.yml` 里的 `nav` 段加一行
3. `git add . && git commit -m "..." && git push` 自动触发部署

## 故障排查

**本地 `mkdocs serve` 报错找不到模块**：
```bash
pip install --upgrade mkdocs-material
```

**部署成功但页面 404**：
- 检查 GitHub Pages 设置 Source 是不是选了 `GitHub Actions`
- 等 5 分钟（DNS 缓存）

**导航里的某篇文档点进去 404**：
- 检查 `mkdocs.yml` 里的路径和实际文件路径一致
- 注意区分 `01-mechanism.md` 和 `01-mechanism-v2.md`
