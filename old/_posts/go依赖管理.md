---
title: go依赖管理
date: 2019-07-02 16:35:45
tags:
	- go
	- 依赖管理
	
---

1 vendor
===

2 dep
===
>- 项目依赖在project/vendor/目录下
>- 必须放在gopath下

```
Dep is a tool for managing dependencies for Go projects

Usage: "dep [command]"

Commands:

  init     Set up a new Go project, or migrate an existing one
  status   Report the status of the project's dependencies
  ensure   Ensure a dependency is safely vendored in the project
  version  Show the dep version information
  check    Check if imports, Gopkg.toml, and Gopkg.lock are in sync

Examples:
  dep init                               set up a new project
  dep ensure                             install the project's dependencies
  dep ensure -update                     update the locked versions of all dependencies
  dep ensure -add github.com/pkg/errors  add a dependency to the project

Use "dep help [command]" for more information about a command.
```
>-
>
```
执行成功之后会生成两个文件 Gopkg.lock、Gopkg.toml和一个文件夹vendor
Gopkg.toml文件记录着current project依赖项project的约束。
```

```
Gopkg.toml参数解释：

[[constraint]]： 这个约束主要体现在到底要采用目标project的某个tag的版本（version），还是某个branch，或者是某个revision，这三个对于一个constraint只能选一个。

[[override]]:有时项目依赖比较复杂，经常会遇到依赖冲突导致 dep ensure 命令无法执行成功，这个时候使用 override 消除单个依赖关系上多个不可调和的constraint声明之间的分歧

[[required]]：列出了必须包含在Gopkg.lock中的一组包

[[ignored]]：列出dep静态分析源代码时忽略的一组包

[[prune]]：prune为依赖关系定义全局和每个项目的prune选项。 这些选项决定写入vendor/时丢弃哪些文件

unused-packages:修剪掉来自于目录中，但是没有出现在包导入图中的文件
non-go：修剪掉非.go文件
go-tests：修剪掉Go的测试文件
Gopkg.lock文件是工具生成的，你不用手工编辑

vendor文件里面存放current project的远程依赖的源代码
在指定version的时候，如果指定semantic version，可选的符号有

=:              只选择对应version
>或<:        大于(或小于)对应版本号
>=或<=:   大于等于(或小于等于)对应版本号
~:               ~1.2.3表示 >=1.2.3,<1.3.0
^:                ^1.2.3表示  >1.2.3,<2.0.0
不指定符号的话，默认为^符号。
```

```
Usage: dep ensure [-update | -add] [-no-vendor | -vendor-only] [-dry-run] [-v] [<spec>...]

Project spec:

  <import path>[:alt source URL][@<constraint>]


Ensure gets a project into a complete, reproducible, and likely compilable state:

  * All imports are fulfilled
  * All rules in Gopkg.toml are respected
  * Gopkg.lock records immutable versions for all dependencies
  * vendor/ is populated according to Gopkg.lock

Ensure has fast techniques to determine that some of these steps may be
unnecessary. If that determination is made, ensure may skip some steps. Flags
may be passed to bypass these checks; -vendor-only will allow an out-of-date
Gopkg.lock to populate vendor/, and -no-vendor will update Gopkg.lock (if
needed), but never touch vendor/.

The effect of passing project spec arguments varies slightly depending on the
combination of flags that are passed.


Examples:

  dep ensure                                 Populate vendor from existing Gopkg.toml and Gopkg.lock
  dep ensure -add github.com/pkg/foo         Introduce a named dependency at its newest version
  dep ensure -add github.com/pkg/foo@^1.0.1  Introduce a named dependency with a particular constraint

For more detailed usage examples, see dep ensure -examples.

Flags:

  -add          add new dependencies, or populate Gopkg.toml with constraints for existing dependencies (default: false)
  -dry-run      only report the changes that would be made (default: false)
  -examples     print detailed usage examples (default: false)
  -no-vendor    update Gopkg.lock (if needed), but do not update vendor/ (default: false)
  -update       update the named dependencies (or all, if none are named) in Gopkg.lock to the latest allowed by Gopkg.toml (default: false)
  -v            enable verbose logging (default: false)
  -vendor-only  populate vendor/ from Gopkg.lock without updating it first (default: false)
```

dep相关命令：
```
dep help XX
dep init
dep ensure -add github.com/xx
dep ensure -v
dep ensure -update packageXX
dep ensure -update
```
[dep github链接](https://github.com/golang/dep)

3 mod
===
A module is a collection of Go packages stored in a file tree with a go.mod file at its root. 
The go.mod file defines the module’s module path, which is also the import path used for the root directory, and its dependency requirements, which are the other modules needed for a successful build.(不同module之间互相引用必须能通过module名拉去到对应的项目)
>- go.mod文件定义了module的module路径，也是根目录&其他module依赖的导入路径
>- go.mod解耦了go项目所在实际目录及其导入目录；不同go项目必须可以通过module路径拉取到对应的包
>- The argument to go mod init is the module path, the location where the module may be found. module path为相应的module可以被找到的路径
>- 支持多个大版本依赖
>- 对于用过其他依赖管理的项目只需要go mod init，mod会自动迁移依赖
>- 多项目共享依赖：go/src/pkg/mod
>- goPath不用来解决import，用来存储下载的依赖和编译命令。downloaded source code (in GOPATH/pkg/mod) and compiled commands (in GOPATH/bin).
>- 引用go mod项目的import依赖与go.mod文件中的module名一致
>- 不通大版本go.mod中路径为XX/v2 XX，引用时路径可不带/v2
>- 依赖版本为v0.0.0时会拉取master最新head
>- go module 安装 package 的原則是先拉最新的 release tag，若无tag则拉最新的commit
>- 用法
>
```
关系goproxy: export 
工作目录：~/gomod/hello 非gopath目录  
1 cd ~/gomod/hello
2 go mod init mytest/hello
3 生成go.mod文件: module mytest/hello
4 go.mod文件仅在module的根目录下：该目录为module path,且为其他引用模块的导入路径的根路径
5 module目录下的其他子目录(eg world)的导入路径为: module path/子目录； 本例为mytest/hello/world
6 tree结构如下：
.
├── cmd
│   └── main.go
├── go.mod
├── main
├── testhello
└── world
    ├── hello.go
    └── hello_test.go
7 cat main.go
package main
import (
"fmt"
world "mytest/hello/world" // go mod下的子目录引用格式
)
func main(){
s := world.Hello()
fmt.Println(s)
}
8 hello目录下：go build -o testhello ./cmd/main.go
9 执行./testhello: 输出Hello, world.
```

>- 依赖需求写为a module path & a specific semantic version.
>
```
module hello
require (
	github.com/BurntSushi/toml v0.3.1
	github.com/PaesslerAG/gval v1.0.1
	github.com/andreburgaud/crypt2go v0.0.0-20190506042608-9396ccdf73c8
	github.com/beorn7/perks v1.0.1
	github.com/certifi/gocertifi v0.0.0-20180118203423-deb3ae2ef261
	github.com/coreos/etcd v3.3.13+incompatible
	github.com/fluge/squirrel v0.0.0-20171213032934-026d18314901
	github.com/getsentry/raven-go v0.2.0
	github.com/go-errors/errors v1.0.1
	github.com/go-redis/redis v0.0.0-20190813142431-c5c4ad6a4cae
	github.com/go-sql-driver/mysql v1.4.1
	github.com/gogo/protobuf v1.2.1
	github.com/golang/mock v0.0.0-20190508161146-9fa652df1129
	github.com/jmoiron/sqlx v1.2.0
	github.com/json-iterator/go v1.1.7
	github.com/konsorten/go-windows-terminal-sequences v1.0.2
	github.com/lann/builder v0.0.0-20180802200727-47ae307949d0
	github.com/lann/ps v0.0.0-20150810152359-62de8c46ede0
	github.com/magiconair/properties v1.8.1
	github.com/matttproud/golang_protobuf_extensions v1.0.1
	github.com/modern-go/concurrent v0.0.0-20180306012644-bacd9c7ef1dd
)
```

>>- [semantic version](https://semver.org/)
>>
```
Consider a version format of X.Y.Z (Major.Minor.Patch). Bug fixes not affecting the API increment the patch version, backwards compatible API additions/changes increment the minor version, and backwards incompatible API changes increment the major version.
//版本号
MAJOR.MINOR.PATCH
Major大版本不兼容更新
minor变更性功能后向兼容
patch后向兼容bug修复
```

>- 依赖在goPath/pkg/mod目录下
>- 
>- 相同项目依赖不同主版本为不同的文件目录: eg rsc.io/quote 其v3版本与v2版本为不同的文件夹；
>- 相同依赖的主版本不能相同: v1.5.2不能与v1.6.0同时存在
>- mod支持相同依赖（eg rsc.io/quote）的不同主版本
>
```
rsc.io/quote v1.5.2
rsc.io/quote/v3 v3.1.0 //不同主版本存放在不同的路径下
```
>- it is impossible for a program to build with both rsc.io/quote v1.5.2 and rsc.io/quote v1.6.0；不可能一个module build两个不同的小版本
>- go mod init packagename: creates a new module, initializing the go.mod file that describes it.
>- go build, go test, and other package-building commands add new dependencies to go.mod as needed.
>- go list -m all : prints the current module’s dependencies.可以查看依赖是否为自己需求的若不是则通过go get获取对应的依赖
>- go get(可带版本): changes the required version of a dependency (or adds a new dependency). go get XX@v1.2.3
>- go mod tidy : 添加import导入的依赖并removes unused dependencies.  
>- go mod why -m rsc.io/quote v1.5.2
>
```
go list -m all
go mod why -m github.com/sirupsen/logrus v1.4.2
go mod graph //依赖关系图
go get github.com/sirupsen/logrus@v1.4.2
go mod tidy //添加依赖（递归添加 依赖的依赖也会添加） 删除无用依赖；添加module的最新版本，需要降级版本时通过go get获取
go get -u 将会升级到最新的次要版本或者修订版本(x.y.z, z是修订版本号， y是次要版本号);仅升级对应包及包import的相关包，递归
go get -u all 更新main包导入的所有依赖包，递归查询
go get -u=patch 将会升级到最新的修订版本
go get package@version 将会升级到指定的版本号version
go get如果有版本的更改，那么go.mod文件也会更改

```
>- 
>
```
go mod init creates a new module, initializing the go.mod file that describes it.
go build, go test, and other package-building commands add new dependencies to go.mod as needed.
go list -m all prints the current module’s dependencies.
go get changes the required version of a dependency (or adds a new dependency).
go mod tidy removes unused dependencies.
```
>
>
总结：

```
1. 使用go 1.13
2. 设置环境变量：利用代理&私服
2.1 go env -w GOPROXY=https://goproxy.cn,direct
2.2 go env -w GOPRIVATE **.**-corp.com //1.13自动设置 GONOPROXY="**.**-corp.com" GONOSUMDB="**.**-corp.com"
3 require格式为版本号v1.3.2 或者commitID前若干位； 不可为分支类型
eg: go get github.com/mqu/go-notify@ef6f6f49
require (
  // other packages
  github.com/mqu/go-notify ef6f6f49
)
commitID hash mod官方为12位，至少给8位
4 replace:只能处理顶层依赖
4.1 本地replace; github.**.com => ../xx或者./xx
本地替换引用为相对路径，且非本module的本地引用需要带go.mod文件，即mod之间相互引用
4.2 网络引用替换: github.aa => github.bb v1.2.0 或者commitID完整位或者前几位
5 proto文件用proto3生成(要求对应proto-gen-go版本)版本3对应的pb.go会写明版本ISVERSION3；对应引用的proto相应的包需要支持proto3；需要将pb.go对应的go程序编译成相应版本的proto文件；
6 avro问题，可以本地replace
7 依赖问题：依赖的依赖版本不对，需要修改，按提示修改即可
8 执行go mod tidy
9 执行go build -o xx xx
10 语义化版本信息vx.x.x
module从v1更新到v2无法保证兼容性时，需要将module改为xx/v2
eg: 
module my-module/v2
require (
  some/pkg/v2 v2.0.0
  some/pkg/v2/mod1 v2.0.0
  my/pkg/v3 v3.0.1
)
v2+的包使用pkg/v2： v2及以上版本需要写成pkg/vN
11 go.sum
go.sum是一个构建状态跟踪文件。它会记录当前module所有的顶层和间接依赖，以及这些依赖的校验和，从而提供一个可以100%复现的构建过程并对构建对象提供安全性的保证。
go.sum同时还会保留过去使用的包的版本信息，以便日后可能的版本回退，这一点也与普通的锁文件不同。所以go.sum并不是包管理器的锁文件。
因此我们应该把go.sum和go.mod一同添加进版本控制工具的跟踪列表，同时需要随着你的模块一起发布。如果你发布的模块中不包含此文件，使用者在构建时会报错，同时还可能出现安全风险（go.sum提供了安全性的校验）
12 +incompatible: etcd的tag v3.3.15版本对应的module文件并不是etcd/v3，默认v3.3向下兼容，若不向下兼容，需要更改module path添加/vN；
github.com/coreos/etcd v3.3.15+incompatible // indirect
eg go.mod module github.com/coreos/etcd/v3
13 若包文件包含大些字符，在mac系统上，pkg/mod目录中包名为!小写字母，编译时找不到对应的包

```
>>- go.mod示例
>>
```
module golang.guazi-corp.com/znkf/imkf-dispatch
go 1.12
require (
	avro v0.0.0-00010101000000-000000000000
	cloud.google.com/go v0.36.0 // indirect
	github.com/LeoLy008/go-orm v0.0.0-20180606041409-beeb895458b8
	github.com/RichardKnop/logging v0.0.0-20181101035820-b1d5d44c82d6 // indirect
	github.com/RyouZhang/async-go v0.1.0
	github.com/alexcesaro/statsd v2.0.0+incompatible // indirect
	github.com/andreburgaud/crypt2go v0.0.0-20170406040737-362bd5db8ea6 // indirect
	github.com/asaskevich/govalidator v0.0.0-20180319081651-7d2e70ef918f // indirect
	github.com/bradfitz/gomemcache v0.0.0-20180710155616-bc664df96737 // indirect
	github.com/certifi/gocertifi v0.0.0-20180118203423-deb3ae2ef261 // indirect
	github.com/confluentinc/confluent-kafka-go v0.11.5-0.20180904122048-1723c23c211e // indirect
	github.com/coreos/bbolt v1.3.3 // indirect
	github.com/coreos/etcd v3.3.15+incompatible // indirect
	github.com/coreos/go-semver v0.3.0 // indirect
	github.com/coreos/pkg v0.0.0-20180928190104-399ea9e2e55f // indirect
	github.com/dgrijalva/jwt-go v3.2.0+incompatible // indirect
	github.com/elazarl/goproxy v0.0.0-20190911111923-ecfe977594f1 // indirect
	github.com/evalphobia/logrus_sentry v0.4.5
	github.com/fatih/structs v1.0.1-0.20180123065059-ebf56d35bba7 // indirect
	github.com/garyburd/redigo v1.6.0 // indirect
	github.com/getsentry/raven-go v0.0.0-20180121060056-563b81fc02b7 // indirect
	github.com/gin-contrib/cors v0.0.0-20180514151808-6f0a820f94be
	github.com/gin-contrib/sse v0.0.0-20170109093832-22d885f9ecc7 // indirect
	github.com/gin-gonic/gin v1.1.5-0.20180531061340-caf3e350a548
	github.com/go-redis/redis v6.12.1-0.20180618105819-4b568cdf1adc+incompatible
	github.com/gogo/protobuf v1.3.0 // indirect
	github.com/golang/groupcache v0.0.0-20190702054246-869f871628b6 // indirect
	github.com/gomodule/redigo v2.0.0+incompatible // indirect
	github.com/google/uuid v1.1.0 // indirect
	github.com/gorilla/websocket v1.4.1 // indirect
	github.com/grpc-ecosystem/go-grpc-middleware v1.0.0 // indirect
	github.com/grpc-ecosystem/go-grpc-prometheus v0.0.0-20170826090648-0dafe0d496ea // indirect
	github.com/grpc-ecosystem/grpc-gateway v1.6.2 // indirect
	github.com/jonboulle/clockwork v0.1.0 // indirect
	github.com/json-iterator/go v0.0.0-20180722035151-10a568c51178
	github.com/kelseyhightower/envconfig v1.3.0 // indirect
	github.com/magiconair/properties v1.8.1
	github.com/mattn/go-isatty v0.0.4 // indirect
	github.com/modern-go/concurrent v0.0.0-20180306012644-bacd9c7ef1dd // indirect
	github.com/modern-go/reflect2 v1.0.1 // indirect
	github.com/moul/http2curl v0.0.0-20161031194548-4e24498b31db // indirect
	github.com/nu7hatch/gouuid v0.0.0-20131221200532-179d4d0c4d8d // indirect
	github.com/onsi/ginkgo v1.10.1 // indirect
	github.com/onsi/gomega v1.7.0 // indirect
	github.com/pkg/errors v0.8.1 // indirect
	github.com/prometheus/client_golang v0.9.2
	github.com/prometheus/common v0.0.0-20181218105931-67670fe90761 // indirect
	github.com/rcrowley/go-metrics v0.0.0-20161128210544-1f30fe9094a5 // indirect
	github.com/satori/go.uuid v1.2.0 // indirect
	github.com/sirupsen/logrus v1.0.4-0.20170822132746-89742aefa4b2
	github.com/smartystreets/goconvey v0.0.0-20190731233626-505e41936337 // indirect
	github.com/soheilhy/cmux v0.1.4 // indirect
	github.com/streadway/amqp v0.0.0-20190827072141-edfb9018d271 // indirect
	github.com/stretchr/testify v1.3.0
	github.com/stvp/tempredis v0.0.0-20181119212430-b82af8480203 // indirect
	github.com/tmc/grpc-websocket-proxy v0.0.0-20190109142713-0ad062ec5ee5 // indirect
	github.com/uber-go/atomic v1.4.0 // indirect
	github.com/ugorji/go v1.1.2-0.20180407103000-f3cacc17c85e // indirect
	github.com/xiang90/probing v0.0.0-20190116061207-43a291ad63a2 // indirect
	github.com/zsais/go-gin-prometheus v0.0.0-20180525164355-3f93884fa240
	go.etcd.io/bbolt v1.3.3 // indirect
	go.uber.org/atomic v1.3.2 // indirect
	go.uber.org/multierr v1.1.0 // indirect
	go.uber.org/ratelimit v0.0.0-20180316092928-c15da0234277 // indirect
	go.uber.org/zap v1.10.0 // indirect
	golang.org/x/net v0.0.0-20190923162816-aa69164e4478
	golang.org/x/sys v0.0.0-20190221075227-b4e8571b14e0 // indirect
	google.golang.org/genproto v0.0.0-20190819201941-24fa4b261c55 // indirect
	google.golang.org/grpc v1.21.1
	gopkg.in/airbrake/gobrake.v2 v2.0.9 // indirect
	gopkg.in/alexcesaro/statsd.v2 v2.0.0 // indirect
	gopkg.in/check.v1 v1.0.0-20180628173108-788fd7840127 // indirect
	gopkg.in/gemnasium/logrus-airbrake-hook.v2 v2.1.2 // indirect
	gopkg.in/go-playground/assert.v1 v1.2.1 // indirect
	gopkg.in/go-playground/validator.v8 v8.18.2 // indirect
	gopkg.in/mgo.v2 v2.0.0-20190816093944-a6b53ec6cb22 // indirect
	gopkg.in/redsync.v1 v1.0.1 // indirect
	gopkg.in/square/go-jose.v1 v1.1.2 // indirect
	gopkg.in/testfixtures.v2 v2.5.3 // indirect
	gopkg.in/yaml.v2 v2.2.2 // indirect
	gotest.tools v2.2.0+incompatible // indirect
	proto v0.0.0-20190924125620-d031cd82657f
	sigs.k8s.io/yaml v1.1.0 // indirect
)
replace (
	avro => ../../../avro
	github.com/gomodule/redigo/redis => github.com/garyburd/redigo v1.3.0
)
```

3.2 其他
---
go mod exclude: 当前版本用的不是tag标准版本，利用exclude，go mod从master拉取相应分支

```
go.mod
module github.com/example/project

require (
    github.com/SermoDigital/jose v0.0.0-20180104203859-803625baeddc
    github.com/google/uuid v1.1.0
)

exclude github.com/SermoDigital/jose v0.9.1

replace github.com/google/uuid v1.1.0 => git.coolaj86.com/coolaj86/uuid.go v1.1.1
exclude
In the case of the github.com/SermoDigital/jose package, it has a proper git tag for v0.9.1, but the current version is v1.1, which is NOT a proper git tag (missing the "patch" version).

By excluding the broken version it causes go mod to fetch from master instead.

replace
Likewise (and truly hypothetical), if I have a patch to github.com/google/uuid, I can create a fork and use replace to get my own version while I wait for the upstream version to accept my patch (or not).
```
[exclude & replace stackoverflow](https://stackoverflow.com/questions/53473290/how-the-exclude-directive-works-in-the-go-mod-file)

[官网参考go.mod](https://blog.golang.org/using-go-modules)  
[go module in 2019](https://blog.golang.org/modules2019)  
[依赖包失败解决方案](https://shockerli.net/post/go-get-golang-org-x-solution/)
[ go 官方command](https://golang.org/cmd/go/#hdr-The_go_mod_file)
[go mod官方详细文档](https://github.com/golang/go/wiki/Modules)
[go mod迁移官方](https://blog.golang.org/migrating-to-go-modules)
[需要proxy代理的依赖,替换为github](https://blog.csdn.net/qq_33296108/article/details/88184060)
[掘金go mod](https://juejin.im/post/5c8e503a6fb9a070d878184a)
[go mod](https://segmentfault.com/a/1190000018398763)
[go代理](https://www.cnblogs.com/sparkdev/p/10649159.html)
[go mod](https://colobu.com/2018/08/27/learn-go-module/)
[go mod细节](https://www.cnblogs.com/apocelipes/p/10295096.html)
