# 📚 Knowledge Vault

> 个人知识库 —— 用于收集、整理和归档各类学习资料、技术分享与笔记。

## 📂 目录结构

```
knowledge-vault/
├── README.md
└── 基于联邦实例的AI提效分享/
    ├── 从HDFS联邦开发看AI编程助手的落地实践-讲稿.md
    └── 从HDFS联邦开发看AI编程助手的落地实践.pptx
```

## 📖 内容索引

| 主题 | 说明 |
|------|------|
| [基于联邦实例的AI提效分享](./基于联邦实例的AI提效分享/) | 从 HDFS 联邦开发看 AI 编程助手的落地实践（含讲稿 & PPT） |

## 🚢 推送到远端服务器

使用 rsync 将项目推送到远端服务器，自动排除不需要同步的文件：

```bash
rsync -avz \
    --exclude='.git/' \
    --exclude='.github/' \
    --exclude='__pycache__/' \
    --exclude='*.pyc' \
    --exclude='venv' \
    /Users/ziwh666/GitHub/knowledge-vault \
    root@182.43.22.165:/data/github/
```

在远端服务器上拉取最新代码：

```bash
git fetch origin && git reset --hard origin/main
```

> 💡 **免密推送**：建议先配置 SSH 密钥认证，执行一次 `ssh-copy-id root@your-server-ip` 即可免密。
