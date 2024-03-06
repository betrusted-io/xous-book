
# OLD NOTES

Here is the flow for how page faults happen in a `swap` enabled kernel:

1. Catch Store, Load or Instruction page fault (inside `kernel/src/arch/riscv/irq.c/trap_handler`)
2. `ensure_page_exists_inner`:
   1. checks if the address makes sense
   2. If the PTE is marked invalid, check the physical address.
      1. Do a single-pass, bottom-up search through memory for the first unallocated physical page.
         1. If the physical address is not in the swap space:
            1. Zero the page, map it, flush the MMU
            2. Stick an `AllocAdvisory` message to the `swapper`
         2. If the physical address is in the swap space:
            1. Zero the page, map it, flush the MMU
            2. Call `swapper` with `MoveToMain`, with this allocated page lended as an argument
            3. Swapper decrypts the requested page and copies it to the lent page
            4. Swapper decrements physical kernel memory tracker
            5. Execution returns to the kernel
3. Return from the page fault and resume execution
4. `swapper` gains a quantum and handles `AllocAdvisory`
   1. Kernel free memory tracker is decremented
   2. If the free memory amount falls below the low water mark, find the least recently used pages, and use the `UncommitPage` syscall.
      1. `EvictPage` is called with the new physical address for the data to be moved
      2. Kernel notes the physical address of the page to be uncommitted
      3. Kernel modifies the target process's PT so that the virtual page as invalid but points to the physical address
      4. Kernel maps the uncommit physical page into the swapper's memory space as the return result of the `UncommitPage` syscall
      5. The uncommit page is encrypted

      6. Decrement the amount of memory available, 
      7. If (1) fails, send a `MoveToSwap` message to the `swapper`
      8. Activate the `swapper` context
      9. `swapper` searches through swap for the first unallocated physical page
      10. `swapper` calls `EvictPage` on the kernel


For contrast, here is the flow for how pages faults happen in a kernel without `swap` enabled:

1. Catch Store or Load page fault (inside `kernel/src/arch/riscv/irq.c/trap_handler`)
2. If the PTE is marked invalid call `ensure_page_exists_inner`:
   1. checks if the address makes sense
   2. If it is demand-paged, allocate a new physical page with `kernel/src/mem.rs/MemoryManager::alloc_page()` -- this does a single-pass, bottom-up search through memory for the first unallocated physical page.
   3. Zero the page, map it, flush the MMU
3. Return from the page fault and resume execution

# Random notes

general idea of how to invoke the swapper:

const HANDLER_PID: PID = PID::new(2);
SystemServices::with_mut(|ss| {
    // Disable all other IRQs and redirect into userspace
    arch::irq::disable_all_irqs();
    ss.make_callback_to(HANDLER_PID,
                        function_address as *const usize,
                        arg_0,
                        arg_1,
    )
}).expect("couldn't switch to handler");
ArchProcess::with_current_mut(|process| {
    crate::arch::syscall::resume(current_pid().get() == 1, process.current_thread())
})

Make a special version of this, for the swapper to return to (i.e., allocate a new magic address for swapper return):
https://github.com/betrusted-io/xous-core/blob/8622fbdf7a1d63924e9fc5fe86ef69e35a3122b1/kernel/src/arch/riscv/irq.rs#L303-L308

This will need patching to go to that magic address:
https://github.com/betrusted-io/xous-core/blob/8622fbdf7a1d63924e9fc5fe86ef69e35a3122b1/kernel/src/services.rs#L649-L670
