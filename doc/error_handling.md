# Error handling
For ease of use, errors are thrown and caught, but they are not exceptions,
within this language we call them errors. 

A caller is responsible for catching all errors that can be thrown by an
expression. The compiler will check if all errors that can be thrown are caught.

Therefor all control flow is local, making it easier to reason about the code.

Features:
 - Although errors are thrown, they are not exceptions. 
 - No non-local control flow.
 - Compiler checks if all errors that can be thrown are caught.
 - Creating a new error is easy.
 - Fast
   - No setup on call.
   - No setup on return.
   - Fast to throw an error.
   - Fast to catch an error.
 - A chain of errors is recorded, so that the error can be traced back to the
   point where it was originally thrown.



## Throwing an error.
The `throw` statement is used to throw an error; its argument is the name of
an error. The act of throwing an error also declares the name of the error,
internally the name will be assigned a unique `__u32__` value.

```
func foo(a: int) -> int {
    if a < 0 {
        throw bound_error
    }
    return a
}
```

You may throw an error from within an `catch` block. Unlike `trap` the error
argument is required, this so that it easy to determine which errors a
function can throw.

```
try {
    var x = foo()
    bar()
} catch {
    throw my_error
}
```

The compiler will track which errors can be thrown (or rethrown) by a function.
It is a static error if a function may throw an error that is not be caught by
the caller.

A function protype can be declared with a list of errors it can throw. The
compiler will make sure that the function will not try to throw an error that is
not in this list. This should be used when passing a function-pointer to a
function.

```
var my_func_type = func (a: int) throws(bound_error, underflow_error) -> int
```

## Trap an error
The `trap` statement is used to trap an error. A trapped error will cause
the CPU to be trapped, and the program to terminate displaying a messsage, stack
trace and error-chain.

```
func foo(a: int) -> int {
    if a < 0 {
        trap bound_error
    }
    return a
}
```

The `trap` statement may be used without an argument when used inside
a `catch` block, to trap the current error.

```
try {
    var x = foo()
    bar()
} catch {
    trap // This will trap the current error and terminate the program.
}
```

## Assertions
The following assert and constract statements throw the following error on
a false result:
 - `assert()` - `assertion_failure`
 - `debug_assert()` - `assertion_failure`
 - `pre()` - `precondition_failure`
 - `debug_pre()` - `precondition_failure`
 - `post()` - `postcondition_failure`
 - `debug_post()` - `postcondition_failure`
 - `invariant()` - `invariant_does_not_hold`
 - `debug_invariant()` - `invariant_does_not_hold`

## Catch clause
Control flow statements can have a `catch` clause. The `catch` clause is used to
catch errors that are thrown in the expression of a control-flow statement. A
`catch` clause must directly follow the clause with an expression that can
throw an error.

The following control-flow statements can have a `catch` clause:
 - `if` statement
 - `while` statement
 - `for` statement
 - `switch` statement
 - `try` statement

An example of a `catch` clause in an `if` statement:
```
if (foo()) {
    // code
} catch (underflow, overflow) {
    // code
} else {
    // code
}
```

The expression of the `catch` clause is a comma `,` separated list of errors. A
thrown error matches if the error is listed in the `catch` expression. The
`catch` clause can also be empty, in which case it will catch any error.

You can exit the `catch` block:
 - using a `throw` statement to rethrow the error possibly with a different
   error code. 
 - using a `trap` statement to trap the error. This will cause the program to
   terminate and call the error handler.
 - using a `return` statement to return from the function. This will cause the
   function to return successfully with a value.
 - executing to the end of the `catch` block. This will exit the current flow
   control statement and continue execution the current function.



The following functions can be used to display the error chain:
 - `std.error_chain(index: __u8__) -> (code: __u32__, address: __ptr__)` -
   returns the error code and address at the given index. Returns `(0, 0)` when
   the index is out of range.
 - `std.error_chain_to_string(code: __u32__, address: __ptr__) -> string` -
   returns a string which shows the error codes and addresses in a human
   readable format.
 - `std.print_error(code: __u32__, address: __ptr__)` - prints the the chain
   of error codes and addresses in a human readable format to stderr.

The following helper function are used to get information about an error-code
these are technically also valid outside of a `catch` statement:
 - `std.get_error_message(code: __u32__) -> string` - returns the error
   message of the error code. The error code must be a valid error code.
 - `std.get_error_name(code: __u32__) -> string` - returns the error name
   of the error code. The error code must be a valid error code.
 - `std.get_error_code(name: string) -> __u32__` - returns the error code of
   the error with the given name. The error name must be a valid error name.
 - `std.is_sub_error(code: __u32__, super_code: __u32__) -> bool` - returns true
   if the error code is a sub-error of the super error code. The error code and
   super error code must be valid error codes.


## try statement
The `try` statement is used to handle errors possibly thrown by multiple
statements or expressions. The `try` block will execute the code and when an
error is thrown, an optional `catch` block may catch the error.
Any error not catched will be rethrown.

```
try {
    var x = foo()
    bar()
} catch (bound_error) {
    std.print("This was a bound error")
} // No default catch block, other errors will be rethrown.
```

A `try { return foo() }` statement is compatible with tail-call optimization.


### try / trap / catch expression
Errors can also be handled within an expression. The `try` and `trap` prefix
operators are used to rethrow, or trap an error.

The binary `catch` operator evaluates the left operand and when it throws an
error, it will evaluate and return the right operand. The `catch` operator can
be used to catch an error and return a default value.

The `try`, `trap` and `catch` operators have a slightly lower precedence
(numerically higher) than the function-call operator. This means that the `try`
and `trap` operators will be applied directly to the result of the function
call.

```
var x = foo() // static error if foo() can throw.
var y = trap foo() // Any error is unhandled, and will trap.
var z = try foo() // Any error is rethrown.
var w = foo() catch 5 // Any error causes the results to be 5.
```

A `return try foo()` statement is compatible with tail-call optimization.

## ABI
On function return an error is signaled by setting an appropriate CPU flag. On
x86 this is done by setting the `CF` flag. The caller can check the flag and if
it is set with a conditional jump or a conditional move.

On error the lower 31 bits of the `RAX` register will be set to the error code.
Bit 31 of the `RAX` register will be set to 1 if the error is rethrown, this is
used as a performance optimization to avoid resetting the error-chain.

The `RDX` register will be set to instruction pointer of the `throw` statement,
possibly using `LEA RDX, [RIP + 0]` instruction.

When a `throw` statement is executed while inside the error-handling-block
RAX:RDX are appended to the error-chain stack. At the end of the
error-handling-block the error-chain stack is cleared, this only needs to
be done for rethrown errors as otherwise the error-chain would already by empty.

The error-chain stack is a stack of error codes and and addresses. This error
chain is used to produce a more complete stack trace, up to the point where the
error was thrown. The error chain stack is 256 entries deep, on overflow the
chain will wrap around and the oldest entries will be overwritten.

An trapped error will first add the error-code/address to the error-chain stack,
and then execute an appropiate trap-instruction to call the error handler. The
run-time services library should setup the trap system to call an trap handler.
The trap handler can determine if it was caused by an error by checking the
error-chain stack.

```

push_error_chain:
            mov rcx, 1
            xadd gs:[error_chain_head], rcx
            and rcx, 0xFF ; Limit the size of the error chain to 256 entries.
            sll rcx, 4
            lea rcx, [error_chain_data + rcx]
            mov gs:[rcx], rax
            mov gs:[rcx + 8], rdx
            ret
            
foo:
            lea rdx, [rip + 0]
            mov rax, unbound_error_value
            stc
            ret

foo_caller:
            call foo()
            jc .failure

success:
            ...
            xor eax, eax ; Clears the CF flag.
            ret

failure:
            cmp eax, unbound_error_value ; Only check the bottom 32 bits of the error code.
            je .bound_error

            cmp eax, assertion_failure_value ; Only check the bottom 32 bits of the error code.
            je .assert_error

            ; Ignore any other error. Clear the error chain.
            xor rdx, rdx
            bt rax, 31
            cmovc gs:[error_chain_head], rdx
            jmp .success

bound_error:
            ; rethrow
            call push_error_chain

            lea rdx, [rip + 0]
            ; Optionally set rax to a new error code.
            bts rax, 31 ; Indicate that the error is rethrown
            stc
            ret

assert_error:
            ; trap
            call push_error_chain
            int 3
```
