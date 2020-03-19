---
layout: post
title:  "【译文】Shell 编码指南"
date:   2020-3-16 17:42:49 +0800
categories: sre linux
---

修订版 2.02
由许多 Google 员工撰写、修订和维护。

# 背景

## 使用哪种 Shell

Bash 是可执行文件唯一允许的 Shell 脚本语言。

可执行文件必须以 `#!/bin/bash` 和最少的标志开头。使用 `set` 来设置 Shell 程序选项，以便以 `bash_script_name` 调用脚本而不会破坏其功能。

将所有可执行的 Shell 脚本限制为 *bash*，可以使我们在所有计算机上都安装一致的 Shell 语言环境。

唯一的例外是，无论您使用哪种编码方式，都会被迫进入。其中一个示例是 Solaris SVR4 软件包，该软件包需要使用纯 Bourne Shell 编写任何脚本。

## 何时使用 Shell

Shell 仅应用于小型使用程序或简单的包装器脚本。

虽然 Shell 脚本并不是一种开发语言，但可用于在整个 Google 上编写各种实用程序脚本。该规范更多地是对其使用的认可，而不是建议将其广泛使用。

一些指南：

* 如果您主要是在调用其他使用程序，并且进行的数据操作相对较少，那么 Shell 是执行次任务的可接受选择。
* 如果性能很重要，请使用非 Shell 的东西。
* 如果编写的脚本长度超过 100 行，或使用非直接控制流逻辑，则应*立即*以结构化的语言重写它。请记住，脚本会不断增长。尽早重写脚本，以避免以后花费更多时间进行重写。
* 在评估代码的复杂性时（例如，决定是否切换语言），请考虑改代码是否易于由作者之外的其他人维护。

# Shell 文件和解释器调用

## 文件扩展名

可执行文件不应具有扩展名（强烈推荐）或 `.sh` 扩展名。库必须具有 `.sh` 扩展名，并且不能执行。

不必在执行程序时知道用哪种语言编写程序，而且 Shell 不需要扩展名，因此我们不希望对可执行文件使用扩展名。

但是，对于库来说，重要的是要知道它是什么语言，有时还需要在相似的库中使用不同的语言。这允许用途相同但语言不同的库文件以相同的名称命名（除了特定于语言的后缀）。

## SUID/SGID

Shell 脚本上*禁止* SUID 和 SGID。

Shell 存在太多的安全问题，几乎不可能完全安全地允许 SUID/SGID。

虽然 bash 确实使运行 SUID 变得困难，但是在某些平台上仍然有可能，这就是为什么我们明确禁止使用它。

如果需要，请使用 `sudo` 提升访问权限。

# 环境

## STDOUT vs STDERR

所有的错误消息都应发送至 `STDERR`。

这样可以更轻松地将正常状态与实际问题区分开。

建议使用打印错误消息以及其他状态信息的函数。

```bash
err() {
  echo "[$(date +'%Y-%m-%dT%H:%M:%S%z')]: $*" >&2
}

if ! do_something; then
  err "Unable to do_something"
  exit 1
fi
```

# 注释

## 文件头

在每个文件的开头加上其内容的描述。

每个文件都必须具有顶部注释，包括其内容的简要概述。版权声明和作者信息是可选的。

例如：

```bash
#!/bin/bash
#
# Perform hot backups of Oracle databases.
```

## 函数注释

任何既不明显也不简短的功能必须加以注释。无论长度或复杂性如何，都必须对库中的所有函数进行注释。

他人无需阅读代码即可通过阅读注释（和自助服务，如果提供）来学习如何使用您的程序或在库中使用函数。

所有函数注释都应使用一下方式描述预期的 API 行为：

* 功能说明。
* 全局变量：使用和修改的全局变量列表。
* 参数：采用的参数。
* 输出：输出到 STDOUT 或 STDERR。
* 返回值：返回的值不是上一次运行命令的默认退出状态。

示例：

```bash
#######################################
# Cleanup files from the backup directory.
# Globals:
#   BACKUP_DIR
#   ORACLE_SID
# Arguments:
#   None
#######################################
function cleanup() {
  …
}

#######################################
# Get configuration directory.
# Globals:
#   SOMEDIR
# Arguments:
#   None
# Outputs:
#   Writes location to stdout
#######################################
function get_dir() {
  echo "${SOMEDIR}"
}

#######################################
# Delete a file in a sophisticated manner.
# Arguments:
#   File to delete, a path.
# Returns:
#   0 if thing was deleted, non-zero on error.
#######################################
function del_thing() {
  rm "$1"
}
```

## 实施注释

注释代码中棘手、不明显、有趣或重要的部分。

这遵循一般的 Google 编码注释惯例。不要注释所有内容。如果算法复杂，或者您要执行的操作与众不同，请在此处简短说明。

## TODO 注释

对于临时的、短期的解决方案或足够好但不完美的代码，请使用 TODO 注释。

这符合 [C++ 指南](https://google.github.io/styleguide/cppguide.html#TODO_Comments)中的约定。

`TODO` 应在所有大写字母中包含字符串 `TODO`，然后是与该 `TODO` 所引用问题最相关的人员的姓名、电子邮件地址或其他标识符。主要目的是要有一个一致的 `TODO`，可以对其进行搜索以找到如何根据请求获取更多详细信息。`TODO` 并不表示相关人员将解决问题。因此，在创建 TODO 时，几乎总是给出您的名字。

示例：

```bash
# TODO(mrmonkey): Handle the unlikely edge cases (bug ####)
```

# 格式化

虽然您应该遵循要修改的文件已存在的样式，但是任何新代码都需要遵循以下规范。

## 缩进

缩进 2 个空格。不使用 tab。

在块之间使用空行以提高可读性。缩进是两个空格。无论您做什么，都不要使用标签。对于现有文件，请遵循现有缩进。

## 长行和长字符串

行的最大长度为 80 个字符。

如果必须编写长度唱过 80 个字符的字符串，则应使用 here 文档或嵌入的换行符来完成。长度必须超过 80 个字符且不能明智地拆分的文字字符串是可以的，但是强烈建议您找到一种方法来缩短它。

```bash
# DO use 'here document's
cat <<END
I am an exceptionally long
string.
END

# Embedded newlines are ok too
long_string="I am an exceptionally
long string."
```

## 管道

如果管道不能全部容纳在一行上，则应将每条管道分开到一行上。

如果管道适合放在一行上，那么它应该在一行上。

如果不是，则应在每条线上的一个管道符（|）前将其拆分，并在换行符后添加管道符（|），并在管道前留出 2 个空格。这适用于使用 `|` 组合的命令链，以及使用 `||` 和 `&&` 的逻辑符。

```bash
# All fits on one line
command1 | command2

# Long commands
command1 \
  | command2 \
  | command3 \
  | command4
```

---

# 备注

* 原文：[https://google.github.io/styleguide/shellguide.html](https://google.github.io/styleguide/shellguide.html)

