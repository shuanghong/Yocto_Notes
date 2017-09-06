# BitBake 语法

详细参考 [http://www.yoctoproject.org/docs/current/bitbake-user-manual/bitbake-user-manual.html](http://www.yoctoproject.org/docs/current/bitbake-user-manual/bitbake-user-manual.html)


## 函数

在大多数语言中, 函数是构建块用来累积操作到 task 中, BitBake 支持以下几种类型的函数:

- Shell Functions: 写在 shell 脚本中, 直接以函数或者以 task 执行, 也可以被其他shell函数调用.
- BitBake-Style Python Functions: 写在Python中, 被 BitBake 执行或者被其他Python函数使用 bb.build.exec_func() 执行.
- Python Functions: 写在Python中并且被Python执行.
- Anonymous(匿名) Python Functions: 在解析过程中自动执行.

无论哪种类型的函数, 只能定义在 class 文件(.bbclass) 和 recipe 文件中(.bb or .inc).

### Shell Functions

例如:

	some_function () {
    	echo "Hello World"
     }

Shell 函数要遵循 shell 编程规则, 脚本被 /bin/sh 执行, /bin/sh 有可能是 bash shell或者dash, 不应该使用 bash特定的脚本.  
重写操作符如 _append, _prepend 同样可用于shell函数. 常用于 .bbappend 文件用来修改主 recipe中的函数, 也可用来修改从 classes 继承来的函数. 例如:
	
	 do_foo() {
         bbplain first //bbplain = bb.plain(msg): Writes msg as is to the log while also logging to stdout.
         fn				// 调用 fn 函数
     }

     fn_prepend() {
         bbplain second
     }

     fn() {
         bbplain third
     }

     do_foo_append() {
         bbplain fourth
     }

运行 do_foo 打印如下:
	
	recipename do_foo: first
    recipename do_foo: second
    recipename do_foo: third
    recipename do_foo: fourth

可以使用 "bitbake -e recipename" 命令查看应用重写之后最终组合的函数.

### BitBake-Style Python Functions

这些函数用Python来写, 被BitBake执行或者其他Python函数使用 bb.build.exec_func()执行. 例如:

	 python some_python_function () {
         d.setVar("TEXT", "Hello World")
         print d.getVar("TEXT")
     }

此类函数中, Python的 "bb" 和 "os" 模块已经被自动 import, 同时 datastore ("d")是全局变量, 总是自动可用.

类似于 shell functions, 你可以使用重写操作符, 如下:

	 python do_foo_prepend() {
         bb.plain("first")
     }
     python do_foo() {
         bb.plain("second")
     }
     python do_foo_append() {
         bb.plain("third")
     }

运行 do_foo 打印如下:

     recipename do_foo: first
     recipename do_foo: second
     recipename do_foo: third

可以使用 "bitbake -e recipename" 命令查看应用重写之后最终组合的函数.

### Python Functions

这些函数用Python来写, 并被其他Python代码执行. 例如:

     def get_depends(d):
     if d.getVar('SOMECONDITION'):
         return "dependencywithcond"
     else:
         return "dependency"

     SOMECONDITION = "1"
     DEPENDS = "${@get_depends(d)}"

执行之后, DEPENDS 将包含 dependencywithcond.

此类函数能带参数,  "bb" 和 "os" 模块已经被自动 import, 但是 datastore 不是自动可用的, 因此必须作为一个参数传递给函数.

### Bitbake-Style Python Functions VS Python Functions

下面是BitBake-style Python函数与 def 定义的正式的Python函数的区别:

- 只有 BitBake-style Python函数可以是 task.
- 重写操作符只能用于 BitBake-style Python.
- 只有正式的Python函数能带参数和返回值.
- Variable flags 如[dirs], [cleandirs], [lockfiles] 可以用在BitBake-style Python函数上, 不能用于正式的Python函数.
- BitBake-style Python函数生成一个单独的 ${T}/run.function-name.pid 来执行函数. 如果作为一个 task执行, 产生一个log文件在 ${T}/log.function-name.pid; 正式的Python函数内联执行, 不产生任何文件.
- 正式的Python函数被Python语法调用, BitBake-style Python函数被 BitBake直接调用或者被其他Python代码使用 bb.build.exec_func()调用, 如
	
		bb.build.exec_func("my_bitbake_style_function", d)

### Anonymous(匿名) Python Functions

用于在recipe解析期间设置变量或者执行其他操作. 例如:

     python () {
         if d.getVar('SOMEVAR') == 'value':
             d.setVar('ANOTHERVAR', 'value2')
     }

Anonymous Python 函数总是在解析结束后执行, 无论他们定义在什么位置. 如果一个 recipe 包含多个匿名函数, 他们将以在 recipe中定义的顺序执行, 如下:

     python () {
         d.setVar('FOO', 'foo 2')
     }

     FOO = "foo 1"

     python () {
         d.appendVar('BAR', ' bar 2')
     }

     BAR = "bar 1"

等同于:

     FOO = "foo 1"
     BAR = "bar 1"
     FOO = "foo 2"
     BAR += "bar 2"

## Tasks

Task是BitBake的执行单元, 是BitBake执行一个recipe的过程, 通常一个 recipe 包含do_fetch, do_patch, do_compile 等 task.

Task只在 recipe和class中支持(.bb 文件中和include .bb或者inherited .bb的文件中). 按照惯例, task 的名字以 "do_" 开头.

### 将一个函数变为 Task

使用 addtask 命令可以将一个  shell functions 或者 BitBake-style Python functions 变成一个 task, 例如:

     python do_printdate () {
         import time
         print time.strftime('%Y%m%d', time.gmtime())
     }
     addtask printdate after do_fetch before do_build
addtask 的第一个参数是函数名字, 如果名字没有以"do_"开始, "do_"会被隐式添加.  
同时上面的语句定义了task之间的依赖关系, do_printdate 依赖于do_fetch, 并且是do_build的依赖, 执行 do_build 会导致 do_printdate 先运行. do_build 是所有 recipe的默认task, 其依赖于所有其他的task.

	注:
	当使用 bitbake recipe 编译 recipe 运行上面的例子时, 会发现 do_printdate 只在第一次的时候运行, 这是因为bitbake认为task已经是最新的了. 
	如果你想让task每次都会运行, 可以使用[nostamp]标志(没有时间戳), 这样bitbake 就会认为do_printdate task 总是过时的.
		do_printdate[nostamp] = "1"

	也可以通过 -f 选项显式执行 task, 如:
	$ bitbake recipe -c printdate -f
	这条命令相当于选择一个 task 执行, 此时可以省略前缀 "do_". 

如果在使用 addtask 时没有指定任何依赖, 如下:
	
	addtask printdate
则task不会在 bitbake recipe 过程中运行, 唯一运行此task的方法是使用 bitbake recipe -c printdate 手动执行.

	$ bitbake recipe -c listtasks # 列出recipe的所有task.

### 删除一个task

使用deltask命令, 如下:
	
	deltask printdate

如果删除的task有依赖, 则 deltask会删除依赖关系. 比如 do_c 依赖与 do_b, do_b 依赖于 do_a, 删除do_b之后则 do_c 并不会依赖于 do_a, do_c 有可能先于 do_a运行.

如果想保持依赖关系, 可以使用 [noexec] 标志禁用 do_b, 如下:

	do_b[noexec] = "1"

### 传递信息到Task执行环境(暂略)

## 变量标志 (Variable Flags)

变量标志用于控制 task的功能和依赖, 如前面用到的 [nostamp], [noexec]. 还有如下这些常用的:

	 


