fork系统调用是Linux系统层面的函数，原理主要涉及到进程的创建和内存管理。

首先，fork()函数用于创建一个新的进程，这个新进程是当前进程的复制品，称为子进程。调用fork()的进程则被称为父进程。

其次，在fork()执行的过程中，子进程的地址空间会复制父进程的地址空间，包括代码段、数据段、堆栈以及寄存器内容等。这样，子进程就拥有了和父进程完全相同的内存镜像。另外，这种复制是“写时才复制”（Copy-On-Write）的，在fork()之后，一开始父进程和子进程仍然是共享相同的物理页面，直到其中一个进程试图修改页面内容时，系统才会复制该页面，这样可以避免不必要的内存拷贝，提高了性能和效率。

最后，fork()函数的一个特点是它“父进程调用一次，返回两次结果”。这是区分父进程和子进程的一个关键特征。
- 第一个结果是在父进程执行fork()返回新创建的子进程的进程ID（PID），
- 第二个结果是在子进程中，fork()则返回0。
fork源码如下：
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int main(int argc, char * argv[])
{
    int pid;
    /* fork another process */
    pid = fork();
    if (pid < 0) 
    { 
        /* error occurred */
        fprintf(stderr,"Fork Failed!");
        exit(-1);
    } 
    else if (pid == 0) 
    {
        /* child process */
        printf("This is Child Process!\n");
    } 
    else 
    {  
        /* parent process  */
        printf("This is Parent Process!\n");
        /* parent will wait for the child to complete*/
        wait(NULL);
        printf("Child Complete!\n");
    }
}
```
