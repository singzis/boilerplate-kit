# 前端发版上线规范流程

## 一、准备阶段

### 1.1 代码分支管理

- **主分支**：`master`（生产环境代码，保持稳定可部署状态）
- **开发分支**：`develop-1`（多套并行开发分支，分别部署不同功能模块，修改末尾数字，目前一共 6 套，分别是`develop-1`、`develop-2`、`develop-3`、`develop-4`、`develop-5`、`develop-6`）
- **功能分支**：`feature/*`（新功能开发，命名格式：`feature/功能名称-日期`）
- **发布分支**：`release/*`（发版专用分支，命名格式：`release/v版本号`）
- **tag**：`v版本号`（版本号标签，命名格式：`v版本号`）
- **修复分支**：`hotfix/*`（生产环境紧急修复，命名格式：`hotfix/问题描述-日期`）

### 1.2 版本号管理（遵循 SemVer 语义化规范）

- 格式：`MAJOR.MINOR.PATCH`（主版本号.次版本号.修订号）
  - **MAJOR**：不兼容的 API 变更（如 v1.0.0 → v2.0.0）
  - **MINOR**：新增功能且向后兼容（如 v2.2.0 → v2.3.0）
  - **PATCH**：向后兼容的 bug 修复（如 v2.3.1 → v2.3.2）
- 版本号更新工具：
  ```bash
  npm version major  # 升级主版本号
  npm version minor  # 升级次版本号
  npm version patch  # 升级修订号（自动提交变更）
  ```

## 二、标准发版流程

### 2.1 开发与提测阶段

1. **功能开发完成**

   - 在`feature/*`分支完成开发，本地自测通过后提交 PR（Pull Request）到`develop`分支。
   - 需通过代码评审（至少 1 名团队成员审核）。

2. **测试环境部署**

   - 合并到`develop`后，触发 CI 自动构建（如`npm run build:test`），部署至测试环境（如`test.xxx.com`）。

3. **测试反馈与修复**
   - QA 在测试环境验证功能，反馈问题后，开发在原`feature/*`分支修复，重复步骤 2 直至测试通过。

### 2.2 创建发布分支

1. **从`develop`分支切出发布分支**：

   ```bash
   git checkout develop
   git pull origin develop
   git checkout -b release/v2.3.1  # 以目标版本号命名
   ```

2. **更新版本号**
   - 方式 1：手动修改`package.json`的`version`字段，提交变更：
     ```bash
     git add package.json
     git commit -m "chore: bump version to v2.3.1"
     ```
   - 方式 2：使用`npm version`自动更新（推荐）：
     ```bash
     npm version patch -m "chore: bump version to %s"  # %s会自动替换为版本号
     ```

### 2.3 预发布环境验证

1. **构建生产包**  
   在发布分支执行生产环境构建（确保与生产环境依赖一致）：

   ```bash
   npm ci  # 优先使用package-lock.json安装依赖，避免版本差异
   npm run build  # 生成dist目录（生产环境资源）
   ```

2. **部署至预发布环境**  
   将构建产物部署到预发布环境（如`staging.xxx.com`），该环境配置与生产一致（数据库、CDN、接口域名等）。

3. **最终验证**
   - QA 执行回归测试、兼容性测试（多浏览器/设备）、性能测试（LCP、FID 等指标）。
   - 产品确认功能符合需求，无线上风险。
   - 开发检查控制台报错、资源加载异常等问题。

### 2.4 合并代码与打 Tag

1. **合并到主分支（`main`）**

   ```bash
   git checkout main
   git pull origin main
   git merge --no-ff release/v2.3.1 -m "chore: merge release v2.3.1 to main"
   git push origin main
   ```

2. **打版本 Tag**

   ```bash
   git tag -a v2.3.1 -m "release: v2.3.1"  # 创建带注释的Tag，便于追溯
   git push origin v2.3.1  # 推送Tag到远程仓库
   ```

3. **合并回`develop`分支**  
   确保开发分支同步发布分支的变更：

   ```bash
   git checkout develop
   git pull origin develop
   git merge --no-ff release/v2.3.1 -m "chore: merge release v2.3.1 to develop"
   git push origin develop
   ```

4. **删除发布分支**  
   发布完成后清理临时分支：
   ```bash
   git branch -d release/v2.3.1
   git push origin --delete release/v2.3.1
   ```

### 2.5 生产环境部署

1. **部署策略选择**

   - **全量部署**：适用于无重大变更的小版本（如补丁修复），直接替换生产资源。
   - **灰度发布**（推荐）：适用于功能迭代，逐步扩大覆盖范围：
     - 阶段 1：部署至 1 台服务器/10%用户（通过负载均衡/CDN 权重控制）。
     - 阶段 2：监控 15 分钟，无异常则扩大至 50%用户。
     - 阶段 3：确认稳定后全量部署（100%用户）。

2. **自动化部署（CI/CD）**  
   通过工具（如 GitHub Actions、Jenkins）触发部署，示例（GitHub Actions 配置片段）：

   ```yaml
   jobs:
     deploy-production:
       runs-on: ubuntu-latest
       needs: build # 依赖构建 job，确保产物正确
       if: github.ref == 'refs/heads/main' # 仅主分支触发
       steps:
         - name: Download build artifacts
           uses: actions/download-artifact@v3
           with:
             name: dist
             path: dist

         - name: Deploy to CDN
           uses: some-cdn-deploy-action # 自定义CDN部署动作（如阿里云OSS、腾讯云COS）
           with:
             cdn-secret: ${{ secrets.CDN_SECRET }}
             local-path: ./dist
   ```

### 2.6 发布后检查

1. **线上环境验证**

   - 访问生产域名（如`xxx.com`），检查页面渲染、功能交互、接口调用是否正常。
   - 验证 CDN 资源是否生效（查看资源 URL 是否为 CDN 路径，如`https://cdn.xxx.com/xxx.js`）。

2. **监控指标跟踪**

   - 错误监控：通过 Sentry 查看 JS 错误率、异常堆栈（15 分钟内无新增错误）。
   - 性能监控：通过 Lighthouse 或 DataDog 查看页面加载时间、资源大小是否符合预期。
   - 业务监控：核心指标（如登录成功率、支付转化率）无异常波动。

3. **文档更新**

   - 维护`CHANGELOG.md`，记录版本变更内容（新增功能、修复 bug、兼容说明）：

     ```markdown
     ## v2.3.1 (2024-05-20)

     ### Features

     - 新增用户头像裁剪功能

     ### Bug Fixes

     - 修复移动端下拉刷新时白屏问题

     ### Docs

     - 更新 API 文档中上传接口的参数说明
     ```

   - 通知相关团队（产品、运营、客服）版本变更内容。

## 三、紧急修复流程（Hotfix）

当生产环境出现严重 bug（如功能阻塞、数据异常），需走紧急修复流程：

1. **创建 hotfix 分支**  
   从主分支（`main`）切出修复分支：

   ```bash
   git checkout main
   git pull origin main
   git checkout -b hotfix/v2.3.2  # 基于当前生产版本号升级patch
   ```

2. **修复与测试**

   - 开发在`hotfix/*`分支修复问题，提交代码。
   - 构建后部署至测试环境，QA 执行针对性测试（重点验证修复点及关联功能）。

3. **合并与发布**
   - 重复「2.4 合并代码与打 Tag」「2.5 生产环境部署」流程，合并目标分支为`main`和`develop`。
   - 发布后重点监控修复效果，确认问题已解决。
