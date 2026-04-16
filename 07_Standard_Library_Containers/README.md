# Standard Library — Containers

STL containers (vector, map, set, unordered_map, etc.), their properties, performance characteristics, and correct usage.

**Topics:** 32

## Contents

- [Implement a custom container satisfying stdstdrangesstdrange](Implement_a_custom_container_satisfying_stdstdrangesstdrange.md)
- [Know how stdunordered map handles load factor and rehashing](Know_how_stdunordered_map_handles_load_factor_and_rehashing.md)
- [Know how to implement a custom iterator satisfying stdrandom access iterator](Know_how_to_implement_a_custom_iterator_satisfying_stdrandom_access_iterator.md)
- [Know stdbitset for fixed-size bit manipulation](Know_stdbitset_for_fixed-size_bit_manipulation.md)
- [Know stddeque internals and when to prefer it over vector](Know_stddeque_internals_and_when_to_prefer_it_over_vector.md)
- [Know stdhive C++26 for stable-address high-throughput element storage](Know_stdhive_C++26_for_stable-address_high-throughput_element_storage.md)
- [Know stdinplace vector C++26 for fixed-capacity stack-allocated vectors](Know_stdinplace_vector_C++26_for_fixed-capacity_stack-allocated_vectors.md)
- [Know stdstdvector resize and overwrite-style patterns for avoiding double-initia](Know_stdstdvector_resize_and_overwrite-style_patterns_for_avoiding_double-initia.md)
- [Know the time complexity guarantees of all standard containers](Know_the_time_complexity_guarantees_of_all_standard_containers.md)
- [Know when to use stdforward list over stdlist](Know_when_to_use_stdforward_list_over_stdlist.md)
- [Understand flat map and flat set C++23 and when they beat stdmap](Understand_flat_map_and_flat_set_C++23_and_when_they_beat_stdmap.md)
- [Understand iterator invalidation rules for all standard containers](Understand_iterator_invalidation_rules_for_all_standard_containers.md)
- [Understand stdstdbasic string resize and overwrite Cpp23](Understand_stdstdbasic_string_resize_and_overwrite_Cpp23.md)
- [Understand stdvectorbools non-standard behavior](Understand_stdvectorbools_non-standard_behavior.md)
- [Understand unordered map load factor rehashing and bucket management](Understand_unordered_map_load_factor_rehashing_and_bucket_management.md)
- [Use emplace vs insert for in-place construction in all containers](Use_emplace_vs_insert_for_in-place_construction_in_all_containers.md)
- [Use heterogeneous lookup in unordered containers C++20](Use_heterogeneous_lookup_in_unordered_containers_C++20.md)
- [Use node-based containers map set with merge and extract C++17](Use_node-based_containers_map_set_with_merge_and_extract_C++17.md)
- [Use priority queue stack and queue as container adaptors](Use_priority_queue_stack_and_queue_as_container_adaptors.md)
- [Use stdarray for fixed-size stack-allocated arrays](Use_stdarray_for_fixed-size_stack-allocated_arrays.md)
- [Use stdflat set C++23 for sorted unique storage with better cache behavior](Use_stdflat_set_C++23_for_sorted_unique_storage_with_better_cache_behavior.md)
- [Use stdlist only when stable iterators are required](Use_stdlist_only_when_stable_iterators_are_required.md)
- [Use stdlistsplice for O1 node transfers between lists](Use_stdlistsplice_for_O1_node_transfers_between_lists.md)
- [Use stdmdspan layouts for stride-based memory access patterns](Use_stdmdspan_layouts_for_stride-based_memory_access_patterns.md)
- [Use stdmultimap and stdmultiset for non-unique keys](Use_stdmultimap_and_stdmultiset_for_non-unique_keys.md)
- [Use stdpmrvector for arena-allocated vectors without template parameter changes](Use_stdpmrvector_for_arena-allocated_vectors_without_template_parameter_changes.md)
- [Use stdstring view to avoid unnecessary string copies in APIs](Use_stdstring_view_to_avoid_unnecessary_string_copies_in_APIs.md)
- [Use stdstrings starts with ends with and contains C++2023](Use_stdstrings_starts_with_ends_with_and_contains_C++2023.md)
- [Use stdvector correctly reserve emplace back and iterator invalidation](Use_stdvector_correctly_reserve_emplace_back_and_iterator_invalidation.md)
- [Use transparent comparators for heterogeneous lookup in ordered containers](Use_transparent_comparators_for_heterogeneous_lookup_in_ordered_containers.md)
- [Use transparent comparators with heterogeneous lookup C++14](Use_transparent_comparators_with_heterogeneous_lookup_C++14.md)
- [Use try emplace and insert or assign C++17 for map operations](Use_try_emplace_and_insert_or_assign_C++17_for_map_operations.md)

## Notes

- std::vector should be the default container — it has the best cache locality
- std::array is stack-allocated with zero-overhead over C arrays — use for fixed sizes
- std::unordered_map is O(1) average but has high constant factor — profile before assuming it's faster
- std::flat_map (C++23) stores keys and values in sorted vectors — better cache performance for small maps
- Reserve capacity in vectors (eserve()) to avoid repeated reallocations
- emplace_back constructs in-place; push_back may copy/move — prefer emplace_back for complex types
- Iterator invalidation rules differ per container — know them to avoid undefined behavior
- std::deque is not contiguous in memory despite random access — use std::vector unless you need front insertion
- 	ry_emplace (C++17) avoids constructing the value if the key already exists
- std::span offers a non-owning view into contiguous containers without template bloat
