GNU Debugger
------------

#### Version

* GDB 7.6
* GCC 4.8.1

### Set compile's config to debug

```
gcc -g main.c -o main
```

```
cmake_minimum_required(VERSION 2.8)
project(helloworld)

set(CMAKE_VERBOSE_MAKEFILE on)
set(CMAKE_CXX_COMPILER "g++")
SET(CMAKE_BUILD_TYPE Debug)
set(CMAKE_CXX_FLAGS "")
set(CMAKE_CXX_FLAGS_DEBUG "-g3 -Wall -DDEBUG")
set(CMAKE_CXX_FLAGS_RELEASE "-O2 -Wall")
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)

aux_source_directory(./ SRC_LIST)
aux_source_directory(./other OTHER_SRC_LIST)
list(APPEND SRC_LIST ${OTHER_SRC_LIST})

include_directories(${PROJECT_SOURCE_DIR}/include)
link_directories(${PROJECT_SOURCE_DIR}/lib)

if(${CMAKE_BUILD_TYPE} MATCHES "Debug")
    add_executable(hellod ${SRC_LIST})
    target_link_libraries(hellod Ad Bd.a Cd.so)
else()
    add_executable(hello ${SRC_LIST})
    target_link_libraries(hello A B.a C.so)
endif()
```

```
//include the following snippet into header file.
#ifdef DEBUG
#define PRINTD(format, args...) printf("[%s:%d] "format, __FILE__, __LINE__, ##args)
#else
#define PRINTD(args...)
#endif
```

```
#ifdef DEBUG
#define DEBUG_MSG(str) do { std::cout << str << std::endl; } while( false )
#else
#define DEBUG_MSG(str) do { } while ( false )
#endif
```

#### Basic Instructions

* list [line number]
	- To show the code.(It needs debug symbols)
* break [line number or function name]
	- To set breakpoint on your code.
* info break
	- To see the breakpoints that you set.
* run
	- To execute the program.
* step
	- If the program stop at breakpoint and you want to jump to the callee function, you should use `step` instead of `next`.
* next
	- If you want to execute code line by line and don't care the detail in the callee function, you should use `step`.
* print [variable]
	- To show the variable's value.
* display [variable]
	- To show the variable's value every time that you execute line by line.
* continue
	- If your program stop at a breakpoint, you should use `continue` to continue your program.
* quit
	- Type `quit` to quit this program.

#### How to set break points

* find the line you want to set
	- layout asm
	- disas main # or disas function_name or disas 0x00943032,+10
	- break *main+24
* check what breakpoints you already set
	- info break
* delete breakpoints
	- delete num

#### Show some information

* show registers
	- layout reg
	- info reg
* show the top of stack
	- x/40wx $esp
* show the register
	- p/x $eip
* show the value at this address in string
	- x/s 0x8032947
* show the value at this address in instruction format
	- x/i 0x00897654

#### Step in ,step over and reverse

* First to record the information to reverse
	- record
* step into
	- s
* step over
	- n
* step into by each instruction
	- si
* step over by each instruction
	- ni
* if you want to reverse
	- rs
	- rn
	- rsi
	- rni
