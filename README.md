> 开发大型软件项目并不容易，但是很多时候这些大型软件的基本思想非常简单，我们自己去实现它是不错的学习方式，至少可以证明你具有成为一位真正的程序员所需要的基本条件。
> 在本文中，我会使用C语言编写一个简单的shell程序来让你理解shell的基本运作方式，shell当然不止这么简单，但是它已足够将你领进门了。

__注意:__ 不要过度依赖这篇文章，倒杯热水，慢慢思考

# INTRUDUCTION

如果你已经非常熟悉shell那就跳过这一段吧，这一段是简单介绍一下shell的，像我一样的linux程序员几乎每天都在使用shell, 那么shell具体是什么呢？如果你看过《Unix环境高级编程》的话，翻开正文第一页，你就能得到这个问题的答案。归纳来讲，shell其实就是操作系统软件用户接口，其主要目的是启动其他程序并控制它们的交互；它是一种将命令执行的内部复杂性与用户分离的方法。内部结构可以改变，而用户体验/界面保持不变。

## Basic lifetime of shell
让我们自顶向下来理解shell的基本生命周期，总体来说，shell只做三件事：
- 初始化（Intiialize）：在这一步中，shell将读取并执行其配置文件，配置文件决定了shell在工作中的各个细节行为
- 解释（Interpret）：在这一步中，shell 从标准输入（可以是交互式的，也可以是文件）读取命令并执行它们
- 终止（Terminate）：在命令执行后，shell 执行所有关闭命令，释放所有内存，然后终止

这些步骤看起来非常通用，许多不是shell的应用程序也具有这样的执行流程，所以我们将使用其作为我们实现的 shell 的基础。这个 shell 将非常简单，不会有任何配置文件，也不会有任何关闭命令，不过如果你跟着本文走到最后，那么可以自己实现一下这些特性，它不会很难。那么现在，我们只需调用循环函数然后终止。但就架构而言，重要的是要记住程序的生命周期不仅仅是循环：

```c
#include <stdio.h>

int
main(int argc, char *argv[])
  {
    lsh_loop();
    return EXIT_SUCCESS;
  }
```

很显然，我们还没有实现`lsh_loop()`，这是我们接下来要介绍的。

## Basic loop of a shell
既然我们已经知道了程序应该如何启动，那么现在需要思考一下程序基本的运行逻辑：shell在其循环期间基本上来看也只做了三件是：
- 读取(Read)：从标准输入读取命令
- 解析(Parse)：解析命令字符串拿到需要运行的程序和对应的参数
- 执行(Execute)：根据解析完成的命令运行对应的应用程序

那么我们根据这个流程来编写`lsh_loop()`函数:
```c
void
lsh_loop(void)
  {
    char *line;
    char **args;
    int status;

    do {
      printf("> ");
      line = lsh_read_line();
      args = lsh_split_line(line);
      status = lsh_execute(args);

      free(line);
      free(args);
    } while (status);
  }
```

需要注意的是，我们使用 `lsh_execute()` 返回的状态变量来确定何时退出。

## Reading a line
接下来我们来实现`lsh_loop()`中的`lsh_read_line()`函数，从标准输入读取一行听起来很简单，但在 C 语言中可能会很麻烦。可悲的是，我们不能提前知道用户将在 shell 中输入多少文本。所以不能简单地分配一个特定大小的内存然后祈祷用户输入的文本不会超过它。我们只能从一个块开始，如果输入的数据超过它，就重新分配更多空间。这是 C 语言中的常用策略，我们将使用它来实现 lsh_read_line()：

```c
char *
lsh_read_line(void)
  {
    int bufsize = LSH_RL_BUFSIZE;
    int position = 0;
    char *buffer = malloc(sizeof(char) * bufsize);
    int c;

    if (!buffer) {
      fprintf(stderr, "lsh: allocation error \n");
      exit(EXIT_FAILURE);
    }

    while (true) {
      c = getchar();

      if (c == EOF || c == '\n') {
	    buffer[position] = '\0';
	    return buffer;
      } else {
	    buffer[position] = c;
      }

      position ++;
      
      if (position >= bufsize) {
	    bufsize += LSH_RL_BUFSIZE;
	    buffer = realloc(buffer, bufsize);

	    if (!buffer) {
	        fprintf(stderr, "lsh: allocation error\n");
	        exit(EXIT_FAILURE);
	    }
      }
    }
  }
```
我习惯性的在函数的起始声明变量，这是旧的C语言风格，不用特别在意。该函数的核心在`while(true)`的死循环内。在循环中，我们读取一个字符（并将其存储为 int，而不是 char，因为 EOF 是整数，而不是字符，如果要拿到它，则需要使用 int，因为我也不知道char到底是unsigned char 还是 signed char）。如果拿到的字符是换行符或 EOF，我们就终止当前字符串并返回它。否则，我们将字符添加到现有字符串中。接下来，检查字符是否会超出我们当前的缓冲区大小。如果超出，就重新新分配缓冲区大小并且检查是否分配成功。

起始GNU C的stdio.h中有一个 getline() 函数，它完成了我们刚刚实现的大部分工作，这个函数是 C 库的 GNU 扩展，2008 年它被添加到规范中，所以大多数现代Unix以及类Unix操作系统应该有这个函数，所以我给出一个getline的实现方案，这个方案更加简单更加清晰，所以接下来我最终使用的`lsh_read_line()`是这个版本：
```c
#define LSH_RL_BUFSIZE 1024

char *
lsh_read_line(void)
  {
    char *line = NULL;
    ssize_t bufsize = 0;

    if (getline(&line, &bufsize, stdin) == -1) {
      if (feof(stdin)) {
	    exit(EXIT_SUCCESS);
      } else {
	    perror("readline");
	    exit(EXIT_FAILURE);
      }
    }
    
    return line;
  }
```

## Parsing the line
现在已经实现了 lsh_read_line()，有了这个函数我们就可以拿到输入的数据了。现在需要将该数据解析为需要运行的程序及其参数列表。我们简化一下工作：不允许在命令行参数中使用引号或反斜杠转义。我们将简单地使用空格来分隔参数，所以当你输入 `echo "this message"` 时不会使用单个 “this message” 参数传递给 echo ，而是使用两个参数 "this 和 message" 传递给echo。

简化之后，我们需要做的就是使用空格作为分隔符来分割字符串:
```c
char **
lsh_split_line(char *line)
  {
    int bufsize = LSH_TOK_BUFSIZE;
    int position = 0;
    char **tokens = malloc(bufsize * sizeof(char *));
    char *token;

    if (!tokens) {
      fprintf(stderr, "lsh: allocation error\n");
      exit(EXIT_FAILURE);
    }

    token = strtok(line, LSH_TOK_DELM);

    while (token != NULL) {
      tokens[position] = token;
      position ++;

      if (position >= bufsize) {
	    bufsize += LSH_TOK_BUFSIZE;
	    tokens = realloc(tokens, bufsize * sizeof(char *));

	      if (!tokens) {
	        fprintf(stderr, "lsh: allocation error\n");
	        exit(EXIT_FAILURE);
	      }
      }

      token = strtok(NULL, LSH_TOK_DELM);
    }

    tokens[position] = NULL;
    return tokens;
  }
```

这段代码看起来似乎和之前的第一版`lsh_read_line()`有些相似，它们俩都创建了一个缓冲区然后动态扩展这个缓冲区。除去一些必要的变量声明和变量有效性检查，函数是从`token = strtok(line, LSH_TOK_DELM);`开始的，strtok函数将会返回指向line中第一个LSH_TOK_DELM的指针，我们会根据LSH_TOK_DELM将line分成一小段一小段存储在tokens中，并且以NULL结束tokens。

# How shell start process
现在我们就差`lsh_execute()`函数了，这个函数的作用是根据解析好的命令去启动对应的程序，并将相应的参数传递给对应的程序。启动进程是我们这个shell的主要功能，编写一个shell需要你确切的知道进程发生了什么以及它们是如何启动的，所以接下来会歪个楼讲讲Unix操作系统中的进程。

在 Unix 上只有两种启动进程的方法。第一个是 Init，这种方案几乎可以忽略不计，当一台 Unix 计算机启动时，它的内核会被加载了。一旦内核被加载并初始化，就会启动了一个称为 Init 的进程。这个进程在计算机运行的整个时间段内运行，它管理加载我们的计算机所需的其他进程。

由于大多数程序不是 Init 程序，因此只有一种方法可以启动进程：fork()，这是一个system call。调用此函数时，操作系统会复制该进程并启动它们。原始进程称为“父进程”，新进程称为“子进程”。 fork() 向子进程返回 0，并向父进程返回其子进程的进程 ID 号 (PID)。从本质上讲，这意味着创建一个新的进程的唯一方法是自我复制现有的进程。

通常，当我们想要运行一个新进程时，我们并不想要同一个程序的另一个副本，而是想要运行不同的程序。这就是 exec() 的作用，这也是一个system call。它会用一个全新的程序替换当前正在运行的程序。这意味着当我们调用 exec 时，操作系统会停止我们的进程，加载新程序，并在其位置启动该程序。进程永远不会从 exec() 调用中返回（除非出现错误）。

有了这两个函数，我们就有了解决方案：首先，将现有进程将自身分成两个独立的进程。然后，子进程使用 exec() 将自己替换为新程序。父进程可以继续做其他事情，甚至可以使用系统调用 wait() 监视其子进程。

先让我们来看看一个`lsh_launch()`函数：
```c
lsh_launch(char **args)
  {
    pid_t pid, wpid;
    int status;
    pid = fork();

    if (pid == 0) {
      if (execvp(args[0], args) == -1) {
	      perror("lsh");
      }

      exit(EXIT_FAILURE);
    } else if (pid < 0) {
      perror("lsh");
    } else {
      do {
	      wpid = waitpid(pid, &status, WUNTRACED);
      } while (!WIFEXITED(status) && !WIFSIGNALED(status));
    }

    return 1;
  }
```
这个函数接收我们之前解析好的命令的参数列表，然后fork当前进程，并保存fork的返回值，一旦拿到了正确的fork的返回值，说明有另一个子进程运行起来了，在子进程中，我们要运行用户给出的命令。因此，我们使用 exec 系统调用的众多变体之一 —— execvp。 exec 的不同变体做的事情略有不同。有些采用可变数量的字符串参数。有些则使用字符串列表。还有一些可以指定进程运行的环境。而execvp则需要一个程序名称和一个字符串参数的数组（也称为vector，因此是exec`v`p）。 'p' 意味着我们不提供要运行的程序的完整文件路径，而是给出它的名称，让操作系统在路径中搜索程序。

如果 exec 命令返回 -1，就说明出现错误了。那么就使用 perror 打印系统的错误消息以及我们的程序名称，以便外界知道错误来自哪里。然后退出以便 shell 可以继续运行。

else表示 fork() 执行成功。父进程会在这里等待子进程执行的命令完成。所以使用 waitpid() 来等待进程的状态改变。不过，进程可以通过多种方式改变状态，但并非所有改变都会导致进程结束。一个进程可以退出，也可以被信号终止。这里我们使用 waitpid() 提供的宏来等待进程退出或终止。

# Shell Builtins
看样子`lsh_launch()`已经实现了`lsh_execute()`的逻辑了，那么`lsh_execute()`应该做些什么呢？事实上，很多shell都可以执行一些 ”内置的“ 命令，这些内置的命令虽然很少但不能没有啊，原因其实很简单。如果要更改目录属性，则需要使用函数 chdir()。可问题是，当前目录是进程的属性。因此，如果我们编写了一个名为 cd 的程序来更改目录，它只会更改自己的当前目录，然后终止。其父进程的当前目录将保持不变。相反，shell 进程本身需要执行 chdir()，以便更新自己的当前目录。当它启动子进程时，它们也将继承该目录。同样，如果有一个名为 exit 的程序，它就无法退出调用它的 shell。该命令还需要内置到 shell 中。此外，大多数 shell 都是通过运行配置脚本来配置的，例如 ~/.bashrc。这些脚本使用改变 shell 操作的命令。这些命令只有在 shell 进程本身实现时才能更改 shell 的操作。

所以我们需要向 shell 本身添加一些内置的命令。先添加 cd、exit 和 help 吧：
```c
char *builtin_str[] = { "cd", "help", "exit" };

int
lsh_num_builtins()
  {
    return sizeof(builtin_str) / sizeof(char *);
  }

int
lsh_cd(char **args)
  {
    if (args[1] == NULL) {
      fprintf(stderr, "lsh: expected argument to \"cd\"\n");
    } else {
      if (chdir(args[1]) != 0) {
	      perror("lsh");
      }
    }

    return 1;
  }

int
lsh_help(char **args)
  {
    int i;
    printf("Stephen Brennan's LSH\n");
    printf("Type program names and arguments, and hit enter.\n");
    printf("The following are built in:\n");

    for (i = 0; i < lsh_num_builtins(); i++) {
      printf("  %s\n", builtin_str[i]);
    }

    printf("Use the man command for information on other programs.\n");
    return 1;
  }

int
lsh_exit(char **args)
  {
    return 0;
  }

int (*builtin_func[]) (char **) = {
  &lsh_cd, &lsh_help, &lsh_exit
};
```

这段代码有点长，大致上有三个部分，第一部分定义了一些内置命令的名字（用字符串表示），第二部分定义了这些内置命令的实现，然后将这些内置命令归纳到一个函数指针数组中。

# Putting together builtins and processes
最后，让我们来实现`lsh_execute()`，这个函数将会启动内置的命令或是其他的进程：
```c
int
lsh_execute(char **args)
  {
    int i;

    if (args[0] == NULL) {
      return 1;
    }

    for (i = 0; i < lsh_num_builtins(); i ++) {
      if (strcmp(args[0], builtin_str[i]) == 0) {
	      return (*builtin_func[i])(args);
      }
    }

    return lsh_launch(args);
  }
```

lsh_execute检查命令是否是内置命令，如果是就运行内置的，否则就调用lsh_launch启动新的进程，除此之外，lsh_execute还会在一开始就检查命令是否是一个空的字符串。

# Putting it all together
你可以在 https://github.com/muqiuhan/Write-a-Shell-in-C 找到本文的代码，运行结果
```shell
> ls
a.out  LICENSE  simpleshell.c
> ls -lh LICENSE
-rw-r--r-- 1 muqiuhan muqiuhan 1.1K  6月28日 08:21 LICENSE
> cd /
> ls
bin  boot  dev  etc  home  lib  lib64  mnt  opt  personal  proc  root  run  sbin  srv  sys  tmp  usr  var  version
> exit
```

- https://en.wiktionary.org/wiki/shell
