## 调用约定

 **ARM64 ABI**（应用程序二进制接口）中规定了在函数调用过程中，哪些寄存器的值需要由**调用方**（caller）负责保存，哪些由**被调用方**（callee）负责保存。

## ARM64 中的寄存器分类

AAPCS64 ABI

| 寄存器        | 类型                 | 别名                 | 说明                                  |
| ------------- | -------------------- | -------------------- | ------------------------------------- |
| **x0 – x7**   | **caller-saved**     | 参数 / 返回值        | 用于传递前8个参数；x0/x1 也用于返回值 |
| x8            | caller-saved         | 间接结果             | 用于某些系统调用或大返回值            |
| x9 – x15      | caller-saved         | 临时寄存器           | callee 可自由使用                     |
| **x16 – x17** | caller-saved         | IP0/IP1              | 用于跳转、链接器内部使用              |
| **x18**       | 特殊                 | 平台寄存器           | 通常保留给平台（如 TLS）              |
| **x19 – x29** | **callee-saved**     | 通用寄存器           | callee 若使用，必须保存并恢复         |
| x30           | callee-saved（部分） | **lr**（链接寄存器） | 存放返回地址，通常需保存              |

```c
// caller 函数
long caller(long a, long b) {
    long temp = a + b;          // 假设 temp 存在 x0
    long result = callee(a);    // 调用 callee
    return temp + result;       // ← 这里仍要用 temp（原 x0 的值）
}
// callee 函数
long callee(long x) {
    return x * 2;               // 编译器可能用 x0 存放 x 和返回值
}
```
编译器生成的汇编（简化）：

```asm
caller:
    add x0, x0, x1        // temp = a + b → 存在 x0
    stp x0, x1, [sp, #-16]!   // ← 保存 x0（temp）！因为 callee 会覆盖它
    bl callee             // 调用 callee（callee 会用 x0 存参数和返回值）
    ldp x2, x1, [sp], #16     // 恢复 temp 到 x2
    add x0, x2, x0        // return temp + result
    ret

callee:
    lsl x0, x0, #1        // x0 = x * 2
    ret                   // 直接返回，不恢复 x0（因为它是 caller-saved）
```

## 设计目的

x0-x7作为参数寄存器，要一层一层的传递给callee，如果在caller做了修改，直接作为参数就传递进callee去了，他们经常需要被传递和修改，就像一个context或者全局变量一样，穿梭在各种函数之间，尤其是x0，x1这种既作为参数又作为返回值的寄存器，使用后不管是常态。

如果某些值需要保留生命周期，留在当前的caller里继续使用，向局部变量那样，则压栈，或者分配到x19–x28（callee-saved）,arm不是像x86那样一开始就开辟栈来保存局部变量。压栈和放到寄存器肯定是优先使用寄存器，当寄存器不够用的时候，才考虑压栈。

caller其实不知道哪些当前使用过的寄存器后续会不会被使用，决定要不要压栈是编译器的事，压栈属于编译器分析代码后的决定。比如一个变量放在x2里被操作，编译器编译过程中把他压栈，后续再使用变量的时候，编译器可能把它恢复到x3里倍操作。