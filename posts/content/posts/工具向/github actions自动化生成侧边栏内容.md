---
date: '2024-03-25'
draft: false
title: 'github actions自动化生成 docsify 侧边栏内容'
tags: ['github action', '方案', 'docsify']
archives: ['2024-03-25']
categories: ["工具向"]
---

## 需求

指定分支的推送代码后,检查指定目录下的内容是否有更改,如果有更改则运行`gen_sidebar.sh`脚本,生成侧边栏内容.

侧边栏内容前后顺序,根据md文件在git中的创建时间进行排序.

生成的文件,进行git push.

## 实现

### 1. 配置基础 github actions

```yaml
# .github/workflows/gen_sidebar.yml
on:
  push:
    branches: [ "gh-pages" ]
    paths:
      - 'docs/**/*.md'

jobs:
  gen_sidebar:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - name: Run generate shell
        run: bash scripts/gen_sidebar.sh
```

- `on.push.branches`指定分支
- `on.push.paths`指定目录
- `jobs.gen_sidebar.runs-on`指定运行环境
- `jobs.gen_sidebar.steps`指定步骤
- `jobs.gen_sidebar.steps.uses`指定使用的actions
- `jobs.gen_sidebar.steps.run`指定运行的命令
- `bash scripts/gen_sidebar.sh` 指定使用bash运行脚本, 如果使用sh,则可能出现不兼容问题

### 2. 过滤目录

```yaml
# .github/workflows/gen_sidebar.yml
on:
  push:
    branches: [ "gh-pages" ]
    paths:
      - 'docs/**/*.md'
      - '!docs/**/_*.md'    # 排除docs目录下的以 _ 开头的文件
      - '!docs/**/LS*.md'   # 排除docs目录下的以 LS 开头的临时文件
```

- `!docs/**/_*.md` 单个action只能使用paths和paths-ignore中的一个,使用`!`表示排除

### 3. 配置 GITHUB_TOKEN

```yaml
#.github/workflows/gen_sidebar.yml
jobs:
  gen_sidebar:
    runs-on: ubuntu-latest
    permissions:
      contents: write # 允许写入仓库内容
```

- `permissions` 指定权限,默认是`read`,需要写入仓库内容,需要指定`write`权限.

### 4. 配置 git 信息

```yaml
#.github/workflows/gen_sidebar.yml
jobs:
...
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # 0表示获取所有提交记录
      - name: Run generate shell
        run: bash scripts/gen_sidebar.sh
      - name: Run commit sidebar.md
        run: |
          git config --global user.name "GitHub Actions"
          git config --global user.email "actions@github.com"
```

- `with.fetch-depth` 指定获取提交记录的深度,0表示获取所有提交记录. `gen_sidebar.sh` 脚本中需要用到git的提交记录.
- `git config` 指定git的用户名和邮箱.

### 5. 配置 git push

```yaml
#.github/workflows/gen_sidebar.yml
jobs:
...
    steps:
      - name: Run commit sidebar.md
        run: |
          git config --global user.name "GitHub Actions"
          git config --global user.email "actions@github.com"
          git add docs/_sidebar.md

          if git diff --cached --quiet; then
            echo "No changes to commit"
          else
            git commit -m "Auto-update sidebar [skip ci]"
            git push
          fi
```

- `git add` 添加文件
- `git commit` 提交文件
- `git push` 推送文件
- `git diff --cached --quiet` 判断是否有更改

### 6. 完整代码

```yaml
#.github/workflows/gen_sidebar.yml
on:
  push:
    branches: [ "gh-pages" ]
    paths:
      - 'docs/**/*.md'
      - '!docs/**/_*.md'
      - '!docs/**/LS*.md'
  workflow_dispatch:

jobs:
  gen_sidebar:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Run generate shell
        run: bash scripts/gen_sidebar.sh
      - name: Run commit sidebar.md
        run: |
          git config --global user.name "GitHub Actions"
          git config --global user.email "actions@github.com"
          git add docs/_sidebar.md

          # 只有在有变更时才提交
          if git diff --cached --quiet; then
            echo "No changes to commit"
          else
            git commit -m "Auto-update sidebar [skip ci]"
            git push
          fi
```

## 运行情况

![GitHub Actions 运行记录](/img/工具向/github_action_record.png)

![GitHub Actions 执行情况](/img/工具向/github_action_runed.png)

![GitHub Actions 执行日志](/img/工具向/github_action_log.png)

## 总结

**🌈 核心功能**
1. **触发条件**：当 `gh-pages` 分支的 `docs/` 目录下 Markdown 文件发生变更时触发（排除以 `_` 或 `LS` 开头的文件）
2. **自动排序**：根据文件在 Git 中的创建时间排序生成侧边栏
3. **自动提交**：检测到侧边栏内容变更后自动提交并推送到仓库

**🌈 关键技术点**
1. **路径过滤**：
   - 使用 `paths` 和 `!` 排除特定模式文件
   - 精确控制只有文档内容变更时才触发
2. **Git 深度配置**：
   - `fetch-depth: 0` 获取完整提交历史
   - 确保能正确获取文件的创建时间
3. **安全推送**：
   - 设置 `contents: write` 权限
   - 配置 GitHub Actions 专用用户信息
   - 使用 `--cached --quiet` 检测实际变更
4. **优化提交**：
   - 只在有实际变更时提交

**🌈 使用效果**
- 完全自动化侧边栏生成和更新
- 变更检测准确，避免无效提交
- 与文档编写流程无缝集成
- 执行记录和日志清晰可查

**🌈 最佳实践建议**
1. 脚本应处理异常情况并返回适当退出码
2. 考虑添加执行超时限制
3. 重要操作可添加通知机制
4. 定期检查工作流执行效率

该方案实现了文档目录结构的自动化维护，显著降低了手动维护成本，确保了侧边栏与文档内容的实时同步。