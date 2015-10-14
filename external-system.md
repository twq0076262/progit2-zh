## A3.10 外部系统

Git 有一些可以与其他的版本控制系统集成的命令。

### git svn

git svn 可以使 Git 作为一个客户端来与 Subversion 版本控制系统通信。 这意味着你可以使用 Git 来检出内容，或者提交到 Subversion 服务器。

Git 与 Subversion 一章深入讲解了此命令。

### git fast-import

对于其他版本控制系统或者从其他任何的格式导入，你可以使用 git fast-import 快速地将其他格式映射到 Git 可以轻松记录的格式。

在 一个自定义的导入器 一节中深入讲解了此命令。