## 7.1 扩展的构成及编译

### 7.1.1 扩展能做什么
扩展是PHP的重要组成部分，它是PHP提供给开发者用于扩展PHP语言功能的主要方式。开发者可以用C/C++定义自己的功能，通过扩展嵌入到PHP中，灵活的扩展能力使得PHP拥有了大量、丰富的第三方组件，这些扩展很好的补充了PHP的功能、特性，大大提高了PHPer的开发效率，使得PHP在web开发中得以大展身手。

C语言是PHP之母，笔者认为：C语言是世界上最好的语言！自它诞生至今，C语言孕育了大量优秀、知名的项目：Linux、Nginx、MySQL、PHP、Redis、Memcached等等，感谢里奇带给这个世界如此伟大的一份礼物。C语言的优秀也折射到PHP身上，但是PHP内核提供的功能终究有限，如果你发现PHP在某些方面已经满足不了你的需求了，那么不妨试试扩展，当然扩展也不是万能的。

常见的，扩展可以在以下几个方面有所作为：
* __介入PHP的编译、执行阶段：__ 可以介入PHP框架执行的那5个阶段，比如opcache，就是重定义了编译函数
* __提供内部函数：__ 可以定义内部函数扩充PHP的函数功能，比如array、date等操作
* __提供内部类__
* __实现RPC客户端：__  实现与外部服务的交互，比如redis、mysql等
* __提升执行性能：__ PHP是解析型语言，在性能方面远不及C语言，可以将耗cpu的操作以C语言代替
* ......

PHP中的扩展分为两类：PHP扩展、Zend扩展，对内核而言这两个分别称之为：模块(module)、扩展(extension)，本章主要介绍是PHP扩展，也就是模块。

### 7.1.2 脚本工具
PHP提供了几个脚本工具用于简化扩展的实现：ext_skel、phpize、php-config，后面两个脚本主要配合autoconf、automake生成Makefile。

#### 7.1.2.1 ext_skel
这个脚本位于PHP源码ext目录下，它的作用是用来生成扩展的基本骨架，帮助开发者快速生成一个规范的扩展结构，可以通过以下命令生成一个扩展结构：
```shell
./ext_skel --extname=扩展名称
```

### 7.1.2.2 php-config

#### 7.1.2.3 phpize

### 7.1.3 编写扩展的基本步骤
编写一个PHP扩展主要分为以下几步：
* 通过ext目录下ext_skel脚本生成扩展的基本框架：`./ext_skel --extname`，这个脚本生成一个新的扩展目录，并生成了扩展所需的基本文件；
* 修改config.m4配置：这个是autoconf语法的配置文件，可以根据不同的编译环境生成不同的宏定义，以及设置依赖的库；
* 编写扩展要实现的功能：按照PHP扩展的格式以及PHP提供的API编写功能；
* 生成configure：扩展编写完成后执行phpize脚本生成configure及其它配置文件，phpize是一个shell脚本；
* 编译&安装：./configure、make、make install，然后将扩展的.so路径添加到php.ini中。

最后就可以在PHP中使用这个扩展了。

### 7.1.4 扩展的组成部分
扩展最核心的部分就是`zend_module_entry`结构，每个扩展拥有这样一个结构，内核通过这个结构得到这个扩展都提供了哪些功能，换句话说，一个扩展可以只包含一个`zend_module_entry`结构，相当于定义了一个什么功能都没有的扩展。
```c
//zend_modules.h
struct _zend_module_entry {
    unsigned short size; //sizeof(zend_module_entry)
    unsigned int zend_api; //ZEND_MODULE_API_NO
    unsigned char zend_debug; //是否开启debug
    unsigned char zts; //是否开启线程安全
    const struct _zend_ini_entry *ini_entry;
    const struct _zend_module_dep *deps;
    const char *name; //扩展名称，不能重复
    const struct _zend_function_entry *functions; //扩展提供的内部函数列表
    int (*module_startup_func)(INIT_FUNC_ARGS); //扩展初始化回调函数，PHP_MINIT_FUNCTION或ZEND_MINIT_FUNCTION定义的函数
    int (*module_shutdown_func)(SHUTDOWN_FUNC_ARGS); //扩展关闭时回调函数
    int (*request_startup_func)(INIT_FUNC_ARGS); //请求开始前回调函数
    int (*request_shutdown_func)(SHUTDOWN_FUNC_ARGS); //请求结束时回调函数
    void (*info_func)(ZEND_MODULE_INFO_FUNC_ARGS); //php_info展示的扩展信息处理函数
    const char *version; //版本
    size_t globals_size;
    void* globals_ptr;
    void (*globals_ctor)(void *global);
    void (*globals_dtor)(void *global);
    int (*post_deactivate_func)(void);
    int module_started;
    unsigned char type;
    void *handle;
    int module_number; //扩展的唯一编号
    const char *build_id;
};
```
这个结构包含很多成员，但并不是所有的都需要自己定义，经常用到的主要有下面几个：
* __name:__ 扩展名称，不能重复
* __functions:__ 扩展定义的内部函数entry
* __module_startup_func:__ PHP在模块初始化时回调的hook函数，可以使扩展介入module startup阶段
* __module_shutdown_func:__ 在模块关闭阶段回调的函数
* __request_startup_func:__ 在请求初始化阶段回调的函数
* __request_shutdown_func:__ 在请求结束阶段回调的函数
* __info_func:__ php_info()函数时调用，用于展示一些配置、运行信息
* __version:__ 扩展版本

除了上面这些需要手动设置的成员，其它部分可以通过`STANDARD_MODULE_HEADER`、`STANDARD_MODULE_PROPERTIES`宏完成标准定义。

### 7.1.5 编译配置
使用C语言编写程序时通常会编写一个Makefile，但是大型项目中Makefile将会非常复杂，而且不同的编译环境下也要对这个文件进行相应的修改，很不方便。很多开源软件编译过程非常简单(三板斧)：configure、make、make install，这就是通过自动编译配置实现的，PHP使用的是automake/autoconf等，GNU的Autotool系列工具非常多，初学起来也比较复杂，这里不作过多说明，只简单介绍下与扩展相关的基本用法。

一个简单的扩展配置只需要以下内容：
```c
PHP_ARG_WITH(扩展名称, for mytest support,
Make sure that the comment is aligned:
[  --with-扩展名称             Include xxx support])

if test "$PHP_扩展名称" != "no"; then
    PHP_NEW_EXTENSION(扩展名称, 源码文件列表, $ext_shared,, -DZEND_ENABLE_STATIC_TSRMLS_CACHE=1)
fi
```
PHP_ARG_WITH()、PHP_NEW_EXTENSION()是PHP自定义的autoconf宏，这些宏封装了autoconf的操作，简化了复杂的配置，它们定义在aclocal.m4、acinclude.m4中，通过AC_DEFUN()定义，

`PHP_ARG_WITH()`有三个参数，第一个是扩展名称，第二个是运行configure时的展示的信息，第三个是运行`configure --help`时展示的信息。`PHP_NEW_EXTENSION()`这个宏

