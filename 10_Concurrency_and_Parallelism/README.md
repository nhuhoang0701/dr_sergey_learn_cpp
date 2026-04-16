# Concurrency & Parallelism

Threads, mutexes, atomics, memory model, lock-free programming, coroutine-based concurrency, and synchronization primitives.

**Topics:** 48

## Contents

- [Avoid deadlocks with stdlock and stdscoped lock C++17](Avoid_deadlocks_with_stdlock_and_stdscoped_lock_C++17.md)
- [Avoid false sharing with cache-line padding in concurrent data structures](Avoid_false_sharing_with_cache-line_padding_in_concurrent_data_structures.md)
- [Combine coroutines with stdexecution for structured async concurrency](Combine_coroutines_with_stdexecution_for_structured_async_concurrency.md)
- [Compare async patterns callbacks vs futures vs coroutines vs senders-receivers](Compare_async_patterns_callbacks_vs_futures_vs_coroutines_vs_senders-receivers.md)
- [Implement a concurrent hash map using striped locking](Implement_a_concurrent_hash_map_using_striped_locking.md)
- [Implement a lock-free stack using compare exchange weak](Implement_a_lock-free_stack_using_compare_exchange_weak.md)
- [Implement a producer-consumer queue with backpressure using counting semaphore](Implement_a_producer-consumer_queue_with_backpressure_using_counting_semaphore.md)
- [Implement a thread-safe singleton using stdcall once](Implement_a_thread-safe_singleton_using_stdcall_once.md)
- [Implement a thread-safe singleton using stdcall once 2](Implement_a_thread-safe_singleton_using_stdcall_once_2.md)
- [Implement a wait-free data structure using fetch add](Implement_a_wait-free_data_structure_using_fetch_add.md)
- [Implement sequence locks seqlock for read-heavy concurrent data](Implement_sequence_locks_seqlock_for_read-heavy_concurrent_data.md)
- [Know RCU Read-Copy-Update patterns before stdstdrcu arrives](Know_RCU_Read-Copy-Update_patterns_before_stdstdrcu_arrives.md)
- [Understand DRF-SC Data Race Freedom implies Sequential Consistency](Understand_DRF-SC_Data_Race_Freedom_implies_Sequential_Consistency.md)
- [Understand and avoid TOCTOU races in concurrent code](Understand_and_avoid_TOCTOU_races_in_concurrent_code.md)
- [Understand coroutine-based concurrency vs thread-based concurrency](Understand_coroutine-based_concurrency_vs_thread-based_concurrency.md)
- [Understand epoch-based memory reclamation for lock-free data structures](Understand_epoch-based_memory_reclamation_for_lock-free_data_structures.md)
- [Understand memory fences and stdatomic thread fence](Understand_memory_fences_and_stdatomic_thread_fence.md)
- [Understand memory fences and stdatomic thread fence 2](Understand_memory_fences_and_stdatomic_thread_fence_2.md)
- [Understand stdhardware destructive interference size and alignment for concurren](Understand_stdhardware_destructive_interference_size_and_alignment_for_concurren.md)
- [Understand stdhazard pointer and stdrcu C++26 preview for lock-free reclamation](Understand_stdhazard_pointer_and_stdrcu_C++26_preview_for_lock-free_reclamation.md)
- [Understand stdstdsynchronized value for mutex-wrapped data access](Understand_stdstdsynchronized_value_for_mutex-wrapped_data_access.md)
- [Understand the ABA problem in lock-free programming](Understand_the_ABA_problem_in_lock-free_programming.md)
- [Understand the C++ memory model and happens-before relationships](Understand_the_C++_memory_model_and_happens-before_relationships.md)
- [Understand the cooperative cancellation model of stdstop token](Understand_the_cooperative_cancellation_model_of_stdstop_token.md)
- [Use lock-free data structures with correct memory ordering](Use_lock-free_data_structures_with_correct_memory_ordering.md)
- [Use relaxed atomics correctly for counters and statistics](Use_relaxed_atomics_correctly_for_counters_and_statistics.md)
- [Use stdasync with a thread pool pattern for task parallelism](Use_stdasync_with_a_thread_pool_pattern_for_task_parallelism.md)
- [Use stdatomic flag as the simplest lock-free primitive](Use_stdatomic_flag_as_the_simplest_lock-free_primitive.md)
- [Use stdatomic for lock-free operations on simple types](Use_stdatomic_for_lock-free_operations_on_simple_types.md)
- [Use stdatomic ref for atomic access to non-atomic objects C++20](Use_stdatomic_ref_for_atomic_access_to_non-atomic_objects_C++20.md)
- [Use stdatomicstdshared ptrT C++20 for safe concurrent pointer updates](Use_stdatomicstdshared_ptrT_C++20_for_safe_concurrent_pointer_updates.md)
- [Use stdatomicwait and notify for efficient waiting C++20](Use_stdatomicwait_and_notify_for_efficient_waiting_C++20.md)
- [Use stdatomicwait notify one notify all C++20 for efficient flag signaling](Use_stdatomicwait_notify_one_notify_all_C++20_for_efficient_flag_signaling.md)
- [Use stdcondition variable for thread synchronization](Use_stdcondition_variable_for_thread_synchronization.md)
- [Use stdcounting semaphore and stdbinary semaphore C++20](Use_stdcounting_semaphore_and_stdbinary_semaphore_C++20.md)
- [Use stdfuture and stdpromise for async result passing](Use_stdfuture_and_stdpromise_for_async_result_passing.md)
- [Use stdfuturewait for for polling with a timeout](Use_stdfuturewait_for_for_polling_with_a_timeout.md)
- [Use stdjthread and stop tokens C++20 for cooperative cancellation](Use_stdjthread_and_stop_tokens_C++20_for_cooperative_cancellation.md)
- [Use stdlatch and stdbarrier C++20 for thread synchronization points](Use_stdlatch_and_stdbarrier_C++20_for_thread_synchronization_points.md)
- [Use stdmutex stdlock guard and stdunique lock correctly](Use_stdmutex_stdlock_guard_and_stdunique_lock_correctly.md)
- [Use stdosyncstream C++20 for synchronized output without interleaving](Use_stdosyncstream_C++20_for_synchronized_output_without_interleaving.md)
- [Use stdpackaged task for deferred execution with future results](Use_stdpackaged_task_for_deferred_execution_with_future_results.md)
- [Use stdpromisevoid for one-shot thread synchronization signals](Use_stdpromisevoid_for_one-shot_thread_synchronization_signals.md)
- [Use stdshared mutex for read-write locking](Use_stdshared_mutex_for_read-write_locking.md)
- [Use stdshared mutex for reader-writer locking](Use_stdshared_mutex_for_reader-writer_locking.md)
- [Use stdthread correctly and always join or detach](Use_stdthread_correctly_and_always_join_or_detach.md)
- [Use thread-local storage for per-thread state](Use_thread-local_storage_for_per-thread_state.md)
- [Use work-stealing thread pools for load-balanced parallel tasks](Use_work-stealing_thread_pools_for_load-balanced_parallel_tasks.md)

## Notes

- std::jthread (C++20) auto-joins on destruction and supports cooperative cancellation via stop tokens
- std::mutex + std::lock_guard / std::scoped_lock is the basic synchronization pattern
- std::atomic provides lock-free access for simple types — know the memory ordering options
- Data races are undefined behavior in C++ — even a single unsynchronized read/write pair
- std::async with std::launch::async forces a new thread — std::launch::deferred runs lazily
- std::counting_semaphore and std::latch/std::barrier (C++20) simplify coordination patterns
- False sharing destroys performance — align shared atomics to cache line boundaries (std::hardware_destructive_interference_size)
- Lock-free programming requires deep understanding of memory ordering — start with seq_cst
- std::condition_variable must always be used with a predicate loop to handle spurious wakeups
- Thread-local storage (	hread_local) avoids sharing and synchronization for per-thread state
