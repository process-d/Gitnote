**task1：Backtrace**
只要获取当前栈帧的指针，通过栈帧来遍历栈。栈帧-8是返回地址，栈帧-16是前一个栈帧地址。
```c
void backtrace(void)
{
  printf("backtrace:\n");
  uint64 cur_fp = r_fp();
  while(cur_fp != PGROUNDDOWN(cur_fp))
  {
    printf("%p\n", *(uint64 *)(cur_fp - 8));
    cur_fp = *(uint64 *)(cur_fp - 16);
  }
}
```
**task2 ：**
在本实验中栈是自顶向下增长的，return address总是在stack frame的第一位，前一个stack frame会在当前stack frame的第二位。Stack Frame中有两个重要的寄存器，第一个是SP（Stack Pointer），它指向Stack的底部并代表了当前Stack Frame的位置。第二个是FP（Frame Pointer），它指向当前Stack Frame的顶部。因为Return address和指向前一个Stack Frame的的指针都在当前Stack Frame的固定位置，所以可以通过当前的FP寄存器寻址到这两个数据。
![[Pasted image 20230904163658.png]]