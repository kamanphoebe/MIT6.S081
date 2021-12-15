# Eliminate allocation from sbrk()

Easy ;)

### My solution

The codes below is the complete version for the whole lab.
The codes for `n < 0` condition can be copied from `uvmdealloc()`.

`sysproc.c`

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;
  struct proc *p = myproc();

  if(argint(0, &n) < 0)
    return -1;
  addr = p->sz;
  if(addr + n < 0){
      // overflow
      myproc()->killed = 1;
      exit(-1);
  }
  p->sz = (uint64)(addr + n);
  if(n < 0){
      int npages = (PGROUNDUP(addr) - PGROUNDUP(p->sz)) / PGSIZE;
      uvmunmap(myproc()->pagetable, PGROUNDUP(p->sz), npages, 1);
  }
  return addr;
}
```

# Lazy allocation

An easy one too! 
Follow the instructions and copy necessary codes from `uvmalloc()` to construct a initial page-fault handler. Then modify `uvmunmap()` to not panic if some pages aren't mapped by easily skipping them.

### My solution

The codes below is the complete version for the whole lab.

`trap.c`

```c
void
usertrap(void)
{
  //...ignore some unmodified codes
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if(r_scause() == 13 || r_scause() == 15){
    // page fault
    // handle by lazy allocation
    uint64 va = r_stval();
    if(va >= p->sz || va < p->trapframe->sp){
			// kill a process if it page-faults on
      // a virtual memory address higher than any allocated with sbrk()
      // or
      // the invalid page below the user stack
      p->killed = 1;
    }
    else{
      char *mem = kalloc();
      if(!mem){
				// if kalloc() fails, kill the current process
        p->killed = 1;
      }
      else{
        memset(mem, 0, PGSIZE);
        if(mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, (uint64)mem,
                    PTE_W|PTE_X|PTE_R|PTE_U) != 0)
            kfree(mem);
      }
    }
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }
  //...ignore some unmodified codes
}
```

`vm.c`

```c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      continue;
    if((*pte & PTE_V) == 0)
      continue;
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free && (*pte & PTE_V)){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```


# Lazytests and Usertests

The fourth requirement is somewhat tricky for me :(

- **Handle negative `sbrk()` arguments:**\
Modify `sbrk()` to unmap pages whose addresses are higher than the new size.
- **Kill a process if it page-faults on a virtual memory address higher than any allocated with `sbrk()`:**\
Add an `if` condition in `usertrap()` to check.
- **Handle the parent-to-child memory copy in `fork()` correctly:**\
Modify `uvmcopy()` using the same way for `uvmunmap()`.
- **Handle the case in which a process passes a valid address from `sbrk()` to a system call such as read or write, but the memory for that address has not yet been allocated:**\
Since system calls are running in kernel, so page-faults on user virtual address during system calls will not lead to  `usertrap()`. In order to work, kernel looks up physical addresses of user virtual addresses through `walkaddr()`. Make use of this, we can place an identical handler in `walkaddr()`.
- **Handle out-of-memory correctly: if `kalloc()` fails in the page fault handler, kill the current process:**\
Add an `if` condition in `usertrap()` to check.
- **Handle faults on the invalid page below the user stack:**\
Add an `if` condition in `usertrap()` to check and kill the process if so.

### My solution

`vm.c`

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      continue;
    if((*pte & PTE_V) == 0)
      continue;
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}

uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  struct proc* p = myproc();
  if(!pte || (*pte & PTE_V) == 0){
    if(va >= p->sz || va < p->trapframe->sp){
      return 0;
    }
    else{
      char *mem = kalloc();
      if(!mem){
        return 0;
      }
      else{
        memset(mem, 0, PGSIZE);
        if(mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, (uint64)mem,
                    PTE_W|PTE_X|PTE_R|PTE_U) != 0
          kfree(mem);
      }
    }
  }

  if((*pte & PTE_U) == 0)
    return 0;
  pa = PTE2PA(*pte);
  return pa;
}
```
