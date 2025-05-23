---
date: '2024-03-25'
draft: false
title: 'docsify 部署配置'
tags: [ 'docsify', 'git 配置', '部署方案']
archives: ['2024-03-25']
categories: ["docsify"]
---

## docsify 初始化

[docsify 官方文档](https://docsify.js.org/#/zh-cn/)

```bash
# 安装 docsify-cli 全局
npm install -g docsify-cli
```

```bash
# 初始化项目, 会生成 docs 目录
docsify init ./docs
```

```bash
# 运行项目预览效果
docsify serve docs
```

## 常用配置

**网站favicon**

favicon.ico 图片文件放置在 `docs` 目录下

```
docs/
|--- favicon.ico
|--- index.html
...
```

**配置文件**

```html
<!-- docs/index.html -->
<!DOCTYPE html>
<html lang="zh-cn">

<head>
  <meta charset="UTF-8">
  <title>网站标题</title>
  <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1" />
  <meta name="description" content="Description">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0">
  <link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/vue.css">
</head>
<body>
  <div id="app">Please wait...</div>

  <script>
    window.$docsify = {
      // 仓库地址
      repo: '账号名/项目名',
      name: '侧边栏标题',
      // 自动跳转到顶部
      auto2top: true,
      // 自动添加标题
      autoHeader: true,
      // 加载侧边栏
      loadSidebar: true,
      // 加载导航栏
      loadNavbar: true,
      // 侧边栏目录折叠
      subMaxLevel: 4,
      // 分页
      pagination: {
        previousText: "上一页",
        nextText: "下一页",
        crossChapter: true,
        crossChapterText: true,
      },
    }
  </script>
  <script src="//cdn.jsdelivr.net/npm/docsify/lib/docsify.min.js"></script>

  <!-- 代码高亮 -->
  <script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-bash.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-python.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-cmake.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-java.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-csharp.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-docker.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-powershell.min.js"></script>

  <!-- 图片放大缩小 -->
  <script src="https://cdn.jsdelivr.net/npm/docsify@4/lib/plugins/zoom-image.min.js"></script>

  <!-- 字数统计 -->
  <script src="https://cdn.jsdelivr.net/npm/docsify-count@latest/dist/countable.min.js"></script>

  <!-- 侧边栏目录折叠 -->
  <script src="https://cdn.jsdelivr.net/npm/docsify-sidebar-collapse/dist/docsify-sidebar-collapse.min.js"></script>

  <!-- 分页 -->
  <script src="//cdn.jsdelivr.net/npm/docsify-pagination/dist/docsify-pagination.min.js"></script>

</body>

</html>
```

## 脚本

### 生成侧边栏目录

**使用 docsify generate 生成**

可以使用 `docsify generate ./docs` 命令生成侧边栏目录

```bash
docsify generate <path>

docsify 的生成器

全局选项
  --help, -h     帮助                              [布尔]
  --version, -v  当前版本号                         [布尔]

Options:
  --sidebar, -s  gen.sidebar
```

**注意**

- 默认生成的侧边栏文件名为 `_sidebar.md`
- 目录下不存在 `_sidebar.md` 文件时, 才能生成,否则会提示`The sidebar file '_sidebar.md' already exists.`
- 对于目录或文件名中包含空格的情况, 生成器并没有处理, 可能导致生成的侧边栏文件无法使用

**使用自定义脚本生成**

```bash
#!/bin/bash

# 输出文件
OUTPUT_FILE="docs/_sidebar.md"
# 文档根目录
DOCS_DIR="docs"

# 清空或创建输出文件
> "$OUTPUT_FILE"

# 遍历根目录
traverse_dir() {
    local dir=$1
    local level=$2
    local indent=""
    # 根据层级生成缩进
    for ((i=1; i<level; i++)); do
        indent="$indent  "
    done
    
    for item in "$dir"/*; do
        # 获取文件/目录名
        local name=$(basename "$item")

        # 跳过根目录本身的md文件
        if [[ "$dir" == "$DOCS_DIR" && "$name" == *.md ]]; then
            continue
        fi

        # 检查路径是否有空格
        if [[ "$item" == *' '* ]]; then
            # 替换空格为-，并反写回去
            new_item=$(echo "$item" | sed 's/ /-/g')
            mv "$item" "$new_item"

            name=$(basename "$new_item")
            item="$new_item"
        fi

        # 如果是目录
        if [[ -d "$item" ]]; then
            echo "\n${indent}- $name" >> "$OUTPUT_FILE"
            traverse_dir "$item" $((level+1))
        # 如果是.md文件
        elif [[ -f "$item" && "$name" == *.md ]]; then
            # 获取相对根目录的路径
            local relative_path="${item#$DOCS_DIR/}"
            relative_path=$(replace_filename "$relative_path")
            echo "${indent}- [${name%.md}]($relative_path)" >> "$OUTPUT_FILE"
        fi
    done
}

# 替换url中的空格和冒号
replace_filename() {
    local keys=(" " ":")
    local values=("%20" "%3A")

    local input="$1"
    local result="$input"

    for i in "${!keys[@]}"; do
        result=${result//${keys[i]}/${values[i]}}
    done
    echo "$result"
}

# 开始遍历
traverse_dir "$DOCS_DIR" 1

echo "目录已生成到 $OUTPUT_FILE"
```

**文档内容**

``` bash
# 根目录 docs, 目录结构
docs/
|--- XXX/
     |--- M1.md
     |--- M2.md
|--- XXXX/
     |--- M3.md
     |--- M4.md
```

```markdown
# docs/_sidebar.md

- XXX
  - [M1](XXX/M1.md)
  - [M2](XXX/M2.md)

- XXXX
  - [M3](XXXX/M3.md)
  - [M4](XXXX/M4.md)
```

### git hooks

<!-- `git commit` 操作时会触发 `.git/hooks/pre-commit` 钩子或者指定的 `.git/config` 中配置的 `core.hooksPath` 的钩子 -->

`git commit` 执行时, 会检查 `.git/hooks/` 目录或者 `.git/config` 的 `core.hooksPath`配置项对应目录下的钩子成功执行完所有钩子后, 才会继续执行 `git commit` 命令

```plain
+-------------------+
| 修改代码           | <- 开发者在本地修改代码
+-------------------+
          |
          v
+-------------------+
| git add           | <- 将修改的文件添加到暂存区
+-------------------+
          |
          v
+-------------------+
| git commit        | <- 开发者执行提交命令
+-------------------+
          |
          v
+-------------------+
| pre-commit hooks  | <- git commit 触发 pre-commit hooks
| 检查代码格式、      |
| 静态分析、测试等     |
+-------------------+
          |
    +-----+-----+
    |           |
    v           v
+---+---+   +---+---+
| 通过   |   | 失败   |
+---+---+   +---+---+
    |
    v
+---+-----+  
| 提交成功 |  <- git commit 继续执行
+----+----+  
```

**生成侧边栏的脚本**

```bash
#!/bin/sh
# 检查是否有新文件被暂存
if ! git diff --cached --name-only --exit-code; then
    echo "检测到 git add 操作，正在生成 _sidebar.md..."
    cd $(git rev-parse --show-toplevel)  # 切换到项目根目录
    sh scripts/gen_sidebar.sh           # 执行生成侧边栏的脚本
    git add docs/_sidebar.md             # 添加生成的侧边栏文件
fi
```

**维护 Git Hooks 配置项**

```bash
#!/bin/bash
git config core.hooksPath scripts/hooks  # Git 2.9+ 直接指定 Hook 目录, scripts/hooks 目录为自定义 Hook 目录
chmod -R +x scripts/hooks/  # 确保所有脚本可执行
echo "Git hooks installed successfully!"
```

