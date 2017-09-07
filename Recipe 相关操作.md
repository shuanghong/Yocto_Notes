# Recipe 相关操作

## 新建 recipe

更多可参考官方文档 [http://www.yoctoproject.org/docs/current/dev-manual/dev-manual.html#new-recipe-writing-a-new-recipe](http://www.yoctoproject.org/docs/current/dev-manual/dev-manual.html#new-recipe-writing-a-new-recipe)

Recipes (.bb) 文件是 Yocto Project 的基本组件, 每个使用 OpenEmbedded 构建的软件组件都需要一个 recipe 来定义此组件.

### 基本流程
下图显示了创建一个recipe的基本流程.

![](http://i.imgur.com/ufqGsQo.png)

有3个方法创建 recipe.

- devtool add 命令, 参考 [http://www.yoctoproject.org/docs/current/dev-manual/dev-manual.html#use-devtool-to-integrate-new-code](http://www.yoctoproject.org/docs/current/dev-manual/dev-manual.html#use-devtool-to-integrate-new-code)
- recipetool create 命令, 参考如上.
- 基于一个类似的 recipe 修改, Yocto 本身自带很多 recipe可供参考

### 基于已有的 recipe 修改

一个 recipe 基本的结构如下:

	 DESCRIPTION = ""
     HOMEPAGE = ""
     LICENSE = ""
     SECTION = ""
     LIC_FILES_CHKSUM = ""

     SRC_URI = "" 			# 源代码的下载地址
	 SRCREV = ""			# commit 的版本号

     DEPENDS = ""			# 所依赖的其他 recipe

## 保存并命名 recipe

recipe 文件的命名遵循这样的格式, 即 basename_version.bb. 如 u-boot_2016.01.bb, 其中 u-boot 是 basename, 2016.01 是 version, 中间以 _ 相连. 

	注:
	PN: 指recipe的名字或者所要生成的package的名字, 如 u-boot_2016.01.bb, ${PN} = u-boot
	PV: 指recipe的version, 如 u-boot_2016.01.bb, ${PV} = 2016.01
	PR: 指recipe的修订版, 默认值是"r0"

## recipe 依赖(Dependencies)

使用变量 DEPENDS 指明当前recipe在构建期所依赖的其他recipe, 一般需要明确指出recipe的所有构建期依赖, 如下:

	 DEPENDS = "zlib lzo e2fsprogs util-linux"	#zlib, lzo 都是 recipe 名字 
同时使用变量  RDEPENDS 指定运行时package依赖. 由于一个recipe的main package 与recipe name相同, 并且recipe name可以通过${PN}获取, 可以设置 RDEPENDS_${PN} 来指定 main package 的依赖, 如下: 

	RDEPENDS_${PN} = "foo (>= 1.2)"	# main package 依赖 package foo 1.2 以上版本
	RDEPENDS_${PN}-tools = "foo (>= 1.2)"# package ${PN}-tools 依赖 package foo 1.2 以上版本

参考 [http://www.yoctoproject.org/docs/current/ref-manual/ref-manual.html#var-RDEPENDS](http://www.yoctoproject.org/docs/current/ref-manual/ref-manual.html#var-RDEPENDS)

## 构建/编译 recipe

	$ bitbake basename 或者 $ bitbake target, 这里的target就是一个recipe名字.
	如: bitbake u-boot 就是构建 u-boot_2016.01.bb

构建的过程就是执行 task 的过程, 参考 [http://www.yoctoproject.org/docs/current/ref-manual/ref-manual.html#bitbake-dev-environment](http://www.yoctoproject.org/docs/current/ref-manual/ref-manual.html#bitbake-dev-environment)
可通过如下命令获取一个recipe的所有task.

	$ bitbake recipe -c listtasks

## Recipe 任务

### 获取源代码, do_fetch 

在构建 recipe的过程中, 由do_fetch任务根据变量 SRC_URI 的值获取源代码, 通常对SRC_URI的赋值如下:
	
	U-BOOT_REPO = "git://git.denx.de/u-boot.git"
	U-BOOT_Branch = "master"
	U-BOOT_REV = "a78387ce228e6d7a8df02dd5e74c4a898759cedf"
	U-BOOT_PROT = "ssh"

	SRC_URI = "${U-BOOT_REPO};protocol=${U-BOOT_PROT};branch=${U-BOOT_BRANCH}"
	SRCREV = "${U-BOOT_REV}"

如果有下面的语句, 是指给代码打 patch(do_patch)

	SRC_URI += "file://0001-u-boot-mpc85xx-u-boot-.lds-remove-_GLOBAL_OFFSET_TAB.patch"




### 配置 recipe, do_configure

- Autotools: 源代码使用 Autotools 构建, 则recipe需要继承 autotools 类, 如下:

		inherit autotools #  poky/meta/classes/autotools.bbclass
	同时 recipe无需包含do_configure任务, 但是可能需要设置EXTRA_OECONF 或者 PACKAGECONFIG_CONFARGS 传递需要的配置选项

- Cmake: 如果被编译的代码使用 Cmake 构建, 则recipe需要集成cmake, 如下:
			
		inherit cmake	# poky/meta/classes/cmake.bbclass
	同时 recipe无需包含do_configure任务, 但是可能需要通过设置变量  EXTRA_OECMAKE 传递任何所需的配置选项.

- 其他: 正常来说recipe中需要提供一个do_configure任务, 除非没有任何东西可配置.

注: 当do_configure task执行完成后, 查看 log.do_configure 文件确保使能了正确的配置选项, 以及 "DEPENDS" 是否完整.
