# Lab traps: Trap

## Using GDB for debugging xv6

I had met some troubles when I tried to use GDB to debug xv6 and I finally tackled the problem using the method below.

```bash
make qemu-gdb
# then go to a new window
echo "add-auto-load-safe-path YOUR_PATH/xv6-labs-2020/.gdbinit " >> ~/.gdbinit
cd xv6-labs-2020
gdb-multiarch
```

Resources:
[https://zhuanlan.zhihu.com/p/342402097](https://zhuanlan.zhihu.com/p/342402097)

## RISC-V assembly

1. **Which registers contain arguments to functions? For example, which register holds `13` in `main`'s call to `printf`?**\
Register `a0` to `a7` are designed for storing arguments of functions. In this case, register `a2` holds 13 for `printf`.
2. **Where is the call to function `f` in the assembly code for main? Where is the call to `g`? (Hint: the compiler may inline functions.)**\
There is no call to both function `f` and `g` because of the optimization of the compiler. The call to `g` is replaced with the direct codes and `f` in `main` turns to be `12` which is the final value instead.
3. **At what address is the function `printf` located?**\
`0x630`
4. **What value is in the register ra just after the `jalr` to printf in `main`?**\
`ra` should contain the address of the next instruction after finishing `printf`, namely the very beginning of `exit()`, that is `0x38`.
5. **Run the following code.
        `unsigned int i = 0x00646c72;`
        `printf("H%x Wo%s", 57616, &i);`
What is the output? Here's an ASCII table that maps bytes to characters.
The output depends on that fact that the RISC-V is little-endian. If the RISC-V were instead big-endian what would you set i to in order to yield the same output? Would you need to change `57616` to a different value?
Here's a description of little- and big-endian and a more whimsical description.**\
The output is `He110 World`.
At the time of printing strings, the addresses are always read form low to high no matter which endian type is but compiler will automatically translate the address to the proper condition for any other types. Therefore, if the RISC-V is set to be big-endian, `i` should change to `0x726c6400` in order to obtain the identical result, whereas `57616` can stay unmodified.
6. **In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?
        `printf("x=%d y=%d", 3);`**\
The value stored in register `a2`, whose specific value depends on the earlier running of the program, is going to be printed due to the lack of the third argument.

## Backtrace

I think the key point in this part is that you have to remember a pointer is actually an address. The return value of  `r_fp()` is the frame pointer in type `uint64` and thus requires us to do some type convertions later when using it.

### My solution

`printf.c`

```c
void
backtrace(void)
{
    uint64 fp = r_fp();
    uint64 currpg = PGROUNDUP(fp);
    uint64 ra;

    printf("backtrace:\n");
    while(fp != currpg){
        ra = *(uint64*)(fp - 8);
        printf("%p\n", ra);
        fp = *(uint64*)(fp - 16);
    }
}
```

## Alarm

Test0 is relatively easy and you just need to follow the instructions step by step to finish it. I had stucked on test1 for some time. It's apparent that we have to save the registers during `usertrap` and restore them through `sys_sigreturn`. However, I moved the saved data directly back into the registers at first(using assembly code) while restoring, instead of moving to trapframe because I forgot `sys_sigreturn` is actually a system call, in other words, a trap too...

### My solution

Below are the main portion of my codes, excluding those in `Makefile`, `user.h`, `defs.h`, `usys.pl`, `syscall.h` and `syscall.c`.

`proc.h`

```c
struct regs {
  uint64 epc;
  uint64 ra;
  uint64 sp;
  uint64 gp;
  uint64 tp;
  uint64 a0;
  uint64 t0;
  uint64 t1;
  uint64 t2;
  uint64 s0;
  uint64 s1;
  uint64 a1;
  uint64 a2;
  uint64 a3;
  uint64 a4;
  uint64 a5;
  uint64 a6;
  uint64 a7;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
  uint64 t3;
  uint64 t4;
  uint64 t5;
  uint64 t6;
};

struct proc {
  //...ignore some unmodified codes
  int interval;                // Alarm interval
  void (*handler)();           // Handler of alarm
  int passticks;               // Number of ticks passed since the last call
  struct regs *regs;           // Save all registers before invoking alarm handler
  int alarmlock;               // Lock to avoid re-entrant calls to alarm handler
};
```

`proc.c`

```c
static struct proc*
allocproc(void)
{
  //...ignore some unmodified codes

	// Allocate a regs page.                              
	if((p->regs = (struct regs *)kalloc()) == 0){
	  release(&p->lock);
	  return 0;
	}	

	// Initialize all arguments refer to alarm to be 0
  p->interval = 0;
  p->handler = 0;
  p->passticks = 0;
  p->alarmlock = 0;

  return p;
}

static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->regs)
      kfree((void*)p->regs);
  p->regs = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}

pagetable_t
proc_pagetable(struct proc *p)
{
	  //...ignore some unmodified codes

  // map the regs just below TRAMPOLINE, for trampoline.S.
  if(mappages(pagetable, TRAPFRAME-PGSIZE, PGSIZE,
              (uint64)(p->regs), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAPFRAME, 1, 0);
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }
  return pagetable;
}

void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmunmap(pagetable, REGS, 1, 0);
  uvmfree(pagetable, sz);
}

int
fork(void)
{
  //...ignore some unmodified codes  

  *(np->regs) = *(p->regs);

  release(&np->lock);

  return pid;
}
```

`riscv.h`

```c
static inline long 
r_sscratch()
{
  long x;
  asm volatile("csrr %0, sscratch" : "=r" (x));
  return x;
}
```

`sysproc.c`

```c
// set a process's interval(the number of ticks)
// of invoking the alarm handler
int
sys_sigalarm(int interval, void (*handler)())
{
    struct proc* p = myproc();
		// the value in a0 register originally(the first argument 
    // of this system call) is now in sscratch 
    int _interval = r_sscratch(); 
    
    if(!_interval && !handler){
        p->interval = 0;
        p->handler = 0;
        p->passticks = 0;
    }
    else{
        p->interval = _interval;
        p->handler = handler;
    }
    return 0;
}

// restore the data to trapframe
int
sys_sigreturn(void)
{
    struct proc* p = myproc();
    p->trapframe->epc = p->regs->epc;
    p->trapframe->ra = p->regs->ra;
    p->trapframe->sp = p->regs->sp;
    p->trapframe->gp = p->regs->gp;
    p->trapframe->tp = p->regs->tp;
    p->trapframe->a0 = p->regs->a0;
    p->trapframe->t0 = p->regs->t0;
    p->trapframe->t1 = p->regs->t1;
    p->trapframe->t2 = p->regs->t2;
    p->trapframe->s0 = p->regs->s0;
    p->trapframe->s1 = p->regs->s1;
    p->trapframe->a0 = p->regs->a0;
    p->trapframe->a1 = p->regs->a1;
    p->trapframe->a2 = p->regs->a2;
    p->trapframe->a3 = p->regs->a3;
    p->trapframe->a4 = p->regs->a4;
    p->trapframe->a5 = p->regs->a5;
    p->trapframe->a6 = p->regs->a6;
    p->trapframe->a7 = p->regs->a7;
    p->trapframe->s2 = p->regs->s2;
    p->trapframe->s3 = p->regs->s3;
    p->trapframe->s4 = p->regs->s4;
    p->trapframe->s5 = p->regs->s5;
    p->trapframe->s6 = p->regs->s6;
    p->trapframe->s7 = p->regs->s7;
    p->trapframe->s8 = p->regs->s8;
    p->trapframe->s9 = p->regs->s9;
    p->trapframe->s10 = p->regs->s10;
    p->trapframe->s11 = p->regs->s11;
    p->trapframe->t3 = p->regs->t3;
    p->trapframe->t4 = p->regs->t4;
    p->trapframe->t5 = p->regs->t5;
    p->trapframe->t6 = p->regs->t6;

    p->alarmlock = 0;

    return 0;
}
```

`trap.c`

```c
// save most of the data from trapframe
void
saveregs(struct proc* p)
{
    p->regs->epc = p->trapframe->epc;
    p->regs->ra = p->trapframe->ra;
    p->regs->sp = p->trapframe->sp;
    p->regs->gp = p->trapframe->gp;
    p->regs->tp = p->trapframe->tp;
    p->regs->a0 = p->trapframe->a0;
    p->regs->t0 = p->trapframe->t0;
    p->regs->t1 = p->trapframe->t1;
    p->regs->t2 = p->trapframe->t2;
    p->regs->s0 = p->trapframe->s0;
    p->regs->s1 = p->trapframe->s1;
    p->regs->a1 = p->trapframe->a1;
    p->regs->a2 = p->trapframe->a2;
    p->regs->a3 = p->trapframe->a3;
    p->regs->a4 = p->trapframe->a4;
    p->regs->a5 = p->trapframe->a5;
    p->regs->a6 = p->trapframe->a6;
    p->regs->a7 = p->trapframe->a7;
    p->regs->s2 = p->trapframe->s2;
    p->regs->s3 = p->trapframe->s3;
    p->regs->s4 = p->trapframe->s4;
    p->regs->s5 = p->trapframe->s5;
    p->regs->s6 = p->trapframe->s6;
    p->regs->s7 = p->trapframe->s7;
    p->regs->s8 = p->trapframe->s8;
    p->regs->s9 = p->trapframe->s9;
    p->regs->s10 = p->trapframe->s10;
    p->regs->s11 = p->trapframe->s11;
    p->regs->t3 = p->trapframe->t3;
    p->regs->t4 = p->trapframe->t4;
    p->regs->t5 = p->trapframe->t5;
    p->regs->t6 = p->trapframe->t6;
}
void
usertrap(void)
{
  //...ignore some unmodified codes

  // give up the CPU if this is a timer interrupt
  // and set the next instruction to be alarm handler if needed.
  if(which_dev == 2){
    if(p->interval > 0){
      p->passticks++;
      if(p->passticks == p->interval && p->alarmlock == 0){
          p->passticks = 0;
          saveregs(p);
          p->trapframe->epc = (uint64)p->handler;
          p->alarmlock = 1;
      }
    }
    yield();
  }

  usertrapret();
}
```
