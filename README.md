# agent-design-analysis

智能体设计与分析工程。

## 目录结构

```
├── codes/     # 参考代码（submodule）
├── docs/      # 文档
└── README.md
```

## 克隆仓库

本项目包含 git submodule，克隆时请使用：

```bash
git clone --recurse-submodules <repo-url>
```

如果已克隆但未拉取 submodule，执行：

```bash
git submodule update --init --recursive
```

## 子模块

| 路径 | 来源 |
|------|------|
| `codes/acpx` | https://github.com/openclaw/acpx |
| `codes/agentapi` | https://github.com/coder/agentapi |
| `codes/AionUi` | https://github.com/iOfficeAI/AionUi |
| `codes/claudecodeui` | https://github.com/siteboon/claudecodeui |
| `codes/jetbrains-cc-gui` | https://github.com/zhukunpenglinyutong/jetbrains-cc-gui |
| `codes/lobehub` | https://github.com/lobehub/lobehub |
| `codes/Proma` | https://github.com/ErlichLiu/Proma |
