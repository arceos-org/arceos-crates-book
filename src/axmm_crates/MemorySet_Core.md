# MemorySet Core

> **Relevant source files**
> * [memory_set/src/set.rs](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_set/src/set.rs)

This document covers the `MemorySet` struct, which serves as the central container for managing collections of memory areas in the memory_set crate. The `MemorySet` provides high-level memory mapping operations similar to Unix mmap/munmap, including sophisticated area management with automatic splitting, merging, and overlap detection.

For details about individual memory areas, see [MemoryArea](/arceos-org/axmm_crates/3.2-memoryarea). For information about the backend abstraction layer, see [MappingBackend](/arceos-org/axmm_crates/3.3-mappingbackend). For practical usage examples, see [Usage Examples and Testing](/arceos-org/axmm_crates/3.4-usage-examples-and-testing).

## Core Data Structure

The `MemorySet<B: MappingBackend>` struct maintains an ordered collection of non-overlapping memory areas using a `BTreeMap` for efficient range-based operations.

### Internal Organization

```mermaid
classDiagram
class MemorySet {
    -BTreeMap~B_Addr_MemoryArea~ areas
    +new() MemorySet
    +len() usize
    +is_empty() bool
    +iter() Iterator
    +overlaps(range) bool
    +find(addr) Option_MemoryArea_
    +find_free_area(hint, size, limit) Option_B_Addr_
    +map(area, page_table, unmap_overlap) MappingResult
    +unmap(start, size, page_table) MappingResult
    +protect(start, size, update_flags, page_table) MappingResult
    +clear(page_table) MappingResult
}

class BTreeMap~B::Addr, MemoryArea~B~~ {
    
    +range(range) Iterator
    +insert(key, value) Option_MemoryArea_
    +remove(key) Option_MemoryArea_
    +retain(predicate) void
}

class MemoryArea {
    
    +start() B_Addr
    +end() B_Addr
    +size() usize
    +va_range() AddrRange
    +map_area(page_table) MappingResult
    +unmap_area(page_table) MappingResult
    +split(at) Option_MemoryArea_
    +shrink_left(new_size, page_table) MappingResult
    +shrink_right(new_size, page_table) MappingResult
}

MemorySet  *--  BTreeMap : contains
BTreeMap  -->  MemoryArea : stores
```

The `BTreeMap` key is the start address of each memory area, enabling efficient range queries and maintaining areas in sorted order. This design allows O(log n) lookup operations and efficient iteration over address ranges.

**Sources:** [memory_set/src/set.rs(L1 - L36)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_set/src/set.rs#L1-L36)

## Memory Mapping Operations

### Map Operation

The `map` method adds new memory areas to the set, with sophisticated overlap handling:

```mermaid
flowchart TD
start["map(area, page_table, unmap_overlap)"]
validate["Validate area range not empty"]
check_overlap["Check overlaps(area.va_range())"]
has_overlap["Overlaps exist?"]
check_unmap_flag["unmap_overlap = true?"]
do_unmap["Call unmap() on overlap range"]
return_error["Return MappingError::AlreadyExists"]
map_area["Call area.map_area(page_table)"]
insert_area["Insert area into BTreeMap"]
success["Return Ok(())"]
end_error["End with error"]
end_success["End successfully"]

check_overlap --> has_overlap
check_unmap_flag --> do_unmap
check_unmap_flag --> return_error
do_unmap --> map_area
has_overlap --> check_unmap_flag
has_overlap --> map_area
insert_area --> success
map_area --> insert_area
return_error --> end_error
start --> validate
success --> end_success
validate --> check_overlap
```

The overlap detection uses the `BTreeMap`'s range capabilities to efficiently check both preceding and following areas without scanning the entire collection.

**Sources:** [memory_set/src/set.rs(L93 - L122)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_set/src/set.rs#L93-L122) [memory_set/src/set.rs(L38 - L51)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_set/src/set.rs#L38-L51)

### Unmap Operation

The `unmap` operation is the most complex, handling area removal, shrinking, and splitting:

```mermaid
flowchart TD
start["unmap(start, size, page_table)"]
create_range["Create AddrRange from start/size"]
validate_range["Validate range not empty"]
remove_contained["Remove areas fully contained in range"]
check_left["Check area intersecting left boundary"]
left_intersect["Area extends past start?"]
left_middle["Unmapped range in middle?"]
split_left["Split area, shrink left part"]
shrink_left["Shrink area from right"]
check_right["Check area intersecting right boundary"]
right_intersect["Area starts before end?"]
shrink_right["Remove and shrink area from left"]
complete["Operation complete"]
success["Return Ok(())"]

check_left --> left_intersect
check_right --> right_intersect
complete --> success
create_range --> validate_range
left_intersect --> check_right
left_intersect --> left_middle
left_middle --> shrink_left
left_middle --> split_left
remove_contained --> check_left
right_intersect --> complete
right_intersect --> shrink_right
shrink_left --> check_right
shrink_right --> complete
split_left --> check_right
start --> create_range
validate_range --> remove_contained
```

The unmap operation maintains three key invariants:

1. All areas remain non-overlapping
2. Area boundaries align with page boundaries
3. The BTreeMap remains sorted by start address

**Sources:** [memory_set/src/set.rs(L124 - L184)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_set/src/set.rs#L124-L184)

## Search and Query Operations

### Address Lookup

The `find` method locates the memory area containing a specific address:

```mermaid
flowchart TD
addr["Target Address"]
range_query["BTreeMap.range(..=addr).last()"]
candidate["Get last area with start ≤ addr"]
contains_check["Check if area.va_range().contains(addr)"]
result["Return Option<&MemoryArea>"]

addr --> range_query
candidate --> contains_check
contains_check --> result
range_query --> candidate
```

This leverages the BTreeMap's ordered structure to find the candidate area in O(log n) time, then performs a simple range check.

**Sources:** [memory_set/src/set.rs(L53 - L57)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_set/src/set.rs#L53-L57)

### Free Space Detection

The `find_free_area` method implements a gap-finding algorithm:

```mermaid
flowchart TD
start["find_free_area(hint, size, limit)"]
init_last_end["last_end = max(hint, limit.start)"]
check_before["Check area before last_end"]
adjust_last_end["Adjust last_end to area.end() if needed"]
iterate["Iterate areas from last_end"]
check_gap["Gap size ≥ requested size?"]
return_address["Return last_end"]
next_area["last_end = area.end(), continue"]
no_more["No more areas?"]
check_final_gap["Final gap ≥ size?"]
return_final["Return last_end"]
return_none["Return None"]
end_success["End with address"]
end_failure["End with None"]

adjust_last_end --> iterate
check_before --> adjust_last_end
check_final_gap --> return_final
check_final_gap --> return_none
check_gap --> next_area
check_gap --> return_address
init_last_end --> check_before
iterate --> check_gap
iterate --> no_more
next_area --> iterate
no_more --> check_final_gap
return_address --> end_success
return_final --> end_success
return_none --> end_failure
start --> init_last_end
```

The algorithm performs a single linear scan through areas in address order, making it efficient for typical usage patterns.

**Sources:** [memory_set/src/set.rs(L59 - L91)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_set/src/set.rs#L59-L91)

## Permission Management

### Protect Operation

The `protect` method changes memory permissions within a specified range, involving complex area manipulation:

```mermaid
flowchart TD
start["protect(start, size, update_flags, page_table)"]
iterate["Iterate through all areas"]
check_flags["Call update_flags(area.flags())"]
has_new_flags["New flags returned?"]
skip["Skip this area"]
check_position["Determine area position vs range"]
fully_contained["Area fully in range"]
straddles_both["Area straddles both boundaries"]
left_boundary["Area intersects left boundary"]
right_boundary["Area intersects right boundary"]
simple_protect["area.protect_area(), set_flags()"]
triple_split["Split into left|middle|right parts"]
right_split["Split area, protect right part"]
left_split["Split area, protect left part"]
continue_loop["Continue to next area"]
insert_new_areas["Queue new areas for insertion"]
all_done["All areas processed?"]
extend_areas["Insert all queued areas"]
success["Return Ok(())"]

all_done --> extend_areas
all_done --> iterate
check_flags --> has_new_flags
check_position --> fully_contained
check_position --> left_boundary
check_position --> right_boundary
check_position --> straddles_both
continue_loop --> all_done
extend_areas --> success
fully_contained --> simple_protect
has_new_flags --> check_position
has_new_flags --> skip
insert_new_areas --> continue_loop
iterate --> check_flags
left_boundary --> right_split
left_split --> insert_new_areas
right_boundary --> left_split
right_split --> insert_new_areas
simple_protect --> continue_loop
skip --> continue_loop
start --> iterate
straddles_both --> triple_split
triple_split --> insert_new_areas
```

The protect operation must handle five distinct cases based on how the protection range aligns with existing area boundaries, potentially creating new areas through splitting operations.

**Sources:** [memory_set/src/set.rs(L195 - L264)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_set/src/set.rs#L195-L264)

## Area Management Properties

The `MemorySet` maintains several critical invariants:

|Property|Implementation|Verification|
| --- | --- | --- |
|Non-overlapping Areas|Overlap detection inmap(), careful splitting inunmap()/protect()|overlaps()method, assertions in insertion|
|Sorted Order|BTreeMap automatically maintains order by start address|BTreeMap invariants|
|Consistent Mappings|All operations call correspondingMemoryAreamethods|Backend abstraction layer|
|Boundary Alignment|Size validation and address arithmetic|AddrRangetype safety|

The design ensures that operations can be composed safely - for example, mapping over an existing area with `unmap_overlap=true` will cleanly remove conflicts before establishing the new mapping.

**Sources:** [memory_set/src/set.rs(L101 - L121)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_set/src/set.rs#L101-L121) [memory_set/src/set.rs(L145 - L152)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_set/src/set.rs#L145-L152)