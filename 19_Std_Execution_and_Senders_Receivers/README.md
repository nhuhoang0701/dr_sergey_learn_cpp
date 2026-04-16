# std::execution & Senders/Receivers

The execution model for async work: schedulers, senders, receivers, and structured concurrency.

**Topics:** 36

## Contents

- [Chain senders with then upon error and upon stopped](Chain_senders_with_then_upon_error_and_upon_stopped.md)
- [Compare stdexecution with coroutines and know when to combine them](Compare_stdexecution_with_coroutines_and_know_when_to_combine_them.md)
- [Implement a custom sender from scratch](Implement_a_custom_sender_from_scratch.md)
- [Implement a custom sender type satisfying the sender concept](Implement_a_custom_sender_type_satisfying_the_sender_concept.md)
- [Implement a custom sender type with the sender concept](Implement_a_custom_sender_type_with_the_sender_concept.md)
- [Integrate Asio with P2300 senders via asioexecution](Integrate_Asio_with_P2300_senders_via_asioexecution.md)
- [Understand cancellation in P2300 via stop tokens and stop callbacks](Understand_cancellation_in_P2300_via_stop_tokens_and_stop_callbacks.md)
- [Understand schedulers and execution contexts in stdexecution](Understand_schedulers_and_execution_contexts_in_stdexecution.md)
- [Understand schedulers and execution contexts in stdexecution 2](Understand_schedulers_and_execution_contexts_in_stdexecution_2.md)
- [Understand stop tokens and cancellation in sender pipelines](Understand_stop_tokens_and_cancellation_in_sender_pipelines.md)
- [Understand structured concurrency and async scope for safe task lifetimes](Understand_structured_concurrency_and_async_scope_for_safe_task_lifetimes.md)
- [Understand structured concurrency with async scope](Understand_structured_concurrency_with_async_scope.md)
- [Understand the senderreceiver execution model P2300](Understand_the_senderreceiver_execution_model_P2300.md)
- [Understand the senderreceiver model and why it replaces futures and callbacks](Understand_the_senderreceiver_model_and_why_it_replaces_futures_and_callbacks.md)
- [Understand the senderreceiver model what a sender is and what a receiver is](Understand_the_senderreceiver_model_what_a_sender_is_and_what_a_receiver_is.md)
- [Use async scope for structured concurrency and lifetime management](Use_async_scope_for_structured_concurrency_and_lifetime_management.md)
- [Use schedulers to control where senders execute](Use_schedulers_to_control_where_senders_execute.md)
- [Use stdexeclet value and let error for monadic sender chaining](Use_stdexeclet_value_and_let_error_for_monadic_sender_chaining.md)
- [Use stdexecon and stdexecstarts on for scheduler affinity](Use_stdexecon_and_stdexecstarts_on_for_scheduler_affinity.md)
- [Use stdexecsplit to multicast a sender to multiple receivers](Use_stdexecsplit_to_multicast_a_sender_to_multiple_receivers.md)
- [Use stdexecsync wait to drive a sender pipeline from synchronous code](Use_stdexecsync_wait_to_drive_a_sender_pipeline_from_synchronous_code.md)
- [Use stdexecthen to chain sender transformations](Use_stdexecthen_to_chain_sender_transformations.md)
- [Use stdexecutionbulk for parallel loop execution over a sender](Use_stdexecutionbulk_for_parallel_loop_execution_over_a_sender.md)
- [Use stdexecutionjust and just error to create leaf senders](Use_stdexecutionjust_and_just_error_to_create_leaf_senders.md)
- [Use stdexecutionjust to create a sender that emits a value](Use_stdexecutionjust_to_create_a_sender_that_emits_a_value.md)
- [Use stdexecutionlet value and let error for monadic sender chaining](Use_stdexecutionlet_value_and_let_error_for_monadic_sender_chaining.md)
- [Use stdexecutionlet value let error and let stopped for dependent senders](Use_stdexecutionlet_value_let_error_and_let_stopped_for_dependent_senders.md)
- [Use stdexecutionon to run a sender on a specific scheduler](Use_stdexecutionon_to_run_a_sender_on_a_specific_scheduler.md)
- [Use stdexecutionsplit to multicast a sender to multiple receivers](Use_stdexecutionsplit_to_multicast_a_sender_to_multiple_receivers.md)
- [Use stdexecutionsync wait to block until a sender completes](Use_stdexecutionsync_wait_to_block_until_a_sender_completes.md)
- [Use stdexecutionthen to chain sender transformations](Use_stdexecutionthen_to_chain_sender_transformations.md)
- [Use stdexecutionwhen all and when any for concurrent fan-out](Use_stdexecutionwhen_all_and_when_any_for_concurrent_fan-out.md)
- [Use stdexecutionwhen all to join multiple senders](Use_stdexecutionwhen_all_to_join_multiple_senders.md)
- [Use stdexecwhen all to join multiple senders](Use_stdexecwhen_all_to_join_multiple_senders.md)
- [Use stdthis threadsync wait to drive a sender pipeline synchronously](Use_stdthis_threadsync_wait_to_drive_a_sender_pipeline_synchronously.md)
- [Write a P2300-compatible thread pool scheduler from scratch](Write_a_P2300-compatible_thread_pool_scheduler_from_scratch.md)

## Notes

- std::execution (P2300) provides a structured concurrency model for C++26
- Senders represent lazy asynchronous work — they describe what to do, not when to do it
- Receivers consume the result of a sender (value, error, or stopped signal)
- std::execution::then chains async operations, similar to continuations
- Operation states hold the state of an in-flight async operation — they must remain stable in memory
- Schedulers represent execution contexts (thread pools, event loops, GPU queues)
- std::execution::when_all composes concurrent senders into a single awaitable
- Structured concurrency ensures child operations complete before their parent scope exits
- stdexec (from NVIDIA) is the reference implementation ahead of standardization
- The sender/receiver model replaces callbacks, futures, and coroutines for high-performance async
