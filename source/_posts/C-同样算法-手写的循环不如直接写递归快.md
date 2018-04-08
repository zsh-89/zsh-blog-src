---
title: 'C++, 同样算法, 手写的循环不如直接写递归快?'
date: 2018-04-08 16:50:29
tags:
 - clang
 - gcc
 - clang++
 - g++
 - compiler optimization
 - recursion flattening
---

## 背景
起因是 leetcode 上的一道题 https://leetcode.com/problems/target-sum/description/ .
这道题本身应该用动态规划做. 不过, 如果用搜索, 这道题仍可以通过, 并不会超时.

zy同学说, 他用暴力搜索算法, 手写的循环怎么写都不如递归快.
于是我也试了一下, 确实如此.


## 代码
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <memory.h>
#include <vector>
#include <map>
#include <queue>
#include <set>
#include <unordered_set>
#include <unordered_map>

using namespace std; 

struct Frame2 {
    Frame2() {}
    Frame2(int _level, int _sum) {
        level = _level;
        sum = _sum;
    }
    int level;
    int sum;
};

class Solution {
public:
    int findTargetSumWaysQ(vector<int>& nums, int target)
    {
        int N = nums.size();
        if (!N) return 0;
        
        int total = 0;
        // vector<Frame2> stk(nums.size()+1);
        Frame2* stk = (Frame2*)malloc((N+1)*sizeof(Frame2));
        stk[0].sum = 0;  stk[0].level = 0;

        int stk_ptr = 0;
        while (stk_ptr>=0) {
            int pre_sum = stk[stk_ptr].sum;
            int level = stk[stk_ptr].level;

            if (level == N) {
                --stk_ptr;
                if (pre_sum == target) {
                    ++total;
                }
                continue;
            }
            int num = nums[level];
            stk[stk_ptr].sum = pre_sum + num;
            stk[stk_ptr].level = 1+level;

            stk[stk_ptr+1].sum = pre_sum - num;
            stk[stk_ptr+1].level = 1+level;

            stk_ptr += 1;
        }
        free(stk);
        return total;
    }

    int findTargetSumWaysR(vector<int>& nums, int target) {
        return findTargetSumWaysRecursiveImpl(nums, 0, 0, target);
    }

    int findTargetSumWaysRecursiveImpl(vector<int>& nums, int level, int sum, int target)
    {
        if (level >= nums.size())
        {
            return (int)(sum == target);
        }
        else
        {
            return findTargetSumWaysRecursiveImpl(nums, level + 1, sum + nums[level], target) +
                    findTargetSumWaysRecursiveImpl(nums, level + 1, sum - nums[level], target);
        }
    }
};
```

+ `findTargetSumWaysQ()` 和 `findTargetSumWaysR()` 算法完全相同
+ `findTargetSumWaysR()` 是zy同学写的递归版
+ `findTargetSumWaysQ()` 是我的手写循环版本. 不敢说写得最好, 但有以下性质: 
  + 只有一次申请 `(N+1)*sizeof(Frame2)` 空间的 memory allocation 操作. 这个空间是我手动维护的 call stack. 
    其他都是简单地使用局部变量和赋值操作. 之后编译时将开启优化, 局部变量都应该是放在寄存器里的.
  + `if` 语句在我看来已经最少了
  
可以去 leetcode 上提交看看, 确实递归版比手写的循环版本更快.


## 测试
### 实验环境
使用 gcc (g++) 和 clang (clang++), 在本地 Ubuntu 做实验
```sh
zsh@zsh-ubuntu:~/workspace/acm_alike$ clang -v
clang version 3.8.0-2ubuntu4 (tags/RELEASE_380/final)
Target: x86_64-pc-linux-gnu
Thread model: posix
InstalledDir: /usr/bin
Found candidate GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/5.4.0
Found candidate GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/6.0.0
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/5.4.0
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/6.0.0
Selected GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/5.4.0
Candidate multilib: .;@m64
Selected multilib: .;@m64


zsh@zsh-ubuntu:~/workspace/acm_alike$ gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc/x86_64-linux-gnu/5/lto-wrapper
Target: x86_64-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Ubuntu 5.4.0-6ubuntu1~16.04.9' --with-bugurl=file:///usr/share/doc/gcc-5/README.Bugs --enable-languages=c,ada,c++,java,go,d,fortran,objc,obj-c++ --prefix=/usr --program-suffix=-5 --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --with-sysroot=/ --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-vtable-verify --enable-libmpx --enable-plugin --with-system-zlib --disable-browser-plugin --enable-java-awt=gtk --enable-gtk-cairo --with-java-home=/usr/lib/jvm/java-1.5.0-gcj-5-amd64/jre --enable-java-home --with-jvm-root-dir=/usr/lib/jvm/java-1.5.0-gcj-5-amd64 --with-jvm-jar-dir=/usr/lib/jvm-exports/java-1.5.0-gcj-5-amd64 --with-arch-directory=amd64 --with-ecj-jar=/usr/share/java/eclipse-ecj.jar --enable-objc-gc --enable-multiarch --disable-werror --with-arch-32=i686 --with-abi=m64 --with-multilib-list=m32,m64,mx32 --enable-multilib --with-tune=generic --enable-checking=release --build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=x86_64-linux-gnu
Thread model: posix
gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.9) 
```

### 测试代码
前面已经贴出了算法部分的代码, 这里贴出 `main()` 函数
`iter_main.cpp`:
```cpp
int main()
{
    Solution s;
    {
        vector<int> nums = {1,1,1,1,1, 1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1};
        printf("%d\n", s.findTargetSumWaysQ(nums, 3));
    }

    return 0;
}
```

`recur_main.cpp`:
```cpp
int main()
{
    Solution s;
    {
        vector<int> nums = {1,1,1,1,1, 1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1};
        printf("%d\n", s.findTargetSumWaysR(nums, 3));
    }

    return 0;
}
```

编译命令分别为:
```sh
clang++ -o ./iter2  -O3 cpp_src/iter_main.cpp   -std=c++11 
clang++ -o ./recur2 -O3 cpp_src/recur_main.cpp  -std=c++11 

g++     -o ./iter3  -O3 cpp_src/iter_main.cpp   -std=c++11 
g++     -o ./recur3 -O3 cpp_src/recur_main.cpp  -std=c++11 
```

测试速度的命令为:
```sh
time  ./iter2
time  ./recur2
time  ./iter3
time  ./recur3
```

### 测试结果
注: 每个情形事实上都跑了多次, 并计算了平均的耗时. 
这里对于每个情形, 仅仅贴出一份能够代表平均耗时的数据, 便于观看.


`g++` 编译的递归版本:
```
zsh@zsh-ubuntu:~/workspace/acm_alike$ time ./recur3
4457400

real	0m0.088s
user	0m0.084s
sys	0m0.004s
```

`clang++` 编译的递归版本:
```
zsh@zsh-ubuntu:~/workspace/acm_alike$ time ./recur2
4457400

real	0m0.230s
user	0m0.204s
sys	0m0.004s
```

`g++` 编译的循环版本:
```
zsh@zsh-ubuntu:~/workspace/acm_alike$ time ./iter3
4457400

real	0m0.109s
user	0m0.084s
sys	0m0.008s
```

`clang++` 编译的循环版本:
```
zsh@zsh-ubuntu:~/workspace/acm_alike$ time ./iter2
4457400

real	0m0.106s
user	0m0.092s
sys	0m0.000s
```


## 分析
性能排序(从好到差): `g++递归` > `clang++循环` = `g++循环` > `clang++递归`.
`clang++` 编译出来的递归版本的性能是符合预期的.
`g++` 究竟做了什么优化, 使得递归版本超越了手写的循环版本呢?

注意到, 可执行文件 `recur3` 大小竟然是 12.9 KB, `recur2` 仅仅是 8.65 KB;
作为比较, `iter2` 大小是 8.66 KB, `iter3` 大小是 8.85 KB. 

递归版本的源码是远远比循环版本要短的. 然而 `g++` 编译出来的 `recur3` 竟然是所有可执行文件中最大的一个.
通过命令 `g++ -S -o recur3.s  -O3 cpp_src/main.cpp  -std=c++11 ` 观察 `g++` 编译出的 `Solution::findTargetSumWaysRecursiveImpl` 函数的汇编代码 (附于本文最后), 
得出结论: `g++` **对递归函数做了展开(Recursion flattening), 略微牺牲了二进制文件的大小, 减少了条件跳转, 换取了性能.**

这个案例有趣之处在于, `g++` 对于 **non-tail recursion** 也能进行展开 (flattening), 并能做到超过手工的优化.
参考阅读: http://32leav.es/?p=674


## `g++` 编译出的 `Solution::findTargetSumWaysRecursiveImpl` 汇编代码
```S
_ZN8Solution30findTargetSumWaysRecursiveImplERSt6vectorIiSaIiEEiii:
.LFB8874:
	.cfi_startproc
	pushq	%r15
	.cfi_def_cfa_offset 16
	.cfi_offset 15, -16
	pushq	%r14
	.cfi_def_cfa_offset 24
	.cfi_offset 14, -24
	movslq	%edx, %rdx
	pushq	%r13
	.cfi_def_cfa_offset 32
	.cfi_offset 13, -32
	pushq	%r12
	.cfi_def_cfa_offset 40
	.cfi_offset 12, -40
	pushq	%rbp
	.cfi_def_cfa_offset 48
	.cfi_offset 6, -48
	pushq	%rbx
	.cfi_def_cfa_offset 56
	.cfi_offset 3, -56
	subq	$232, %rsp
	.cfi_def_cfa_offset 288
	movq	(%rsi), %r13
	movq	8(%rsi), %rax
	movq	%rdi, 16(%rsp)
	movq	%rsi, 24(%rsp)
	movl	%edx, 124(%rsp)
	movl	%ecx, 100(%rsp)
	subq	%r13, %rax
	movl	%r8d, 12(%rsp)
	movq	%rdx, 136(%rsp)
	sarq	$2, %rax
	cmpq	%rdx, %rax
	movq	%rax, (%rsp)
	jbe	.L21
	movl	$0, 216(%rsp)
	movq	%r13, %r15
	movq	%rdx, %rax
.L3:
	movl	(%r15,%rax,4), %edx
	addl	$1, 124(%rsp)
	addq	$1, %rax
	movq	%rax, 136(%rsp)
	movl	%edx, 128(%rsp)
	addl	100(%rsp), %edx
	cmpq	(%rsp), %rax
	movl	%edx, 96(%rsp)
	movl	124(%rsp), %edx
	jnb	.L4
	movq	%rax, 152(%rsp)
	movl	%edx, 104(%rsp)
	movl	$0, 220(%rsp)
.L14:
	movl	(%r15,%rax,4), %edx
	addl	$1, 104(%rsp)
	addq	$1, %rax
	movq	%rax, 152(%rsp)
	movl	%edx, 160(%rsp)
	addl	96(%rsp), %edx
	cmpq	%rax, (%rsp)
	movl	%edx, 92(%rsp)
	movl	104(%rsp), %edx
	jbe	.L5
	movq	%rax, 168(%rsp)
	movl	%edx, 112(%rsp)
	movl	$0, 212(%rsp)
.L15:
	movl	(%r15,%rax,4), %edx
	addl	$1, 112(%rsp)
	addq	$1, %rax
	movl	112(%rsp), %esi
	movq	%rax, 168(%rsp)
	movl	%edx, 164(%rsp)
	addl	92(%rsp), %edx
	cmpq	%rax, (%rsp)
	movl	%edx, 88(%rsp)
	jbe	.L6
	movq	%rax, 192(%rsp)
	movl	%esi, 116(%rsp)
	movl	$0, 204(%rsp)
.L16:
	movl	(%r15,%rax,4), %edx
	addl	$1, 116(%rsp)
	addq	$1, %rax
	movl	116(%rsp), %esi
	movq	%rax, 192(%rsp)
	movl	%edx, 180(%rsp)
	addl	88(%rsp), %edx
	cmpq	%rax, (%rsp)
	movl	%edx, 84(%rsp)
	jbe	.L7
	movq	%rax, 184(%rsp)
	movl	%esi, 120(%rsp)
	movl	%edx, %edi
	movl	$0, 176(%rsp)
.L17:
	movl	(%r15,%rax,4), %esi
	addl	$1, 120(%rsp)
	addq	$1, %rax
	movl	120(%rsp), %edx
	movq	%rax, 184(%rsp)
	addl	%esi, %edi
	cmpq	%rax, (%rsp)
	movl	%esi, 200(%rsp)
	movl	%edi, 80(%rsp)
	jbe	.L8
	movq	%rax, 144(%rsp)
	movl	%edx, 108(%rsp)
	movl	$0, 132(%rsp)
.L18:
	movl	(%r15,%rax,4), %edx
	addl	$1, 108(%rsp)
	addq	$1, %rax
	movl	108(%rsp), %esi
	movq	%rax, 144(%rsp)
	addl	%edx, %edi
	cmpq	%rax, (%rsp)
	movl	%edx, 208(%rsp)
	movl	%edi, 60(%rsp)
	jbe	.L9
	movl	%esi, 56(%rsp)
	movq	%rax, 64(%rsp)
	movl	%edi, %esi
	movl	$0, 76(%rsp)
.L20:
	movl	(%r15,%rax,4), %edx
	addl	$1, 56(%rsp)
	addq	$1, %rax
	movq	%rax, 64(%rsp)
	addl	%edx, %esi
	cmpq	%rax, (%rsp)
	movl	%edx, 72(%rsp)
	movl	%esi, 44(%rsp)
	movl	56(%rsp), %edx
	jbe	.L10
	movq	%rax, 32(%rsp)
	movl	%edx, 40(%rsp)
	movl	%esi, %r14d
	movl	$0, 52(%rsp)
	.p2align 4,,10
	.p2align 3
.L19:
	movl	(%r15,%rax,4), %edi
	addl	$1, 40(%rsp)
	addq	$1, %rax
	movl	40(%rsp), %esi
	movq	%rax, 32(%rsp)
	addl	%edi, %r14d
	cmpq	%rax, (%rsp)
	movl	%edi, 48(%rsp)
	jbe	.L11
	movq	%rax, %rbx
	xorl	%ebp, %ebp
	movl	%esi, %r12d
	movl	%r14d, %r13d
	.p2align 4,,10
	.p2align 3
.L13:
	movl	(%r15,%rbx,4), %r14d
	movl	12(%rsp), %r8d
	addl	$1, %r12d
	movq	24(%rsp), %rsi
	movq	16(%rsp), %rdi
	movl	%r12d, %edx
	addq	$1, %rbx
	leal	0(%r13,%r14), %ecx
	subl	%r14d, %r13d
	call	_ZN8Solution30findTargetSumWaysRecursiveImplERSt6vectorIiSaIiEEiii
	addl	%eax, %ebp
	cmpq	%rbx, (%rsp)
	ja	.L13
	xorl	%eax, %eax
	movl	48(%rsp), %edx
	subl	%edx, 44(%rsp)
	cmpl	%r13d, 12(%rsp)
	movl	44(%rsp), %r14d
	sete	%al
	addl	%eax, %ebp
	movq	32(%rsp), %rax
	addl	%ebp, 52(%rsp)
	jmp	.L19
.L11:
	movl	44(%rsp), %eax
	subl	48(%rsp), %eax
	movl	12(%rsp), %esi
	movl	72(%rsp), %edx
	subl	%edx, 60(%rsp)
	cmpl	%esi, %eax
	sete	%dl
	cmpl	%r14d, %esi
	movl	60(%rsp), %esi
	sete	%al
	movzbl	%dl, %edx
	movzbl	%al, %eax
	addl	52(%rsp), %eax
	addl	%edx, %eax
	addl	%eax, 76(%rsp)
	movq	64(%rsp), %rax
	jmp	.L20
.L10:
	movl	12(%rsp), %edi
	movl	208(%rsp), %esi
	subl	%esi, 80(%rsp)
	movl	44(%rsp), %esi
	movl	60(%rsp), %edx
	cmpl	%esi, %edi
	sete	%al
	subl	72(%rsp), %edx
	movzbl	%al, %eax
	addl	76(%rsp), %eax
	cmpl	%edi, %edx
	movl	80(%rsp), %edi
	sete	%dl
	movzbl	%dl, %edx
	addl	%edx, %eax
	addl	%eax, 132(%rsp)
	movq	144(%rsp), %rax
	jmp	.L18
.L9:
	movl	200(%rsp), %edx
	movl	12(%rsp), %esi
	subl	%edx, 84(%rsp)
	movl	60(%rsp), %edx
	movl	84(%rsp), %edi
	cmpl	%edx, %esi
	movl	80(%rsp), %edx
	sete	%al
	subl	208(%rsp), %edx
	movzbl	%al, %eax
	addl	132(%rsp), %eax
	cmpl	%esi, %edx
	sete	%dl
	movzbl	%dl, %edx
	addl	%edx, %eax
	addl	%eax, 176(%rsp)
	movq	184(%rsp), %rax
	jmp	.L17
.L8:
	movl	12(%rsp), %esi
	movl	%edi, %edx
	movl	180(%rsp), %edi
	subl	%edi, 88(%rsp)
	cmpl	%edx, %esi
	movl	84(%rsp), %edx
	sete	%al
	subl	200(%rsp), %edx
	movzbl	%al, %eax
	addl	176(%rsp), %eax
	cmpl	%esi, %edx
	sete	%dl
	movzbl	%dl, %edx
	addl	%edx, %eax
	addl	%eax, 204(%rsp)
	movq	192(%rsp), %rax
	jmp	.L16
.L7:
	movl	12(%rsp), %edi
	movl	164(%rsp), %esi
	subl	%esi, 92(%rsp)
	cmpl	%edx, %edi
	movl	88(%rsp), %edx
	sete	%al
	subl	180(%rsp), %edx
	movzbl	%al, %eax
	addl	204(%rsp), %eax
	cmpl	%edi, %edx
	sete	%dl
	movzbl	%dl, %edx
	addl	%edx, %eax
	addl	%eax, 212(%rsp)
	movq	168(%rsp), %rax
	jmp	.L15
.L6:
	movl	92(%rsp), %eax
	subl	164(%rsp), %eax
	xorl	%edx, %edx
	movl	12(%rsp), %esi
	movl	160(%rsp), %edi
	subl	%edi, 96(%rsp)
	cmpl	%esi, %eax
	sete	%dl
	xorl	%eax, %eax
	cmpl	88(%rsp), %esi
	sete	%al
	addl	212(%rsp), %eax
	addl	%edx, %eax
	addl	%eax, 220(%rsp)
	movq	152(%rsp), %rax
	jmp	.L14
.L5:
	movl	128(%rsp), %edi
	movl	92(%rsp), %esi
	subl	%edi, 100(%rsp)
	movl	12(%rsp), %edi
	movl	96(%rsp), %edx
	cmpl	%esi, %edi
	sete	%al
	subl	160(%rsp), %edx
	movzbl	%al, %eax
	addl	220(%rsp), %eax
	cmpl	%edi, %edx
	sete	%dl
	movzbl	%dl, %edx
	addl	%edx, %eax
	addl	%eax, 216(%rsp)
	movq	136(%rsp), %rax
	jmp	.L3
.L4:
	movl	128(%rsp), %edx
	xorl	%eax, %eax
	subl	%edx, 100(%rsp)
	movl	96(%rsp), %edi
	cmpl	%edi, 12(%rsp)
	movl	100(%rsp), %edx
	sete	%al
	addl	216(%rsp), %eax
.L2:
	cmpl	%edx, 12(%rsp)
	sete	%dl
	addq	$232, %rsp
	.cfi_remember_state
	.cfi_def_cfa_offset 56
	movzbl	%dl, %edx
	popq	%rbx
	.cfi_def_cfa_offset 48
	addl	%edx, %eax
	popq	%rbp
	.cfi_def_cfa_offset 40
	popq	%r12
	.cfi_def_cfa_offset 32
	popq	%r13
	.cfi_def_cfa_offset 24
	popq	%r14
	.cfi_def_cfa_offset 16
	popq	%r15
	.cfi_def_cfa_offset 8
	ret
.L21:
	.cfi_restore_state
	xorl	%eax, %eax
	movl	%ecx, %edx
	jmp	.L2
	.cfi_endproc
```

