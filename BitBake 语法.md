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

Task 是BitBake的执行单元, 组成了BitBake执行一个recipe的步骤. Task只在 recipe和class中支持(.bb 文件中和include .bb或者inherited .bb的文件中). 按照惯例, task 的名字以 "do_" 开头.

