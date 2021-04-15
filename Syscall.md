# Lab syscall: System calls

I think that understanding what you need to build through coding is much more important and harder than coding, in all cases. During this lab, I spent hours to figure out what usage I have to implement for each function. One of the reasons is that I misunderstood the instructions....So I wrote a complete procedure below about how the new system call is expected to work, what you need to do and also some tips for implementing. If you're being confused too, hope this can help :)

## Using GDB for debugging xv6

```bash
make qemu-gdb
# then go to a new window
cd xv6-labs-2020/kernel
gdb kernel
```

Resources:\
[https://stackoverflow.com/questions/10534798/debugging-user-code-on-xv6-with-gdb](https://stackoverflow.com/questions/10534798/debugging-user-code-on-xv6-with-gdb)
[https://www.cse.iitd.ernet.in/~sbansal/os/previous_years/2014/lec/l3-hw1.html](https://www.cse.iitd.ernet.in/~sbansal/os/previous_years/2014/lec/l3-hw1.html)

## System call tracing

### **What will happen after running `trace {mask} {another command}` ?**

First, `user/trace.c` will be run. On the way of it, `sys_trace` will be called by `syscall()` in `kernel/syscall.c`(*I don't know exactly when, since I can't see any invocation in trace.c. I will come back to check after getting more familiar with gdb.*). `sys_trace` will retrieve the trace mask from user space, store it into the current process data and then return. After that, `trace.c` will execute `{another command}`. Whenever a system call is called and return back to `syscall()`, the mask will be used to check whether it should print the trace output. If `fork` system call is called, then the parent will copy all data to the child process, including the mask, which should be operate by `fork()` in `kernel/proc.c`.

### You need to

1. Follow the instructions to fix the compilation issues.
2. Add a new variable in the proc structure.
3. Add a `sys_trace` function in `kernel/sysproc.c` which retrieve a single argument, a.k.a. the trace mask, from user space and save it into the new variable created just now.
4. Modify `fork()` to copy the trace mask from the parent to the child process.
5. Modify `syscall()` to print the trace output when nesscessary. An array of syscall names has to be added.

### My solutions

`kernel/sysproc.c`

```c
int
sys_trace(void)
{
    struct proc *p = myproc();
    argint(0, &(p->mask));
    return 0;
}
```

`kernel/proc.c`

```c
int
fork(void)
{
	// ...
	// the original codes 
	// ...

	np->mask = p->mask;

	return pid;
}
```

`kernel/syscall.c`

```c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    if((1 << num) & p->mask)
        printf("%d: syscall %s -> %d\n", p->pid, syscallname[num-1], p->trapframe->a0);
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

## Sysinfo

I found this one is easier(to understand) than the previous one.

### You need to

1. Follow the instructions to fix the compilation issues.
2. Write the two helper functions.
3. Add a `sys_trace` function in `kernel/sysproc.c` which invokes the two helper functions to fill out a `sysinfo` structure and copy it back to user space using `copyout()`.

### Tips

- Before writing the two functions, have a look of the variables and some(not all) of the other functions in `kalloc.c` and `proc.c` in order to have an general idea about what tools you have.
- Don't forget to modify `kernel/syscall.c`.
- Memory is allocated/freed page by page and each of pages' size is `PGSIZE` bytes.
- `extern` keyword might be helpful.
- Remember to include `sysinfo.h`.

### My solutions

`kernel/kalloc.c`

```c
#include "sysinfo.h"

// Collect the number of bytes of free memory
uint64 
countfreemem(void){
    int count = 0;
    struct run *r = kmem.freelist;
    while(r && (uint64)r < PHYSTOP){
        count += PGSIZE;
        r = r->next; 
    }
    return count;
}
```

`kernel/proc.c`

```c
#include "sysinfo.h"

// Count the number of processes whose state is not UNUSED
uint64
countproc(void){
    int count = 0;
    struct proc *p;
    for(p=proc; p<&proc[NPROC]; p++){
      if(p->state != UNUSED)             
          count++;
    }
    return count;
}
```

`kernel/sysproc.c`

```c
#include "sysinfo.h"

extern uint64 countfreemem(void);
extern uint64 countproc(void);

int
sys_sysinfo(void){
    uint64 uaddr;
    struct sysinfo si;
    struct proc *p = myproc();
    si.freemem = countfreemem();
    si.nproc = countproc();
    if(argaddr(0, &uaddr) < 0)
        return -1;
    if(copyout(p->pagetable, uaddr, (char*)&si, sizeof(si)) < 0)
        return -1;
    return 0;
}
```
