# Slot Map

[![Actions Status](https://github.com/SergeyMakeev/slot_map/workflows/build/badge.svg)](https://github.com/SergeyMakeev/slot_map/actions)
[![Build status](https://ci.appveyor.com/api/projects/status/i00kv17e3ia5jr7q?svg=true)](https://ci.appveyor.com/project/SergeyMakeev/slot-map)
[![codecov](https://codecov.io/gh/SergeyMakeev/slot_map/branch/main/graph/badge.svg?token=3GRAFTRYQU)](https://codecov.io/gh/SergeyMakeev/slot_map)
![MIT](https://img.shields.io/badge/license-MIT-blue.svg)

A Slot Map is a high-performance associative container with persistent unique keys to access stored values. Upon insertion, a key is returned that can be used to later access or remove the values. Insertion, removal, and access are all guaranteed to take O(1) time (best, worst, and average case) Great for storing collections of objects that need stable, safe references but have no clear ownership.

The difference between a `std::unordered_map` and a `dod::slot_map` is that the slot map generates and returns the key when inserting a value. A key is always unique and will only refer to the value that was inserted.

  Usage example:
  ```cpp
  slot_map<std::string> strings;
  auto red = strings.emplace("Red");
  auto green = strings.emplace("Green");
  auto blue = strings.emplace("Blue");

  const std::string* val1 = strings.get(red);
  if (val1)
  {
    printf("red = '%s'\n", val1->c_str());
  }

  strings.erase(green);
  printf("%d\n", strings.has(green));
  printf("%d\n", strings.has(blue));
  ```

  Output:
  ```
  red = 'Red'
  0
  1
  ```
  
# Implementation details

The slot map container will allocate memory in pages (default page size = 4096 elements) to avoid memory spikes during growth and be able to deallocate pages that are no longer needed.
Also, the page-based memory allocator is very important since it guarantees "pointers stability"; hence, we never move values in memory.


Keys are always uses `uint64_t` type and technically typless, but we "artificially" make them typed to get a few extra compile-time checks.
i.e., the following code will produce a compiler error
```cpp
slot_map<std::string> strings;
slot_map<int> numbers;
slot_map<int>::key numKey =  numbers.emplace(3);
const std::string* value = strings.get(numKey);   //  <---- can not use slot_map<int>::key to index slot_map<std::string> !
```

When a slot is reused, its version is automatically incremented (to invalidate all existing keys that refers to the same slot).
But since we only use 16-bits for version counter, there is a possibility that the version counter will wrap around,
and a new item will get the same key as a removed item.

To mitigate this potential issue, once the version counter overflows, we disable that slot so that no new keys are returned for this slot
(this gives us a guarantee that there are no key collisions)

To prevent version overflow from happening too often, we need to ensure that we don't reuse the same slot too often.
So we do not reuse recently freed slot-indices as long as their number is below a certain threshold (`kMinFreeIndices = 64`).

Keys also can carry an extra 24 bits of information provided by a user called `tag`. That might be handy to add application-specific data to keys.

For example:
```cpp
  slot_map<std::string> strings;
  auto red = strings.emplace("Red");
  red.set_tag(12345);
  
  auto tag = red.get_tag();
  assert(tag == 12345);
```

Here is how internal key structure is look like

| Component      |  Number of bits     |
| ---------------|---------------------|
| tag            |  24                 |
| version        |  16 (0..65,535)     |
| index          |  24 (0..16,777,215) |



# API
  
  todo
  
# References
  
  Niklas Gray  
  Building a Data-Oriented Entity System (part 1), 2014  
  http://bitsquid.blogspot.com/2014/08/building-data-oriented-entity-system.html  

  Noel Llopis  
  Managing Data Relationships, 2010  
  https://gamesfromwithin.com/managing-data-relationships  

  Stefan Reinalter  
  Adventures in data-oriented design - Part 3c: External References, 2013  
  https://blog.molecular-matters.com/2013/07/24/adventures-in-data-oriented-design-part-3c-external-references/  

  Niklas Gray  
  Managing Decoupling Part 4 - The ID Lookup Table, 2011  
  https://bitsquid.blogspot.com/2011/09/managing-decoupling-part-4-id-lookup.html  

  Sander Mertens  
  Making the most of ECS identifiers, 2020  
  https://ajmmertens.medium.com/doing-a-lot-with-a-little-ecs-identifiers-25a72bd2647  

  Michele Caini  
  ECS back and forth. Part 9 - Sparse sets and EnTT, 2020  
  https://skypjack.github.io/2020-08-02-ecs-baf-part-9/  

  Andre Weissflog  
  Handles are the better pointers, 2018  
  https://floooh.github.io/2018/06/17/handles-vs-pointers.html  

  Allan Deutsch  
  C++Now 2017: "The Slot Map Data Structure", 2017  
  https://www.youtube.com/watch?v=SHaAR7XPtNU  

  Jeff Gates  
  Init, Update, Draw - Data Arrays, 2012  
  https://greysphere.tumblr.com/post/31601463396/data-arrays  
