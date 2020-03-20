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

## 循环

将 `; do` 和 `; then` 放在 `while`、`for` 或 `if` 的同一行。

Shell 中的循环有些不同，但是在声明函数时，我们遵循与花括号相同的原则。也就是说，`; then` 和 `; do` 应该和 `if/for/while` 在同一行。`else` 则应独占一行，并且闭合语句应与打开语句垂直对齐。

例如：

```bash
# If inside a function, consider declaring the loop variable as
# a local to avoid it leaking into the global environment:
# local dir
for dir in "${dirs_to_cleanup[@]}"; do
  if [[ -d "${dir}/${ORACLE_SID}" ]]; then
    log_date "Cleaning up old files in ${dir}/${ORACLE_SID}"
    rm "${dir}/${ORACLE_SID}/"*
    if (( $? != 0 )); then
      error_message
    fi
  else
    mkdir -p "${dir}/${ORACLE_SID}"
    if (( $? != 0 )); then
      error_message
    fi
  fi
done
```

## Case 语句

* 选项缩进两个空格。
* 单行选项在匹配项的圆括号后和 `;;` 之前需要一个空格。
* 长命令或命令选项应将匹配项、操作和 `;;` 分成多行，分别占据一行。

匹配的表达式比 `case` 和 `esac` 缩进一级。多行命令再次缩进一级。通常无需引用匹配表达式。模式表达式不应在圆括号之前。避免使用 `;&` 和 `;;&`。

```bash
case "${expression}" in
  a)
    variable="…"
    some_command "${variable}" "${other_expr}" …
    ;;
  absolute)
    actions="relative"
    another_command "${actions}" "${other_expr}" …
    ;;
  *)
    error "Unexpected expression '${expression}'"
    ;;
esac
```

简单的命令可以与匹配项和 `;;` 放在同一行。只要表达式能够保持可读性即可。这通常使用用单字母现象处理。当命令不能单行显示时，将选项单独放在一行上，然后是命令，然后是 `;;`。与命令在同一行上时，请在模式的右括号后使用空格，在 `;;` 之前使用另一个空格。

```bash
verbose='false'
aflag=''
bflag=''
files=''
while getopts 'abf:v' flag; do
  case "${flag}" in
    a) aflag='true' ;;
    b) bflag='true' ;;
    f) files="${OPTARG}" ;;
    v) verbose='true' ;;
    *) error "Unexpected option ${flag}" ;;
  esac
done
```

## 变量范围

优先顺序：与发现的内容保持一致；引用您的变量；优先使用 `"${var}"` 而不是 `"$var"`。

这些是强烈建议的准则，而不是强制性法规。不过，这是一项推荐而非强制性要求，并不意味着应轻视或轻描淡写。

它们按优先顺序列出。

* 与现有代码的格式保持一致。
* 引用变量，请参见下面的引用部分。
* 不要用大括号分隔单个字符的 shell 特殊字符/位置参数，除非严格要求，否则请避免造成深深的困惑。

最好用大括号分隔所有其他变量。

```bash
# Section of *recommended* cases.

# Preferred style for 'special' variables:
echo "Positional: $1" "$5" "$3"
echo "Specials: !=$!, -=$-, _=$_. ?=$?, #=$# *=$* @=$@ \$=$$ …"

# Braces necessary:
echo "many parameters: ${10}"

# Braces avoiding confusion:
# Output is "a0b0c0"
set -- a b c
echo "${1}0${2}0${3}0"

# Preferred style for other variables:
echo "PATH=${PATH}, PWD=${PWD}, mine=${some_var}"
while read -r f; do
  echo "file=${f}"
done < <(find /tmp)
# Section of *discouraged* cases

# Unquoted vars, unbraced vars, brace-delimited single letter
# shell specials.
echo a=$avar "b=$bvar" "PID=${$}" "${1}"

# Confusing use: this is expanded as "${1}0${2}0${3}0",
# not "${10}${20}${30}
set -- a b c
echo "$10$20$30"
```

**注意**：在 `{var}` 中使用大括号不是引用形式。还必须使用双引号。

## 引用

* 始终引用包含变量、命令替换、空格或 shell 元字符的字符串，除非需要仔细的无引号扩展或它是 shell 程序内部的证书（请参阅下一点）。
* 使用数组来安全引用元素列表，尤其是命令行标志。请参阅下面的数组。
* （可选）引用定义为整数的 shell 内部制度特殊变量：`$?`、`$#`、`$$`、`$!`（man bash）。最好引用“命名”内部整数变量，例如 PPID 以保持一致性。
* 最好用引号引起来的字符串是“单词”（与命令选项或路径名相反）。
* 请勿引用文字整数。
* 请注意 `[[...]]` 中模式匹配的引用规则。请参见下面的测试，`[...]` 和 `[[...]]` 部分。
* 除非您有特定的原因要使用 `$*`，否则请使用 `"$@"`，例如仅将参数附加到消息或日志中的字符串上。

```bash
# 'Single' quotes indicate that no substitution is desired.
# "Double" quotes indicate that substitution is required/tolerated.

# Simple examples

# "quote command substitutions"
# Note that quotes nested inside "$()" don't need escaping.
flag="$(some_command and its args "$@" 'quoted separately')"

# "quote variables"
echo "${flag}"

# Use arrays with quoted expansion for lists.
declare -a FLAGS
FLAGS=( --foo --bar='baz' )
readonly FLAGS
mybinary "${FLAGS[@]}"

# It's ok to not quote internal integer variables.
if (( $# > 3 )); then
  echo "ppid=${PPID}"
fi

# "never quote literal integers"
value=32
# "quote command substitutions", even when you expect integers
number="$(generate_number)"

# "prefer quoting words", not compulsory
readonly USE_INTEGER='true'

# "quote shell meta characters"
echo 'Hello stranger, and well met. Earn lots of $$$'
echo "Process $$: Done making \$\$\$."

# "command options or path names"
# ($1 is assumed to contain a value here)
grep -li Hugo /dev/null "$1"

# Less simple examples
# "quote variables, unless proven false": ccs might be empty
git send-email --to "${reviewers}" ${ccs:+"--cc" "${ccs}"}

# Positional parameter precautions: $1 might be unset
# Single quotes leave regex as-is.
grep -cP '([Ss]pecial|\|?characters*)$' ${1:+"$1"}

# For passing on arguments,
# "$@" is right almost every time, and
# $* is wrong almost every time:
#
# * $* and $@ will split on spaces, clobbering up arguments
#   that contain spaces and dropping empty strings;
# * "$@" will retain arguments as-is, so no args
#   provided will result in no args being passed on;
#   This is in most cases what you want to use for passing
#   on arguments.
# * "$*" expands to one argument, with all args joined
#   by (usually) spaces,
#   so no args provided will result in one empty string
#   being passed on.
# (Consult `man bash` for the nit-grits ;-)

(set -- 1 "2 two" "3 three tres"; echo $#; set -- "$*"; echo "$#, $@")
(set -- 1 "2 two" "3 three tres"; echo $#; set -- "$@"; echo "$#, $@")
```

# 特性和 Bug


---

# 备注

* 原文：[https://google.github.io/styleguide/shellguide.html](https://google.github.io/styleguide/shellguide.html)

