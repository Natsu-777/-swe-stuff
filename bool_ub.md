Let's look at some pretty intresting piece of code

```cpp
std::string bool_to_string(bool value){
    return value ? "true" : "false"; 
}
```

This seems like a normal standard function without any issues here, Or is it ?
Let's see what the compiler can do here with this piece of code

## 1. Looking at the Assembly
```asm
lea rax, ["false"]   ;load effective address of "false" into rax register
lea rbx, ["true"]    ;load effective address of "true" into rbx register

test rdi, rdi        ;test if rdi is zero and set ZF flag

cnmove rax, rbx.     ;if (ZF==0) rax = rbx
```

Memory Refresher:
- `rdi` contains the first function argument
- `test rdi rdi` performs the bitwise AND of the operands and discards the result, and updates the CPU flags. Set's the Zero flag (ZF) if result is zero
-  `cmovne` stands for conditional move if not equal, basically if (ZF==0) rax = rbx

Nice little trick by compiler to avoid the branching here by using `cmovne`

## 2. But What About the String Length?
Along with the adress of the string we also need length of the string to construct `std::string`

The lengths are

- `strlen("true")` = 4
- `str("false")` = 5

```asm
mov rcx, rdi
xor rcx, 5     ;wtf is this?
```

Take a closer look why above assembly is true
```text
value = 0 -> rdi = 0 -> rcx^5 = 0^5 = 5 = strlen(false)
value = 1 -> rdi = 1 -> rcx^5 = 1^5 = 4 = strlen(true)
```

Again the compiler cleverly removed the branching with a clever bit trick.

## 3. But wait is there anything wrong with this?

What if we pass uninitialized `bool`? 

Let's say somehow the value in `rdi` ended up as 2 then length calculation becomes

```text
rcx = 2^5 = 10^101 = 111 = 7
```
Selected string is still "true" because test result is non-zero but the length calculated is now 7

This could result in an out-of-bounds read when constructing the `std::string`, since we're attempting to read 7 characters from a string literal that only contains 5 bytes, including the null terminator.

That's a potential memory safety issue!

## 4. How Can This Happen?

In C++, a `bool` object can only hold two valid values: `true` and `false`.

Reading an uninitialized `bool` is undefined behavior. Think how we can make the value in `rdi` register as 2?

An important distinction: passing an integer with the value `2` to a function that accepts `bool` does not itself create an invalid `bool`. The implicit conversion converts it to `true`.

The problem arises when an invalid representation is somehow introduced into a `bool` object.

Reference : https://youtu.be/degCN1CSd5w?si=r-2zHiPoZRn1bbRR