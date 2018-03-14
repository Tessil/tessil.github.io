---
layout: post
title:  "Benchmark of major hash maps implementations"
date:   2016-08-29 11:00:00 +0100
comments: true
---

<script language="javascript" type="text/javascript" src="{{ site.url }}/js/jquery.min.js"></script>
<script language="javascript" type="text/javascript" src="{{ site.url }}/js/jquery.flot.min.js"></script>
<script language="javascript" type="text/javascript" src="{{ site.url }}/other/hash_table_benchmark_post_data.js"></script>
<script>
$(function () {
    plot_all_charts();
});
</script>

<style>
    div.chart {
        float: left;
        width: 100%;
        height: 340px;
    }
    .choices li {
        margin-left: 5%;
        display: inline-block;
        width: 45%;
    }
    div.xaxis-title {
        width: 100%;
        text-align: center;
        font-style: italic;
        font-size: small;
        color: #666;
    }
    div.chart-after-space {
        margin-bottom: 2.0em;
    }
</style>
<sup>*Updated on Oct 05, 2017.*</sup>

This benchmark compares different C++ implementations of hashmaps. The main contestants are 
[`tsl::hopscotch_map`](https://github.com/Tessil/hopscotch-map) (hopscotch hashing, v1.4), 
[`tsl::robin_map`](https://github.com/Tessil/robin-map) (linear robin hood probing, v0.1), 
[`tsl::sparse_map`](https://github.com/Tessil/sparse-map) (sparse quadratic probing, v0.1),
[`std::unordered_map`](http://en.cppreference.com/w/cpp/container/unordered_map) (chaining, libstdc++ implementation, v3.4),
[`google::dense_hash_map`](https://github.com/sparsehash/sparsehash) (quadratic probing, v2.0) and
[`QHash`](https://doc.qt.io/qt-5/qhash.html) (chaining, v4.8). We will see how they perform in a large range of operations, both in terms of speed and memory usage.

If you just want to know which hash map you should choose, you can skip to the [last section](#which-hash-map-should-i-choose) which offers some recommendations depending on your use case.

For the benchmark we will use the <http://incise.org/hash-table-benchmarks.html> benchmark but with a few modifications to fix some of its shortcomings.

* The glib, python and ruby hash maps were removed and other C++ hash maps were added.
* We now use `std::string` as key instead of `const char *` for the strings tests.
* Multiple tests were added (reads misses, reads after deletes, iteration, ...).
* We use `std::hash<T>` as hash function for all hash maps for a fair comparison.
* Compiled with `-O3 -march=native -DNDEBUG` flags (`-march=native` includes the `-mpopcnt` flag on the CPU used for the benchmark, important for some hash maps implementations).

Even though they are not on this page to avoid too much jumble on the charts, other hash maps were tested along with different max load factors (which is important to take into account when comparing two hash maps): 
[`ska::flat_hash_map`](https://github.com/skarupke/flat_hash_map) (linear robin hood probing), 
[`spp::sparse_hash_map`](https://github.com/greg7mdp/sparsepp) (sparse quadratic probing),
[`tsl::ordered_map`](https://github.com/Tessil/ordered-map) (linear robin hood probing with keys-values outside the bucket array, v0.4), 
[`boost::unordered_map`](http://www.boost.org/doc/libs/1_62_0/doc/html/unordered.html) (chaining, v1.62),
[`google::sparse_hash_map`](https://github.com/sparsehash/sparsehash) (sparse quadratic probing, v2.0),
[`emilib::HashMap`](https://github.com/emilk/emilib) (linear probing) and 
[`tsl::array_map`](https://github.com/Tessil/array-hash) (array hash table, specialized for strings, v0.3).
You can find all these **additional tests** [here]({{ site.url }}/other/hash_table_benchmark.html) (warning, the page is a bit heavy) with the possibility to easily select which hash maps you want to compare.

Note that even if the benchmark uses C++ implementations, the benchmark is also useful to compare different collision resolution strategies in hash maps (though there may be some variations due to the quality of the implementations).

The code of the benchmark can be found on [GitHub](https://github.com/Tessil/hash-table-shootout) and the raw results of the charts can be found [here]({{ site.url }}/other/hopscotch_map_benchmark_raw_results.csv).

The benchmark was compiled with Clang 5.0 and ran on Linux 4.11 x64 with an Intel i5-5200u and 8 Go of RAM. Best of five runs was taken.

## Benchmark

### Integers
For the integers tests, we use hash maps with `int64_t` as key and `int64_t` as value. The `std::hash<int64_t>` of Clang with libstdc++ used by the benchmark is an identity function (the hash of the '42' integer will return '42').




#### Random shuffle inserts: execution time (integers)
Before the test, we generate a vector with the values [0, nb_entries) and shuffle this vector. 
Then for each value k in the vector, we insert the key-value pair (k, 1) in the hash map.

<div class="chart" id="insert_random_shuffle_range_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="insert_random_shuffle_range_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Random full inserts: execution time (integers)
Before the test, we generate a vector of nb_entries size where each value is randomly taken from an uniform random number generator from all possible positive values an `int64_t` can hold.
Then for each value k in the vector, we insert the key-value pair (k, 1) in the hash map.

<div class="chart" id="insert_random_full_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="insert_random_full_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Random full inserts with reserve: execution time (integers)
Same as the random full inserts test but the reserve method of the hash map is called beforehand to avoid any rehash during the insertion. It provides a fair comparison even if the growth factor of each hash map is different.

<div class="chart" id="insert_random_full_reserve_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="insert_random_full_reserve_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Random full deletes: execution time (integers)
Before the test, we insert nb_entries elements in the same way as in the random full insert test. 
We then delete each key one by one in a different and random order than the one they were inserted.

<div class="chart" id="delete_random_full_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="delete_random_full_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Random shuffle reads: execution time (integers)
Before the test, we insert nb_entries elements in the same way as in the random shuffle inserts test. 
We then read each key-value pair in a different and random order than the one they were inserted.

<div class="chart" id="read_random_shuffle_range_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="read_random_shuffle_range_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Random full reads: execution time (integers)
Before the test, we insert nb_entries elements in the same way as in the random full inserts test. 
We then read each key-value pair in a different and random order than the one they were inserted.

<div class="chart" id="read_random_full_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="read_random_full_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Random full reads misses: execution time (integers)
Before the test, we insert nb_entries elements in the same way as in the random full inserts test.
We then generate another vector of nb_entries random elements different from the inserted elements and we try to search for these unknown elements in the hash map.

<div class="chart" id="read_miss_random_full_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="read_miss_random_full_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Random full reads after deleting half: execution time (integers)
Before the test, we insert nb_entries elements in the same way as in the random full inserts test before deleting half of these values randomly. We then try to read all the original values in a different order which will lead to 50% hits and 50% misses.

<div class="chart" id="read_random_full_after_delete_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="read_random_full_after_delete_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Random full iteration: execution time (integers)
Before the test, we insert nb_entries elements in the same way as in the random full inserts test. 
We then use the hash map iterators to read all the key-value pairs.

<div class="chart" id="iteration_random_full_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="iteration_random_full_runtime_choices"></ul>
<div class="chart-after-space"></div>




#### Memory usage of random full inserts (integers)
Before the random full inserts benchmark finishes, we measure the memory that the hash map is using.

<div class="chart" id="insert_random_full_memory"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="insert_random_full_memory_choices"></ul>
<div class="chart-after-space"></div>









### Small strings

For the small string tests, we use hash maps with `std::string` as key and `int64_t` as value. 

Each string is a random generated string of 15 alphanumeric characters (+1 for the null terminator). A generated key may look like "ju1AOoeWT3LdJxL". The generated string doesn't need any extra heap allocation as Clang 5.0 (with libstdc++) will use the [small string optimization](https://stackoverflow.com/questions/10315041/meaning-of-acronym-sso-in-the-context-of-stdstring) for any string smaller or equal to 16 characters. This allows hash maps using open addressing to potentially avoid cache-misses on strings comparisons.

The size of each `std::string` object is 32 bytes on the used compiler.

#### Inserts: execution time (small strings)
For each entry in the range  [0, nb_entries), we generate a string as key and insert it with the value 1.

<div class="chart" id="insert_small_string_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="insert_small_string_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Inserts with reserve: execution time (small strings)
Same as the inserts test but the reserve method of the hash map is called beforehand to avoid any rehash during the insertion. It provides a fair comparison even if the growth factor of each hash map is different. 

<div class="chart" id="insert_small_string_reserve_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="insert_small_string_reserve_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Deletes: execution time (small strings)
Before the test, we insert nb_entries elements in the hash map as in the inserts test. 
We then delete each key one by one in a different and random order than the one they were inserted.

<div class="chart" id="delete_small_string_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="delete_small_string_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Reads: execution time (small strings)
Before the test, we insert nb_entries elements in the hash map as in the inserts test. 
We then read each key-value pair in a different and random order than the one they were inserted.

<div class="chart" id="read_small_string_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="read_small_string_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Reads misses: execution time (small strings)
Before the test, we insert nb_entries elements in the same way as in the inserts test. We then generate nb_entries strings different from the inserted elements and we try to search for these unknown elements in the hash map.

<div class="chart" id="read_miss_small_string_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="read_miss_small_string_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Reads after deleting half: execution time (small strings)
Before the test, we insert nb_entries elements in the same way as in the inserts test before deleting half of these values randomly. We then try to read all the original values in a different order which will lead to 50% hits and 50% misses.

<div class="chart" id="read_small_string_after_delete_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="read_small_string_after_delete_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Memory usage (small strings)
Before the inserts benchmark finishes, we measure the memory that the hash map is using.

<div class="chart" id="insert_small_string_memory"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="insert_small_string_memory_choices"></ul>
<div class="chart-after-space"></div>








### Strings

For the strings tests, we use hash maps with `std::string` as key and `int64_t` as value. 

Each string is a random generated string of 50 alphanumeric characters (+1 for the null terminator). A generated key may look like "nv46iTRp7ur6UMbdgEkCHpoq7Qx7UU9Ta0u1ETdAvUb4LG6Xu6". The generated string is long enough so that Clang can't use the small string optimization and has to store it in a heap allocated area. Each string has also the same length so that each comparison will go through a trip to a heap allocated area (with its potential cache-miss).

The goal of the test is to see how the hash maps behave when comparing keys is slow.


#### Inserts: execution time (strings)
For each entry in the range [0, nb_entries), we generate a string as key and insert it with the value 1.

<div class="chart" id="insert_string_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="insert_string_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Inserts with reserve: execution time (strings)
Same as the inserts test but the reserve method of the hash map is called beforehand to avoid any rehash during the insertion. It provides a fair comparison even if the growth factor of each hash map is different.

<div class="chart" id="insert_string_reserve_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="insert_string_reserve_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Deletes: execution time (strings)
Before the test, we insert nb_entries elements in the hash map as in the inserts test. 
We then delete each key one by one in a different and random order than the one they were inserted.

<div class="chart" id="delete_string_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="delete_string_runtime_choices"></ul>
<div class="chart-after-space"></div>




#### Reads: execution time (strings)
Before the test, we insert nb_entries elements in the hash map as in the inserts test. 
We then read each key-value pair in a different and random order than the one they were inserted.

<div class="chart" id="read_string_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="read_string_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Reads misses: execution time (strings)
Before the test, we insert nb_entries elements in the same way as in the inserts test. We then generate nb_entries strings different from the inserted elements and we try to search for these unknown elements in the hash map.

<div class="chart" id="read_miss_string_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="read_miss_string_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Reads after deleting half: execution time (strings)
Before the test, we insert nb_entries elements in the same way as in the inserts test before deleting half of these values randomly. We then try to read all the original values in a different order which will lead to 50% hits and 50% misses.

<div class="chart" id="read_string_after_delete_runtime"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="read_string_after_delete_runtime_choices"></ul>
<div class="chart-after-space"></div>



#### Memory usage (strings)
Before the inserts benchmark finishes, we measure the memory that the hash map is using.

<div class="chart" id="insert_string_memory"></div>
<div class="xaxis-title">number of entries in hash table</div>
<ul class="choices" id="insert_string_memory_choices"></ul>
<div class="chart-after-space"></div>







## Analysis

We can see that the hash maps using open addressing provide an advantageous alternative to chaining due to there cache-friendliness. On the integers and small strings read tests, most of them are able to find the key while only loading one or two cache lines which make a significant difference. On insert, they can also avoid a lot of allocations compared to hash maps using chaining which have to allocate the memory for a node at each insert (a custom allocator could improve things).

In the strings tests, we can see that storing the hash alongside the values can offer a huge boost on insertions, as we don't have to recalculate the hash on rehashes, and on lookups, as we only compare two strings when the stored hashes are equal avoiding expensive comparisons. Note that `tsl::robin_map` automatically stores the hash and uses it on rehashes (but not on lookups without an explicit `StoreHash`) if it can detect that it will not take more memory to do so due to alignment. It explains why the strings inserts test is so much faster even without the `StoreHash` parameter.

Regarding the load factor, most open addressing schemes get bad results when the load factor is higher than 0.5, even with robin hood probing (see the [additional tests]({{ site.url }}/other/hash_table_benchmark.html)). Only `tsl::hopscotch_map` is able to cope well with a high load factor like 0.9 without loosing too much in lookup speed offering a really good compromise between speed and memory usage.

Regarding the memory usage, `tsl::sparse_map` beats `google::sparse_hash_map` in every speed test for the price of a little memory increase. And even if it is a bit slow on inserts, it offers an impressive balance between memory usage and lookup speed.


In the benchmark, we are using the Clang implementation of `std::hash` as hash function in all our tests. This implementation of the hash just uses the identity function, some other hash functions may give better results on some hash maps implementations (notably `emilib::HashMap` and `google::sparse_hash_map` which have terrible results on the random shuffle integers inserts test). A more robust hash function could be tested. A poor hash function could be tested as well to check how each hash map is able to cope with a bad hash distribution.

The benchmark was exclusively oriented toward hash maps. Better structures like tries could be used to map strings to values, but `std::string` is a familiar example to test bigger keys than `int64_t` and may incur a cache-miss on comparison if big enough due to its memory indirection.

In conclusion, even though `std::unordered_map` is a good implementation, it may be worth to check the alternatives if you need better performances or if your hash map is using too much memory.


### Which hash map should I choose?

Each hash map has its advantages and inconveniences so it may be difficult to pick-up the right one. Here are some general recommendations depending on your use case.

**By default.** Before choosing a hash map, just try out `std::unordered_map`. Even though it is not the fastest hash map out there due to the cache-unfriendliness of chaining, the standard hash map just works well in most cases. External libraries are an extra maintenance cost and if you are not doing a whole lot of operations on the hash map, `std::unordered_map` will do just fine.

**For speed efficiency.** A hash map using an open addressing scheme should be your choice and I would recommend either hopscotch hashing with `tsl::hopscotch_map` or linear robin hood hashing with `tsl::robin_map` or `ska::flat_hash_map`.

Both have quite similar lookup speed at low load factor but `tsl::hopscotch_map` has the main advantage of being able to cope much better with a high load factor (> 0.6) providing a better compromise between speed and memory usage.

The main drawback of hopscotch hashing is that it can suffer quite a bit of clustering in the neighborhood of a bucket which may cause extensive rehashes. When storing the hash with the `StoreHash` template parameter, it also needs to reduce the size of the neighborhood which may deepens the previous problem. But this should not be a problem with a good hash function.

On the other hand, `tsl::robin_map` can store the hash at no extra cost in most cases and will automatically do so when these cases are detected to speed up the rehash process. As the map only need a few bytes in the bucket for bookkeeping, it uses the rest of the space left due to memory alignment to store part of the hash. The `tsl::robin_map` also offers a faster insertion speed than `tsl::hopscotch_map` and is able to cope better with a poor hash function.

Quadratic probing with `google::dense_hash_map` may also be a good candidate but can't cope well with a high load factor thus needing more memory. It also do quite poorly on reads misses. Linear probing with `emilib::HashMap` suffers from the same problems.

So in the end I would recommend to try out `tsl::hopscotch_map` or `tsl::robin_map` (with a preference for `tsl::hopscotch_map` as it uses less memory) and see which one work the best for your use case.

**For memory efficiency.** If you are storing small objects (< 32 bytes) with a trivial key comparator, `tsl::sparse_map` should be your go to hash map. Even though it is quite slow on insertions, it offers a good balance between lookup speed and memory usage, even at low load factor. It is also faster than both `google::sparse_hash_map` and `spp::sparse_hash_map` while providing more functionalities.

When dealing with larger objects with a non-trivial key comparator, `tsl::sparse_map` will do fine too, but you may also want to try `tsl::ordered_map` even if you don't need the order of insertion to be kept. It can grow the map quite fast as it never needs to move the keys-values outside of deletions and provides good performances on lookups while keeping a low memory usage. For smaller objects with a trivial key comparator, it is only as good as `std::unordered_map` for lookups.

**For strings as key.** If you are using strings as key, the above recommendations still hold true but you may also want to try `tsl::array_map`. It offers one of the best lookup speed on large strings while having the lowest memory usage. The main drawback is that the rehash process is slow and will need some spare memory to copy the strings from the old map to the new map (it can't use `std::move` as the other hash maps using `std::string` as key). But if you know the number of items beforehand, you can call the `reserve` function to avoid the problem.

If you need an even more compact way to store the strings, you may also consider a trie, notably [`tsl::htrie_map`](https://github.com/Tessil/hat-trie), even if you don't need to do any prefix search. The HAT-trie provides a really memory efficient way to store the strings without losing too much on lookup speed.

**For large objects.** When dealing with large objects which take time to copy or move around, using open addressing is not a good idea. On insertion the values may have to be moved around either because it is part of the insertion process (hopscotch hashing, robin hood hashing, cuckoo hashing, ...) or due to a rehash. Best to stick to `std::unordered_map` which can just moves pointers to nodes around or eventually `tsl::ordered_map` which only needs to move one element on deletion.


In the end these are some basic advices based on a benchmark using some artificial use cases with a specific compiler. The best is still to pick-up some candidates and test them with your code in your environment.

