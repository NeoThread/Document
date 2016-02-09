#進入到 Long Mode

[translated from](http://os.phil-opp.com/entering-longmode.html)

在上一篇中我們建立了一個小型的 multiboot kernel。他會印出 `OK` 然後停住。在這邊我們要試著擴充他並且讓他可以呼叫 64-bit 的 Rust 程式碼，但是現在 CPU 是 protected mode，只能執行 32-bit 的指令，而且記憶體受限 4GiB，所以我們必須去設定好分頁機制 (Paging) 然後把 CPU 切換成 64-bit 的 long mode，才能夠進行下一步。

我會盡量讓說明越詳細越好，並且讓程式碼盡可能的簡潔。如果你有任何的問題、建議或是 issues，你可以留言或是在 Github 上建立 issue。這些原始碼也都公開放在 Github 上。

更正：我們不再使用 1GiB 的 pages，因為這會有些相容性問題。而 identity mapping 會透過 2MiB pages 來完成。

(identity mapping：linear address 直接對應到 physical address)

###Some Tests

為了要避免一些 bugs 跟奇怪錯誤在舊規格的 CPU 上，我們應該去確認 processor (CPU) 是不是都有支援這個功能，如果沒有，kernel 就應該要終止並且顯示錯誤訊息。去處理這些錯誤資訊滿簡單的，我們在 `boot.asm` 中建立處理錯誤的 procedure。他會印出基本的訊息 `ERR: X` 然後停止執行，訊息中 X 是錯誤代碼，存在 al (eax) 中。

```
; Prints `ERR: ` and the given error code to screen and hangs.
; parameter: error code (in ascii) in al
error:
    mov dword [0xb8000], 0x4f524f45
    mov dword [0xb8004], 0x4f3a4f52
    mov dword [0xb8008], 0x4f204f20
    mov byte  [0xb800a], al
    hlt
```

VGA 文字 buffer 位在 0xb8000 的記憶體上，他是一個 array 存放著顯示字元，最後再透過顯示卡顯示在螢幕上。未來的教學文會涵蓋到 VGA buffer 細節，並且會說明對此如何建立 Rust 的介面來控制。不過對於現在要做的，一個 bit 一個 bit 設定是比較好的選擇。

一個顯示字元包含著 8 bit 的色碼以及 8 bit ASCII 字元。我們對所有的字元都採用 4f 的色碼，用紅底白字表示。而在 ASCII 字元中 `0x52` 是 `R`、`0x45` 是 `E`、`0x3a` 是 `:`，而 `0x20` 是空白字元。至於第二個空白字元會被給定的 ASCII 字元改寫掉。最後 CPU 會被 hlt 指令停止執行。

現在我們可以加一些檢查的 functions。 fuction 其實就只是一個普通的 label，最後面再加上一個 ret (return) 指令做結尾，然後可以透過 call 指令來呼叫他。不像 jmp 指令只是跳到記憶體位置，call 指令會先把要返回的位址 (return address) push 到 stack (然後到了 ret 指令會跳到這個位址去)。不過我們目前還沒有 stack 可以來完成這個動作，因此我們要用 stack pointer 來建立 stack， 一個存放在 esp 暫存器中的 pointer，但這個暫存器有可能會指到正確記憶體位址也有可能會指到不合法的記憶體位址。所以我們必須要去更新他，讓他可以指到合法的 stack 記憶體位址。

### Creating a Stack

為了建立 stack 記憶體空間，我們在 boot.asm 後面配置幾個 bytes 的記憶體大小：

```
...
section .bss
stack_bottom:
    resb 64
stack_top:
```

Stack 不需要被初始化，這是因為當我們使用 pop (從記憶體取出) 之前一定要使用 push (放到記憶體) 才行，所以也不用配置太大給他。如何設定呢？配置 stack 記憶體是宣告在 executable 檔案中。透過使用 .bss section 以及 resb (reserve byte) 指令，我們可以宣告 64 bytes 的未初始化記憶體空間給 stack。當在 GRUB 載入 executable 時，他會去配置 section 中所宣告的記憶體大小。

怎麼使用 stack？我們必須在程式進入點一開始 (start) 更新 esp 暫存器 (stack pointer)。

```
global start

section .text
bits 32
start:
    mov esp, stack_top

    ; print `OK` to screen
    ...
```

我們使用 stack_top (高位) 來更新 esp，因為 stack 是往低位長的：舉例來說當我 `push eax` 時，會進行兩個動作，首先將 `esp` 減 4 ，再來使用 mov [esp], eax，把 eax 的值放到目前 esp 所指的記憶體空間 (eax 是一個 general purpose register).

現在我們有可用的 stack 可以用來呼叫 function。下面有列一些檢查用的 function ，在這裡放上來只是為了完整性，我不會解釋太詳細。基本上他們作法差不多：就是去檢查功能有沒有支援，沒有的話就會跳去顯示錯誤碼。

### Multiboot check

在之後的文章中，我們會依賴 Multiboot 的一些功能，所以一開始要先確定 kernel 是否真的有被 bootloader 載入，這時候我們可以透過 eax 暫存器來確定。根據 Multiboot 規格書 (PDF) ，在 bootloader 載入 kernel 之前會先把 magic value `0x36d76289` 寫到 eax 暫存器中。所以要驗證有無載入，我們可以新增一個簡單 function：

```
check_multiboot:
    cmp eax, 0x36d76289
    jne .no_multiboot
    ret
.no_multiboot:
    mov al, "0"
    jmp error
```

我們使用了 `cmp` 指令來比較 eax 是不是等於 magic value。如果值相等，`cmp` 指令會去把在 FLAGS 暫存器的 zero flag 設定起來。再由 jne ("jump if not equal") 指令去讀 zero flag，這樣一來，如果他沒被設定起來，那麼就會跳到給定的位址。 因此當 eax 值不是 magic value 時，我們就會跳去 .no_multiboot label 執行。

在 no_multiboot 中，我們使用 jmp ("jump") 指令來跳到要顯示錯誤碼的 function。我們其實也可以用 call 指令，但是這道指令會去 push 返回的位址，這點我們是不需要的，因為錯誤發生的時候是不用返回的。然而還要把錯誤碼 (=0) 傳給 error function，我們必須要在跳到 error function 之前，將錯誤碼傳進 al 中 (error function 會從 al 讀出錯誤碼)。

### CPUID check

CPUID is a CPU instruction that can be used to get various information about the CPU. But not every processor supports it. CPUID detection is quite laborious, so we just copy a detection function from the OSDev wiki:

```
check_cpuid:
    ; Check if CPUID is supported by attempting to flip the ID bit (bit 21) in
    ; the FLAGS register. If we can flip it, CPUID is available.

    ; Copy FLAGS in to EAX via stack
    pushfd
    pop eax

    ; Copy to ECX as well for comparing later on
    mov ecx, eax

    ; Flip the ID bit
    xor eax, 1 << 21

    ; Copy EAX to FLAGS via the stack
    push eax
    popfd

    ; Copy FLAGS back to EAX (with the flipped bit if CPUID is supported)
    pushfd
    pop eax

    ; Restore FLAGS from the old version stored in ECX (i.e. flipping the ID bit
    ; back if it was ever flipped).
    push ecx
    popfd

    ; Compare EAX and ECX. If they are equal then that means the bit wasn't
    ; flipped, and CPUID isn't supported.
    cmp eax, ecx
    je .no_cpuid
    ret
.no_cpuid:
    mov al, "1"
    jmp error
```

Basically, the CPUID instruction is supported if we can flip some bit in the FLAGS register. We can't operate on the flags register directly, so we need to load it into some general purpose register such as eax first. The only way to do this is to push the FLAGS register on the stack through the pushfd instruction and then pop it into eax. Equally, we write it back through push ecx and popfd. To flip the bit we use the xor instruction to perform an exclusive OR. Finally we compare the two values and jump to .no_cpuid if both are equal (je – “jump if equal”). The .no_cpuid code just jumps to the error function with error code 1.

Don't worry, you don't need to understand the details.

### Long Mode check

Now we can use CPUID to detect whether long mode can be used. I use code from OSDev again:

```
check_long_mode:
    ; test if extended processor info in available
    mov eax, 0x80000000    ; implicit argument for cpuid
    cpuid                  ; get highest supported argument
    cmp eax, 0x80000001    ; it needs to be at least 0x80000001
    jb .no_long_mode       ; if it's less, the CPU is too old for long mode

    ; use extended info to test if long mode is available
    mov eax, 0x80000001    ; argument for extended processor info
    cpuid                  ; returns various feature bits in ecx and edx
    test edx, 1 << 29      ; test if the LM-bit is set in the D-register
    jz .no_long_mode       ; If it's not set, there is no long mode
    ret
.no_long_mode:
    mov al, "2"
    jmp error
```

Like many low-level things, CPUID is a bit strange. Instead of taking a parameter, the cpuid instruction implicitely uses the eax register as argument. To test if long mode is available, we need to call cpuid with 0x80000001 in eax. This loads some information to the ecx and edx registers. Long mode is supported if the 29th bit in edx is set. Wikipedia has detailed information.

If you look at the assembly above, you'll probably notice that we call cpuid twice. The reason is that the CPUID command started with only a few functions and was extended over time. So old processors may not know the 0x80000001 argument at all. To test if they do, we need to invoke cpuid with 0x80000000 in eax first. It returns the highest supported parameter value in eax. If it's at least 0x80000001, we can test for long mode as described above. Else the CPU is old and doesn't know what long mode is either. In that case, we directly jump to .no_long_mode through the jb instruction (“jump if below”).

### Putting it together

We just call these check functions right after start:

```
global _start

section .text
bits 32
_start:
    mov esp, stack_top

    call check_multiboot
    call check_cpuid
    call check_long_mode

    ; print `OK` to screen
    ...
```

When the CPU doesn't support a needed feature, we get an error message with an unique error code. Now we can start the real work.

### Paging

Paging is a memory management scheme that separates virtual and physical memory. The address space is split into equal sized pages and a page table specifies which virtual page points to which physical page. If you never heard of paging, you might want to look at the paging introduction (PDF) of the Three Easy Pieces OS book.

In long mode, x86 uses a page size of 4096 bytes and a 4 level page table that consists of:

* the Page-Map Level-4 Table (PML4),
* the Page-Directory Pointer Table (PDP),
* the Page-Directory Table (PD),
* and the Page Table (PT).

As I don't like these names, I will call them P4, P3, P2, and P1 from now on.

Each page table contains 512 entries and one entry is 8 bytes, so they fit exactly in one page (512*8 = 4096). To translate a virtual address to a physical address the CPU1 will do the following2:

[img](#)

1. Get the address of the P4 table from the CR3 register
2. Use bits 39-47 (9 bits) as an index into P4 (2^9 = 512 = number of entries)
3. Use the following 9 bits as an index into P3
4. Use the following 9 bits as an index into P2
5. Use the following 9 bits as an index into P1
6. Use the last 12 bits as page offset (2^12 = 4096 = page size)

But what happens to bits 48-63 of the 64-bit virtual address? Well, they can't be used. The “64-bit” long mode is in fact just a 48-bit mode. The bits 48-63 must be copies of bit 47, so each valid virtual address is still unique. For more information see Wikipedia.

An entry in the P4, P3, P2, and P1 tables consists of the page aligned 52-bit physical address of the frame or the next page table and the following bits that can be OR-ed in:

[table]

### Set Up Identity Paging

When we switch to long mode, paging will be activated automatically. The CPU will then try to read the instruction at the following address, but this address is now a virtual address. So we need to do identity mapping, i.e. map a physical address to the same virtual address.

The huge page bit is now very useful to us. It creates a 2MiB (when used in P2) or even a 1GiB page (when used in P3). So we could map the first gigabytes of the kernel with only one P4 and one P3 table by using 1GiB pages. Unfortunately 1GiB pages are relatively new feature, for example Intel introduced it 2010 in the Westmere architecture. Therefore we will use 2MiB pages instead to make our kernel compatible to older computers, too.

To identity map the first gigabyte of our kernel with 512 2MiB pages, we need one P4, one P3, and one P2 table. Of course we will replace them with finer-grained tables later. But now that we're stuck with assembly, we choose the easiest way.

We can add these two tables at the beginning3 of the .bss section:

```
...

section .bss
align 4096
p4_table:
    resb 4096
p3_table:
    resb 4096
p2_table:
    resb 4096
stack_bottom:
    resb 64
stack_top:
```

The resb command reserves the specified amount of bytes without initializing them, so the 8KiB don't need to be saved in the executable. The align 4096 ensures that the page tables are page aligned.

When GRUB creates the .bss section in memory, it will initialize it to 0. So the p4_table is already valid (it contains 512 non-present entries) but not very useful. To be able to map 2MiB pages, we need to link P4's first entry to the p3_table and P3's first entry to the the p2_table:

```
set_up_page_tables:
    ; map first P4 entry to P3 table
    mov eax, p3_table
    or eax, 0b11 ; present + writable
    mov [p4_table], eax

    ; map first P3 entry to P2 table
    mov eax, p2_table
    or eax, 0b11 ; present + writable
    mov [p3_table], eax

    ; TODO map each P2 entry to a huge 2MiB page
    ret
```

We just set the present and writable bits (0b11 is a binary number) in the aligned P3 table address and move it to the first 4 bytes of the P4 table. Then we do the same to link the first P3 entry to the p2_table.

Now we need to map P2's first entry to a huge page starting at 0, P2's second entry to a huge page starting at 2MiB, P2's third entry to a huge page starting at 4MiB, and so on. It's time for our first (and only) assembly loop:

```
set_up_page_tables:
    ...
    ; map each P2 entry to a huge 2MiB page
    mov ecx, 0         ; counter variable

.map_p2_table:
    ; map ecx-th P2 entry to a huge page that starts at address 2MiB*ecx
    mov eax, 0x200000  ; 2MiB
    mul ecx            ; start address of ecx-th page
    or eax, 0b10000011 ; present + writable + huge
    mov [p2_table + ecx * 8], eax ; map ecx-th entry

    inc ecx            ; increase counter
    cmp ecx, 512       ; if counter == 512, the whole P2 table is mapped
    jne .map_p2_table  ; else map the next entry

    ret
```

Maybe I first explain how an assembly loop works. We use the ecx register as a counter variable, just like i in a for loop. After mapping the ecx-th entry, we increase ecx by one and jump to .map_p2_table again if it's still smaller 512.

To map a P2 entry we first calculate the start address of its page in eax: The ecx-th entry needs to be mapped to ecx * 2MiB. We use the mul operation for that, which multiplies eax with the given register and stores the result in eax. Then we set the present, writable, and huge page bits and write it to the P2 entry. The address of the ecx-th entry in P2 is p2_table + ecx * 8, because each entry is 8 bytes large.

Now the first gigabyte (512 * 2MiB) of our kernel is identity mapped and thus accessible through the same physical and virtual addresses.

### Enable Paging

To enable paging and enter long mode, we need to do the following:

1. write the address of the P4 table to the CR3 register (the CPU will look there, see the paging section)
2. long mode is an extension of Physical Address Extension (PAE), so we need to enable PAE first
3. Set the long mode bit in the EFER register
4. Enable Paging

The assembly function looks like this (some boring bit-moving to various registers):

```
enable_paging:
    ; load P4 to cr3 register (cpu uses this to access the P4 table)
    mov eax, p4_table
    mov cr3, eax

    ; enable PAE-flag in cr4 (Physical Address Extension)
    mov eax, cr4
    or eax, 1 << 5
    mov cr4, eax

    ; set the long mode bit in the EFER MSR (model specific register)
    mov ecx, 0xC0000080
    rdmsr
    or eax, 1 << 8
    wrmsr

    ; enable paging in the cr0 register
    mov eax, cr0
    or eax, 1 << 31
    mov cr0, eax

    ret
```

The or eax, 1 << X is a common pattern. It sets the bit X in the eax register (<< is a left shift). Through rdmsr and wrmsr it's possible to read/write to the so-called model specific registers at address ecx (in this case ecx points to the EFER register).

Finally we need to call our new functions in start:

```
...
start:
    mov esp, stack_top

    call check_multiboot
    call check_cpuid
    call check_long_mode

    call set_up_page_tables ; new
    call enable_paging     ; new

    ; print `OK` to screen
    mov dword [0xb8000], 0x2f4b2f4f
    hlt
...
```

To test it we execute make run. If the green OK is still printed, we have successfully enabled paging!

### The Global Descriptor Table

After enabling Paging, the processor is in long mode. So we can use 64-bit instructions now, right? Wrong. The processor is still in some 32-bit compatibility submode. To actually execute 64-bit code, we need to set up a new Global Descriptor Table. The Global Descriptor Table (GDT) was used for Segmentation in old operating systems. I won't explain Segmentation but the Three Easy Pieces OS book has good introduction (PDF) again.

Today almost everyone uses Paging instead of Segmentation (and so do we). But on x86, a GDT is always required, even when you're not using Segmentation. GRUB has set up a valid 32-bit GDT for us but now we need to switch to a long mode GDT.

A GDT always starts with a 0-entry and contains an arbitrary number of segment entries afterwards. An entry has the following format:

[table]

We need one code and one data segment. They have the following bits set: descriptor type, present, and read/write. The code segment has additionally the executable and the 64-bit flag. In Long mode, it's not possible to actually use the GDT entries for Segmentation and thus the base and limit fields must be 0. Translated to assembly the long mode GDT looks like this:

```
section .rodata
gdt64:
    dq 0 ; zero entry
    dq (1<<44) | (1<<47) | (1<<41) | (1<<43) | (1<<53) ; code segment
    dq (1<<44) | (1<<47) | (1<<41) ; data segment
```

We chose the .rodata section here because it's initialized read-only data. The dq command stands for define quad and outputs a 64-bit constant (similar to dw and dd). And the (1<<44) is a bit shift that sets bit 44.

### Loading the GDT

To load our new 64-bit GDT, we have to tell the CPU its address and length. We do this by passing the memory location of a special pointer structure to the lgdt (load GDT) instruction. The pointer structure looks like this:

```
gdt64:
    ...
    dq (1<<44) | (1<<47) | (1<<41) ; data segment
.pointer:
    dw $ - gdt64 - 1
    dq gdt64
```

The first 2 bytes specify the (GDT length - 1). The $ is a special symbol that is replaced with the current address (it's equal to .pointer in our case). The following 8 bytes specify the GDT address. Labels that start with a point (such as .pointer) are sub-labels of the last label without point. To access them, they must be prefixed with the parent label (e.g., gdt64.pointer).

Now we can load the GDT in start:

```
start:
    ...
    call enable_paging

    ; load the 64-bit GDT
    lgdt [gdt64.pointer]

    ; print `OK` to screen
    ...
```

When you still see the green OK, everything went fine and the new GDT is loaded. But we still can't execute 64-bit code: The selector registers such as the code selector cs and the data selector ds still have the values from the old GDT. To update them, we need to load them with the GDT offset (in bytes) of the desired segment. In our case the code segment starts at byte 8 of the GDT and the data segment at byte 16. Let's try it:

```
    ...
    lgdt [gdt64.pointer]

    ; update selectors
    mov ax, 16
    mov ss, ax  ; stack selector
    mov ds, ax  ; data selector
    mov es, ax  ; extra selector

    ; print `OK` to screen
    ...
```

It should still work. The segment selectors are only 16-bits large, so we use the 16-bit ax subregister. Notice that we didn't update the code selector cs. We will do that later. First we should replace this hardcoded 16 by adding some labels to our GDT:

```
section .rodata
gdt64:
    dq 0 ; zero entry
.code: equ $ - gdt64 ; new
    dq (1<<44) | (1<<47) | (1<<41) | (1<<43) | (1<<53) ; code segment
.data: equ $ - gdt64 ; new
    dq (1<<44) | (1<<47) | (1<<41) ; data segment
.pointer:
    ...
```

We can't just use normal labels here, as we need the table offset. We calculate this offset using the current address $ and set the labels to this value using equ. Now we can use gdt64.data instead of 16 and gdt64.code instead of 8 and these labels will still work if we modify the GDT.

Now there is just one last step left to enter the true 64-bit mode: We need to load cs with gdt64.code. But we can't do it through mov. The only way to reload the code selector is a far jump or a far return. These instructions work like a normal jump/return but change the code selector. We use a far jump to a long mode label:

```
global start
extern long_mode_start
...
start:
    ...
    lgdt [gdt64.pointer]

    ; update selectors
    mov ax, gdt64.data
    mov ss, ax
    mov ds, ax
    mov es, ax

    jmp gdt64.code:long_mode_start
...
```

The actual long_mode_start label is defined as extern, so it's part of another file. The jmp gdt64.code:long_mode_start is the mentioned far jump.

I put the 64-bit code into a new file to separate it from the 32-bit code, thereby we can't call the (now invalid) 32-bit code accidentally. The new file (I named it long_mode_init.asm) looks like this:

```
global long_mode_start

section .text
bits 64
long_mode_start:
    ; print `OKAY` to screen
    mov rax, 0x2f592f412f4b2f4f
    mov qword [0xb8000], rax
    hlt
```

You should see a green OKAY on the screen. Some notes on this last step:

* As the CPU expects 64-bit instructions now, we use bits 64
* We can now use the extended registers. Instead of the 32-bit eax, ebx, etc. we now have the 64-bit rax, rbx, …
* and we can write these 64-bit registers directly to memory using mov qword (quad word)

Congratulations! You have successfully wrestled through this CPU configuration and compatibility mode mess :).

### What's next?

It's time to finally leave assembly behind4 and switch to some higher level language. We won't use C or C++ (not even a single line). Instead we will use the relatively new Rust language. It's a systems language without garbage collections but with guaranteed memory safety. Through a real type system and many abstractions it feels like a high-level language but can still be low-level enough for OS development. The next post describes the Rust setup.

---

1. In the x86 architecture, the page tables are hardware walked, so the CPU will look at the table on its own when it needs a translation. Other architectures, for example MIPS, just throw an exception and let the OS translate the virtual address.
 
2. Image source: Wikipedia, with modified font size, page table naming, and removed sign extended bits. The modified file is licensed under the Creative Commons Attribution-Share Alike 3.0 Unported license.
 
3. Page tables need to be page-aligned as the bits 0-11 are used for flags. By putting these tables at the beginning of .bss, the linker can just page align the whole section and we don't have unused padding bytes in between.
