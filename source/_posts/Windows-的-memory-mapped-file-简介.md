---
title: Windows 的 memory-mapped file 简介
date: 2017-11-13 17:59:55
tags:
  - memory-mapped file
  - memory management
  - Windows
---


## what & why
memory-mapped file (MMF): 将一个文件映射到一个进程的 virtual memory address 中,
像读写一片连续的内存空间一样读写文件.

### 优点
读写方便:
writing data to MMF:   `*p = 23`
reading date from MMF: `x = *p`

由 virtual-memory manager (VMM) 管理, 优化 read/write 性能(减少了磁盘 I/O 的次数), 
减少了碎片读, 碎片写带来的问题: 
+ Much of the file input/output (I/O) is cached, to improve general system performance.
  Several smaller reads or writes are effectively cached into one larger operation.
  使用 `FlushViewOfFile()` 可以强制执行 "写回 disk" 操作.
+ 读相当于有 lazy-load/on-demand-load 特性: 
  一次将过多内容一次性读入内存, 很多时候既慢又没有必要;
  on-demand 地按 page 大小 (一般 4k) 读, 快速而经济
+ system performs all data transfers for it in 4K pages of data

### 常见用途
+ 作为 windows 系统的 "两个或多个 process 间共享数据" 的方案.
+ 快速读写大文件: 
+ 方便地 persist 内存中复杂的数据结构


## 原理: 充分利用 virtual-memory manager (VMM)
Windows 有段页式的 virtual memory 机制.
exe 文件本身, 就是一个程序的部分 code page 和部分的 data page  
(还有其他的 code page 可能来自于 shared library, 等等) 等构成的.

windows 的 system pagefile 和应用程序的所有的 executable image 加在一起, 就可以认为是
该程序的 virtual memory space 在磁盘上的映射.

MMF 本身充分利用了操作系统的 virtual-memory manager, 
所以 :
+ "only an extension of an existing, internal memory management component",
  本身是建立在一个 OS 必备的, 而且稳定的, 被广泛使用的 "virtual memory management 系统" 之上;
  所以 MMF 也稳定可靠
+ "no overhead to manage", 本来 VMM 就一直存在, 为每个进程提供 virtual memory 服务


## Windows API
+ CreateFileMapping
+ OpenFileMapping
+ MapViewOfFile
+ MapViewOfFileEx
+ UnmapViewOfFile
+ FlushViewOfFile
+ CloseHandle


## Example code
功能: 建立 20M 大小的 `D:\mm.file` 文件, 用 memory-mapped file 的方式写入一个字符串.
程序运行&退出后, `D:\mm.file` 的头几个字符是 `"HELLO, WORLD!"` 剩余的都是 `\0`.

```cpp
#include <iostream>
#include <windows.h>
#include <stdio.h>
#include <tchar.h>

using namespace std;

void try_mmf()
{
    LPSECURITY_ATTRIBUTES p_sec_attr = NULL;  // no secure customize
    HANDLE h_template_file = NULL;  // no template file
    HANDLE file_handle = CreateFileW(
        L"D:\\mm.file"
        , GENERIC_WRITE | GENERIC_READ
        , FILE_SHARE_WRITE
        , p_sec_attr
        , CREATE_ALWAYS
        , FILE_ATTRIBUTE_NORMAL
        , h_template_file
    );

    if (file_handle == INVALID_HANDLE_VALUE) {
        fprintf(stderr, "Failed to create file");
        return;
    }

    // high-order bits of the maximum size of the file mapping object
    DWORD max_size_high_bits = 0;
    // low-order bits of the maximum size of the file mapping object
    // For x64 architecture, we need 2 32 bit integer to represent the size of memory-mapped file
    DWORD max_size_low_bits = 0x01400000; // 20 MB mapping file

    HANDLE mmf_handle = CreateFileMappingW(
        file_handle
        , p_sec_attr
        , PAGE_READWRITE
        , max_size_high_bits
        , max_size_low_bits
        , L"hello_world_mmf"
    );

    if (mmf_handle == NULL) {
        fprintf(stderr, "Failed to create file");
        return;
    }

    // The handle to the MMF object is used to map views of the file to your process's address space.
    // Views can be mapped and un-mapped at will while the MMF object exists
    //
    // When a view of the file is mapped, system resources are finally allocated.
    // A continuous range of addresses, large enough to span the size of the file view,
    // are now committed in your process's address space.
    //
    // virtual memory address 的一段地址被征用了, 但是物理内存依旧是 on demand 地被征用; 所以高效
    DWORD file_offset_high_bits = 0;
    DWORD file_offset_low_bits = 0;
    DWORD bytes_to_map = 0;    // map all
    void *map_view = MapViewOfFile(
        mmf_handle
        , FILE_MAP_READ | FILE_MAP_WRITE
        , file_offset_high_bits
        , file_offset_low_bits
        , bytes_to_map
    );

    if (map_view == NULL) {
        fprintf(stderr, "Failed to get mapped view!");
        return;
    }

    // 现在, 操作 "从 map_view 指针为起始的 20M 内存空间", 等同于在操作文件.
    wchar_t *write_ptr = (wchar_t *)map_view;
    const wchar_t *data = L"HELLO, WORLD!";
    while (*data) {
        *write_ptr++ = *data++;
    }

    SIZE_T bytes_to_flush = 0; // flush all
    FlushViewOfFile(map_view, bytes_to_flush);

    UnmapViewOfFile(map_view);
    CloseHandle(mmf_handle);
}

int main()
{
    try_mmf();
    return 0;
}
```


## Reference
+ https://msdn.microsoft.com/en-us/library/ms810613.aspx
  良心文档, with good detail

