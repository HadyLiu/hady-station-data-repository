# Git 常用操作

> 适用：所有平台。以「常用 + 容易踩坑」为主，不追求覆盖全部子命令。

## 1. 初始配置

```bash
git -v
git config --global user.name  "<你的名字>"
git config --global user.email "<你的邮箱>"
git config --global credential.helper store     # 记住凭证，明文存 ~/.git-credentials

# 查看
git config --global --list
```

`credential.helper store` 是明文保存，安全性一般；更稳妥的是配 SSH key 或用 `manager-core`（Windows）。

## 2. 仓库基本流程

```bash
git init                  # 初始化
git clone <url>           # 克隆

git status                # 看工作区/暂存区状态
git add .                 # 全部加入暂存区
git add <file>            # 指定文件
git rm --cached <file>    # 从暂存区撤回（保留工作区文件）

git commit -m "说明"
git log --oneline --graph -20
git diff                  # 工作区 vs 暂存区
git diff --staged         # 暂存区 vs 上次提交
```

`.git` 目录结构：

| 路径 | 内容 |
| --- | --- |
| `.git/index` | 暂存区 |
| `.git/objects` | 本地对象库（提交、树、blob） |
| `.git/refs` | 分支与标签指针 |

删掉 `.git` 就等于放弃全部版本历史（`rm -rf .git`），慎用。

## 3. 远程

```bash
git remote -v
git remote add origin <url>
git remote set-url origin <url>    # 换地址（改域名、换协议）
git remote rm origin

git fetch origin --prune           # 拉取并清理已删除的远端分支引用
git pull --rebase                  # 变基拉取，避免无意义的 merge 提交
git push origin <分支名>
```

换仓库地址后如果仍提示旧账号，是凭证缓存问题：

```bash
git credential-manager 或删除 ~/.git-credentials 里对应行
```

## 4. 回退

```bash
git reset --soft HEAD^      # 撤销提交，保留改动在暂存区
git reset --mixed HEAD^     # 撤销提交和暂存，改动留在工作区（默认）
git reset --hard HEAD^      # 撤销提交并丢弃改动 ← 不可恢复
git reset --hard <commit>   # 回到指定提交

git revert <commit>         # 生成一个反向提交，已推送的分支用这个
```

已 `push` 过的历史不要用 `reset --hard` + 强推去改，会破坏别人本地仓库。

## 5. 删除历史中的大文件 / 敏感文件

```bash
# 需要 git-filter-repo（pip install git-filter-repo）
git filter-repo --path <路径> --invert-paths

git push origin --force --all
git push origin --force --tags
```

**注意**：rewrite 只是让旧对象在新历史里不可达，GitHub 上仍可能通过 commit hash 访问一阵子。真正的密钥泄漏必须**先去吊销/更换密钥**，再清历史。

## 6. SSH 密钥

```bash
ssh-keygen -t rsa -b 4096        # 交互里可指定文件名和 passphrase
cat ~/.ssh/<名字>.pub            # 把这段贴到 GitHub → Settings → SSH keys
ssh -T git@github.com            # 测试
```

或直接用 CLI 登录（会走浏览器 OAuth）：

```bash
gh auth login
```

## 7. 鉴权失败排查

```bash
unset GIT_ASKPASS
unset SSH_ASKPASS
```

图形界面的 askpass 弹窗卡住或密码永远错时，清掉这两个变量再试。

## 8. TLS 证书路径失效（换过 Git 安装位置）

换过 Git 安装目录后，`http.sslCAInfo` 指向的 `ca-bundle.crt` 不存在，报 `unable to get local issuer certificate`：

```bash
git config --global --unset http.sslCAInfo
git config --system --unset http.sslCAInfo

# 重新指定新路径
git config --global http.sslCAInfo "<Git安装路径>/mingw64/etc/ssl/certs/ca-bundle.crt"
```

`--system` 那层需要管理员权限。

## 9. 坑

| 现象 | 原因 |
| --- | --- |
| `fatal: not a git repository` | 不在仓库目录，或 `.git` 被删 |
| 提交显示旧用户名 | 仓库级 `git config user.name` 覆盖了 global |
| `upstream branch ... does not match` | 本地分支无跟踪关系：`git push -u origin <分支>` |
| 中文文件名显示为 `\344\275\240` | `git config --global core.quotepath false` |
| 换行符导致整文件 diff | `core.autocrlf` 设置与团队不一致，统一用 `.gitattributes` |
