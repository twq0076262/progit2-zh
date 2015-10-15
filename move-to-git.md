## 9.2 迁移到 Git

如果你现在有一个正在使用其他 VCS 的代码库，但是你已经决定开始使用 Git，必须通过某种方式将你的项目迁移至 Git。 这一部分会介绍一些通用系统的导入器，然后演示如何开发你自己定制的导入器。 你将会学习如何从几个大型专业应用的 SCM 系统中导入数据，不仅因为它们是大多数想要转换的用户正在使用的系统，也因为获取针对它们的高质量工具很容易。

### Subversion

如果你阅读过前面关于 git svn 的章节，可以轻松地使用那些指令来 git svn clone 一个仓库，停止使用 Subversion 服务器，推送到一个新的 Git 服务器，然后就可以开始使用了。 如果你想要历史，可以从 Subversion 服务器上尽可能快地拉取数据来完成这件事（这可能会花费一些时间）。

然而，导入并不完美；因为花费太长时间了，你可能早已用其他方法完成导入操作。 导入产生的第一个问题就是作者信息。 在 Subversion 中，每一个人提交时都需要在系统中有一个用户，它会被记录在提交信息内。 在之前章节的例子中几个地方显示了 schacon，比如 blame 输出与 git svn log。 如果想要将上面的 Subversion 用户映射到一个更好的 Git 作者数据中，你需要一个 Subversion 用户到 Git 用户的映射。 创建一个 users.txt 的文件包含像下面这种格式的映射：

```
schacon = Scott Chacon <schacon@geemail.com>
selse = Someo Nelse <selse@geemail.com>
```

为了获得 SVN 使用的作者名字列表，可以运行这个：

```
$ svn log --xml | grep author | sort -u | \
  perl -pe 's/.*>(.*?)<.*/$1 = /'
```

这会将日志输出为 XML 格式，然后保留作者信息行、去除重复、去除 XML 标记。 （很显然这只会在安装了 grep、sort 与 perl 的机器上运行。） 然后，将输出重定向到你的 users.txt 文件中，这样就可以在每一个记录后面加入对应的 Git 用户数据。

你可以将此文件提供给 git svn 来帮助它更加精确地映射作者数据。 也可以通过传递 --no-metadata 给 clone 与 init 命令，告诉 git svn 不要包括 Subversion 通常会导入的元数据。 这会使你的 import 命令看起来像这样：

```
$ git svn clone http://my-project.googlecode.com/svn/ \
      --authors-file=users.txt --no-metadata -s my_project
```

现在在 my_project 目录中应当有了一个更好的 Subversion 导入。 并不像是下面这样的提交：

```
commit 37efa680e8473b615de980fa935944215428a35a
Author: schacon <schacon@4c93b258-373f-11de-be05-5f7a86268029>
Date:   Sun May 3 00:12:22 2009 +0000

    fixed install - go to trunk

    git-svn-id: https://my-project.googlecode.com/svn/trunk@94 4c93b258-373f-11de-
    be05-5f7a86268029
```

反而它们看起来像是这样：

```
commit 03a8785f44c8ea5cdb0e8834b7c8e6c469be2ff2
Author: Scott Chacon <schacon@geemail.com>
Date:   Sun May 3 00:12:22 2009 +0000

    fixed install - go to trunk
```

不仅是 Author 字段更好看了，git-svn-id 也不在了。

之后，你应当做一些导入后的清理工作。 第一步，你应当清理 git svn 设置的奇怪的引用。 首先移动标签，这样它们就是标签而不是奇怪的远程引用，然后你会移动剩余的分支这样它们就是本地的了。

为了将标签变为合适的 Git 标签，运行

```
$ cp -Rf .git/refs/remotes/origin/tags/* .git/refs/tags/
$ rm -Rf .git/refs/remotes/origin/tags
```

这会使原来在 remotes/origin/tags/ 里的远程分支引用变成真正的（轻量）标签。

接下来，将 refs/remotes 下剩余的引用移动为本地分支：

```
$ cp -Rf .git/refs/remotes/* .git/refs/heads/
$ rm -Rf .git/refs/remotes
```

现在所有的旧分支都是真正的 Git 分支，并且所有的旧标签都是真正的 Git 标签。 最后一件要做的事情是，将你的新 Git 服务器添加为远程仓库并推送到上面。 下面是一个将你的服务器添加为远程仓库的例子：

```
$ git remote add origin git@my-git-server:myrepository.git
```

因为想要上传所有分支与标签，你现在可以运行：

```
$ git push origin --all
```

通过以上漂亮、干净地导入操作，你的所有分支与标签都应该在新 Git 服务器上。

### Mercurial

因为 Mercurial 与 Git 在表示版本时有着非常相似的模型，也因为 Git 拥有更加强大的灵活性，将一个仓库从 Mercurial 转换到 Git 是相当直接的，使用一个叫作“hg-fast-export”的工具，需要从这里拷贝一份：

```
$ git clone http://repo.or.cz/r/fast-export.git /tmp/fast-export
```

转换的第一步就是要先得到想要转换的 Mercurial 仓库的完整克隆：

```
$ hg clone <remote repo URL> /tmp/hg-repo
```

下一步就是创建一个作者映射文件。 Mercurial 对放入到变更集作者字段的内容比 Git 更宽容一些，所以这是一个清理的好机会。 只需要用到 bash 终端下的一行命令：

```
$ cd /tmp/hg-repo
$ hg log | grep user: | sort | uniq | sed 's/user: *//' > ../authors
```

这会花费几秒钟，具体要看项目提交历史有多少，最终 /tmp/authors 文件看起来会像这样：

```
bob
bob@localhost
bob <bob@company.com>
bob jones <bob <AT> company <DOT> com>
Bob Jones <bob@company.com>
Joe Smith <joe@company.com>
```

在这个例子中，同一个人（Bob）使用不同的名字创建变更集，其中一个实际上是正确的，另一个完全不符合 Git 提交的规范。 Hg-fast-export 通过向我们想要修改的行尾添加 ={new name and email address} 来修正这个问题，移除任何我们想要保留的用户名所在的行。 如果所有的用户名看起来都是正确的，那我们根本就不需要这个文件。 在本例中，我们会使文件看起来像这样：

```
bob=Bob Jones <bob@company.com>
bob@localhost=Bob Jones <bob@company.com>
bob jones <bob <AT> company <DOT> com>=Bob Jones <bob@company.com>
bob <bob@company.com>=Bob Jones <bob@company.com>
```

下一步是创建一个新的 Git 仓库，然后运行导出脚本：

```
$ git init /tmp/converted
$ cd /tmp/converted
$ /tmp/fast-export/hg-fast-export.sh -r /tmp/hg-repo -A /tmp/authors
```

-r 选项告诉 hg-fast-export 去哪里寻找我们想要转换的 Mercurial 仓库，-A 标记告诉它在哪找到作者映射文件。 这个脚本会分析 Mercurial 变更集然后将它们转换成 Git“fast-import”功能（我们将在之后详细讨论）需要的脚本。 这会花一点时间（尽管它比通过网格 更 快），输出相当的冗长：

```
$ /tmp/fast-export/hg-fast-export.sh -r /tmp/hg-repo -A /tmp/authors
Loaded 4 authors
master: Exporting full revision 1/22208 with 13/0/0 added/changed/removed files
master: Exporting simple delta revision 2/22208 with 1/1/0 added/changed/removed files
master: Exporting simple delta revision 3/22208 with 0/1/0 added/changed/removed files
[…]
master: Exporting simple delta revision 22206/22208 with 0/4/0 added/changed/removed files
master: Exporting simple delta revision 22207/22208 with 0/2/0 added/changed/removed files
master: Exporting thorough delta revision 22208/22208 with 3/213/0 added/changed/removed files
Exporting tag [0.4c] at [hg r9] [git :10]
Exporting tag [0.4d] at [hg r16] [git :17]
[…]
Exporting tag [3.1-rc] at [hg r21926] [git :21927]
Exporting tag [3.1] at [hg r21973] [git :21974]
Issued 22315 commands
git-fast-import statistics:
---------------------------------------------------------------------
Alloc'd objects:     120000
Total objects:       115032 (    208171 duplicates                  )
      blobs  :        40504 (    205320 duplicates      26117 deltas of      39602 attempts)
      trees  :        52320 (      2851 duplicates      47467 deltas of      47599 attempts)
      commits:        22208 (         0 duplicates          0 deltas of          0 attempts)
      tags   :            0 (         0 duplicates          0 deltas of          0 attempts)
Total branches:         109 (         2 loads     )
      marks:        1048576 (     22208 unique    )
      atoms:           1952
Memory total:          7860 KiB
       pools:          2235 KiB
     objects:          5625 KiB
---------------------------------------------------------------------
pack_report: getpagesize()            =       4096
pack_report: core.packedGitWindowSize = 1073741824
pack_report: core.packedGitLimit      = 8589934592
pack_report: pack_used_ctr            =      90430
pack_report: pack_mmap_calls          =      46771
pack_report: pack_open_windows        =          1 /          1
pack_report: pack_mapped              =  340852700 /  340852700
---------------------------------------------------------------------

$ git shortlog -sn
   369  Bob Jones
   365  Joe Smith
```

那看起来非常好。 所有 Mercurial 标签都已被转换成 Git 标签，Mercurial 分支与书签都被转换成 Git 分支。 现在已经准备好将仓库推送到新的服务器那边：

```
$ git remote add origin git@my-git-server:myrepository.git
$ git push origin --all
```

### Perforce

下一个将要看到导入的系统是 Perforce。 就像我们之前讨论过的，有两种方式让 Git 与 Perforce 互相通信：git-p4 与 Perforce Git Fusion。

### Perforce Git Fusion

Git Fusion 使这个过程毫无痛苦。 只需要使用在 Git Fusion 中讨论过的配置文件来配置你的项目设置、用户映射与分支，然后克隆整个仓库。 Git Fusion 让你处在一个看起来像是原生 Git 仓库的环境中，如果愿意的话你可以随时将它推送到一个原生 Git 托管中。 如果你喜欢的话甚至可以使用 Perforce 作为你的 Git 托管。

### Git-p4

Git-p4 也可以作为一个导入工具。 作为例子，我们将从 Perforce 公开仓库中导入 Jam 项目。 为了设置客户端，必须导出 P4PORT 环境变量指向 Perforce 仓库：

```
$ export P4PORT=public.perforce.com:1666
```

**Note**

为了继续后续步骤，需要连接到 Perforce 仓库。 在我们的例子中将会使用在 public.perforce.com 的公开仓库，但是你可以使用任何你有权限的仓库。

运行 git p4 clone 命令从 Perforce 服务器导入 Jam 项目，提供仓库、项目路径与你想要存放导入项目的路径：

```
$ git-p4 clone //guest/perforce_software/jam@all p4import
Importing from //guest/perforce_software/jam@all into p4import
Initialized empty Git repository in /private/tmp/p4import/.git/
Import destination: refs/remotes/p4/master
Importing revision 9957 (100%)
```

这个特定的项目只有一个分支，但是如果你在分支视图（或者说一些目录）中配置了一些分支，你可以将 --detect-branches 选项传递给 git p4 clone 来导入项目的所有分支。 查看 分支 来了解关于这点的更多信息。

此时你几乎已经完成了。 如果进入 p4import 目录中并运行 git log，可以看到你的导入工作：

```
$ git log -2
commit e5da1c909e5db3036475419f6379f2c73710c4e6
Author: giles <giles@giles@perforce.com>
Date:   Wed Feb 8 03:13:27 2012 -0800

    Correction to line 355; change </UL> to </OL>.

    [git-p4: depot-paths = "//public/jam/src/": change = 8068]

commit aa21359a0a135dda85c50a7f7cf249e4f7b8fd98
Author: kwirth <kwirth@perforce.com>
Date:   Tue Jul 7 01:35:51 2009 -0800

    Fix spelling error on Jam doc page (cummulative -> cumulative).

    [git-p4: depot-paths = "//public/jam/src/": change = 7304]
```

你可以看到 git-p4在每一个提交里都留下了一个标识符。 如果之后想要引用 Perforce 的修改序号的话，标识符保留在那里也是可以的。 然而，如果想要移除标识符，现在正是这么做的时候 - 在你开始在新仓库中工作之前。 可以使用 git filter-branch 将全部标识符移除。

```
$ git filter-branch --msg-filter 'sed -e "/^\[git-p4:/d"'
Rewrite e5da1c909e5db3036475419f6379f2c73710c4e6 (125/125)
Ref 'refs/heads/master' was rewritten
```

如果运行 git log，你会看到所有提交的 SHA-1 校验和都改变了，但是提交信息中不再有 git-p4 字符串了：

```
$ git log -2
commit b17341801ed838d97f7800a54a6f9b95750839b7
Author: giles <giles@giles@perforce.com>
Date:   Wed Feb 8 03:13:27 2012 -0800

    Correction to line 355; change </UL> to </OL>.

commit 3e68c2e26cd89cb983eb52c024ecdfba1d6b3fff
Author: kwirth <kwirth@perforce.com>
Date:   Tue Jul 7 01:35:51 2009 -0800

    Fix spelling error on Jam doc page (cummulative -> cumulative).
```

现在导入已经准备好推送到你的新 Git 服务器上了。

### TFS

如果你的团队正在将他们的源代码管理从 TFVC 转换为 Git，你们会想要最高程度的无损转换。 这意味着，虽然我们在之前的交互章节介绍了 git-tfs 与 git-tf 两种工具，但是我们在本部分只能介绍 git-tfs，因为 git-tfs 支持分支，而使用 git-tf 代价太大。

**Note**

这是一个单向转换。 这意味着 Git 仓库无法连接到原始的 TFVC 项目。

第一件事是映射用户名。 TFVC 对待变更集作者字段的内容相当宽容，但是 Git 需要人类可读的名字与邮箱地址。 可以通过 tf 命令行客户端来获取这个信息，像这样：

```
PS> tf history $/myproject -recursive > AUTHORS_TMP
```

这会将历史中的所有变更集抓取下来并放到 AUTHORS_TMP 文件中，然后我们将会将 User 列（第二个）取出来。 打开文件找到列开始与结束的字符并替换，在下面的命令行中，cut 命令的参数 11-20 就是我们找到的：

```
PS> cat AUTHORS_TMP | cut -b 11-20 | tail -n+3 | uniq | sort > AUTHORS
```

cut 命令只会保留每行中第 11 个到第 22 个字符。 tail 命令会跳过前两行，就是字段表头与 ASCII 风格的下划线。 所有这些的结果通过管道送到 uniq 来去除重复，然后保存到 AUTOHRS 文件中。 下一步是手动的；为了让 git-tfs 有效地使用这个文件，每一行必须是这种格式：

```
DOMAIN\username = User Name <email@address.com>
```

左边的部分是 TFVC 中的 “User” 字段，等号右边的部分是将被用作 Git 提交的用户名。

一旦有了这个文件，下一件事就是生成一个你需要的 TFVC 项目的完整克隆：

```
PS> git tfs clone --with-branches --authors=AUTHORS https://username.visualstudio.com/DefaultCollection $/project/Trunk project_git
```

接下来要从提交信息底部清理 git-tfs-id 区块。 下面的命令会完成这个任务：

```
PS> git filter-branch -f --msg-filter 'sed "s/^git-tfs-id:.*$//g"' -- --all
```

那会使用 Git 终端环境中的 sed 命令来将所有以 “git-tfs-id:” 开头的行替换为 Git 会忽略的空白。

全部完成后，你就已经准备好去增加一个新的远程仓库，推送你所有的分支上去，然后你的团队就可以开始用 Git 工作了。

### 一个自定义的导入器

如果你的系统不是上述中的任何一个，你需要在线查找一个导入器 - 针对许多其他系统有很多高质量的导入器，包括 CVS、Clear Case、Visual Source Safe，甚至是一个档案目录。 如果没有一个工具适合你，需要一个不知名的工具，或者需要更大自由度的自定义导入过程，应当使用 git fast-import。 这个命令从标准输入中读取简单指令来写入特定的 Git 数据。 通过这种方式创建 Git 对象比运行原始 Git 命令或直接写入原始对象（查看 Git 内部原理 了解更多内容）更容易些。 通过这种方式你可以编写导入脚本，从你要导入的系统中读取必要数据，然后直接打印指令到标准输出。 然后可以运行这个程序并通过 git fast-import 重定向管道输出。

为了快速演示，我们会写一个简单的导入器。 假设你在 current 工作，有时候会备份你的项目到时间标签 back_YYYY_MM_DD 备份目录中，你想要将这些导入到 Git 中。 目录结构看起来是这样：

```
$ ls /opt/import_from
back_2014_01_02
back_2014_01_04
back_2014_01_14
back_2014_02_03
current
```

为了导入一个 Git 目录，需要了解 Git 如何存储它的数据。 你可能记得，Git 在底层存储指向内容快照的提交对象的链表。 所有要做的就是告诉 fast-import 哪些内容是快照，哪个提交数据指向它们，以及它们进入的顺序。 你的策略是一次访问一个快照，然后用每个目录中的内容创建提交，并且将每一个提交与前一个连接起来。

如同我们在 使用强制策略的一个例子 里做的，我们将会使用 Ruby 写这个，因为它是我们平常工作中使用的并且它很容易读懂。 可以使用任何你熟悉的东西来非常轻松地写这个例子 - 它只需要将合适的信息打印到 标准输出。 然而，如果你在 Windows 上，这意味着需要特别注意不要引入回车符到行尾 - git fast-import 非常特别地只接受换行符（LF）而不是 Windows 使用的回车换行符（CRLF）。

现在开始，需要进入目标目录中并识别每一个子目录，每一个都是你要导入为提交的快照。 要进入到每个子目录中并为导出它打印必要的命令。 基本主循环像这个样子：

```
last_mark = nil

# loop through the directories
Dir.chdir(ARGV[0]) do
  Dir.glob("*").each do |dir|
    next if File.file?(dir)

    # move into the target directory
    Dir.chdir(dir) do
      last_mark = print_export(dir, last_mark)
    end
  end
end
```

在每个目录内运行 print_export，将会拿到清单并标记之前的快照，然后返回清单并标记现在的快照；通过这种方式，可以将它们合适地连接在一起。 “标记” 是一个给提交标识符的 fast-import 术语；当你创建提交，为每一个提交赋予一个标记来将它与其他提交连接在一起。 这样，在你的 print_export 方法中第一件要做的事就是从目录名字生成一个标记：

```
mark = convert_dir_to_mark(dir)
```

可以创建一个目录的数组并使用索引做为标记，因为标记必须是一个整数。 方法类似这样：

```
$marks = []
def convert_dir_to_mark(dir)
  if !$marks.include?(dir)
    $marks << dir
  end
  ($marks.index(dir) + 1).to_s
end
```

既然有一个整数代表你的提交，那还要给提交元数据一个日期。 因为目录名字表达了日期，所以你将会从中解析出日期。 你的 print_export 文件的下一行是

```
date = convert_dir_to_date(dir)
convert_dir_to_date 定义为

def convert_dir_to_date(dir)
  if dir == 'current'
    return Time.now().to_i
  else
    dir = dir.gsub('back_', '')
    (year, month, day) = dir.split('_')
    return Time.local(year, month, day).to_i
  end
end
```

那会返回每一个目录日期的整数。 最后一项每个提交需要的元数据是提交者信息，它将会被硬编码在全局变量中：

```
$author = 'John Doe <john@example.com>'
```

现在准备开始为你的导入器打印出提交数据。 初始信息声明定义了一个提交对象与它所在的分支，紧接着一个你生成的标记、提交者信息与提交信息、然后是一个之前的提交，如果它存在的话。 代码看起来像这样：

```
# print the import information
puts 'commit refs/heads/master'
puts 'mark :' + mark
puts "committer #{$author} #{date} -0700"
export_data('imported from ' + dir)
puts 'from :' + last_mark if last_mark
```

我们将硬编码时区信息（-0700），因为这样很容易。 如果从其他系统导入，必须指定为一个偏移的时区。 提交信息必须指定为特殊的格式：

```
data (size)\n(contents)
```

这个格式包括文本数据、将要读取数据的大小、一个换行符、最终的数据。 因为之后还需要为文件内容指定相同的数据格式，你需要创建一个帮助函数，export_data：

```
def export_data(string)
  print "data #{string.size}\n#{string}"
end
```

剩下的工作就是指定每一个快照的文件内容。 这很轻松，因为每一个目录都是一个快照 - 可以在目录中的每一个文件内容后打印 deleteall 命令。 Git 将会适当地记录每一个快照：

```
puts 'deleteall'
Dir.glob("**/*").each do |file|
  next if !File.file?(file)
  inline_data(file)
end
```

注意：因为大多数系统认为他们的版本是从一个提交变化到另一个提交，fast-import 也可以为每一个提交执行命令来指定哪些文件是添加的、删除的或修改的与新内容是哪些。 可以计算快照间的不同并只提供这些数据，但是这样做会很复杂 - 也可以把所有数据给 Git 然后让它为你指出来。 如果这更适合你的数据，查阅 fast-import man 帮助页来了解如何以这种方式提供你的数据。

这种列出新文件内容或用新内容指定修改文件的格式如同下面的内容：

```
M 644 inline path/to/file
data (size)
(file contents)
```

这里，644 是模式（如果你有可执行文件，反而你需要检测并指定 755），inline 表示将会立即把内容放在本行之后。 你的 inline_data 方法看起来像这样：

```
def inline_data(file, code = 'M', mode = '644')
  content = File.read(file)
  puts "#{code} #{mode} inline #{file}"
  export_data(content)
end
```

可以重用之前定义的 export_data 方法，因为它与你定义的提交信息数据的方法一样。

最后一件你需要做的是返回当前的标记以便它可以传给下一个迭代：

```
return mark
```

**Note**

如果在 Windows 上还需要确保增加一个额外步骤。 正如之前提到的，Windows 使用 CRLF 作为换行符而 git fast-import 只接受 LF。 为了修正这个问题使 git fast-import 正常工作，你需要告诉 ruby 使用 LF 代替 CRLF：

$stdout.binmode
就是这样。 这是全部的脚本：

```
#!/usr/bin/env ruby

$stdout.binmode
$author = "John Doe <john@example.com>"

$marks = []
def convert_dir_to_mark(dir)
    if !$marks.include?(dir)
        $marks << dir
    end
    ($marks.index(dir)+1).to_s
end


def convert_dir_to_date(dir)
    if dir == 'current'
        return Time.now().to_i
    else
        dir = dir.gsub('back_', '')
        (year, month, day) = dir.split('_')
        return Time.local(year, month, day).to_i
    end
end

def export_data(string)
    print "data #{string.size}\n#{string}"
end

def inline_data(file, code='M', mode='644')
    content = File.read(file)
    puts "#{code} #{mode} inline #{file}"
    export_data(content)
end

def print_export(dir, last_mark)
    date = convert_dir_to_date(dir)
    mark = convert_dir_to_mark(dir)

    puts 'commit refs/heads/master'
    puts "mark :#{mark}"
    puts "committer #{$author} #{date} -0700"
    export_data("imported from #{dir}")
    puts "from :#{last_mark}" if last_mark

    puts 'deleteall'
    Dir.glob("**/*").each do |file|
        next if !File.file?(file)
        inline_data(file)
    end
    mark
end


# Loop through the directories
last_mark = nil
Dir.chdir(ARGV[0]) do
    Dir.glob("*").each do |dir|
        next if File.file?(dir)

        # move into the target directory
        Dir.chdir(dir) do
            last_mark = print_export(dir, last_mark)
        end
    end
end
```

如果运行这个脚本，你会得到类似下面的内容：

```
$ ruby import.rb /opt/import_from
commit refs/heads/master
mark :1
committer John Doe <john@example.com> 1388649600 -0700
data 29
imported from back_2014_01_02deleteall
M 644 inline README.md
data 28
# Hello

This is my readme.
commit refs/heads/master
mark :2
committer John Doe <john@example.com> 1388822400 -0700
data 29
imported from back_2014_01_04from :1
deleteall
M 644 inline main.rb
data 34
#!/bin/env ruby

puts "Hey there"
M 644 inline README.md
(...)
```

为了运行导入器，将这些输出用管道重定向到你想要导入的 Git 目录中的 git fast-import。 可以创建一个新的目录并在其中运行 git init 作为开始，然后运行你的脚本：

```
$ git init
Initialized empty Git repository in /opt/import_to/.git/
$ ruby import.rb /opt/import_from | git fast-import
git-fast-import statistics:
---------------------------------------------------------------------
Alloc'd objects:       5000
Total objects:           13 (         6 duplicates                  )
      blobs  :            5 (         4 duplicates          3 deltas of          5 attempts)
      trees  :            4 (         1 duplicates          0 deltas of          4 attempts)
      commits:            4 (         1 duplicates          0 deltas of          0 attempts)
      tags   :            0 (         0 duplicates          0 deltas of          0 attempts)
Total branches:           1 (         1 loads     )
      marks:           1024 (         5 unique    )
      atoms:              2
Memory total:          2344 KiB
       pools:          2110 KiB
     objects:           234 KiB
---------------------------------------------------------------------
pack_report: getpagesize()            =       4096
pack_report: core.packedGitWindowSize = 1073741824
pack_report: core.packedGitLimit      = 8589934592
pack_report: pack_used_ctr            =         10
pack_report: pack_mmap_calls          =          5
pack_report: pack_open_windows        =          2 /          2
pack_report: pack_mapped              =       1457 /       1457
---------------------------------------------------------------------
```

正如你所看到的，当它成功完成时，它会给你一串关于它完成内容的统计。 这本例中，一共导入了 13 个对象、4 次提交到 1 个分支。 现在，可以运行 git log 来看一下你的新历史：

```
$ git log -2
commit 3caa046d4aac682a55867132ccdfbe0d3fdee498
Author: John Doe <john@example.com>
Date:   Tue Jul 29 19:39:04 2014 -0700

    imported from current

commit 4afc2b945d0d3c8cd00556fbe2e8224569dc9def
Author: John Doe <john@example.com>
Date:   Mon Feb 3 01:00:00 2014 -0700

    imported from back_2014_02_03
```

做得很好 - 一个漂亮、干净的 Git 仓库。 要注意的一点是并没有检出任何东西 - 一开始你的工作目录内并没有任何文件。 为了得到他们，你必须将分支重置到 master 所在的地方：

```
$ ls
$ git reset --hard master
HEAD is now at 3caa046 imported from current
$ ls
README.md main.rb
```

可以通过 fast-import 工具做很多事情 - 处理不同模式、二进制数据、多个分支与合并、标签、进度指示等等。 一些更复杂情形下的例子可以在 Git 源代码目录中的 contrib/fast-import 目录中找到。