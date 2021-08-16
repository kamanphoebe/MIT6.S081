# Lab pgtbl: Page tables

This lab is really hard to debug that I spent days to finish...\
For this lab, I only modified `proc.c`, `vm.c`, `exec.c`, `proc.h` and `defs.h`.

## Print a page table

The first part is an easy one. The idea of `vmprint` is similar to `freewalk`, both use recursion. What's more, from the codes of `freewalk`, we can find that a pte which points to next level page table must be `(pte & (PTE_R|PTE_W|PTE_X)) == 0`.

### My solutions

`vm.c`

```c
// Print a pagetable, level should be 1.
void
vmprint(pagetable_t pagetable, int level)
{
    if(level == 1){
        printf("page table %p\n", pagetable);
    }
    for(int i=0; i<512; i++){
        pte_t pte = pagetable[i];
        // skip invalid pte
        if(!(pte & PTE_V))
            continue;
        // print the prefix ".."                             
        for(int j=0; j<level; j++){                  
            if(j>0)                                      
                printf(" ");                             
            printf("..");                                
        }                                                
        uint64 pa = PTE2PA(pte);                      
        printf("%d: pte %p pa %p\n", i, pte, pa);     
        // if current pte point to the next level of page table    
        if((pte & PTE_V) && ((pte & (PTE_R|PTE_W|PTE_X)) == 0)){
            vmprint((pagetable_t)pa, level+1);
        }
    }
}
```

### Questions

Explain the output of vmprint in terms of Fig 3-4 from the text. What does page 0 contain? What is in page 2? When running in user mode, could the process read/write the memory mapped by page 1?\
**Ans:**<br/>


## A kernel page table per process

The goal of this part is to let every process gets its own kernel page table which is identical with the global kernel page table. The mapping of kernel stack should be moved too. Notice that the global kernel page table still has to exist since the instruction said that "`scheduler()` should use `kernel_pagetable` when no process is running".

### My solutions

There are plenty of modifications so I list nearly all of them and respectively give a brief conclusion below, mainly introduce the differences of old functions and the usage of new functions. The complete codes(in the last version) can be found in the [files](./kernel).
Don't forget to define all your new functions in `defs.h`.

`proc.c`

`procinit()`: remove the set up of kernel stack

`allocproc()`: invoke `proc_kvminit` to create a kernel page table for the process and set up a kernel stack

`freeproc()`: free the process's kernel page table and kernel stack

`scheduler()`: load the process's kernel page table into the `satp` register before running the process and switch back to the global kernel page table when no process is running. Be careful about the timing of the later. I first wrote it under the `if(found==0)` condition and a error showed up at `kernelvec`. A whole day had gone when I finally found out what's the matter...

`proc_kvminit`: create and initialize a kernel page table which is the same as the global one

`proc_kvmmap`: a process version of `kvmmap`

`proc_freekernelpgtb`: invoke `kvmfreepgtb` to free a process's kernel page table

<br/>

`vm.c`

`kvmpa()`: replace the global kernel page table with the process's one

`kvmunmapleaf()`: unmap all leaf mappings of a page table without freeing the physical memory it refers to

`kvmfreepgtb()`: invoke `kvmunmapleaf()` to remove leaf mappings and then free the whole page table by `freewalk()`

`freekstack()`: free a kernel stack's memory


## Simplify copyin/copyinstr

The main job here is to copy the mappings from a process's user page table to its kernel page table. In order to limit the size of user space, I changed the `MAXVA` of the user page tables at first,  but it led to many difficulties in coding. Later I found that this requirment can be easily reached by checking the size every time before copying the mappings(because `sbrk` also need the copying).  

### My solutions

Same as the previous part, check the codes in [files](./kernel) if needed. Below is a brief introduction:

`proc.c`

`proc_kvminit()`: map the trapframe to kernel page table and remove the mapping of `CLINT`

`userinit()`: add a call of `copyusrmap()`

`growproc()`: add a call of `copyusrmap()`

`fork()`: add a call of `copyusrmap()`

<br/>

`vm.c`

`copyusrmap()`: copy the mappings from a process's user page table to its kernel page table

`copyin()`: replace the body with a call of `copyin_new` 

`copyinstr()`: replace the body with a call of `copyinstr_new` 

`exec.c`

`exec()`: add a call of `copyusrmap()`

### Questions

Explain why the third test srcva + len < srcva is necessary in copyin_new(): give values for srcva and len for which the first two test fail (i.e., they will not cause to return -1) but for which the third one is true (resulting in returning -1).\
**Ans:**<br/>
The third test is set to prevent overflow. Here is the example with output `0xfffffffe`:
```c
#include <stdio.h>

int main(){
    unsigned long srcva = 0x2fffffffff, len = 0xfffffffff;
    if(srcva + len < srcva)
        printf("%lx\n", srcva+len);
    return 0;
}
```
