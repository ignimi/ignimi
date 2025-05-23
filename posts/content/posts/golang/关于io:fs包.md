---
date: '2024-04-23T16:30:06+08:00'
draft: false
title: '关于io:fs包'
tags: ['go','go io:fs包']
archives: ['2024-04-23']
categories: ["Golang"]
---

## 引言

Go 1.16 引入了 io/fs 包,提供了一组接口和实用函数,用于在 Go 程序中操作文件系统。本文将介绍 io/fs 包的核心接口和实用函数,以及如何使用它们来操作文件系统。

**抽象文件系统操作**
- 提供了一组通用接口来抽象文件系统操作
- 允许代码不直接依赖于具体的文件系统实现（如本地文件系统、内存文件系统、网络文件系统等）

**支持嵌入文件系统**
- 特别为 Go 1.16 引入的 //go:embed 指令设计

**统一接口**
- 定义了 fs.FS 接口作为最小文件系统接口
- 其他文件系统操作可以基于这个基本接口构建

**测试友好**
- 可以轻松创建内存中的模拟文件系统进行测试
- 不需要操作真实的文件系统

## 接口和函数

**核心接口**
- `fs.FS` 最基础的接口,表示一个只读的文件系统树。任何实现了Open方法的类型都可以被视为一个文件系统
- `fs.File` 表示一个打开的文件,提供了读取文件内容和获取文件信息的基本操作
- `fs.FileInfo` 提供文件的基本信息,与os.FileInfo接口相同
- `fs.DirEntry` 表示目录中的一个条目,用于高效地遍历目录

**扩展接口**
- `fs.GlobFS` 提供了 Glob 方法,用于匹配文件路径
- `fs.ReadDirFS` 提供了 ReadDir 方法,用于读取目录中的条目
- `fs.ReadDirFile` 提供了 ReadDir 方法,用于读取目录中的条目
- `fs.ReadFileFS` 提供了 ReadFile 方法,用于读取文件内容
- `fs.StatFS` 提供了 Stat 方法,用于获取文件信息

**实用函数**
- `fs.FormatDirEntry` 返回一个格式化的 dir 版本,以便于阅读
- `fs.FormatFileInfo` 返回一个格式化的 info 版本, 文件的元数据
- `fs.Glob` 返回所有与 pattern 匹配的文件的名称,如果没有匹配的文件,则返回 nil
- `fs.ReadDir` 读取目录中的条目,返回一个 DirEntry 切片
- `fs.ReadFile` 从文件系统 fs 读取指定文件并返回其内容。成功调用将返回 nil 错误,而不是io.EOF
- `fs.Stat` 获取文件信息,返回一个 FileInfo 对象
- `fs.ValidPath` 报告给定的路径名​​是否可用于调用 Open
- `fs.WalkDir` 遍历 fs 中的文件树,为每个文件调用 walkFn

**错误枚举**
```go
var (
 	ErrInvalid = errInvalid()       // 参数无效
 	ErrPermission = errPermission() // 权限被拒绝
 	ErrExist = errExist()           // 文件已存在
 	ErrNotExist = errNotExist()     // 文件不存在
 	ErrClosed = errClosed()         // 文件已关闭
 )
```

## 应用示例

### fs.FS 接口

```go
package main

import (
	"io/fs"
	"os"
)

func main() {
	// 使用本地文件系统
	var localFS fs.FS = os.DirFS("/path/to/dir")

	// 打开文件
	file, err := localFS.Open("example.txt")
	if err != nil {
		panic(err)
	}
	defer file.Close()

	// 读取文件内容...
}
```
- `os.DirFS` 创建一个本地文件系统的根目录. `os.DirFS` 实现了 `fs.FS` 接口.

```go
// os/file.go
package os

func DirFS(dir string) fs.FS {
	return dirFS(dir)
}

type dirFS string

func (dir dirFS) Open(name string) (fs.File, error) {
	fullname, err := dir.join(name)
	if err != nil {
		return nil, &PathError{Op: "open", Path: name, Err: err}
	}
	f, err := Open(fullname)
	if err != nil {
		err.(*PathError).Path = name
		return nil, err
	}
	return f, nil
}
```

### fs.File, fs.FileInfo 接口
```go
package main

import (
	"fmt"
	"io/fs"
	"os"
)

func main() {
	// 使用本地文件系统
	var localFS fs.FS = os.DirFS("/path/to/dir")

	// 打开文件
	file, err := localFS.Open("example.txt")
	if err != nil {
		panic(err)
	}
	defer file.Close()

	// 获取文件信息
	fInfo, err := file.Stat()
	if err != nil {
		panic(err)
	}
	fmt.Println(fInfo.Name())
}
```
- `localFS.Open` 返回的对象实现了 `fs.File`.
- 通过 `file.Stat()` 获取文件信息,包括文件名,大小,修改时间等信息.

```go
// io/fs/fs.go
type FileInfo interface {
	Name() string       // base name of the file
	Size() int64        // length in bytes for regular files; system-dependent for others
	Mode() FileMode     // file mode bits
	ModTime() time.Time // modification time
	IsDir() bool        // abbreviation for Mode().IsDir()
	Sys() any           // underlying data source (can return nil)
}
```

### fs.DirEntry 接口

```go
package main

import (
	"io/fs"
	"os"
)

func main() {
	// 使用本地文件系统
	var localFS fs.FS = os.DirFS("/path/to/dir")

	// 读取目录下的文件
	entries, err := fs.ReadDir(localFS, ".")
	if err != nil {
		panic(err)
	}

	for _, entry := range entries {
		info, err := entry.Info()
		if err != nil {
			panic(err)
		}
		println(info.Name())
	}
}
```
- `fs.ReadDir` 函数返回一个 `DirEntry` 切片,包含目录下的所有文件和子目录.
- `DirEntry` 接口提供了一些方法,用于获取文件的基本信息,包括文件名,是否是目录等.

```go
// io/fs/fs.go
type DirEntry interface {
	Name() string
	IsDir() bool
	Type() FileMode
	Info() (FileInfo, error)
}

// io/fs/readdir.go
func ReadDir(fsys FS, name string) ([]DirEntry, error) {
	if fsys, ok := fsys.(ReadDirFS); ok {
		return fsys.ReadDir(name)
	}

	file, err := fsys.Open(name)
	if err != nil {
		return nil, err
	}
	defer file.Close()

	dir, ok := file.(ReadDirFile)
	if !ok {
		return nil, &PathError{Op: "readdir", Path: name, Err: errors.New("not implemented")}
	}

	list, err := dir.ReadDir(-1)
	slices.SortFunc(list, func(a, b DirEntry) int {
		return bytealg.CompareString(a.Name(), b.Name())
	})
	return list, err
}
```

### fs.GlobFS 接口,fs.Glob 函数
```go
package main

import (
	"io/fs"
	"os"
)

func main() {
	// 使用本地文件系统
	var localFS fs.FS = os.DirFS("/path/to/dir")
	// 模式匹配
	globFs, err := fs.Glob(localFS, "*.txt")
	if err != nil {
		panic(err)
	}
	for _, f := range globFs {
		println(f)
	}
}
```

```go
// io/fs/glob.go
type GlobFS interface {
	FS

	Glob(pattern string) ([]string, error)
}

func Glob(fsys FS, pattern string) (matches []string, err error) {
	return globWithLimit(fsys, pattern, 0)
}

// This limit is added to prevent stack exhaustion issues. See
// CVE-2022-30630.
func globWithLimit(fsys FS, pattern string, depth int) (matches []string, err error)
```
- `GlobFS` 接口扩展了 `FS` 接口,添加了 `Glob` 方法
- `fs.Glob` 函数接受一个 `fs.FS` 对象和一个模式字符串作为参数,返回一个字符串切片,包含所有与给定模式匹配的文件的名称.
- 可以断言 `fs.FS` 对象为 `GlobFS` 接口,以便调用 `Glob` 方法.

### fs.ReadDirFS,fs.ReadDirFile,fs.ReadFileFS 接口,fs.ReadDir,fs.ReadFile 函数

```go
package main

import (
	"io/fs"
	"os"
)

func main() {
	// 使用本地文件系统
	var localFS fs.FS = os.DirFS("/path/to/dir")

	// 读取目录
	if fsys, ok := localFS.(fs.ReadDirFS); ok {
		entries, err := fsys.ReadDir(".")
		if err != nil {
			panic(err)
		}
		for _, entry := range entries {
			println(entry.Name())
		}
	}
}
```

```go
// io/fs/readdir.go
type ReadDirFS interface {
	FS
	ReadDir(name string) ([]DirEntry, error)
}
func ReadDir(fsys FS, name string) ([]DirEntry, error)
```
- `ReadDirFS` 接口扩展了 `FS` 接口,添加了 `ReadDir` 方法
- `fs.ReadDir` 函数接受一个 `fs.FS` 对象和一个目录名作为参数,返回一个 `DirEntry` 切片,包含所有与给定目录匹配的文件的名称.
- 可以断言 `fs.FS` 对象为 `ReadDirFS` 接口,以便调用 `ReadDir` 方法.

```go
// io/fs/readfile.go
type ReadFileFS interface {
	FS
	ReadFile(name string) ([]byte, error)
}

func ReadFile(fsys FS, name string) ([]byte, error)
```
- `ReadFileFS` 接口扩展了 `FS` 接口,添加了 `ReadFile` 方法
- `fs.ReadFile` 函数接受一个 `fs.FS` 对象和一个文件名作为参数,返回一个字节切片,包含文件的内容.
- 可以断言 `fs.FS` 对象为 `ReadFileFS` 接口,以便调用 `ReadFile` 方法.

```go
// io/fs/fs.go
type ReadDirFile interface {
	File
	ReadDir(n int) ([]DirEntry, error)
}
```
- `ReadDirFile` 接口扩展了 `File` 接口,添加了 `ReadDir` 方法
- `ReadDir` 方法接受一个整数参数,表示要读取的目录条目数.
- 如果参数为负数,则表示读取所有目录条目.
- 返回一个 `DirEntry` 切片,包含所有与给定目录匹配的文件的名称.
- 可以断言 `File` 对象为 `ReadDirFile` 接口,以便调用 `ReadDir` 方法.

### fs.StatFS 接口,fs.Stat 函数

```go
// io/fs/stat.go
type StatFS interface {
	FS
	Stat(name string) (FileInfo, error)
}

func Stat(fsys FS, name string) (FileInfo, error)
```
- `StatFS` 接口扩展了 `FS` 接口,添加了 `Stat` 方法
- `fs.Stat` 函数接受一个 `fs.FS` 对象和一个文件名作为参数,返回一个 `FileInfo` 对象,包含文件的基本信息.
- 可以断言 `fs.FS` 对象为 `StatFS` 接口,以便调用 `Stat` 方法.

### fs.FormatDirEntry 函数

```go
package main

import (
	"fmt"
	"io/fs"
	"os"
)

func main() {
	entries, _ := os.ReadDir(".") // os.DirEntry 实现了 fs.DirEntry
	for _, entry := range entries {
		fmt.Println(fs.FormatDirEntry(entry))
	}
}
```

```plain
- main.go
d test/
```
- `fs.FormatDirEntry` 函数接受一个 `fs.DirEntry` 对象作为参数,返回一个格式化的字符串.便于阅读.

### fs.FormatFileInfo 函数

```go
package main

import (
	"io/fs"
	"os"
)

func main() {
	// 使用本地文件系统
	var localFS fs.FS = os.DirFS(".")

	dirs, err := fs.ReadDir(localFS, ".")
	if err != nil {
		panic(err)
	}
	for _, dir := range dirs {
		info, err := dir.Info()
		if err != nil {
			panic(err)
		}
		println(fs.FormatFileInfo(info))
	}
}
```
```plain
-rw-r--r-- 481 2024-05-01 13:08:09 main.go
drwxr-xr-x 96 2024-05-01 13:04:04 test/
```
- `fs.FormatFileInfo` 函数接受一个 `fs.FileInfo` 对象作为参数,返回一个格式化的字符串.
- 确保不同文件系统实现有相同的显示格式

### fs.ValidPath 函数

```go
// io/fs/fs.go
func ValidPath(name string) bool {
	if !utf8.ValidString(name) {
		return false
	}

	if name == "." {
		// special case
		return true
	}

	// Iterate over elements in name, checking each.
	for {
		i := 0
		for i < len(name) && name[i] != '/' {
			i++
		}
		elem := name[:i]
		if elem == "" || elem == "." || elem == ".." {
			return false
		}
		if i == len(name) {
			return true // reached clean ending
		}
		name = name[i+1:]
	}
}
```
- 不检查路径是否存在，仅检查格式是否合法
- 具体限制使用的字符查看 `fs.ValidPath` 的实现
- 预处理,在调用 Open 等文件系统方法前确保路径合法
- 安全检查,在处理用户提供的路径前进行验证

### fs.WalkDir 函数

```go
package main

import (
	"io/fs"
	"os"
)

func main() {
	// 使用本地文件系统
	var localFS fs.FS = os.DirFS(".")

	err := fs.WalkDir(localFS, "test", func(path string, d fs.DirEntry, err error) error {
		println(path)
		return nil
	})
	if err != nil {
		panic(err)
	}
}
```

```go
// io/fs/walk.go
func WalkDir(fsys FS, root string, fn WalkDirFunc) error {
	info, err := Stat(fsys, root)
	if err != nil {
		err = fn(root, nil, err)
	} else {
		err = walkDir(fsys, root, FileInfoToDirEntry(info), fn)
	}
	if err == SkipDir || err == SkipAll {
		return nil
	}
	return err
}

// WalkDirFunc 是 WalkDir 的回调函数类型.    
type WalkDirFunc func(path string, d DirEntry, err error) error

// walkDir 递归遍历目录,调用 fn 函数处理每个文件.
func walkDir(fsys FS, name string, d DirEntry, walkDirFn WalkDirFunc) error {
	if err := walkDirFn(name, d, nil); err != nil || !d.IsDir() {
		if err == SkipDir && d.IsDir() {
			// Successfully skipped directory.
			err = nil
		}
		return err
	}

	dirs, err := ReadDir(fsys, name)
	if err != nil {
		// Second call, to report ReadDir error.
		err = walkDirFn(name, d, err)
		if err != nil {
			if err == SkipDir && d.IsDir() {
				err = nil
			}
			return err
		}
	}

	for _, d1 := range dirs {
		name1 := path.Join(name, d1.Name())
		if err := walkDir(fsys, name1, d1, walkDirFn); err != nil {
			if err == SkipDir {
				break
			}
			return err
		}
	}
	return nil
}
```
- `fs.WalkDir` 函数接受一个 `fs.FS` 对象,一个根目录和一个回调函数作为参数.
- 回调函数接受一个路径,一个 `fs.DirEntry` 对象和一个错误作为参数.
- `fs.WalkDir` 函数递归遍历目录,调用回调函数处理每个文件.
- 回调函数返回一个错误,如果错误不为空,则停止遍历.
- 回调函数返回 `fs.SkipDir` 错误,则跳过当前目录.
- 回调函数返回 `fs.SkipAll` 错误,则停止遍历.

## 实践示例
### 压缩文件处理
处理 zip 等压缩文件作为文件系统

```go
package main

import (
	"archive/zip"
	"io/fs"
)

func main() {
	r, _ := zip.OpenReader("testdata.zip")
	defer r.Close()

	// 使用 zip 文件作为文件系统
	fs.WalkDir(r, ".", func(path string, d fs.DirEntry, err error) error {
		// 处理文件或目录
		return nil
	})
}
```

### 文件系统遍历
使用 WalkDir 函数遍历目录结构
```go
package main
import (
	"io/fs"
	"os"
)
func main() {
	// 使用本地文件系统
	var localFS fs.FS = os.DirFS(".")
	// 遍历目录
	fs.WalkDir(localFS, ".", func(path string, d fs.DirEntry, err error) error {
        // 处理文件或目录
		return nil
	})
}
```

### 嵌入式资源访问
```go
package main
import (
	"embed"
	"io/fs"
)
//go:embed testdata
var testdata embed.FS
func main() {
	// 使用嵌入式资源作为文件系统
	fs.WalkDir(testdata, ".", func(path string, d fs.DirEntry, err error) error {
		// 处理文件或目录
		return nil
	})
}
```

### 与标准库集成
```go
package main

import (
	"io/fs"
	"net/http"
	"os"
	"text/template"
)

func main() {
	// 使用本地文件系统
	var localFS fs.FS = os.DirFS(".")

	// 解析模板
	temp, err := template.ParseFS(localFS, "templates/*.html")
	if err != nil {
		panic(err)
	}
	// 渲染模板
	temp.ExecuteTemplate(os.Stdout, "index.html", nil)

	// 使用网络文件系统
	httpFS := http.FS(localFS)
	http.Handle("/", http.FileServer(httpFS))
	http.ListenAndServe(":8080", nil)
}
```

许多标准库函数接受 fs.FS 接口:
- http.FileServer 可以通过 http.FS 包装 fs.FS
- template.ParseFS 从文件系统解析模板
- text/template 和 html/template 的 ParseFS 方法

### 测试场景

```go
package main

import (
	"fmt"
	"testing/fstest"
	"time"
)

func main() {
	testFS := fstest.MapFS{
		"hello.txt": {
			Data:    []byte("hello world"),
			Mode:    0444,
			ModTime: time.Now(),
		},
	}
	info, err := testFS.Stat("hello.txt")
	if err != nil {
		panic(err)
	}
	fmt.Println(info.Name())
}
```

### 云存储读取
将S3等云存储挂载为本地文件系统.这样可以使用标准库的文件操作函数进行操作.根据实际情况选择合适的方式:
- 使用第三方库,如 `go-cloud` 库,在程序中挂载云存储为本地文件系统.
- 使用系统命令,如 `mount` 命令,将云存储挂载为本地文件系统.

```go
package main
import (
	"context"
	"fmt"
	"io/fs"
	"log"
	"gocloud.dev/blob"
	_ "gocloud.dev/blob/gcs" // 引入 GCS 驱动，如果使用其他云存储服务，请引入相应的驱动
)
func main() {
	// 设置 S3 存储桶 URL
	bucketURL := "gs://my-bucket"
	// 创建一个 blob.Bucket
	bucket, err := blob.OpenBucket(context.Background(), bucketURL)
	if err != nil {
		log.Fatal(err)
	}
	defer bucket.Close()
	// 创建一个 fs.FS
	fsys := blob.NewFS(bucket)
	// 现在可以使用 fsys 进行文件操作
	err = fs.WalkDir(fsys, ".", func(path string, d fs.DirEntry, err error) error {
		if err != nil {
			return err
		}
		fmt.Println("Walking:", path)
		return nil
	})
	if err != nil {
		log.Fatal(err)
	}
}
```

## 总结
Go语言的io/fs包为文件系统操作提供了一个优雅的抽象层，其主要价值体现在：
- 统一接口：通过定义一组最小接口(fs.FS, fs.File等)，统一了不同文件系统的访问方式
- 解耦设计：应用程序不再依赖具体文件系统实现，可以轻松切换本地文件系统、内存文件系统、嵌入式文件系统等
- 测试友好：通过fstest.MapFS等工具可以轻松创建测试用的虚拟文件系统
- 标准集成：与Go标准库深度整合，如//go:embed、net/http、text/template等

io/fs包代表了Go语言对文件系统抽象的一次重要演进，它平衡了简单性与扩展性，是Go标准库中接口设计的典范。通过合理使用这个包，可以编写出更灵活、更易测试且与具体存储解耦的Go应用程序。

## 参考

[Go 1.16 io/fs 接口](https://golang.org/pkg/io/fs/)

[鸟窝 快速了解 Go io/fs 包](https://colobu.com/2025/01/30/some-notes-about-go-io-fs-package/)
