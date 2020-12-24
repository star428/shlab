# shlab

write a sample shell in linux

    首先考虑相关要写的代码。

## relevant function

    eval：解析和解释命令行的主例程（70 lines）
    builtin_cmd：识别和解释内置命令（25 lines）
    do_bgfg: 实现 bg 和 fg 内置命令（50 lines）
    waitfg：等待前台工作完成（20 lines）
    sigchld_handler：处理 SIGCHILD 信号（80 lines）
    sigint_handler：处理 SIGINT 信号（15 lines）
    sigtstp_handler：处理 SIGTSTP 信号（15 lines）

### eval

首先我们eval函数需要完成以下功能：

* 分析命令行给出时候为前台进程或者后台进程，同时把相关的处理交给argv

```c
bg = parseline(buf, argv);
```

* 下一步为判定`argv[0]`是否为内置命令，是转入`builtin_cmd(argv)`进行处理，不是则认为`argv[0]`为可执行文件路径，使用`fork()`和`execve()`产生子进程。

> Attention：这个时候应该注意信号屏蔽与取消屏蔽的时间，在加入job之前主进程都应该屏蔽SIGCHLD信号直到加入后才能取消屏蔽，同时在子进程fork后execve之前应该取消屏蔽因为子进程会继承父进程的屏蔽序列。

* 下一步为判定是否为前台进程，如果是，那么调用`waitfg`显式的等待子进程终止，为后台进程就直接输出相关信息不需要显示等待它的结束。

### builtin_cmd

函数的功能为分析是否为内置功能，内置功能有四个，分别为`quit`，`bg`， `fg`， `jobs`，对于每个内置命令，使用如下指令判断`argv[0]`中的值是否符合，若符合则转相关的处理代码。

```c
!strcmp(argv[0], "quit");//以quit为例
```

> Attention: 注意在访问全局变量时应阻塞所有的信号以免发生错误（参照安全的信号处理规则）。

### waitfg

该函数处理前台进程，等待前台进程直到停止，此时为合理占用CPU资源使用`sigsuspend`函数，该函数暂时使用`prev`的阻塞同时挂起进程直到收到一个信号（无论信号类型）。

```c
while (pid == fgpid(jobs)) //当前pid是前台进程进入，加入while是防止多个sigint信号冲入
  {                          //fgpid有就会返回前台子进程的pid，没有就返回0
    sigsuspend(&prev);       //挂起主shell进程，一直运行前台子进程，同时暂时设置屏蔽为prev
  }   
```

> Attention: 注意在访问全局变量jobs的时候要注意阻塞所有信号。

### sigchld_handler

该函数处理子进程发送`SIGCHLD`信号，此时子进程无论是停止或者终止都会向父进程发送`SIGCHLD`信号，所以处理该信号是我们的`waitpid`应该使用`options`选项，将`options`设置为`WNOHANG | WUNTRACED`,效果为立即返回，如果等待集合里面的子进程都没有停止或终止，那么返回0,如果有一个停止或终止，返回pid，如果没有这个的话如果有停止的子进程则会一直运行卡死在这个循环(默认设置只处理终止)。

对于`SIGCHLD`信号有三种情况，分别要考虑：

* 正常终止，此时应该在`jobs`中删除相关任务。

    ```c
    if WIFEXITED (status)
    {
      deletejob(jobs, pid);
    }
    ```

* 异常终止，前台作业组接收到父进程（shell）发送的`SIGINT`信号，然后子进程异常终止向父进程发送`SIGCHLD`信号处理。删除相关任务。
  
    ```C
    else if (WIFSIGNALED(status)) //非0表示程序异常终止
    {
      printf("Job [%d] (%d) terminated by signal %d\n", pid2jid(pid), pid, WTERMSIG(status));
      deletejob(jobs, pid);
    }
    ```

* 子进程停止：同异常终止，此时应该把前台子进程状态设置为ST，然后就会停止运行（前台处理的`waitfg`会返回0此时没有前台进程，然后开始下一个`tsh>`命令行。

### sigint_handler && sigtstp_handler

首先要注意这两个函数都是针对我们目前自己写的`shell`的前台进程决定的，此时我们是在标准`Unix shell`上运行，我们的shell程序自然会被当作为标准`Unix shell`的前台作业，当键入`ctrl+c` `ctrl+z`的时候，`Unix shell`会自动将`SIGINT` `SIGTSTP`信号传递给我们的shell进程，此时我们的`shell`如果在处理它的前台作业在使用`sigsuspend(&prev)`在等待信号，那么就会接收到`SIGINT` `SIGTSTP`，然后转相关的信号处理程序。此时我们的信号处理程序需要做到的为向我们的`shell`的所有前台作业内的进程发送`SIGINT` `SIGTSTP`信号，那么使用`kill`函数来给前台进程发送信号。

  ~~~c
  kill(-pid, SIGINT); //kill后前台job接受信号然后异常终止
  ~~~

> Attention: 注意在获取前台进程的pid的时候访问了jobs全局变量此时应该阻塞所有信号。

### do_bgfg

此时处理的为最难的部分，首先我们要分析给出的为`JID`还是`PID`，
然后再取出这个后台作业的所有信息（包括`state`，`pid`，`jid`）。

  ```C
  else if (argv[1][0] == '%')//处理jid
  {
    jid = atoi(argv[1] + 1); // 跳过第一个字符从下一个开始选
    job = getjobjid(jobs, jid);
    if (job == NULL)
    {
      printf("/% %d: No such job\n", jid);
      return;
    }
    pid = job->pid;
  }
  ```

然后根据`bg`和`fg`的相关信息设置不同的操作，对于`bg`，为重启该后台进程同时在后台重新执行，那么只需要改变该`job`的状态为`BG`同时向该作业（进程组）发送`SIGCONT`信号即可.对于`fg`比较复杂，此时重启的作业需要在前台运行，此时需要显示的调用`waitfg`（因为能输入`bg`内置命令意味着此时没有前台进程），此时显式等待等待该进程结束发送`SIGCHLD`此时父进程`shell`再处理信号再结束开始下一个`tsh>`。

  ~~~c
  if (!strcmp(argv[0], "bg"))
  {
    job->state = BG;
    printf("[%d] (%d) %s\n", jid, pid, job->cmdline);
    //sigprocmask(SIG_SETMASK, &prev, NULL);
    kill(-pid, SIGCONT);
    return;
  }
  ~~~
  
  ~~~c
  else if (!strcmp(argv[0], "fg"))
  {
    job->state = FG;
    sigprocmask(SIG_SETMASK, &prev, NULL);
    kill(-pid, SIGCONT);
    waitfg(pid);
    return;
  }
  ~~~

> Attention: 此时同时得注意在访问jobs全局变量的时候需设置屏蔽，但是在fg处理时因为要显示的接收信号所以得取消相关的屏蔽。









