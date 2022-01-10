# How to on starting processes (mostly in Linux)

Reformat [How to on starting processes (mostly in Linux)](https://iximiuz.com/en/posts/how-to-on-processes/) by [Ivan Velichko](https://twitter.com/iximiuz)


## Intro

Do you want to run an executable file from your program? Or execute a shell command programmatically? Or maybe just parallelize your code? Have you read a lot of information regarding [`execve()`](https://en.wikipedia.org/wiki/Exec_(system_call)) functions family and [`fork()`](https://en.wikipedia.org/wiki/Fork_(system_call)) but still have a mess in your head? Then this article is for you.

## How to start Linux process

### System calls

Let's keep it simple and start from the beginning. We are developing a program for Linux. Let's have a look on so called [system calls](https://en.wikipedia.org/wiki/System_call) - the interface Linux provides us to request [kernel](https://en.wikipedia.org/wiki/Linux_kernel) functionalities.

Linux has the next system calls to work with processes:

*   `fork(void)` (`man 2 fork`) - creates a full copy of the calling process. Sounds ineffecient because of need of copying the enterie process's address space, but uses copy-on-write optimization. **This is the only (ideological) way to create a process in Linux**. However, in fresh versions of the kernel `fork()` is implemented on top of tricky `clone()` system call and now it's possible to use `clone()` directly to create processes, but for simplicity we are going to skip these details.
*   `execve(path, args, env)` (`man 2 execve`) - transforms the calling process into a new process by executing a file under the specified `path`. In effect, it replaces the current process image with a new process image and doesn't create any new processes.
*   `pipe(fildes[2] __OUT)` (`man 2 pipe`) - creates a pipe which is an inter-process communication primitive. Usually pipes are unidirectional data flows. The first element of the array connects to the _read end_ of the pipe, and the second element connects to the _write end_. The data written to `fildes[1]` can be read from the `fildes[0]`.

We are not going to have a look at the aforementioned system calls source code because it's a part of the kernel and could be hardly understandable.

Also an important part of our consideration is the Linux [**shell**](https://en.wikipedia.org/wiki/Unix_shell) - a command interpreter utility (i.e. regular program). The shell process constantly reads from the stdin. A user usually interacts with the shell by typing some commands and pressing `enter` key. The shell process then executes provided commands. Standard outputs of these processes are connected to the stdout of the shell process. However, the shell process can be launched as a subprocess by itself and the command to execute can be specified via `-c` argument. Eg. `bash -c "date"`.

### C standard library

Of course we are developing our program in `C` to be as close to the OS-level primitives as possible. `C` has a so called [standard library](https://en.wikipedia.org/wiki/C_standard_library) `libc` - a broad set of functions to simplify writing programs in this language. Also it provides wrapping around syscalls.

C standard library has the next functions (on Debian-based distros `apt-get download glibc-source`):

* `system(command)` (`man 3 system`) - launches a `shell` process to execute provided `command`. The calling process is blocked till the end of the execution of the underlying `shell` process. `system()` returns an exit code of the shell process. Let's have a look on the [implementation](http://www.retro11.de/ouxr/211bsd/usr/src/lib/libc/gen/system.c.html) of this function in the stdlib: 

```c
int system(char *command) { 
    // ... skip signals tricks for simplicity ...
    
    switch(pid = vfork()) { 
        case -1: // error 
            // ... 
        case 0: // child 
            execl("/bin/sh", "sh", "-c", command, (char *)NULL); 
            _exit(127); // will be called only if execl() returns, i.e. a syscall faield.
    }
    
    // ... skip signals tricks for simplicity ...
    
    waitpid(pid, (int *)&pstat, 0); // waiting for the child process, i.e. shell.
    return pstat.w_status;
}
```

So in effect, `system()` just uses the combination of the `fork()` + `exec()` + `waitpid()`.
    
* `popen(command, mode = 'r|w')` (`man 3 popen`) - forks and replaces the forked process with a shell instance executing provided command. Sounds pretty the same like the `system()`? The difference is an abilitiy to communicate with the child process via its stdin or stdout. But usually in the unidirectional way. To communicate with this process a `pipe` is used. Real implementations can be found [here](http://www.retro11.de/ouxr/211bsd/usr/src/lib/libc/gen/popen.c.html) and [here](https://github.com/bminor/glibc/blob/09533208febe923479261a27b7691abef297d604/libio/iopopen.c) but the main idea is the following:
    
```c
FILE * popen(char *program, char *type)
{ 
    int pdes\[2\], fds, pid;

    pipe(pdes);  // create a pipe
    
    switch (pid = vfork()) { // fork the current process
    case -1:            // error
        // ...
    case 0:             // child
        if (*type == 'r') {
            dup2(pdes[1], fileno(stdout));  // bind stdout of the child process to the writing end of the pipe
            close(pdes[1]);
            close(pdes[0]);                 // close reading end of the pipe on the child side
        } else {
            dup2(pdes[0], fileno(stdin));  // bind stdin of the child process to the reading end of the pipe
            close(pdes[0]);
            close(pdes[1]);                // close writing end of the pipe on the child side
        }
        execl("/bin/sh", "sh", "-c", program, NULL);  // replace the child process with the shell running our command
        _exit(127);  // will be called only if execl() returns, i.e. a syscall faield.
    }
    
    // parent
    if (*type == 'r') {
        result = pdes[0];
        close(pdes[1]);
    } else {
        result = pdes[1];
        close(pdes[0]);
    }
    return result;
}
```
    
Congratulations, that it!
    
**NB1**: shell implementation of the subprocess launching is pretty the same. i.e. `fork()` + `execve()`.
    
**NB2**: it's good to mention that other programming languages usually implement bindings to the OS's libc (and do some wrappings for convenience) to provide OS-specific funtionality.
    
## Why to start Linux process
    
### Parallelize execution
    
The simplest one. We need only `fork()`. Call of the `fork()` in effect duplicates your program process. But since this process uses completely separate address space to communicate with it we anyway need [inter-process communication primitives](https://en.wikipedia.org/wiki/Inter-process_communication). Even the instructions set of the forked process is the same as the parent's one, it's a different instance of the program.
    
### Just run a program from your code

If you need just to run a program, without communicating with its stdin/stdout the libc `system()` function is the simplest solution. Yep, you also can `fork()` your process and then run `exec()` in the child process, but since it's a quite common scenario there is `system()` function.
    
### Run a process and read its stdout (or write to its stdin)

We need `popen()` libc function. Yep, you still can achieve the goal just by combining `pipe()` + `fork()` + `exec()` as shown above, but `popen()` is here to reduce the amount of boilerplate code.

### Run a process, write to its stdin and read from its stdout
    
The most interesting one. For some reasons default `popen()` implementation is usually unidirectional. But looks like we can easily come up with the bidirectional solution: we need two pipes, first will be attached to child's stdin and the second one to the child's stdout. The remaining part is to `fork()` a child process, connect pipes via `dup2()` to IO descriptors and `execve()` the command. One of the potential implementations can be found on my GitHub [popen2() project](https://github.com/iximiuz/popen2). An extra thing you should be aware while developing such function is [*leaking*](https://gist.github.com/iximiuz/65c7d2d128c374ef83d885dfef74bed7) of open file descriptors of pipes from previously opened via `popen()` processes. If we forget to close explicitly foreign file descriptors in each child fork, there will be a possibility to do IO operations with the siblings' `stdin`s and `stdout`s. Sounds like a vulnerability. To be able to close all those file descriptors we have to track them. I used a `static` variable with the linked list of such descriptors:
    

```c
static files_chain_t *files_chain;

file_t *popen2(const char *command) { 
    file_t *fp = malloc(); // allocate new element of the chain

    _do_popen(fp, command);
    
    // add the current result to the chain
    fp->next = files_chain;
    files_chain = fp;
}

_do_popen() { 
    // open pipes 
    // fork() 
    // if is_child: 
    // for (fp in files_chain): 
    // close(fp->in); 
    close(fp->out); 
}

int pclose2(file_t *fp) { 
    // if (fp in files_chain): 
    / ... do payload ... 
    // remove fp from the chain
    free(fp);  // DO NOT FORGET TO FREE THE MEMORY WE ALLOCATED DURING popen2() CALL
}
```
    
## A few words about Windows

Windows OS family have a slightly different paradigm for working with processes. If we skip neoteric [Unix compatibility layer](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux) introduced on Windows 10 and attempts to [port](http://www.cygwin.com/) POSIX API support for Windows we will have only two functions from the oldschool WinAPI:

- [`CreateProcess(filename)`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms682425(v=vs.85).aspx) - starts a brand new process for a given executable.

- [`ShellExecute(Ex)(command)`](https://msdn.microsoft.com/en-us/library/windows/desktop/bb762153(v=vs.85).aspx) - starts a shell (yep, Windows also has a shell concept) process to execute provided command.
    
So, no `fork`s and `execve`s. However, to communicate with the started processes pipes also can be used.
    
    
## Instead of conclusions

Make code not war!