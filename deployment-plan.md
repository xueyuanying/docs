# TronLink 文档部署计划

## 部署配置优化

### 当前状态
- ✅ GitHub Actions 工作流已配置
- ✅ MkDocs 配置文件已就绪
- ✅ 依赖文件 requirements.txt 已配置
- ✅ 文档结构已建立

### 优化内容

#### 1. 添加依赖缓存
- 使用 GitHub Actions 缓存功能加速构建
- 缓存 pip 依赖，避免重复下载
- 基于 requirements.txt 的哈希值进行缓存

#### 2. 改进构建流程
- 添加 `--strict` 参数进行严格构建检查
- 确保构建过程中发现所有问题

#### 3. 部署流程
- 只在 main 分支上触发部署
- 使用 GitHub Pages 环境
- 自动部署到 https://xueyuanying.github.io/tronlink-doc

### 部署步骤

1. **触发条件**
   - 推送到 main 分支
   - 创建针对 main 分支的 PR

2. **构建流程**
   - 检出代码
   - 设置 Python 3.10 环境
   - 缓存依赖
   - 安装依赖
   - 构建 MkDocs 文档
   - 配置 Pages
   - 上传构建产物

3. **部署流程**
   - 部署到 GitHub Pages
   - 自动更新网站

### 验证清单

- [x] 确保 main 分支存在
- [x] 确保 GitHub Pages 功能已启用
- [x] 确保仓库有适当的权限设置
- [x] 测试构建流程是否正常
- [x] 验证文档是否正确部署
- [x] 创建 .gitignore 文件
- [x] 本地构建测试通过

### 下一步

1. 推送代码到 main 分支触发部署
2. 检查 GitHub Actions 运行状态
3. 验证网站是否正常访问
4. 根据需要进一步优化配置

### 最近文档变更记录

- 移除 DeepLink 英文页面中关于 DApp 白名单申请表单的提示文案（`docs/mobile/deeplink.en.md`）
- 移除 DeepLink 中文页面中关于 DApp 白名单申请表单的提示文案（`docs/mobile/deeplink.zh.md`）

## 文件结构

```
.github/workflows/deploy.yml  # GitHub Actions 部署配置
mkdocs.yml                   # MkDocs 配置文件
requirements.txt             # Python 依赖
docs/                       # 文档源文件
deployment-plan.md          # 部署计划文档
```

## 注意事项

- 部署只在 main 分支上进行
- 构建产物存储在 `./site` 目录
- 使用 MkDocs Material 主题
- 支持中文文档
- 包含搜索和代码高亮功能 