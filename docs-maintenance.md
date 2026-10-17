# 独立维护与手动升级

维护仓库：https://github.com/muyichun/Xboard-Node ，维护分支：`muyichun`。
`origin` 指向自己的仓库，`upstream` 仅用于查看官方更新；不自动合并。
GitHub 的 fork 关联和原有 Go module/import 名称保留，不影响独立发布。

首个固定版本：`v1.13.1-muyichun.1`，包含 `46d1475` 在线设备快照续期修复。
该修复避免同 IP 长连接的设备记录过期，不改变流量字节统计算法。

## 已安装的 systemd 节点：首次从官方迁移

测试机和生产机使用同一套步骤。先在测试机执行并观察，再逐台升级生产机。
下面操作适用于官方脚本安装的 `/usr/local/bin/xboard-node`、`xbctl` 和
`/etc/xboard-node/config.yml`，不适用于 Docker 容器。升级会短暂重启节点。
不要先运行旧版 `xbctl upgrade`，旧程序内置的下载地址仍然是官方仓库。

在节点服务器以 root 执行以下 Bash 命令。下载在全新目录内进行，避免脚本误选当前目录中的旧二进制。

```bash
sudo bash
set -e
NODE_VERSION=v1.13.1-muyichun.1
NODE_WORKDIR=$(mktemp -d /tmp/xboard-node-upgrade.XXXXXX)
cd "$NODE_WORKDIR"
case "$(uname -m)" in
  x86_64|amd64) NODE_ARCH=amd64 ;;
  aarch64|arm64) NODE_ARCH=arm64 ;;
  *) echo "不支持的架构"; exit 1 ;;
esac
NODE_RELEASE="https://github.com/muyichun/Xboard-Node/releases/download/$NODE_VERSION"
for asset in install.sh SHA256SUMS "xboard-node-linux-$NODE_ARCH" "xbctl-linux-$NODE_ARCH"; do
  curl -fL --retry 3 "$NODE_RELEASE/$asset" -o "$asset"
done
# 只校验当前架构的三个文件；任一缺失或校验失败即停止。
for asset in install.sh "xboard-node-linux-$NODE_ARCH" "xbctl-linux-$NODE_ARCH"; do
  awk -v f="$asset" '$2 == f {print}' SHA256SUMS > "$asset.sha256"
  test -s "$asset.sha256"
  sha256sum -c "$asset.sha256"
done
chmod +x "xboard-node-linux-$NODE_ARCH" "xbctl-linux-$NODE_ARCH"
./"xboard-node-linux-$NODE_ARCH" -v
./"xbctl-linux-$NODE_ARCH" version

# 下载、校验完成后，下面这一条才真正升级并重启服务。
bash ./install.sh upgrade --version "$NODE_VERSION" \
  --binary "$NODE_WORKDIR/xboard-node-linux-$NODE_ARCH" \
  --xbctl-binary "$NODE_WORKDIR/xbctl-linux-$NODE_ARCH"

/usr/local/bin/xboard-node -v
/usr/local/bin/xbctl version
systemctl is-active xboard-node
journalctl -u xboard-node -n 50 --no-pager
exit
```

安装脚本保留原有配置、凭据及绑定关系，在 `/etc/xboard-node/backups/时间戳/`
备份原二进制、配置与服务文件，并在升级失败时尝试回滚。新版本的 `xbctl` 已指向自己的 Releases。
成功后需保持客户端连接并持续传输超过五分钟，确认面板在线设备数仍正常、流量继续上报。
本次构建不等于该在线场景已经验证；需要升级测试机后完成。

## 手动回滚

先查看 `/etc/xboard-node/backups/`，选择本次升级前的目录，不要随意选择其他历史备份。
以下命令只回滚二进制（本次升级不改变配置），以 root 在 Bash 中执行：

```bash
set -e
NODE_BACKUP=/etc/xboard-node/backups/替换为本次备份目录
 test -x "$NODE_BACKUP/xboard-node"
 test -x "$NODE_BACKUP/xbctl"
systemctl stop xboard-node
install -m 755 "$NODE_BACKUP/xboard-node" /usr/local/bin/xboard-node
install -m 755 "$NODE_BACKUP/xbctl" /usr/local/bin/xbctl
systemctl start xboard-node
systemctl is-active xboard-node
```

## 后续升级和新节点

迁移完成后可以通过自己的更新源升级，建议指定已经测试的固定版本：

```bash
sudo xbctl upgrade --version v1.13.1-muyichun.1
```

不传 `--version` 时下载自己仓库的 latest Release。安装脚本也是如此。
Xboard 后台生成的安装命令已改为 `muyichun/Xboard-Node/muyichun/install.sh`。
按原流程在后台创建服务器并复制安装命令即可。此来源调整需要部署新版 Xboard 镜像后才会出现在页面上。
脚本和 `xbctl` 中的 Release 下载源均为 `muyichun/Xboard-Node`。

## 构建与发布

要求 Go 版本满足 go.mod（目前 1.26）。amd64 和 arm64 分别构建，不混用。

```bash
git switch muyichun
go test -race ./internal/...
make build-all VERSION=v1.13.1-muyichun.1
```

CI 在 `muyichun` 分支上测试和构建，只有推送 `v*` 标签才发布 Release 和 Docker 镜像。
Release 包含两个架构的节点、xbctl、install.sh 和 SHA256SUMS。版本标签不能重用；
后续版本应新建标签，例如 `v1.13.1-muyichun.2`，确认内容后再推送。

```bash
git push origin muyichun
git tag -a v1.13.1-muyichun.2 -m 'Describe the tested changes'
git push origin v1.13.1-muyichun.2
```

如需参考官方更新，先 `git fetch upstream`，阅读差异，再选择提交 cherry-pick 到 `muyichun`，测试后发布。
不要将上游 `dev` 直接覆盖到维护分支，也不要使用官方 Release 更新自己的节点。
