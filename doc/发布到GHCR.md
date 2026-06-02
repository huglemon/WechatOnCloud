# 发布到 GHCR

> 返回 [← README](../README.md)

把两个镜像（`woc-panel`、`wechat-on-cloud`）发布到 GitHub Container Registry，供他人 `docker compose up -d` 直接拉取。两种方式任选其一。

---

## 方式 A · GitHub Actions（推荐）

仓库自带 GitHub Actions（[.github/workflows/release.yml](../.github/workflows/release.yml)），在你**推送 `vX.Y.Z` 标签或发布 Release** 时，自动构建多架构（amd64+arm64）镜像并推到 GHCR：

```bash
git tag v1.0.0
git push origin v1.0.0     # 触发 Actions，产出 ghcr.io/<owner>/woc-panel:1.0.0 等标签
# 或在 GitHub 上 Publish 一个 Release（会额外打 latest）：
gh release create v1.0.0 --title v1.0.0 --notes "..."
```

> 注意：单纯 push tag 只产出 `X.Y.Z / X.Y / X`，**不会更新 `latest`**；要更新 `latest` 请改用 **发布 Release** 或在 Actions 里手动 `workflow_dispatch`。

---

## 方式 B · 本机 buildx 手动构建并推送（不走 Actions）

适合想立刻出包、或不依赖 CI 的场景。需要 Docker Buildx（Docker Desktop 自带；纯 Linux 跨架构需先装 QEMU：`docker run --privileged --rm tonistiigi/binfmt --install all`）。

```bash
# 1) 登录 GHCR（PAT 需 write:packages 权限）
echo <YOUR_GITHUB_PAT> | docker login ghcr.io -u <github 用户名> --password-stdin

# 2) 首次创建并启用多架构构建器（已建过改用 docker buildx use woc）
docker buildx create --name woc --use

# 3) 构建并推送两个镜像（amd64 + arm64）。VER 与 git tag 保持一致（不带 v）
VER=1.0.1
docker buildx build --platform linux/amd64,linux/arm64 \
  -t ghcr.io/gloridust/woc-panel:$VER -t ghcr.io/gloridust/woc-panel:latest \
  --push ./panel
docker buildx build --platform linux/amd64,linux/arm64 \
  -t ghcr.io/gloridust/wechat-on-cloud:$VER -t ghcr.io/gloridust/wechat-on-cloud:latest \
  --push ./docker
```

> 把 `gloridust` 换成你的 GHCR 命名空间（与 `docker-compose.yml` / `WOC_IMAGE_PREFIX` 一致）。
> 只想本机自用、不推 GHCR，用 [`./scripts/build-local.sh`](../scripts/build-local.sh) 构建本机架构单架构镜像即可。

---

## 发布后：把包设为公开

首次发布后还需把 GHCR 包设为公开，否则别人 `docker compose up -d` 会报 `denied`：

1. 打开 GitHub → 你的头像 → **Packages** → 分别进入 `woc-panel`、`wechat-on-cloud`；
2. **Package settings → Change visibility → Public**。

> 若想保持私有，则使用者需先 `docker login ghcr.io`（用具备 `read:packages` 的 PAT）才能拉取。
> 在镜像发布之前，本地用 [`./scripts/build-local.sh`](../scripts/build-local.sh) 自构建即可，无需等待发布。
