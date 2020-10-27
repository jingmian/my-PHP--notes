#wait
首先简单介绍一下,在操作系统中底层其实已经封装一个父进程复制一个子进程,通过`fork()`函数.而`wait()`函数也是在`c`语言中实现的,具体为了等待子进程退出后,父进程收到一个子进程退出信号从而释放系统资源,**如果子进程一直不退出,父进程就一直阻塞等待**
##pcntl_wait
`pcntl_wait()`其实就是封装了底层`c`语言提供的`wait()`函数.[点击手册](https://www.php.net/manual/zh/function.pcntl-wait.php)进入,可以看到如下解释
```sh
wait函数刮起当前进程的执行直到一个子进程退出或接收到一个信号要求中断当前进程或调用一个信号处理函数。 如果一个子进程在调用此函数时已经退出（俗称僵尸进程），此函数立刻返回。子进程使用的所有系统资源将 被释放。关于wait在您系统上工作的详细规范请查看您系统的wait（2）手册。
```
###wait(2)手册
最后一句话提示更多详情参考`您系统上工作的详细规范请查看您系统的wait（2）手册`.
或许可能看到你还不知道从哪里看这些手册信息.
方法在linux系统输入:
```sh
man wait  
```
我只摘录上半部分简单说一下,更多的信息请自行手动一遍
```man
WAIT(2)            Linux Programmer's Manual           WAIT(2)

NAME
       wait,  waitpid,  waitid  -  wait  for process to change
       state

SYNOPSIS
       #include <sys/types.h>
       #include <sys/wait.h>

       pid_t wait(int *wstatus);

       pid_t waitpid(pid_t pid, int *wstatus, int options);

       int waitid(idtype_t idtype, id_t id, siginfo_t  *infop,
       int options);
                       /*  This  is the glibc and POSIX inter‐
       face; see
                          NOTES for  information  on  the  raw
       system call. */

   Feature   Test  Macro  Requirements  for  glibc  (see  fea‐
   ture_test_macros(7)):

       waitid():
           Since glibc 2.26: _XOPEN_SOURCE >= 500 ||
               _POSIX_C_SOURCE >= 200809L
           Glibc 2.25 and earlier:
               _XOPEN_SOURCE
                   || /* Since glibc 2.12: */
           _POSIX_C_SOURCE >= 200809L
                   || /* Glibc versions <= 2.19: */
           _BSD_SOURCE

DESCRIPTION
       All of these system calls are used to  wait  for  state
       changes  in  a child of the calling process, and obtain
       information about the child whose state has changed.  A
       state change is considered to be: the child terminated;
       the child was stopped by a signal;  or  the  child  was
       resumed  by  a  signal.   In  the  case of a terminated
       child, performing a wait allows the system  to  release
       the  resources  associated with the child; if a wait is
       not performed, then the terminated child remains  in  a
       "zombie" state (see NOTES below).

wait() and waitpid()
       The wait() system call suspends execution of the  call‐
       ing  process until one of its children terminates.  The
       call wait(&wstatus) is equivalent to:

           waitpid(-1, &wstatus, 0);

       The waitpid() system call  suspends  execution  of  the
       calling process until a child specified by pid argument
       has changed state.  By default,  waitpid()  waits  only
       for  terminated  children, but this behavior is modifi‐
       able via the options argument, as described below.

       The value of pid can be:

       < -1   meaning wait for any child process whose process
              group ID is equal to the absolute value of pid.

       -1     meaning wait for any child process.

       0      meaning wait for any child process whose process
              group  ID  is  equal  to  that  of  the  calling
              process.

       > 0    meaning  wait  for the child whose process ID is
              equal to the value of pid.

       The value of options is an OR of zero or  more  of  the
       following constants:

       WNOHANG     return immediately if no child has exited.

       WUNTRACED   also return if a child has stopped (but not
                   traced via ptrace(2)).  Status  for  traced
                   children  which  have  stopped  is provided
                   even if this option is not specified.

       WCONTINUED (since Linux 2.6.10)
                   also return if a  stopped  child  has  been
                   resumed by delivery of SIGCONT.

       (For Linux-only options, see below.)

       If wstatus is not NULL, wait() and waitpid() store sta‐
       tus information in the int to which  it  points.   This
```
留心观察下几个参数
`WNOHANG`如果子进程没有退出立马返回

所以在php手册中
`pcntl_wait ( int $pid , int &$status [, int $options = 0 ] ) : int`中`$options`对应这个参数的,以下实例代码就可以解释为什么`wait()`函数没有阻塞等待,是因为添加`WNOHANG`参数

```php
<?php


$sock = stream_socket_server("tcp://0.0.0.0:9501", $errno, $errstr);

$pids = [];


for ($i=0; $i<2; $i++) {

    $pid = pcntl_fork();
    $pids[] = $pid;

    if ($pid == 0) {
        for ( ; ; ) {
            $conn = stream_socket_accept($sock);

            $write_buffer = "HTTP/1.0 200 OK\r\nServer: my_server\r\nContent-Type: text/html; charset=utf-8\r\n\r\nhello!world";

            fwrite($conn, $write_buffer);

            fclose($conn);
        }

        exit(0);
    }

}
$num=0;
while(count($pids) > 0) {
    $num++;
    foreach($pids as $key => $pid) {
        $res = pcntl_wait($status);
        var_dump("--{$num}--",$status,$res);
        // If the process has already exited
        if($res == -1 || $res > 0)
            unset($pids[$key]);
    }

    sleep(5);
}
```

