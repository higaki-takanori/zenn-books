---
title: "LESSコマンドの前処理はいつ設定される?"
emoji: "🕒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["shell", "command", "less"]
published: false
publication_name: "levtech"
---

この記事は [レバテック開発部 Advent Calendar 2025](https://qiita.com/advent-calendar/2025/levtech) 22日目の記事です。

# 結論

bashかつUbuntu環境では、`/etc/skel/.bashrc`に記載された`[ -x /usr/bin/lesspipe ] && eval "$(SHELL=/bin/sh lesspipe)"`によってデフォルトの前処理が設定される。

# lessコマンド

https://packages.debian.org/ja/sid/less

>一度に画面全体 にテキストを表示できるメモリ効率の良いユーティリティです。
> less には基本的な ページャ "more" よりも多くの機能があります。
> GNU プロジェクトの一部として、 このプログラムは UNIX 派生システムの標準的なページャと広くみなされています。

:::message
`less`にはいろんな機能がありますが、今回は前処理に関する部分に注目します。
:::

`less`は`more`コマンドより多くの機能を持ったページャです。

設定次第ですが、前処理によって対象のファイルが圧縮されているかに関わらずテキストの中身を確認できます。

## 環境

- Proxmox VE
    - Ubuntu 24.04

## まずはlessを実行してみる

普通のテキストファイルと圧縮されたファイルの両方を読み込んでみます。

まずは、適当なサンプルテキストを`less`で読み込んでみます。

```shell
$ echo "this is sample text" > sample.txt
$ less sample.txt
```

```shell
this is sample text
~
~
```

次に、圧縮した場合も見てみます。

```shell
$ gzip sample.txt
$ less sample.txt.gz
```

```shell
this is sample text
~
~
```

同様にsample.txtの内容が見れています。

## lessのマニュアルを見る

ここで、`less`のマニュアルを見てみます。

```shell
$ man less
```

今回の範囲に関係しそうな部分を抜粋すると、

```
INPUT PREPROCESSOR
       You may define an "input preprocessor" for less.  Before less opens a file, it first gives your input preprocessor a chance to modify the way the contents of the file are
       displayed.  An input preprocessor is simply an executable program (or shell script), which writes the contents of the file to a different file, called the replacement
       file.  The contents of the replacement file are then displayed in place of the contents of the original file.  However, it will appear to the user as if the original file
       is opened; that is, less will display the original filename as the name of the current file.
...

ENVIRONMENT VARIABLES

...

       LESSOPEN
              Command line to invoke the (optional) input-preprocessor.

```

`less`は圧縮ファイルの解凍など「less実行の前処理」を行うことができます。

どのような前処理を行うかは環境変数の`LESSOPEN`に記載されています。

:::message
`LESSSECURE`を設定してセキュリティモードにすることで、この辺りの挙動が変わりますがこの記事では範囲外とします。
:::

:::details DESCRIPTION

```
DESCRIPTION
       Less  is a program similar to more(1), but it has many more features.  Less does not have to read the entire input file before starting, so with large input files it starts up
       faster than text editors like vi(1).  Less uses termcap (or terminfo on some systems), so it can run on a variety of terminals.  There is even  limited  support  for  hardcopy
       terminals.  (On a hardcopy terminal, lines which should be printed at the top of the screen are prefixed with a caret.)
```

:::

:::details INPUT PREPROCESSOR

```
INPUT PREPROCESSOR
       You may define an "input preprocessor" for less.  Before less opens a file, it first gives your input preprocessor a chance to modify the way the contents of the file are
       displayed.  An input preprocessor is simply an executable program (or shell script), which writes the contents of the file to a different file, called the replacement
       file.  The contents of the replacement file are then displayed in place of the contents of the original file.  However, it will appear to the user as if the original file
       is opened; that is, less will display the original filename as the name of the current file.

       An input preprocessor receives one command line argument, the original filename, as entered by the user.  It should create the replacement file, and when finished, print
       the name of the replacement file to its standard output.  If the input preprocessor does not output a replacement filename, less uses the original file, as normal.  The
       input preprocessor is not called when viewing standard input.  To set up an input preprocessor, set the LESSOPEN environment variable to a command line which will invoke
       your input preprocessor.  This command line should include one occurrence of the string "%s", which will be replaced by the filename when the input preprocessor command
       is invoked.

       When less closes a file opened in such a way, it will call another program, called the input postprocessor, which may perform any desired clean-up action (such as
       deleting the replacement file created by LESSOPEN).  This program receives two command line arguments, the original filename as entered by the user, and the name of the
       replacement file.  To set up an input postprocessor, set the LESSCLOSE environment variable to a command line which will invoke your input postprocessor.  It may include
       two occurrences of the string "%s"; the first is replaced with the original name of the file and the second with the name of the replacement file, which was output by
       LESSOPEN.

       For example, on many Unix systems, these two scripts will allow you to keep files in compressed format, but still let less view them directly:

       lessopen.sh:
            #! /bin/sh
            case "$1" in
            *.Z) TEMPFILE=$(mktemp)
                 uncompress -c $1  >$TEMPFILE  2>/dev/null
                 if [ -s $TEMPFILE ]; then
                      echo $TEMPFILE
                 else
                      rm -f $TEMPFILE
                 fi
                 ;;
            esac

       lessclose.sh:
            #! /bin/sh
            rm $2

       To use these scripts, put them both where they can be executed and set LESSOPEN="lessopen.sh %s", and LESSCLOSE="lessclose.sh %s %s".  More complex LESSOPEN and LESSCLOSE
       scripts may be written to accept other types of compressed files, and so on.
 
        It is also possible to set up an input preprocessor to pipe the file data directly to less, rather than putting the data into a replacement file.  This avoids the need to
       decompress the entire file before starting to view it.  An input preprocessor that works this way is called an input pipe.  An input pipe, instead of writing the name of
       a replacement file on its standard output, writes the entire contents of the replacement file on its standard output.  If the input pipe does not write any characters on
       its standard output, then there is no replacement file and less uses the original file, as normal.  To use an input pipe, make the first character in the LESSOPEN
       environment variable a vertical bar (|) to signify that the input preprocessor is an input pipe.  As with non-pipe input preprocessors, the command string must contain
       one occurrence of %s, which is replaced with the filename of the input file.
 
       For example, on many Unix systems, this script will work like the previous example scripts:
 
       lesspipe.sh:
            #! /bin/sh
            case "$1" in
            *.Z) uncompress -c $1  2>/dev/null
                 ;;
            *)   exit 1
                 ;;
            esac
            exit $?
 
       To use this script, put it where it can be executed and set LESSOPEN="|lesspipe.sh %s".
 
       Note that a preprocessor cannot output an empty file, since that is interpreted as meaning there is no replacement, and the original file is used.  To avoid this, if
       LESSOPEN starts with two vertical bars, the exit status of the script becomes meaningful.  If the exit status is zero, the output is considered to be replacement text,
       even if it is empty.  If the exit status is nonzero, any output is ignored and the original file is used.  For compatibility with previous versions of less, if LESSOPEN
       starts with only one vertical bar, the exit status of the preprocessor is ignored.
 
       When an input pipe is used, a LESSCLOSE postprocessor can be used, but it is usually not necessary since there is no replacement file to clean up.  In this case, the
       replacement file name passed to the LESSCLOSE postprocessor is "-".
 
       For compatibility with previous versions of less, the input preprocessor or pipe is not used if less is viewing standard input.  However, if the first character of
       LESSOPEN is a dash (-), the input preprocessor is used on standard input as well as other files.  In this case, the dash is not considered to be part of the preprocessor
       command.  If standard input is being viewed, the input preprocessor is passed a file name consisting of a single dash.  Similarly, if the first two characters of LESSOPEN
       are vertical bar and dash (|-) or two vertical bars and a dash (||-), the input pipe is used on standard input as well as other files.  Again, in this case the dash is
       not considered to be part of the input pipe command.
```

:::

:::details SECURITY

```
SECURITY
       When the environment variable LESSSECURE is set to 1, less runs in a "secure" mode.  This means these features are disabled:

              !      the shell command

              |      the pipe command

              :e     the examine command.

              v      the editing command

              s  -o  log files

              -k     use of lesskey files

              -t     use of tags files

                     metacharacters in filenames, such as *

                     filename completion (TAB, ^L)

       Less can also be compiled to be permanently in "secure" mode.
```

:::

:::details ENVIRONMENT VARIABLES

```
ENVIRONMENT VARIABLES
       Environment variables may be specified either in the system environment as usual, or in a lesskey(1) file.  If environment variables are defined in more than one place,
       variables defined in a local lesskey file take precedence over variables defined in the system environment, which take precedence over variables defined in the system-
       wide lesskey file.

       COLUMNS
              Sets the number of columns on the screen.  Takes precedence over the number of columns specified by the TERM variable.  (But if you have a windowing system which
              supports TIOCGWINSZ or WIOCGETD, the window system's idea of the screen size takes precedence over the LINES and COLUMNS environment variables.)

       EDITOR The name of the editor (used for the v command).

       HOME   Name of the user's home directory (used to find a lesskey file on Unix and OS/2 systems).

       HOMEDRIVE, HOMEPATH
              Concatenation of the HOMEDRIVE and HOMEPATH environment variables is the name of the user's home directory if the HOME variable is not set (only in the Windows
              version).

       INIT   Name of the user's init directory (used to find a lesskey file on OS/2 systems).

       LANG   Language for determining the character set.

       LC_CTYPE
              Language for determining the character set.

       LESS   Options which are passed to less automatically.

       LESSANSIENDCHARS
              Characters which may end an ANSI color escape sequence (default "m").

       LESSANSIMIDCHARS
              Characters which may appear between the ESC character and the end character in an ANSI color escape sequence (default "0123456789:;[?!"'#%()*+ ".

       LESSBINFMT
              Format for displaying non-printable, non-control characters.

       LESSCHARDEF
              Defines a character set.
 
       LESSCHARSET
              Selects a predefined character set.
       LESSCLOSE
              Command line to invoke the (optional) input-postprocessor.
 
       LESSECHO
              Name of the lessecho program (default "lessecho").  The lessecho program is needed to expand metacharacters, such as * and ?, in filenames on Unix systems.
 
       LESSEDIT
              Editor prototype string (used for the v command).  See discussion under PROMPTS.
 
       LESSGLOBALTAGS
              Name of the command used by the -t option to find global tags.  Normally should be set to "global" if your system has the global(1) command.  If not set, global
              tags are not used.
 
       LESSHISTFILE
              Name of the history file used to remember search commands and shell commands between invocations of less.  If set to "-" or "/dev/null", a history file is not
              used.  The default is "$HOME/.lesshst" on Unix systems, "$HOME/_lesshst" on DOS and Windows systems, or "$HOME/lesshst.ini" or "$INIT/lesshst.ini" on OS/2 systems.
 
       LESSHISTSIZE
              The maximum number of commands to save in the history file.  The default is 100.
 
       LESSKEY
              Name of the default lesskey(1) file.
 
       LESSKEY_SYSTEM
              Name of the default system-wide lesskey(1) file.
 
       LESSMETACHARS
              List of characters which are considered "metacharacters" by the shell.
 
       LESSMETAESCAPE
              Prefix which less will add before each metacharacter in a command sent to the shell.  If LESSMETAESCAPE is an empty string, commands containing metacharacters will
              not be passed to the shell.
 
       LESSOPEN
              Command line to invoke the (optional) input-preprocessor.
 
       LESSSECURE
              Runs less in "secure" mode.  See discussion under SECURITY.
        LESSSEPARATOR
              String to be appended to a directory name in filename completion.
 
       LESSUTFBINFMT
              Format for displaying non-printable Unicode code points.
 
       LESS_IS_MORE
              Emulate the more(1) command.
 
       LINES  Sets the number of lines on the screen.  Takes precedence over the number of lines specified by the TERM variable.  (But if you have a windowing system which
              supports TIOCGWINSZ or WIOCGETD, the window system's idea of the screen size takes precedence over the LINES and COLUMNS environment variables.)
 
       MORE   Options which are passed to less automatically when running in more compatible mode.
 
       PATH   User's search path (used to find a lesskey file on MS-DOS and OS/2 systems).
 
       SHELL  The shell used to execute the ! command, as well as to expand filenames.
 
       TERM   The type of terminal on which less is being run.
 
       VISUAL The name of the editor (used for the v command).
```

:::

## `LESSOPEN`を見てみる

では`LESSOPEN`に何が指定されているか見てみます。

```shell
$ env | grep LESSOPEN
LESSOPEN=| /usr/bin/lesspipe %s
```

環境によって違うと思いますが、自分の環境では上記が設定されていました。

自分の環境では`less`の前処理には`lesspipe`を使用していそうです。

https://sources.debian.org/src/less/668-1/debian/lesspipe

しかし、自分には`LESSOPEN`を設定した記憶はありません。どのタイミングで設定されていたのでしょうか。

## `LESSOPEN`はいつ設定された?

ここで、`/etc/skel/.bashrc`を見てみると、 以下の記載がありました。

```shell
$ cat /etc/skel/.bashrc | grep lesspipe
# make less more friendly for non-text input files, see lesspipe(1)
[ -x /usr/bin/lesspipe ] && eval "$(SHELL=/bin/sh lesspipe)"
```

`/etc/skel`とは、「新しく作成するユーザのホームディレクトリの雛形」なので、Ubuntuにユーザを追加した時点で`.bashrc`に設定されています。

**参考**

https://envader.plus/course/12/scenario/1129

### lesspipe

まずは`lesspipe`について実行してみます。

`SHELL=/bin/sh lesspipe`は環境変数`SHELL`に`/bin/sh`を設定して、`lesspipe`を実行するコマンドです。

```shell
$ SHELL=/bin/sh lesspipe
export LESSOPEN="| /usr/bin/lesspipe %s";
export LESSCLOSE="/usr/bin/lesspipe %s %s";
```

どうやら`lesspipe`は引数なしで実行すると「環境変数に`LESSOPEN`と`LESSCLOSE`を設定するコマンド」が出力されるようです。

:::details 引数が0個の場合のlesspipe

`$SHELL=/bin/sh` を設定しているので、`*)`に該当して、`$BASENAME`は`lessfile`ではないので、以下が実行されます。

```shell
echo "export LESSOPEN=\"| $FULLPATH %s\";"
echo "export LESSCLOSE=\"$FULLPATH %s %s\";"
```

```shell
..

BASENAME=`basename $0`
LESSFILE=lessfile

..

elif [ $# -eq 0 ] ; then
	#
	# must setup shell to use LESSOPEN/LESSCLOSE
	#
	# I have no idea how some of the more esoteric shells (es, rc) do
	# things. If they don't do things in a Bourne manner, send me a patch
	# and I'll incorporate it.
	#

	# first determine the full path of lessfile/lesspipe
	# if you can determine a better way to do this, send me a patch, I've
	# not shell-scripted for many a year.
	FULLPATH=`cd \`dirname $0\`;pwd`/$BASENAME

	case "$SHELL" in
		*csh)
			if [ $BASENAME = $LESSFILE ]; then
				echo "setenv LESSOPEN \"$FULLPATH %s\";"
				echo "setenv LESSCLOSE \"$FULLPATH %s %s\";"
			else
				echo "setenv LESSOPEN \"| $FULLPATH %s\";"
				echo "setenv LESSCLOSE \"$FULLPATH %s %s\";"
			fi
			;;
		*)
			if [ $BASENAME = $LESSFILE ]; then
				echo "export LESSOPEN=\"$FULLPATH %s\";"
				echo "export LESSCLOSE=\"$FULLPATH %s %s\";"
			else
				echo "export LESSOPEN=\"| $FULLPATH %s\";"
				echo "export LESSCLOSE=\"$FULLPATH %s %s\";"
			fi
			;;
	esac

	#echo "# If you tried to view a file with a name that starts with '#', you"
	#echo "# might see this message instead of the file's contents."
	#echo "# To view the contents, try to put './' ahead of the filename when"
	#echo "# calling less."
```

https://sources.debian.org/src/less/668-1/debian/lesspipe

:::

ただこちらは標準出力に表示されただけで実際にこのコマンドが実行されたわけではありません。

### eval

これらは`eval`で実際に実行されます。

```shell
$ eval "$(SHELL=/bin/sh lesspipe)"

実質こうなる

$ export LESSOPEN="| /usr/bin/lesspipe %s";
$ export LESSCLOSE="/usr/bin/lesspipe %s %s";
```

これで、`LESSOPEN`と`LESSCLOSE`が環境変数に設定されました。

### 条件評価`[`

最後に `[ -x /usr/bin/lesspipe ]` の部分を見てみます。

`[`もコマンドなので、`man [`でマニュアルを見てみます。

```shell
$ man [
```

```
NAME
     test, [ – condition evaluation utility
     
SYNOPSIS
     test expression
     [ expression ]

DESCRIPTION
     The test utility evaluates the expression and, if it evaluates to true, returns a zero (true) exit status; otherwise it returns 1 (false).  If there is no expression, test also returns 1 (false).

     All operators and flags are separate arguments to the test utility.

     The following primaries are used to construct expression:
...          
        -x file       True if file exists and is executable.  True indicates only that the execute flag is on.  If file is a directory, true indicates that file can be searched.
```

条件を評価するコマンドとして動作するようです。

`-x`オプションをつけると、ファイルが存在する、かつ、実行可能であるときに`true`を返します。

```shell
$ [ -x /usr/bin/lesspipe ]
```

こちらは`/usr/bin/lesspipe`が存在し、実行可能である場合に`true`を返すコマンドとなります。


### 処理の流れ

最後に`.bashrc`の記載をまとめると、以下のようになります。

```shell
[ -x /usr/bin/lesspipe ] && eval "$(SHELL=/bin/sh lesspipe)"
```

`/usr/bin/lesspipe`が存在し、実行可能である場合に`LESSOPEN`と`LESSCLOSE`を環境変数に設定するコマンドが実行されます。

処理の流れを記載していくと、

```shell
[ -x /usr/bin/lesspipe ] && eval "$(SHELL=/bin/sh lesspipe)"
```
```shell:/usr/bin/lesspipeが存在し、実行可能である
true && eval "$(SHELL=/bin/sh lesspipe)"
```
```shell:lesspipeが引数0個で実行される
eval "$(SHELL=/bin/sh lesspipe)"
```
```shell
export LESSOPEN="| /usr/bin/lesspipe %s";
export LESSCLOSE="/usr/bin/lesspipe %s %s";
```

## 圧縮ファイルの内容を確認する処理

せっかくなので、`lesspipe`がどのように圧縮ファイルの内容を確認しているかも見てみます。

```shell
$ lesspipe sample.txt.gz
this is sample text
```

```shell:引数が1つのlesspipeの処理抜粋
..

if [ $# -eq 1 ] ; then
	# we were called as LESSOPEN
        ..
	(
	        ..
		# Decode file for less
		case `echo "$1" | tr '[:upper:]' '[:lower:]'` in
		
		        ..
                                
			# Note that this is out of alpha order so that we don't catch
			# the gzipped tar files.
			*.gz|*.z|*.dz)
				gzip -dc "$1" ;;

		        ..            
		esac
	) 2>/dev/null	
..
```

まず、`echo "$1" | tr '[:upper:]' '[:lower:]'`の部分で、大文字のファイル名も処理できるようにファイル名の大文字を小文字に揃えています。

```shell
$ echo sample.txt.GZ | tr '[:upper:]' '[:lower:]'
sample.txt.gz
```

次に、実際に解凍する処理です。

今回の例だと`*.gz` に該当する部分が実行されています。

```shell
$ gzip -dc sample.txt.gz
this is sample text
```

この出力を`less`コマンドのプロセスに渡すことで、`less`コマンドは圧縮ファイルも処理できるようになっています。

# まとめ

`less`コマンドの前処理がどこで設定されているかを見ていきました。

この記事が誰かの参考になれば幸いです。

# 参考

https://qiita.com/jmatsuzawa/items/0cb53a5e555652ae78d3

https://sources.debian.org/src/less/668-1/debian/lesspipe

https://packages.debian.org/ja/sid/less

https://envader.plus/course/12/scenario/1129
