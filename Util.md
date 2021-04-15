# Lab util: Unix utilities
## sleep
The first one is used for warming up.  Highlights of the instruction:

> - If the user forgets to pass an argument, sleep should print an error message.
> - Use the system call sleep.

Note that `kernel/types.h` and `kernel/stat.h` should be included into headers(you can look at other programs like `user/echo.c` or `user/grep.c` for reference) or you would get a compile error.

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char* argv[]){
    if(argc != 2){
        fprintf(2, "sleep: should be one argument\n");
        exit(1);
    }
    sleep(atoi(argv[1])); 
    exit(0);   
}
```

## pingpong

Recall that parent and child share the same file descriptors after fork which can be visualized as below. So parent have to wait for child to read and rewrite the byte, otherwise parent might read the byte written by itself and then exit before child does anything.

(Image from: [https://www.geeksforgeeks.org/pipe-system-call/](https://www.geeksforgeeks.org/pipe-system-call/))

![https://media.geeksforgeeks.org/wp-content/uploads/sharing-pipe.jpg](https://media.geeksforgeeks.org/wp-content/uploads/sharing-pipe.jpg)

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(){
    int fd[2];
    char buf[1], str[20];
    pipe(fd);
    if(fork() == 0){
        read(fd[0], buf, 1); // Wait for parent sends the byte.
        str[0] = getpid() + '0';
        strcpy(str+1, ": received ping\n");
        write(1, str, 17);
        write(fd[1], buf, 1);
    }
    else{
        buf[0] = 'a';
        write(fd[1], buf, 1); 
        // Wait for child to read and rewrite,
        // otherwise parent might read the byte written by itself
        // and then exit before child does anything.
        wait(0);
        read(fd[0], buf, 1);
        str[0] = getpid() + '0';
        strcpy(str+1, ": received pong\n");
        write(1, str, 17);
    }
    exit(0);
}
```

## primes

Be aware that before starting reading from the left neighbour(while loop in `sieve()`), you **must** close the write-side of the left(old) pipe for child in order to gaurantee that `read` will return 0 at the end.

Highlights of the instruction:

> - Once the first process reaches 35, it should wait until the entire pipeline terminates.
> - `read` returns zero when the write-side of a pipe is closed.
> - Directly write 32-bit (4-byte) ints to the pipes, rather than using formatted ASCII I/O.

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void sieve(int *fd){
    int num;
    read(fd[1], &num, 4);
    // The output below would not be interrupted.
    printf("prime %d\n", num);
    // 32 is the last prime number for this program.
    if(num == 32){
        close(fd[2]);
        close(fd[1]);
        return;
    }
    int nextfd[3];
    pipe(nextfd);
    if(fork() == 1){
        // Release previous fd to prevent running out of resources
        close(fd[1]);
        close(fd[2]);
        // No need to close nextfds since they will be closed 
        // in the next process(as parent)
        sieve(nextfd);
        exit(0);
    }
    else{
        // The first num that is read would be the current prime number.
        int prime = num;
        // Close the write-side of pipe in order to guarantee 
				// read will return 0 at the end. 
        close(fd[1]);
        while(read(fd[0], &num, 4) > 0){
            // If the current num is not a multiple of prime,
            // then pass it to the next process,
            // otherwise just ignore it.
            if(num % prime != 0){
                write(nextfd[1], &num, 4);
            }
        }
        close(fd[0]);
        close(nextfd[1]);
        close(nextfd[0]);
        wait(0);
        // This parent will then return to be
        // child in the former level and exit(line 19).
    }
}

int main(){
    int fd[2];
    pipe(fd);
    if(fork() == 0){
        // No need to close fds since they will be closed in sieve()
        sieve(fd); 
    }
    else{
        int i = 2;
        for(;i<=35;i++){
            write(fd[1], &i, 4);
        }
        close(fd[1]);
        close(fd[0]);
        wait(0);
    }
    exit(0);
}
```

## find

`user/ls.c` is USEFUL. Also check `kernel/fs.h` for definitions that are used. 

Highlights of the instruction:

> - Use recursion to allow find to descend into sub-directories.
> - Don't recurse into "." and "..".

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

char* fmtname(char* path){
    static char buf[DIRSIZ+1];
    char *p;
    // Find first character after last slash.
    p = path + strlen(path);
    while(p >= path && *p != '/')
        p--;
    p++;
    // Return blank-padded name.
    if(strlen(p) >= DIRSIZ)
        return p;
    memmove(buf, p, strlen(p));
    memset(buf+strlen(p), ' ', DIRSIZ-strlen(p));
    return buf;
}

void find(char* path, char* target){
    int fd;
    char buf[512], *p;
    struct stat st;
    struct dirent de;

    if((fd = open(path, 0)) < 0){
        fprintf(2, "find: cannot open %s\n", path);
        exit(1);
    }
    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        exit(1);
    }
    switch(st.type){
        // If path is a file, then just check whether 
        // it's name is the same as target.
        case T_FILE:
            if(strcmp(fmtname(path), target) == 0){
                write(1, path, sizeof(path));
                write(1, "\n", 1);
            }
        // If path is a directory, then loop for each file under it 
        // to check their names; for subdirectories, invoke find() again.
        case T_DIR:
            if(strlen(path) + 1 + DIRSIZ + 1 > sizeof(buf)){
               printf("find: path too long\n");
               break;
            }
            strcpy(buf, path);
            // Put a slash at the end of buf.
            p = buf + strlen(buf);
            *p++ = '/';
            // Loop for each entry under current directory;
            // note that "a directory is a sequence of dirent struct"(see fs.h).
            while(read(fd, &de, sizeof(de)) == sizeof(de)){
                if(de.inum == 0)
                    continue;
                // Omit '.' and '..'
                if(strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
                    continue;
                // Construct complete path to current file.
                memcpy(p, de.name, DIRSIZ);  // memcpy is equal to memmove here(see ulib.c).
                p[DIRSIZ] = '\0';
                if(strcmp(de.name, target) == 0){
                    printf("%s\n", buf);
                }
                if(stat(buf, &st) < 0){
                    printf("find: cannot stat %s\n", buf);
                    continue;
                }
                // Recurse for subdirectories
                if(st.type == T_DIR){
                    find(buf, target);
                }
            }
            break;
    }
    close(fd);
}
                
int main(int argc, char* argv[]){
    if(argc != 3){
        fprintf(2, "find: must be 2 argumants\n");
        exit(1);
    }
    find(argv[1], argv[2]);
    exit(0);
}
```

## xargs

The trickiest part for me is to construct the new argv array. Just remember to `malloc` memory for an uninitialized pointer before using it lol.

Highlights of the instruction:

> - read lines from the standard input
> - Use `fork` and `exec` to invoke the command on each line of input. Use `wait` in the parent to wait for the child to complete the command.
> - To read individual lines of input, read a character at a time until a newline ('`\n`') appears.
> - `kernel/param.h` declares `MAXARG`, which may be useful if you need to declare an argv array.

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"

int main(int argc, char* argv[]){
    if(argc == 1){
        fprintf(2, "xargs: missing argument(s)\n");
        exit(1);
    }
    char *newArgv[MAXARG], buf[512], *p;
    int i = 0;
    // Copy arguments given in xargs.
    for(; i<argc-1; i++){
        newArgv[i] = malloc(strlen(argv[i+1]));
        strcpy(newArgv[i], argv[i+1]);
    }
    memset(buf, '\0', 512);
    p = buf;
    // Read from stdin byte by byte until the end.
    while(read(0, p, 1)){
        // A new argument.
        if(*p == ' '){
            *p = '\0';
            newArgv[i] = malloc(strlen(buf));
            strcpy(newArgv[i++], buf);
            p = buf;
            memset(buf, '\0', strlen(buf));
        }       
        // When finish reading a line, execute the command.
        else if(*p == '\n'){
            *p = '\0';
            newArgv[i] = malloc(strlen(buf));
            strcpy(newArgv[i], buf);
            if(fork() == 0){
                exec(argv[1], newArgv); 
            }   
            else{
                wait(0);
            }
            // Reset
            for(int j=argc-1; j<MAXARG; j++){
                if(strcmp(newArgv[j], "\0") == 0)
                    break;
                memset(newArgv[j], '\0', strlen(newArgv[j]));
            }
            memset(buf, '\0', strlen(buf));
            p = buf;
            i = argc;
        } 
        else
            p++;
    }
    exit(0);
}
```
