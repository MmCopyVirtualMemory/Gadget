# Gadget
**I hope to post some useful gadgets that I find here for the eager reverse engineering enthusiasts to utilize and take inspiration from. If you would like to contribute your own gadget(s), a PR is welcome.**

# KsecDD Table Gadget(s):
![image](https://user-images.githubusercontent.com/88007716/231940159-dc0ca94a-1789-4244-9b32-c66cc5ce6584.png)
```
48 83 EC 28 48 8B 05 ? ? ? ? 48 85 C0 74 11 48 8B 40 18 48 85 C0 74 08 FF 15
```
This gadget is pretty cool for a couple of reasons. Firstly, if you didn't already catch it from the disassembly, the gadget moves a table function pointer into rax and then via __guard_dispatch_icall_fptr, the function is called. Therefore by writing to this table we can jump to our code. Secondly, it's more complex than a simple jmp, mov->call, or mov->jmp gadget which means that it will bypass common gadget checks. In addition, this uses a table which means this is far from the only function that can be used.

Usage Example:
We need to keep in mind that the there will be a return address pushed on the stack when __guard_dispatch_icall_fptr is called. This means that it is imperative that we pop a value off the stack before passing execution to our handler. My fixup looks as follows:
```
pop rax
mov rax, HandlerFunction
jmp rax
```
Once the stack is repaired, we can handle execution appropriately. To show this, working, I chose to hook the IOCTL major irp from KsecDD. Here is a snippet of my code:
```cpp
static U64 dummy_table[512] = { 0 };
memcpy(&dummy_table[0], (void*)function_table, sizeof(dummy_table));
dummy_table[3] = (U64)&fixup_asm[0];
*(U64*)((U64)ksecdd.dos + table) = (U64)&dummy_table[0];
driver_object->MajorFunction[IRP_MJ_DEVICE_CONTROL] = (PDRIVER_DISPATCH)(ksecdd + gadget);
```
When I was debugging, I found that writing to the table directly caused a BSOD (PG maybe?). To fix this, I copied to contents of the table and changed the table to point to my new replica table. This is fine because I am only modifying the table pointer which resides in the .data section.

Finally the dispatch points to the gadget. When executed, the gadget grabs our handler from the table and calls it. Using a little bit of assembly, the stack is repaired and our hook is executed without the dispatch ever pointing outside of the proper section.

