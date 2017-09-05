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

例子如下:

	some_function () {
    	echo "Hello World"
     }

Shell 函数要遵循 shell 编程规则, 脚本被 /bin/sh 执行, /bin/sh 有可能是 bash shell或者dash, 不应该使用 bash特定的脚本.  
重写操作符如 _append, _prepend 同样可用于shell函数. 常用于 .bbappend 文件用来修改主 recipe中的函数, 也可用来修改从 classes 继承来的函数. 例子如下:
	
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