## A3.2 获取与创建项目

有几种方式获取一个 Git 仓库。 一种是从网络上或者其他地方拷贝一个现有的仓库，另一种就是在一个目录中创建一个新的仓库。

### git init

你只需要简单地运行 git init 就可以将一个目录转变成一个 Git 仓库，这样你就可以开始对它进行版本管理了。

我们一开始在 获取 Git 仓库 一节中介绍了如何创建一个新的仓库来开始工作。

在 远程分支 一节中我们简单的讨论了如何改变默认分支。

在 把裸仓库放到服务器上 一节中我们使用此命令来为一个服务器创建一个空的祼仓库。

最后，我们在 底层命令和高层命令 一节中介绍了此命令背后工作的原理的一些细节。

### git clone

git clone 实际上是一个封装了其他几个命令的命令。 它创建了一个新目录，切换到新的目录，然后 git init 来初始化一个空的 Git 仓库， 然后为你指定的 URL 添加一个（默认名称为 origin 的）远程仓库（git remote add），再针对远程仓库执行 git fetch，最后通过 git checkout 将远程仓库的最新提交检出到本地的工作目录。

git clone 命令在本书中多次用到，这里只列举几个有意思的地方。

在 克隆现有的仓库 一节中我们通过几个示例详细介绍了此命令。

在 在服务器上搭建 Git 一节中，我们使用了 --bare 选项来创建一个没有任何工作目录的 Git 仓库副本。

在 打包 一节中我们使用它来解包一个打包好的 Git 仓库。

最后，在 克隆含有子模块的项目 一节中我们学习了使用 --recursive 选项来让克隆一个带有子模块的仓库变得简单

虽然在本书的其他地方都有用到此命令，但是上面这些用法是特例，或者使用方式有点特别。