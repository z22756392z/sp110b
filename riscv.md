[11-riscv/mini-riscv-os · master · ccc110 / Sp · GitLab](https://gitlab.com/ccc110/sp/-/tree/master/11-riscv/mini-riscv-os)

* MakeFile

```makefile
CC = riscv64-unknown-elf-gcc
CFLAGS = -nostdlib -fno-builtin -mcmodel=medany -march=rv32ima -mabi=ilp32

QEMU = qemu-system-riscv32
QFLAGS = -nographic -smp 4 -machine virt -bios none

OBJDUMP = riscv64-unknown-elf-objdump

all: os.elf

os.elf: start.s os.c
	$(CC) $(CFLAGS) -T os.ld -o os.elf $^

qemu: $(TARGET)
	@qemu-system-riscv32 -M ? | grep virt >/dev/null || exit
	@echo "Press Ctrl-A and then X to exit QEMU"
	$(QEMU) $(QFLAGS) -kernel os.elf

clean:
	rm -f *.elf

```

* os.id 連結組合語言與C語言 

```
OUTPUT_ARCH( "riscv" ) /* 處理器架構為 RISCV */

ENTRY( _start ) /* 進入點從 _start 開始執行 */

MEMORY
{
  ram   (wxa!ri) : ORIGIN = 0x80000000, LENGTH = 128M /* RAM 從 0x80000000 開始，共 128M BYTES */
}

PHDRS
{
  text PT_LOAD; /* text 段要載入*/ /* PT_LOAD (1) Indicates that this program header describes a segment to be loaded from the file. 參考 -- https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_node/ld_23.html */
  data PT_LOAD; /* data 段要載入*/
  bss PT_LOAD;  /* bss 段要載入*/
}

SECTIONS
{
  .text : {
    PROVIDE(_text_start = .); /* 設定 _text_start 為 .text 段開頭位址。 */
    *(.text.init) *(.text .text.*) /* 將所有程式碼放在這段。 */
    PROVIDE(_text_end = .); /* 設定 _text_end 為 .text 段開頭位址。 */
  } >ram AT>ram :text

  .rodata : { /* 唯讀資料段 */
    PROVIDE(_rodata_start = .);
    *(.rodata .rodata.*)
    PROVIDE(_rodata_end = .);
  } >ram AT>ram :text

  .data : { /* 資料段 */
    . = ALIGN(4096);
    PROVIDE(_data_start = .);
    *(.sdata .sdata.*) *(.data .data.*)
    PROVIDE(_data_end = .);
  } >ram AT>ram :data

  .bss :{ /* 未初始化資料段 */
    PROVIDE(_bss_start = .);
    *(.sbss .sbss.*) *(.bss .bss.*)
    PROVIDE(_bss_end = .);
  } >ram AT>ram :bss

  PROVIDE(_memory_start = ORIGIN(ram));
  PROVIDE(_memory_end = ORIGIN(ram) + LENGTH(ram));
}

```

* start.s 每個核心都會跑這個start 但只要 hart id != 0 就進去park (無窮迴圈) 代表程式中只會有一個核心再跑

```
# 來源 -- https://matrix89.github.io/writes/writes/experiments-in-riscv/
.equ STACK_SIZE, 8192

.global _start

_start:
    # setup stacks per hart // hart: hardware thread, 差不多就是 core, 設定此 core 的堆疊
    csrr t0, mhartid                # read current hart id // 讀取 core 的 id
    slli t0, t0, 10                 # shift left the hart id by 1024 // t0 = t0*1024
    la   sp, stacks + STACK_SIZE    # set the initial stack pointer // 將堆疊指標 sp 放到堆疊底端
                                    # to the end of the stack space
    add  sp, sp, t0                 # move the current hart stack pointer // 設定 hart 的堆疊位置
                                    # to its place in the stack space

    # park harts with id != 0
    csrr a0, mhartid                # read current hart id
    bnez a0, park                   # if we're not on the hart 0 // 如果不是 hart 0 ，就進入無窮迴圈。
                                    # we park the hart

    j    os_main                    # hart 0 jump to c // 堆疊設好了，讓 hart 0 進入 C 語言主程式。

park:                               # 無窮迴圈，停住!
    wfi
    j park

stacks:
    .skip STACK_SIZE * 4            # allocate space for the harts stacks // 分配堆疊空間

```



* os.c

```c
//記憶體映射 => 顯示螢幕
#include <stdint.h>

#define UART        0x10000000
#define UART_THR    (uint8_t*)(UART+0x00) // THR:transmitter holding register
#define UART_LSR    (uint8_t*)(UART+0x05) // LSR:line status register
#define UART_LSR_EMPTY_MASK 0x40          // LSR Bit 6: Transmitter empty; both the THR and LSR are empty

int lib_putc(char ch) {
	while ((*UART_LSR & UART_LSR_EMPTY_MASK) == 0);
    // 10000 & 100.....05 確認第六個bit是否為零 零代表不能寫入緩衝區 1則反之
    // UART_LSR 是外部修改的 會由另一個地方 修改這個位置的值 來告知可不可以寫入
	return *UART_THR = ch;
}

void lib_puts(char *s) {
	while (*s) lib_putc(*s++);
}

int os_main(void)
{
	lib_puts("Hello OS!\n");
	while (1) {}
	return 0;
}
```

## **ContextSwitch**

* os.id

```
OUTPUT_ARCH( "riscv" )

ENTRY( _start )

MEMORY
{
  ram   (wxa!ri) : ORIGIN = 0x80000000, LENGTH = 128M
}

PHDRS
{
  text PT_LOAD;
  data PT_LOAD;
  bss PT_LOAD;
}

SECTIONS
{
  .text : {
    PROVIDE(_text_start = .);
    *(.text.init) *(.text .text.*)
    PROVIDE(_text_end = .);
  } >ram AT>ram :text

  .rodata : {
    PROVIDE(_rodata_start = .);
    *(.rodata .rodata.*)
    PROVIDE(_rodata_end = .);
  } >ram AT>ram :text

  .data : {
    . = ALIGN(4096);
    PROVIDE(_data_start = .);
    *(.sdata .sdata.*) *(.data .data.*)
    PROVIDE(_data_end = .);
  } >ram AT>ram :data

  .bss :{
    PROVIDE(_bss_start = .);
    *(.sbss .sbss.*) *(.bss .bss.*)
    PROVIDE(_bss_end = .);
  } >ram AT>ram :bss

  PROVIDE(_memory_start = ORIGIN(ram));
  PROVIDE(_memory_end = ORIGIN(ram) + LENGTH(ram));
}

```

* start.s

```
.equ STACK_SIZE, 8192

.global _start

_start:
    # setup stacks per hart
    csrr t0, mhartid                # read current hart id
    slli t0, t0, 10                 # shift left the hart id by 1024
    la   sp, stacks + STACK_SIZE    # set the initial stack pointer 
                                    # to the end of the stack space
    add  sp, sp, t0                 # move the current hart stack pointer
                                    # to its place in the stack space

    # park harts with id != 0
    csrr a0, mhartid                # read current hart id
    bnez a0, park                   # if we're not on the hart 0
                                    # we park the hart

    j    os_main                    # hart 0 jump to c

park:
    wfi
    j park

stacks:
    .skip STACK_SIZE * 4            # allocate space for the harts stacks

```



```c
#include "os.h"

#define STACK_SIZE 1024
uint8_t task0_stack[STACK_SIZE];
struct context ctx_os;
struct context ctx_task;

extern void sys_switch();

void user_task0(void)
{
	lib_puts("Task0: Context Switch Success !\n");
	while (1) {} // stop here.
}

int os_main(void)
{
	lib_puts("OS start\n");
	ctx_task.ra = (reg_t) user_task0; //setup return address
	ctx_task.sp = (reg_t) &task0_stack[STACK_SIZE-1];//stack pointer
	sys_switch(&ctx_os, &ctx_task);//define in sys.s
	return 0;
}
```



* sys_swtich 在 sys.s 組合語言完成

![image-20220525175842894](C:\Users\z22756392z\AppData\Roaming\Typora\typora-user-images\image-20220525175842894.png)

```s
# This Code derived from xv6-riscv (64bit)
# -- https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/swtch.S

# ============ MACRO ==================
.macro ctx_save base	# save current status
        sw ra, 0(\base) 	#ra == return address 
        sw sp, 4(\base)
        sw s0, 8(\base)
        sw s1, 12(\base)
        sw s2, 16(\base)
        sw s3, 20(\base)
        sw s4, 24(\base)
        sw s5, 28(\base)
        sw s6, 32(\base)
        sw s7, 36(\base)
        sw s8, 40(\base)
        sw s9, 44(\base)
        sw s10, 48(\base)
        sw s11, 52(\base)
.endm

.macro ctx_load base
        lw ra, 0(\base)
        lw sp, 4(\base)
        lw s0, 8(\base)
        lw s1, 12(\base)
        lw s2, 16(\base)
        lw s3, 20(\base)
        lw s4, 24(\base)
        lw s5, 28(\base)
        lw s6, 32(\base)
        lw s7, 36(\base)
        lw s8, 40(\base)
        lw s9, 44(\base)
        lw s10, 48(\base)
        lw s11, 52(\base)
.endm

# ============ Macro END   ==================
 
# Context switch
#
#   void sys_switch(struct context *old, struct context *new);
# 
# Save current registers in old. Load from new.

.globl sys_switch
.align 4
sys_switch: # 對應到 C 為 sys_switch(struct context *old, struct context *new); 其中 old=>a0, new=>a1 
        ctx_save a0  # a0 => struct context *old 
        ctx_load a1  # a1 => struct context *new
        ret          # pc=ra; switch to new task (new->ra)

```



## **MultiTasking**

* os.ld

```
OUTPUT_ARCH( "riscv" )

ENTRY( _start )

MEMORY
{
  ram   (wxa!ri) : ORIGIN = 0x80000000, LENGTH = 128M
}

PHDRS
{
  text PT_LOAD;
  data PT_LOAD;
  bss PT_LOAD;
}

SECTIONS
{
  .text : {
    PROVIDE(_text_start = .);
    *(.text.init) *(.text .text.*)
    PROVIDE(_text_end = .);
  } >ram AT>ram :text

  .rodata : {
    PROVIDE(_rodata_start = .);
    *(.rodata .rodata.*)
    PROVIDE(_rodata_end = .);
  } >ram AT>ram :text

  .data : {
    . = ALIGN(4096);
    PROVIDE(_data_start = .);
    *(.sdata .sdata.*) *(.data .data.*)
    PROVIDE(_data_end = .);
  } >ram AT>ram :data

  .bss :{
    PROVIDE(_bss_start = .);
    *(.sbss .sbss.*) *(.bss .bss.*)
    PROVIDE(_bss_end = .);
  } >ram AT>ram :bss

  PROVIDE(_memory_start = ORIGIN(ram));
  PROVIDE(_memory_end = ORIGIN(ram) + LENGTH(ram));
}

```

* start.s

```s
.equ STACK_SIZE, 8192

.global _start

_start:
    # setup stacks per hart
    csrr t0, mhartid                # read current hart id
    slli t0, t0, 10                 # shift left the hart id by 1024
    la   sp, stacks + STACK_SIZE    # set the initial stack pointer 
                                    # to the end of the stack space
    add  sp, sp, t0                 # move the current hart stack pointer
                                    # to its place in the stack space

    # park harts with id != 0
    csrr a0, mhartid                # read current hart id
    bnez a0, park                   # if we're not on the hart 0
                                    # we park the hart

    j    os_main                    # hart 0 jump to c

park:
    wfi
    j park

stacks:
    .skip STACK_SIZE * 4            # allocate space for the harts stacks

```

* os.c

![multiTask](./image/multiTask.png)

```c
#include "task.h"
#include "lib.h"

uint8_t task_stack[MAX_TASK][STACK_SIZE]; //  task堆疊
struct context ctx_os; // 作業系統的 context
struct context ctx_tasks[MAX_TASK]; // 各個 task 的 context
struct context *ctx_now; // 指向目前 task
int taskTop=0;  // 目前 task 數量

// create a new task // 創建新 task
int task_create(void (*task)(void))
{
	int i=taskTop++;
	ctx_tasks[i].ra = (reg_t) task;
	ctx_tasks[i].sp = (reg_t) &task_stack[i][STACK_SIZE-1];
	return i;
}

// switch to task[i]  // 切換到 task[i]
void task_go(int i) {
	ctx_now = &ctx_tasks[i];
	sys_switch(&ctx_os, &ctx_tasks[i]);
}

// switch back to os // 切換回 os
void task_os() {
	struct context *ctx = ctx_now;
	ctx_now = &ctx_os; //address
	sys_switch(ctx, &ctx_os);
}

```

```c
#include "os.h"

void user_task0(void) // task0
{
	lib_puts("Task0: Created!\n");
	lib_puts("Task0: Now, return to kernel mode\n");
	os_kernel();
	while (1) {
		lib_puts("Task0: Running...\n");
		lib_delay(1000); // 延遲一秒鐘
		os_kernel(); // 呼叫 os_kernel 主動將控制權交回給 OS
	}
}

void user_task1(void) // task1
{
	lib_puts("Task1: Created!\n");
	lib_puts("Task1: Now, return to kernel mode\n");
	os_kernel();
	while (1) {
		lib_puts("Task1: Running...\n");
		lib_delay(1000);
		os_kernel();
	}
}

void user_init() {
	task_create(&user_task0);
	task_create(&user_task1);
}

```

```c
#include "os.h"

void os_kernel() {
	task_os();
}

void os_start() {
	lib_puts("OS start\n");
	user_init();
}

int os_main(void)
{
	os_start();
	
	int current_task = 0;
	while (1) {
		lib_puts("OS: Activate next task\n");
		task_go(current_task);
		lib_puts("OS: Back to OS\n");
		current_task = (current_task + 1) % taskTop; // Round Robin Scheduling
		lib_puts("\n");
	}
	return 0;
}
```

