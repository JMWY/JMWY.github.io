---
layout: post
title: "[unix] 进程环境"
comments: true
categories:
- unix
---

# 进程环境

* [C 程序启动（main 函数）](#1)
* [进程终止](#2)        
　　[exit 函数](#2.1)       
　
　[atexit 函数 - 登记终止处理程序](#2.2)      
　　[C程序启动与终止图](#2.3)       
* [环境变量](#3)        
　　[环境变量的设置](#3.1)
* [共享库](#4)
* [C 程序内存空间](#5)
* [跳转函数](#6)
* [进程资源限制](#7)
* [习题](#8)


<h2 id="1"></h2>

## 1. C 程序启动（main 函数）

main 函数是 C 程序进入点。
### 函数原型

(ISO C 和 POSIX.1 支持的)原型有两个:  

    int main(void) { /* ... */ };
    int main(int argc, char *argv[]) { /* ... */ };
    
>历史上，多数 UNIX 系统支持:
    
    int main(int argc, char *argv[], char *envp[]) { /* ... */ };    

>第三个参数是环境表地址。现在通常用 getenv 和 putenv 函数来访问特定环境变量。

### 调用 main 函数
- 内核（通过一个 exec 函数）执行 C 程序时，首先调用一个特殊的启动例程（start-up routine），再调用 main 函数。  
- **C 编译器**调用**链接编辑器**，然后**链接编辑器**设置**执行程序文件**的**起始地址**为**启动例程**。
- **启动例程**（常用汇编语言编写）如果用 C 代码形式表示，大概为：  

        exit(main(argc, argv))

### 命令行参数
执行程序时，**调用 exec 的进程**将命令行参数传递给新程序。*Note:* echo 程序通常不回送第 0 个参数。
``` c
#include <stdio.h>
int main (int argc, char *argv[])
{   
    int i;
    for (int i = 0; i < argc; ++i) {
        printf("argv[%d]: %s\n", i, argv[i]);
    }
    /*note: use exit replace return maybe generate warnings in some C complier or UNIX lint(1) programs*/
    exit(0);
}
```

----------------------------------------------------------------
<h2 id="2"></h2>
## 2. 进程终止
进程终止（termination）方式有 8 种：正常终止 5 种方式，异常终止 3 种:
> (1) 从 main 返回      
> (2) exit      
> (3) _exit     
> (4) 最后一个线程从启动例程返回        
> (5) 最后一个线程调用 pthread_exit     
> **以下为异常终止方式**        
> (6) abort     
> (7) 收到（终止）信号      
> (8) 最后一个线程响应取消请求     

<h3 id="2.1"></h3>

### exit 函数 

在头文件 [stdlib.h](http://pubs.opengroup.org/onlinepubs/7908799/xsh/stdlib.h.html) 中声明的原型为：

        void exit(int status); // (ISO C)
        void _Exit(int status); // Note: not found in file stdlib.h

在头文件 [unistd.h](http://pubs.opengroup.org/onlinepubs/7908799/xsh/unistd.h.html) 中声明的原型为：

        void _exit(int status); //(POSIX.1)

- 参数 satus：终止状态（或退出状态，exit status）
- **exit 函数**先调用**终止处理程序**，并关闭所有 I/O 流（为所有打开流调用 fclose 函数），然后调用 _exit(或_Exit) 进入内核

<h3 id="2.2"></h3>

### atexit 函数 - 登记**终止处理程序**

在头文件 [stdlib.h](http://pubs.opengroup.org/onlinepubs/7908799/xsh/stdlib.h.html) 中声明的原型为：

        int atexit(void (*func)(void));

- 参数 func : 所登记函数的地址
- 调用顺序与登记顺序相反
- ISO C 规定，一个进程至少支持登记 32 个函数，即**终止处理程序(exit handler)**。为了确定平台支持的最大**终止处理程序数**，可查看 [sysconf 函数](http://pubs.opengroup.org/onlinepubs/009695399/functions/sysconf.html)中变量 ATEXIT_MAX。

``` c
/*@ test_atexit.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static void my_exit1(void);
static void my_exit2(void);

int main (void) 
{
    if (atexit(my_exit1) != 0) {
        perror("can't register my_exit1");
    }
    if (atexit(my_exit2) != 0) {
        perror("can't register my_exit1");
    }

    printf("main is done\n");
    return 0;
}

static void my_exit1(void) 
{
    printf("1st exit handler\n");   
}

static void my_exit2(void) 
{
    printf("2nd exit handler\n");   
}
```

![运行结果](https://raw.githubusercontent.com/JMWY/MyBlog/master/AdvancedProgrammingInTheUnixEnvironment_v3/images/chapter7/test_atexit.png)        

<h3 id="2.3"></h3>

### C程序启动与终止图   

<br />
![C程序启动与终止](https://raw.githubusercontent.com/JMWY/MyBlog/master/AdvancedProgrammingInTheUnixEnvironment_v3/images/chapter7/C%E7%A8%8B%E5%BA%8F%E5%90%AF%E5%8A%A8%E4%B8%8E%E7%BB%88%E6%AD%A2.PNG)


------------------------------------------------------------------
<h2 id="3"></h2>

## 3. 环境变量

每个程序都有一张**环境表（environment list）**，地址为全局变量 environ:
    
    extern char **environ; 
    
可用于查看整个环境。其大致形式如下：

![环境表例图](https://raw.githubusercontent.com/JMWY/MyBlog/master/AdvancedProgrammingInTheUnixEnvironment_v3/images/chapter7/%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F.PNG)

<br />

**Single UNIX Specification 定义的环境变量**

![环境变量](https://raw.githubusercontent.com/JMWY/MyBlog/master/AdvancedProgrammingInTheUnixEnvironment_v3/images/chapter7/%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%E7%B1%BB%E5%9E%8B.PNG)
<br />

头文件 stdlib.h 中，定义了环境变量的访问函数：

    char *getenv(const char *name);  // return: SUCESS, 对应 value 的指针; FAIL: NULL
    int putenv(char *str);
    int setenv(const char *name, const char *value, int rewrite);
    int unsetenv(const char *name); // return: SUCESS, 0; FAIL: 非0 

**解释**：      
`putenv`: 字符串形式为 name=value，放入环境表。若 name 已存在，则先删除原定义。     
`setenv`: name 存在时，(a) rewrite 非0，删除原定义 (b) rewirte 为0，不删除原定义，返回0 。      
`unsetenv`: 删除 name 定义。不存在也返回0 。        
<br />
**函数对环境表的修改策略:**
- **环境表** 和 **环境字符串**通常放在进程存储空间顶部(栈之上)，该空间长度却不能增加(不能向上扩展也不能移动栈)。
- 修改已有 `name` :**(1)** `value` <= 现有 `value`， 直接写  **(2)** 否则，`malloc` 安置新串，释放老串（新串内容在**堆**中）。
- 添加新 `name` :**(1)**第一次添加，`malloc` 并复制原环境表(一列指针)到**堆**  **(2)** 否则，`remalloc` 增加一个指针空间。
<br />

<h3 id="3.1"></h3>

### 环境变量的设置

> Effective for all users: `/etc/profile` ; Just for current user: `~/.bashrc` or `~/bash_profile`

1. executable file(bin/): add in **PATH**               
2. head file(include/.h): **C_INCLUDE_PATH** or **CPLUS_INCLUDE_PATH**
3. dynamic lib(lib/.so): **LD_LIBRARY_PATH**
4. static lib(lib/.a):  **LIBRARY_PATH**        
<br />

**PS:** 
(《Shell 脚步攻略》)推荐函数（写在 .bashrc,用来添加环境变量）prepend:       
`prepend() { [ -d "$2" ] && eval $1=\"$2\$\{$1:+':'\$$1\}\" && export $1 ; }`       
`Used as: prepend PATH /opt/myapp/bin`



------------------------------------------------------------------
<h2 id="4"></h2>

## 4. 共享库

**共享库**： 运行时加载的动态链接库(.so文件)。      
**优点**：
- 减少可执行文件长度（不用包含公用的**库例程**）
- 库函数**新版本**可直接代替**老版本**使用（无需重新编译和链接相关的**可执行程序**）
- 节省内存空间（内存维护**库例程**的**一个副本**，让**所有进程**都可引用）      

**不足**:       
- 增加了运行时间开销（产生在第一次调用库函数时）

**Example**

```c
/* util.c */
#include <stdio.h>

void print(const char *message) 
{
    if (!message) {
        printf("<null>\n");
    } else {
        printf("%s\n", message);
    }
}

int subtract(const int minuend, const int subtrahend)
{   
    return (minuend - subtrahend);
}

/* util.h */
#ifndef __util_h
#define __util_h

extern void print(const char* mes);
extern int subtract(const int minuend, const int subtrahend);

#endif

/* DemoMain.c */
#include "util.h"
#include <string.h>
#include <stdio.h>

int main(void) 
{
    char *str = "Message: How do you do.";
    int A = 100;
    int B = 1;
    
    print(str);     
    printf("subtract: %d\n", subtract(A, B));
    
    return 0;
}

```
![运行结果](https://raw.githubusercontent.com/JMWY/MyBlog/master/AdvancedProgrammingInTheUnixEnvironment_v3/images/chapter7/shared_lib.png)



--------------------------------------------------------------------
<h2 id="5"></h2>

## 5. C 程序内存空间

### C 程序内存布局

内存中，C 程序组成： **正文段（Text segment）**，**初始化数据段（Initialized data segment）**，**非初始化数据段（Uninitialized data segment）**，**栈（Stack）**，**堆（Heap）**。

**内存布局如下：**      


![内存布局](https://github.com/JMWY/MyBlog/blob/master/AdvancedProgrammingInTheUnixEnvironment_v3/images/chapter7/%E5%86%85%E5%AD%98%E5%AE%89%E6%8E%92.PNG)  


- 正文段：CPU 执行的机器指令；通常可共享；只读。
- 初始化数据段： 函数外明确初始化的变量。
- 非初始化数据段：称为 bss(block started by symbol) 段；函数外的声明；初始化为 0.
- 其它类型段：符号表的段，调试信息的段，动态库链接表的段等不装载。
- **注意**磁盘上的程序文件只有正文段和初始化数据段。

### 内存分配

**JMWY 注：内存管理是一个很重要的课题，有很多相关的工具和编程技巧。**       
在文件 stdlib.h  中，定义了三个内存空间动态分配函数（ISO C）:

    void *malloc(size_t size);
    void *calloc(size_t nobj, size_t size);
    void *realloc(void *ptr, size_t newsize); // RETURN: SUCESS, 非空; FAIL: NULL.
    
    void free(void *ptr);

- `calloc`: 分配 nobj * size , 空间初始化为 0.
- `realloc`: 更改分配区长度（增或减）。当增加长度时，原存储区后没有足够的扩充空间，则分配另一个足够大的存储区，释放原存储区，返回新存储区域指针，所以参数 `ptr` 可能会失效。另外，新增区域初始值不确定。
- `free`: 忘记调用 `free` 是内存泄露根源，常用的检测工具有 **Valgrind** 等。

--------------------------------------------------------------------
<h2 id="6"></h2>

## 6. 跳转函数

goto 语句可以实现**函数内调转**（常用于 `RETRY:` 和对（异常等）信号的跳转处理）。       
头文件 [setjmp.h](http://pubs.opengroup.org/onlinepubs/007908799/xsh/setjmp.h.html) 定义的执行**非局部的跳转**的函数有：

    int setjmp(jmp_buf env); // RETURN: 直接调用, 0; 从 `longjmp` 调用， 非0.
    void longjmp(jmp_buf env, int val);

-  `jmp_buf`: 数组，存放栈状态所有信息，在调用 `longjmp` 时用来恢复环境。
-  `env`: 因为需在另一函数引用，所以规范处理应定义为全局变量。
-  `val`: 存在的原因是一个 `setjmp` 可以有多个 `longjmp`。

**应用环境示例：**
    
```c
/* test_jump.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <setjmp.h>

#define SPER    "------------"
#define MAXLINE 10240

#define TOK_ADD 5

void do_line(char *ptr); 
void cmd_add(void); 
int get_token(void);


jmp_buf jmpbuffer;

int main(void) 
{   
    char line[MAXLINE];
    if (setjmp(jmpbuffer) != 0) 
        printf("error\n " SPER);
    while (fgets(line, MAXLINE, stdin) != NULL) 
        do_line(line);
    exit(0);
}

char *tok_ptr;

void do_line(char *ptr) 
{
    int cmd;
    tok_ptr = ptr; // assignent value for get_token()
    while ((cmd = get_token()) > 0) {
        switch (cmd) {
        case TOK_ADD:
            cmd_add();
            break;
        }
    }
}

void cmd_add(void) 
{   
    int token;
    if ((token = get_token()) < 0) 
        longjmp(jmpbuffer, 1);
    printf("excute: add <var>\n");
}

int get_token(void) 
{
    size_t i = 0;
    char *cmd_buff = (char *)malloc(strlen(tok_ptr) + 1);
    if (NULL == cmd_buff) 
        return -2;
    while(tok_ptr[i] != '\0' && tok_ptr[i] != ' ') {
        cmd_buff[i] = tok_ptr[i++];
    }
    if (0 == i) 
        return -1;
    tok_ptr += tok_ptr[i] ? i+1 : i;

    if (strcmp(cmd_buff, "add") == 0)
        return TOK_ADD;
    
    enum{ OTHER_OPTION };
    return OTHER_OPTION;
    
}

```
![运行结果](https://github.com/JMWY/MyBlog/blob/master/AdvancedProgrammingInTheUnixEnvironment_v3/images/chapter7/test_jump.png)  


**对 Automatic, Register, and Volatile 类型变量的影响**

```c
#include <stdio.h>
#include <stdlib.h>
#include <setjmp.h>

static jmp_buf jmpbuffer;
static int globval;

static void f2(void) 
{
    longjmp(jmpbuffer, 1);
}

static void f1 (int i, int j, int k, int l)
{
        printf("in f1():\n");
        printf("globval = %d, autoval = %d, regival = %d,"
                "volaval = %d, statval = %d\n",
                globval, i, j, k, l);
        f2();
}

int main(void) 
{
    int autoval;
    register int regival;
    volatile int volaval;
    static int statval;

    globval = 1; autoval = 2; regival = 3; volaval = 4; statval = 5;

    if (setjmp(jmpbuffer) != 0) {
        printf("after longjmp:\n");
        printf("globval = %d, autoval = %d, regival = %d,"
                "volaval = %d, statval = %d\n",
                globval, autoval, regival, volaval, statval);
        exit(0);
    }
    
    globval = 95; autoval = 96; regival = 97; volaval = 98; statval = 99;
    
    f1(autoval, regival, volaval, statval);
    exit(0);
}
```
![运行结果](https://github.com/JMWY/MyBlog/blob/master/AdvancedProgrammingInTheUnixEnvironment_v3/images/chapter7/test_var.png)  


- **内存**中的变量具有 `longjmp` 时的值；**CPU** 和**浮点寄存器**中的变量会恢复为调用 `setjmp` 时的值。
- So，编写非局部跳转的可移植程序，必须用 `volatile` 属性。





--------------------------------------------------------------------
<h2 id="7"></h2>

## 7. 进程资源限制

通常是在系统初始化时由**进程 0** 建立，而后被其它进程继承的。头文件 [sys/resource.h](http://pubs.opengroup.org/onlinepubs/007908799/xsh/sysresource.h.html) 定义了对资源限制**查询**和**更改**的函数:

    int getrlimit(int resource, struct rlimit *rlptr);
    int setrlimit(int resource, const struct rlimit *rlptr); // RETURN: SUCESS, 0

其中，
```c
struct rlimit {
    rlim_t  rlim_cur; /* soft limit: current limit */
    rlim_t  rlim_max; /* hard limit: maximum value for rlim_cur */
};
```
更改资源限制的三条规则：        
1. 修改软限制值时， <= 硬限制值     
2. 降低硬限制值时， >= 软限制值     
3. 只超级用户进程才可提高硬限制值。         
（最大值常量： RLIM_INFINITY）      

**参数 `resource` 取值如图：**
![对资源限制的支持](https://raw.githubusercontent.com/JMWY/MyBlog/master/AdvancedProgrammingInTheUnixEnvironment_v3/images/chapter7/resoure_limit_support_by_OS.PNG)

|Limit          |    Description|
| :----------   | :----------:  |
|RLIMIT_AS *(RLIMIT_VMEM)* |可用内存最大总长度（字节），会影响 `sbrk` (1.11节)和 `mmap` (14.8节)函数|
|RLIMIT_CORE    | core 文件的最大值（字节）|
|RLIMIT_CPU     |CPU 时间最大量(s)，超出软限制，会收到 `SIGXCPU` 信号 |
|RLIMIT_DATA    |初始化数据、非初始化数据及堆的总和的最大值（字节）|
|RLIMIT_FSIZE   |创建的文件的最大值(字节)，超出软限制，会收到 `SIGXFSZ` 信号 |
|RLIMIT_MEMLOCK |使用 `mlock` 能锁定内存中最大长度（字节）|
|RLIMIT_MSGQUEUE|分配给POSIX消息队列的最大内存（字节）|
|RLIMIT_NICE    |影响调度优先级的进程 `nice`(8.16节)值上限|
|RLIMIT_NOFILE  |打开的最大文件数。更改会影响 `sysconf(_SC_OPEN_MAX)` 的返回值 |
|RLIMIT_NPROC   |每个真实用户ID可拥有最大子进程数。更改会影响 `sysconf(_SC_CHILD_MAX)` 的返回值 |
|RLIMIT_NPTS    |一个用户一次可以打开伪终端（pseudo terminals）最大数（19章）|
|RLIMIT_RSS     |最大驻内存集（resident set size）（字节），物理内存少时，内核取回其超出**RSS**部分|
|RLIMIT_SBSIZE   |用户可占有的最大套接字缓存（socket buffers）（字节）|
|RLIMIT_SIGPENDING|等待进程的最大信号数，该限制被 `sigqueue` 函数强制执行|
|RLIMIT_STACK   | 栈的最大长度（字节）|
|RLIMIT_SWAP    |用户占有的交换空间最大量(字节)|

**Example**  

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/resource.h>

/* # is string-creation operator */
#define doit(name) pr_limits(#name, name)
#define info_limit(name) doit(RLIMIT_##name)

static void pr_limits(char *, int);

int main (void) 
{
#ifdef RLIMIT_AS    
    info_limit(AS);
#endif                  
#ifdef RLIMIT_NPROC 
    info_limit(NPROC);
#endif                  
#ifdef RLIMIT_RSS
    info_limit(STACK);
#endif                  
#ifdef RLIMIT_RSS
    info_limit(STACK);
#endif                  
#ifdef RLIMIT_VMEM
    info_limit(VMEM);
#endif              
    printf("------\n");
    info_limit(CORE);
    info_limit(CPU);
    info_limit(DATA);
    info_limit(FSIZE);
    info_limit(STACK);
    info_limit(NOFILE);
    exit(0);
}

static void pr_limits(char *name, int resource)
{
    struct rlimit limit;
    unsigned long long lim; // uniform format: "%10lld"
    if (getrlimit(resource, &limit) != 0) {
        printf("getrlimit error for %s", name);
        exit(-1);
    }
    printf("%-14s   ", name);
    if (RLIM_INFINITY == limit.rlim_cur)
        printf("<infinite>  ");
    else {
        lim = limit.rlim_cur;
        printf("%10lld  ", lim);
    }
    if (RLIM_INFINITY == limit.rlim_max)
        printf("<infinite>");
    else {
        lim = limit.rlim_max;
        printf("%10lld  ", lim);
    }
    putchar((int)'\n');
}

```

![](https://github.com/JMWY/MyBlog/blob/master/AdvancedProgrammingInTheUnixEnvironment_v3/images/chapter7/test_res.png)  


-------------------------------------------------------------------
<h2 id="8"></h2>

## 8. 习题

**习题 7.1**

这种情况下，Interl x86系统，FreeBSD 和 Linux 会将 `printf` 返回值(*打印字节数*)变成 `main` 返回值。

``` c
#include <stdio.h>
main (int argc, char *argv[]) 
{
    printf("%d\n",printf("hello world!\n"));
}
```

![](https://github.com/JMWY/MyBlog/blob/master/AdvancedProgrammingInTheUnixEnvironment_v3/images/chapter7/7.1.png)

**习题 7.7**        
Q: size 命令的输出结果中，为什么没有堆和栈的大小？
> 只有通过 `exec` 函数执行一个程序时，才会分配堆和堆栈。

**习题 7.8**        
Q: size 命令的输出结果中，为什么文件大小不等于文本和数据大小之和？
> 可执行文件包含了用于调试 core 文件的符号表信息，用 `strip` 命令可以删除这些信息。



