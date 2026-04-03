# In particular, a memory-safe assembly block may only access the following memory ranges:

Memory allocated by yourself using a mechanism like the allocate function described above.
Memory allocated by Solidity, e.g. memory within the bounds of a memory array you reference.
The scratch space between memory offset 0 and 64 mentioned above.
Temporary memory that is located after the value of the free memory pointer at the beginning of the assembly block, i.e. memory that is “allocated” at the free memory pointer without updating the free memory pointer.