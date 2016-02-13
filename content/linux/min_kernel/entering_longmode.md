#進入到 Long Mode

[translated from](http://os.phil-opp.com/entering-longmode.html)

在上一篇中我們建立了一個小型的 multiboot kernel。他會印出 `OK` 然後停住。在這邊我們要試著擴充他並且讓他可以呼叫 64-bit 的 Rust 程式碼，但是現在 CPU 是 protected mode，只能執行 32-bit 的指令，而且記憶體受限 4GiB，所以我們必須去設定好分頁機制 (Paging) 然後把 CPU 切換成 64-bit 的 long mode，才能夠進行下一步。

我會盡量讓說明越詳細越好，並且讓程式碼盡可能的簡潔。如果你有任何的問題、建議或是 issues，你可以留言或是在 Github 上建立 issue。這些原始碼也都公開放在 Github 上。

更正：我們不再使用 1GiB 大小的 pages，因為這會有些相容性問題。所以 identity mapping 會透過 2MiB 大小的 pages 來完成。

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

CPUID 是一個 CPU 指令，可以被用來去得到 CPU 相關資訊，但並不是每個 processor 有支援。其實要寫一個 CPUID 有無支援是有點困難的，所以我們直接從 OSDev wiki 複製下來：

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

基本上，檢查 CPUID 指令有無支援，就是去修改 FLAGS 暫存器的某個 bit (0->1, 1->0)，如果可以成功改值，就是有支援。但我們並不能直接對 flags 暫存器操作，所以我們只能先將他載入到 general purpose register 像是 eax，如何達成？唯一做法就是先用 `pushfd` 指令把 FLAGS 暫存器 push 到 stack 中，接著再 `pop` 到 `eax` 。同樣的，我們要寫回 FLAGS 暫存器則是要透過像是 push ecx 以及 popfd 指令來完成。還有如何去改特定的 bit (0->1, 1->0) 呢？在這我們使用 xor 指令去完成一個互斥或的動作 (遇到 true 改值，反之保留)。最後我們再去比較 eax 跟 ecx 的值，如果相等的話，就跳去 .no_cpuid (je - "jump if equal")。.no_cupid 會把錯誤碼 (=1) 傳到 error function 並且跳去顯示。

如果不懂不用擔心，你暫時還不需要去了解細節。

### Long Mode check

現在我們可以使用 CPUID 指令來測試是否能用 long mode。這邊程式碼是從 OSDev 複製下來： 

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

CPUID 跟一些底層的機制很像，有些奇妙設計。像是 CPUID 看起來不需要參數，但是 cpuid 指令本身其實還是會把 eax 暫存器當引數去執行。要測試 long mode 能不能用，我們只需要把 eax 設定成 0x80000001 並且呼叫 cpuid 指令， 接著這道指令會把一些資訊載入到 ecx 以及 edx 暫存器，如果可以支援 long mode，那麼 edx 的第 29 個 bit 就會被設定起來，詳細可以看 Wikipedia。

如果你有看上面的程式碼，那麼你會發現我們呼叫了 cpuid 兩次。原因是因為 CPUID 指令只能一次執行一個功能，所以不同功能要在下一次執行。又由於有些較舊的 processors 其實是不支援 0x80000001 的引數，所以在第一次呼叫中，為了要測試有無支援，會先將 eax 設定成 0x80000000 再呼叫 cpuid，他會將結果設定在 eax 中，如果結果大於等於 0x80000001，那麼我們可以用上述方法繼續測試 long mode，反之就是不支援了，這裡是使用 jb ("jump if below") 指令跳去 .no_long_mode。

### Putting it together

接下來我們把這些檢查 functions 接續 start 後面：

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

當 CPU 有功能不支援時，我們就會得到錯誤訊息。現在我們總算可以進行重要的步驟了。

### Paging

Paging 是一個記憶體管理的機制，這個機制區分了虛擬記憶體和實體記憶體，並且把 address space 切分成許多相等大小的 page，再由 page talbe 去描述這些虛擬 page 他們所相對應的實體 page。 如果你從未聽過 paging，你可能需要去看 Three Easy Pieces OS 書中的 paging 相關介紹。

在 long mode 中，x86 一個 page 大小為 4096 bytes，所用到的 page table 是四層架構，組成如下：

* Page-Map Level-4 Table (PML4)
* Page-Directory Pointer Table (PDP)
* Page-Directory Table (PD)
* Page Table (PT)

因為我不太喜歡這些名字，從現在我會用 P4、P3、P2、P1 來稱呼這些 page table。

每個 page table 都有 512 個 entries，然後每個 entry 大小都是 8 bytes，所以一個 page table 大小剛好為一個 page (512*8 = 4096)。要如何把 virtual address 轉換成 physical address？CPU[1] 會做下面步驟[2]：

[img](#)

1. 從 CR3 暫存器中得到 P4 table 的位址
2. 把 virtual address 中第 39-47 (9 bits) 個 bits 作為引索查詢 P4 (2^9 = 512 = entries 的數量)
3. 把 virtual address 中接下來 9 個 bits 作為引索查詢 P3
4. 把 virtual address 中接下來 9 個 bits 作為引索查詢 P2
5. 把 virtual address 中接下來 9 個 bits 作為引索查詢 P1
6. 最後再把剩下的 12 個 bits 當作 page offset (2^12 = 4096 = page 大小)

在 64-bit virtual address 中，你一定會發現第 48-63 bits 沒有被提到，這是因為他們不會被用到。一般常說的 "64-bit" long mode 更正確來講其實是一個 48-bit mode，對於第 48-63 bits 的值全部都跟第 47 bit 一樣，也因為如此所有合法的 virtual address 都是唯一。更多資訊請參考 Wikipedia。

在 P4、P3、P2、P1 tables 中的 entry 可切分成 52-bit 和剩下的 12 bit，而 52-bit 可能是存 frame (以 page 為單位，因為 page 大小可能改變) 的 physical address 或是存下一層 page table 的 physical address，所有的 bits 綜合來看：

Bit(s)  | Name                  | Meaning
------- | --------------------- | ----------------------------------
0       | present               | 這個 page 目前在記憶體中
1       | writable              | 容許寫入這個 page
2       | user accessible       | 如果沒被設定起來，只有 kernel mode code 可以存取這個 page
3       | write through caching | 寫入會直接寫到 cache 和記憶體中
4       | disable cache         | 這個 page 沒有使用 cache
5       | accessed              | 當 page 被使用時，CPU 會把這個 bit 設定起來
6       | dirty                 | 當發生寫入到這個 page 時，CPU 會把這個 bit 設定起來
7       | huge page/null        | P1 跟 P4 的這個 bit 一定要是 0，如果被設定起來，在 P3 中就是 1GiB page，在 P2 中就是 2MiB page
8       | global                | 在 address space 轉換下 (kernel/user)，這個 page 在 cache 中不會被清掉 (在 CR4 暫存器中 PGE bit 一定要被設定起來)
9-11    | available             | 讓 OS 自由使用
52-62   | available             | 讓 OS 自由使用
63      | no execute            | 這個 page 禁止被執行 (在 EFER 暫存器中 NXE bit 一定要先被設定起來)

### Set Up Identity Paging

當我們轉換到 long mode，paging 這個機制就會被自動啟動。所以接下來要從一個位址讀出指令來執行的這位址已經是一個 virtual address。所以我們要做 identity mapping，像是把一個 physical address 對應到 virtual address。

```
bootloader 一開始會把 kernel 載入到實體中，所有的資料都是用 physical address 來處理，當啟動 paging 時，只能對 virtual address 操作，但是在沒做任何設定之前，我們不知道 virtual address 怎麼對應到 physical address，因此我們要執行 identity mapping，把 physical address 對應到 virtual address。
```

It creates a 2MiB (when used in P2) or even a 1GiB page (when used in P3). So we could map the first gigabytes of the kernel with only one P4 and one P3 table by using 1GiB pages. Unfortunately 1GiB pages are relatively new feature, for example Intel introduced it 2010 in the Westmere architecture. Therefore we will use 2MiB pages instead to make our kernel compatible to older computers, too.
這時候 `huge page bit` 對我們來說非常好用，他可以讓 page 大小可以是 2MiB (透過 P2) 或是 1GiB (透過 P3)。當我們想要透過一個 P4 和一個 P3 使用 1GiB page 來對應到 kernel 前 1 gigabytes 時，很不幸的 1GiB pages 是相對來的新功能，他是 Intel 在 2010 發表 Westmere 架構中提到。因此我們只能使用 2MiB pages 來讓我們所寫的 kernel 相容於更老的機器。

```
我猜 kernel address space 大小剛好是 1 GiB，所以才一次 map 這麼多。
```

為了完成 identity map，我們使用了 512 個 2MiB pages 來 map kernel 前 1 gigabyte，所以我們需要一個 P4、一個 P3、一個 P2 來進行，當然我們將會之後會用更完善設計的 tables 來取代他們。但是現在，我們仍然被限制使用組合語言來完成，所以我們先要找一個簡單方法來實作。

我們可以新增這兩個 tables 在 .bss section 的一開始[3]：

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

`resb` 指令可以不用初始化方式來配置特定大小的 bytes，所以 8KiB (兩個 page table 大小) 不用一開始就儲存在 executable 中。而 align 4096 可以確保 page table 可以以 page 為單位對齊 (p4_table 位址為 4096 倍數)。

當 GRUB 把 .bss section 的記憶體建立起來的時候，他會先把記憶體初始化成 0。所以目前 `p4_table` 是可用的狀態 (他有 512 個 non-present entries)，但不是很好用。為了可以 map 2MiB pages，我們必須要把 P4 的第一個 entry 連結到 `p3_table` 的位址並且把 P3 的第一個 entry 連結到 `p2_table` 的位址。

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

如何做呢?我們一開始就把 P3 table 位址的 present 和 writable bits 設定起來 (因為 P3 table 有做 align 所以後面 12 bit 為 0，用來描述 page)，這部分透過 or 跟 0b11 來達成 (0b11 是一個二進位表示方式)，最後再放到 P4 table 的前 4 個 bytes。接下來我們就用一樣的步驟把 `p2_table` 的位址放到 P3 entry。

現在我們必須要 map P2 的第一個 entry 到一個較大的 page 而且起始位址是 0，而 P2 的第二個 entry 所對應的 page 起始位址為 2 MiB，P2 的第三個 entry 所對應的 page 起始位址為 4 MiB，以此類推。現在就來寫第一個 (也只有一個) 組合語言的 loop：

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

這邊我先解釋一下如何用組合語言實作 loop，我們使用 ecx 暫存器來當作記數的變數，就像 for loop 中 i 變數的角色。當把第 ecx 個 entry 做完對應之後，我們會把 ecx 加一，如果 ecx 小於 512 的話，那就跳到 .map_p2_table 繼續做對應。

為了找到 P2 entry 所對應的位址，所以我們要去計算所對應的 page 的起始位置，然後放在 eax 暫存器中：基本上第 ecx 個 entry 所對應到的位址為 ecx * 2MiB，我們會需要用到 mul 指令，他會將給定的暫存器乘上 eax 並且將結果存在 eax 中。接下我們把 present、writable、huge page bits 設定起來，最後再寫進 P2 entry，而 P2 的第 ecx 個 entry 的位址就是 p2_table + ecx * 8，因為每個 entry 大小是 8 bytes。

現在我們已經 identity map kernel 的第一個 gigabyte (512 * 2MiB)，而且對於存取來說 physical address 跟 virtual address 是一致的。

### Enable Paging

要開啟 paging 並且進入到 long mode，我們需要以下步驟：

1. 把 P4 table 的位址寫進 CR3 暫存器 (CPU 會去用到這個暫存器，忘記可以回頭看 paging 那一段)
2. long mode 是 Physical Address Extension (PAE) 機制的延伸，所以我們要先啟動 PAE
3. 把在 EFER 暫存器中的 long mode bit 設定起來
4. 啟動 Paging

用組合語言寫成的 function 會像這樣 (一些對於不同暫存器的 bit 處理)：

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

or eax, 1 << X 是一個常見的做法，他把暫存器的第 X bit (從 0 開始) 設定起來 (<< 是一個 left shift 的操作)。透過 rdmsr 和 wrmsr 我們可以去讀或寫 model specific registers，其中這兩道指令會使用到 ecx 指到我們要讀寫的 msr，然後使用 eax 來讀出或寫到 msr (在這邊 ecx 是指到 EFER 暫存器)。 

最後，我們在 start 呼叫剛剛所寫的 function：

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

如果要測試是否成功，就看有沒有印出綠色的 OK，如果有那麼我們就是成功的啟動 paging 了！

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
