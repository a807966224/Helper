# Windows SSH + GitHub 使用文档

## 1. 配置文件路径
```
C:\Users\<USERNAME>\.ssh\config
```

## 2. 推荐完整配置
```
Host github.com
    HostName github.com
    User git
    IdentityFile C:\Users\<USERNAME>\.ssh\id_rsa
    IdentitiesOnly yes
    Port 22
```

## 3. 测试 SSH 连接
```
ssh -T git@github.com
ssh -vvv git@github.com
```

## 4. 常见错误说明
### ❌ Could not resolve hostname github
原因：`Host github` 未指定 HostName 或拼写错误  
解决：改为 `Host github.com` 并加上 `HostName github.com`

### ❌ CreateProcessW failed error:2 / posix_spawnp
原因：SSH 可执行路径异常  
解决：确保下面路径优先：
```
C:\Windows\System32\OpenSSH\
```

## 5. 检查 SSH 版本与路径
```
where ssh
ssh -V
```

## 6. Git 配置 GitHub SSH
```
git config --global user.name "yourname"
git config --global user.email "you@example.com"
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

## 7. 生成 SSH KEY
```
ssh-keygen -t ed25519 -C "you@example.com"
```
生成文件：
```
C:\Users\<USERNAME>\.ssh\id_ed25519
C:\Users\<USERNAME>\.ssh\id_ed25519.pub
```

## 8. 添加公钥到 GitHub
打开：  
GitHub → Settings → SSH and GPG Keys → New SSH key  
粘贴 `id_ed25519.pub` 内容即可。

---

# nc.exe 下载
你可以从官方项目获取最新版：  
**https://nmap.org/ncat/**

如果你需要我帮你准备一个稳定版 nc.exe，我也可以提供。


🔹 克隆命令模板
1️⃣ 使用主账号（github）
git clone github:用户名/仓库名.git


示例：

git clone github:a807966224/2025-h2-solidity-native-polkadot-homework.git


使用这个命令时，Git 会自动使用 C:/Users/zxy80/.ssh/github 这个 Key。







在 WSL 中直接引用 Windows 的 SSH 文件（推荐，避免复制多份）

WSL 可以直接访问 Windows 文件系统：

Windows SSH 文件路径：C:\Users\zxy80\.ssh\

在 WSL 下路径：/mnt/c/Users/zxy80/.ssh/

操作步骤：

在 WSL 中创建 ~/.ssh/config，内容如下（引用 Windows 的 key）：

Host github
    HostName ssh.github.com
    User git
    Port 443
    IdentityFile /mnt/c/Users/zxy80/.ssh/github
    IdentitiesOnly yes