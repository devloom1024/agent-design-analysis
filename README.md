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
| `codes/jetbrains-cc-gui` | https://github.com/zhukunpenglinyutong/jetbrains-cc-gui |
