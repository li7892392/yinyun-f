# 上游同步与维护指南（my-artist-feature 分支）

## 1. 当前状态

| 项目 | 内容 |
|------|------|
| 本地分支 | `my-artist-feature` |
| 自定义提交 | `0a69a6e`（feat: 本地音乐按歌手浏览视图） |
| 基础 | 上游 `main`（作者仓库 bobcc4/yinyun-lxserver） |

该提交实现了「本地音乐按歌手浏览视图」，只改动以下 4 个文件：

| 文件 | 改动 |
|------|------|
| `src/server/server.ts` | 新增接口 `POST /api/v1/player/music/local/artists`，并新增一处 import |
| `public/music/index.html` | 歌手面板 DOM |
| `public/music/js/local_music.js` | 歌手视图逻辑 |
| `.dockerignore` | 排除 `*.bak` |

### 重要：这个提交只存在于你这台电脑上

`0a69a6e` 是**本地提交**，没有推送到任何远端——既不在你的 GitHub 上，也不在作者的仓库里。只要这台电脑的磁盘坏了，这个功能就没了。**除非你把分支推送到自己的 GitHub 仓库（见第 3 节），否则它没有任何备份。**

## 2. 同步作者的上游更新

作者以后发布新版本时，按以下步骤把你的功能“搬”到新代码上：

```bash
git checkout main            # 切到 main 分支
git pull origin main         # 拉取作者仓库的最新代码
git checkout my-artist-feature   # 切回你的功能分支
git rebase main              # 把你的提交重放到最新代码之上
```

**rebase 做了什么**：它把你的提交 `0a69a6e` 摘下来，先让分支指向最新的 `main`，再把你的提交重新应用上去。结果是你的功能始终“叠在”最新上游代码的顶部，历史保持线性。

**冲突**：如果作者也改了同一文件的同一处代码，rebase 会停下来报冲突。最可能冲突的文件是：

- `src/server/server.ts`（上游也常改这个文件）
- `public/music/js/local_music.js`

**解决冲突**：

1. 打开 git 提示的文件，找到 `<<<<<<<` / `=======` / `>>>>>>>` 标记。
2. 保留两边都需要的逻辑（你的歌手视图代码 + 上游的新改动），删掉标记。
3. 执行 `git add <文件>`，然后 `git rebase --continue`。
4. 如果搞砸了想全部放弃、回到 rebase 之前的状态：`git rebase --abort`。

**rebase 成功后必须重新构建镜像**（代码变了，Docker 镜像还是旧的）：

```bash
docker build -t yinyun-f:latest .
```

## 3. 备份：推送到你自己的 GitHub

为了防止磁盘故障丢失功能，建议创建自己的 GitHub 仓库并推送：

- 方式一：在 GitHub 上 fork `bobcc4/yinyun-lxserver`。
- 方式二：新建一个**私有仓库**（Private）。

然后：

```bash
git remote add myfork <你的仓库地址>   # 只需执行一次
git push myfork my-artist-feature     # 把功能分支推到你自己的仓库
```

**`origin` 与 `myfork` 的区别**：

| 远端 | 指向 | 用途 |
|------|------|------|
| `origin` | 作者仓库 bobcc4/yinyun-lxserver | 只用来 `pull` 上游更新 |
| `myfork` | 你自己的仓库 | 用来 `push` 备份你的分支 |

**警告：永远不要执行 `git push origin`**——那是往作者的仓库推代码，没有权限必然失败，且不是你想要的操作。

## 4. Docker 部署要点

- 镜像：`yinyun-f:latest`，用仓库自带的 `Dockerfile` 构建。
- `docker-compose.yml` 中把 `image: bobcc4/yinyun-lxserver:latest` 改为 `image: yinyun-f:latest`。
- volumes（`data` / `logs` / `cache` / `music`）不用动，现有数据全部保留。
- 每次 rebase + 代码变化后，重新执行 `docker build -t yinyun-f:latest .` 再重启容器。
- 也可以直接在 NAS 上 `git clone -b my-artist-feature https://github.com/li7892392/yinyun-f.git` 后在仓库目录执行 `docker build -t yinyun-f:latest .`，无需传 tar。

**迁移镜像到 NAS**（可选）：

```bash
docker save yinyun-f:latest -o yinyun-f.tar    # 导出
docker load -i yinyun-f.tar                    # 在 NAS 上导入
```

## 5. 自动构建（GitHub Actions）

仓库里有一个 `.github/workflows/docker-custom.yml`（独立文件，与上游的 `docker.yml` 互不干扰，同步上游不会冲突）。它做的事：**每次 `git push` 到 `main` 或 `my-artist-feature` 分支，GitHub 自动构建 Docker 镜像并发布到 ghcr.io**。也可以在 GitHub 网页上手动触发（Actions → Build Docker Image (fork) → Run workflow）。

构建成功后，镜像地址是：

```
ghcr.io/li7892392/yinyun-f:latest
```

### 5.1 一次性设置：把镜像包改为 Public

第一次 push 触发构建后，GitHub 会在你的账号下创建一个包（package），**默认是私有的**——私有包 NAS 拉取需要登录，麻烦。建议改成 Public（代码仓库本来就是公开的，镜像公开没有额外风险）：

1. 打开 GitHub，进入**你的 fork 仓库** `li7892392/yinyun-f` 页面。
2. 右侧栏找到 **Packages**，点开包 **yinyun-f**（注意是 ghcr.io 的包，不是 Docker Hub）。
3. 右下角 **Package settings**。
4. 拉到页面底部 **Danger Zone** → **Change visibility** → **Public**，输入包名确认。

改完后任何机器都能匿名 `docker pull` 这个镜像。

> 备选：如果坚持保持 Private，也可以，但 NAS 每次拉取前要先登录一次：
> `docker login ghcr.io -u li7892392`，密码用 GitHub 的 Personal Access Token（需要 `read:packages` 权限）。对新手来说没必要，推荐 Public。

### 5.2 新的更新流程（替代本机构建）

以后更新 NAS 上的服务只需三步：

```bash
# 1. 本机：提交并推送（推送到你的 fork）
git push myfork my-artist-feature

# 2. 等几分钟：GitHub Actions 自动构建（进度看仓库的 Actions 标签页，出现绿勾即完成）

# 3. NAS 上：拉新镜像并重启
docker pull ghcr.io/li7892392/yinyun-f:latest
docker compose up -d
```

第 3 步也可以在飞牛 fnOS 的 Docker 界面里做：找到容器使用的镜像，点**检查更新 / 拉取最新镜像**，然后重建容器即可，效果一样。

注意：fnOS 上 `docker-compose.yml` 里的 `image:` 要改成 `ghcr.io/li7892392/yinyun-f:latest`，volumes 等其他配置保持不变，数据全部保留。

### 5.3 回滚到某次具体的构建

每次构建还会打一个 `sha-开头` 的不可变标签，对应当次推送的 commit。万一新版本有问题：

```bash
# 去 GitHub 仓库 → Actions → 找到出问题那次构建，记下它的短 commit 号（如 sha-a1b2c3d）
docker pull ghcr.io/li7892392/yinyun-f:sha-a1b2c3d
# 把 docker-compose.yml 里 image: 的标签改成 sha-a1b2c3d，然后
docker compose up -d
```

### 5.4 旧的"NAS 上构建"流程（方案 B，备用）

自动构建不可用时（比如 GitHub Actions 排队、或想在 NAS 本地构建），旧流程仍然有效：

```bash
# NAS 上
git clone -b my-artist-feature https://github.com/li7892392/yinyun-f.git
cd yinyun-f
docker build -t yinyun-f:latest .
```

本机 `docker save` / `docker load` 传 tar 包的方式也依然可用（见第 4 节）。

