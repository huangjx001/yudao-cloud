# Jenkins + Harbor 自动化（开箱即用版）

这个方案的目标是：**代码提交后，Jenkins 自动构建 -> 镜像推送到 Harbor -> 在目标机自动拉取并重启容器**。

## 1. 你会得到什么

- `script/ci/Jenkinsfile.harbor`：可直接用于 Pipeline 的 Jenkinsfile 模板
- 支持参数化构建（选择模块、镜像名、tag）
- 支持 Harbor 推送
- 支持远程主机从 Harbor 拉取并部署

## 2. 前置准备

### Jenkins 插件

建议至少安装：

- Pipeline
- Credentials Binding
- SSH Agent（或 SSH Credentials 相关插件）
- Workspace Cleanup

### Jenkins 节点环境

执行构建的节点上要有：

- JDK 17+
- Maven 3.8+
- Docker CLI（并可访问 Docker daemon）
- 到 Harbor 的网络连通性

### Harbor 准备

1. 创建 Harbor 项目（如：`yudao`）
2. 准备一个可推拉镜像账号（建议机器人账号）
3. 记录 Harbor 域名（如：`harbor.example.com`）

### Jenkins 凭据

在 Jenkins `Manage Credentials` 里创建：

1. `harbor-credentials`（Username with password）
   - 用户名：Harbor 用户名或机器人用户名
   - 密码：对应密码或 token
2. `deploy-ssh`（SSH Username with private key，可选）
   - 用于远程部署阶段 SSH 登录

> 如果你只想推送镜像，不想自动部署，把 `DEPLOY_ENABLED` 设为 `false` 即可。

## 3. 使用方式

### 方式 A：Multibranch/SCM Pipeline（推荐）

1. 把 `script/ci/Jenkinsfile.harbor` 提交到仓库
2. Jenkins 新建 Pipeline Job
3. Pipeline script from SCM
4. Script Path 填：`script/ci/Jenkinsfile.harbor`

### 方式 B：直接复制 Jenkinsfile

把 `Jenkinsfile.harbor` 内容复制到 Jenkins 的 Pipeline Script 文本框。

## 4. 关键参数说明

- `MODULE_PATH`：要构建的模块路径，目录内必须有 Dockerfile
- `IMAGE_NAME`：镜像名（不含 Harbor 前缀）
- `IMAGE_TAG`：留空会自动生成 `BUILD_NUMBER-GIT_SHORT_SHA`
- `DEPLOY_ENABLED`：是否启用部署（拉取镜像并重启容器）
- `RUN_ARGS`：`docker run` 额外参数（端口、环境变量、挂载等）

## 5. 一次完整流程示例

例如你构建 `yudao-server`：

1. Jenkins 执行 Maven 打包
2. Docker 在 `yudao-server/` 目录构建镜像
3. 推送到：
   `harbor.example.com/yudao/yudao-service:123-abcd123`
4. 远程服务器执行：
   - `docker login harbor`
   - `docker pull` 新镜像
   - 停掉旧容器
   - `docker run` 新容器

## 6. 常见问题（友好排障）

1. **`docker login` 失败**
   - 检查 Harbor 地址是否带了错误协议前缀
   - 检查 Jenkins 凭据 ID 是否和 Jenkinsfile 一致

2. **`docker push` 权限不足**
   - Harbor 项目里确认该账号有 push 权限
   - 机器人账号是否过期/被禁用

3. **远程部署卡在 SSH**
   - 确认 Jenkins 节点能连通 `DEPLOY_HOST:DEPLOY_PORT`
   - 确认 `deploy-ssh` 私钥和用户名正确

4. **容器启动后立刻退出**
   - 查看目标机容器日志：`docker logs <容器名>`
   - 检查 `RUN_ARGS` 中端口和环境变量是否正确

## 7. 建议的生产实践

- 镜像 tag 用 `分支名-构建号-短SHA`，避免覆盖
- Harbor 开启镜像保留策略
- Jenkins 构建失败时通过企业微信/钉钉/飞书通知
- 部署阶段改造成蓝绿或滚动发布，减少中断

---

如果你愿意，我可以下一步再给你一版：

- **按分支自动区分 dev/test/prod Harbor 项目与部署主机**
- **支持 K8s（Deployment）而不是 docker run**
- **支持并行构建多个模块镜像**
