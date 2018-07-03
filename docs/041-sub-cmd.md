# main函数 和 sub command

## main函数

main 函数其实很明了。

```golang
package main

import (
    "runtime"

    "os"

    "github.com/gohugoio/hugo/commands"  // hugo 子指令都在这个包里面
)

func main() {
    runtime.GOMAXPROCS(runtime.NumCPU())
    // 关键代码逻辑
    resp := commands.Execute(os.Args[1:])

    if resp.Err != nil {
        if resp.IsUserError() {
            resp.Cmd.Println("")
            resp.Cmd.Println(resp.Cmd.UsageString())
        }
        os.Exit(-1)
    }
}
```

main 包和 command 包通过 `Execute` 函数和 `commands.Respone` 结构交互。

## sub command

先简述下后续的逻辑：

hugo 如果提供了sub command，则执行，否则默认动作是build.

`command/hugo.go`

```golang
// Execute adds all child commands to the root command HugoCmd and sets flags appropriately.
// The args are usually filled with os.Args[1:].
func Execute(args []string) Response {
    hugoCmd := newCommandsBuilder().addAll().build()
    cmd := hugoCmd.getCommand()
    cmd.SetArgs(args)

    c, err := cmd.ExecuteC()

    var resp Response

    if c == cmd && hugoCmd.c != nil {
        // Root command executed
        resp.Result = hugoCmd.c.hugo
    }

    if err == nil {
        errCount := int(jww.LogCountForLevelsGreaterThanorEqualTo(jww.LevelError))
        if errCount > 0 {
            err = fmt.Errorf("logged %d errors", errCount)
        } else if resp.Result != nil {
            errCount = resp.Result.NumLogErrors()
            if errCount > 0 {
                err = fmt.Errorf("logged %d errors", errCount)
            }
        }

    }

    resp.Err = err
    resp.Cmd = c

    return resp
}
```

这段信息量比较大。

关键在这几句

```golang
    hugoCmd := newCommandsBuilder().addAll().build()
    cmd := hugoCmd.getCommand()
    cmd.SetArgs(args)

    c, err := cmd.ExecuteC()
```

1. 这里先构造了一个`CommandsBuilder`
2. 然后给`CommandsBuilder`添加了一系列动作
3. 生成最终的 hugoCmd

### commandsBuilder生成关系

从我的理解看，commandsBuilder是一个工厂模型，构建了下面这些 xxxCmd .

```text
commandsBuilder
    |
    +- serverCmd
    +- versionCmd
    +- envCmd
    +- configCmd
    +- convertCmd
    +- newCmd
    +- listCmd
    +- importCmd
    +- genCmd
```

这里先看下 `commandsBuilder` 的详细定义。

```golang
type flagsToConfigHandler interface {
    flagsToConfig(cfg config.Provider)
}

// 上面的 xxxCmd 都实现了这个接口
type cmder interface {
    flagsToConfigHandler
    getCommand() *cobra.Command
}

// commandsBuilder继承它
// xxxCmd 也继承并使用它的各属性
type hugoBuilderCommon struct {
    source  string
    baseURL string

    buildWatch bool

    gc bool

    /// ...
}

type commandsBuilder struct {
    hugoBuilderCommon

    commands []cmder
}
```

###  工厂模式构建`xxxCmd`

```golang
func (b *commandsBuilder) addAll() *commandsBuilder {
    b.addCommands(
        b.newServerCmd(),
        newVersionCmd(),
        newEnvCmd(),
        newConfigCmd(),
        newCheckCmd(),
        b.newBenchmarkCmd(),
        newConvertCmd(),
        newNewCmd(),
        newListCmd(),
        newImportCmd(),
        newGenCmd(),
        createReleaser(),
    )

    // 返回的还是自己的指针
    return b
}
```
### xxxCmd 详细定义

每一个`newXXXCmd`返回的都是实现了`cmder`接口的结构，

从继承结构看，实际分为三类cmd

- 仅继承 `*baseCmd`
- 同时继承 `*hugoBuilderCommon`, `*baseCmd`
- 通过继承 `*baseBuilderCmd` <BR>
    同时继承 `*commandsBuilder`, `*baseCmd`

详细代码

```golang
type baseCmd struct {
    cmd *cobra.Command
}

type baseBuilderCmd struct {
    *baseCmd
    *commandsBuilder
}


// hugoCmd是最特殊的一个Cmd
type hugoCmd struct {
    *baseBuilderCmd
    c *commandeer
}
// serverCmd是另外一个特殊Cmd
type serverCmd struct {
    *baseBuilderCmd
}
type versionCmd struct {
    *baseCmd
}
type envCmd struct {
    *baseCmd
}
type configCmd struct {
    hugoBuilderCommon
    *baseCmd
}
type convertCmd struct {
    hugoBuilderCommon

    outputDir string
    unsafe    bool

    *baseCmd
}
type newCmd struct {
    hugoBuilderCommon
    contentEditor string
    contentType   string

    *baseCmd
}
type listCmd struct {
    hugoBuilderCommon
    *baseCmd
}
// 其它略
```

这里，最神奇的就是 `hugoCmd`, `serverCmd`，它们都继承于 `*baseBuilderCmd`。

而 baseBuilderCmd 自身又内嵌了 `*commandsBuilder` 。

从逻辑上看，就是 `commandsBuilder` 生成了`serverCmd`，而`serverCmd`内部有一个`commandsBuilder`.

```text
commandsBuilder --create-> &serverCmd {
    baseBuilderCmd: &baseBuilderCmd{
        baseCmd: &baseCmd{},
        commandsBuilder: &commandsBuilder{},
    }
}
```

实际相关代码：

```golang
func (b *commandsBuilder) newServerCmd() *serverCmd {
    return b.newServerCmdSignaled(nil)
}

func (b *commandsBuilder) newServerCmdSignaled(stop <-chan bool) *serverCmd {
    cc := &serverCmd{stop: stop}

    cc.baseBuilderCmd = b.newBuilderCmd(&cobra.Command{
        Use:     "server",
        Aliases: []string{"serve"},
        Short:   "A high performance webserver",
        Long: `Hugo provides its own webserver which builds and serves the site.
While hugo server is high performance, .......`,
        RunE: cc.server,
    })

    cc.cmd.Flags().IntVarP(&cc.serverPort, "port", "p", 1313, "port on which the server will listen")
    cc.cmd.Flags().IntVar(&cc.liveReloadPort, "liveReloadPort", -1, "port for live reloading (i.e. 443 in HTTPS proxy situations)")

    // ...
}

func (b *commandsBuilder) newBuilderCmd(cmd *cobra.Command) *baseBuilderCmd {
    bcmd := &baseBuilderCmd{commandsBuilder: b, baseCmd: &baseCmd{cmd: cmd}}
    bcmd.hugoBuilderCommon.handleFlags(cmd)
    return bcmd
}
```

私以为，此处代码相当绕。

`commandsBuilder.newBuilderCmd` 返回 `*baseBuilderCmd` ，而 `baseBuilderCmd` 实际是继承了 `commandsBuilder`。

有种鸡生蛋蛋生鸡的感觉。

也可能是OOP经验腐蚀了我。

### 生成最终的 hugoCmd

```golang
func (b *commandsBuilder) build() *hugoCmd {
    h := b.newHugoCmd()
    addCommands(h.getCommand(), b.commands...)
    return h
}
```

### 继承图

```text

cobra.Command
baseCmd             cmder                   hugoBuilderCommon
|                   |                       |
|                   |                       |
|                   +-----------------------+
|                   |                       |
|                   |                       |
|                   |                       |
|                   commandsBuilder         +---- deps.DepsCfg
|                   |                       +------------ hugoSites
+--------+----------+                       +------------ hugo
         |                                  |
         |                                  |
         |                                  |
         baseBuilderCmd                     commandeer
         |                                  |
         +--------+-------------------------+
                  |
                  |
                  hugoCmd

```