+++
title="Frequently used methods"
date=2025-05-28

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++


A collection of functions frequently used when solving leetcode problems with Rust

## std
### Struct [String](https://doc.rust-lang.org/std/string/struct.String.html) in std::string module

[push](https://doc.rust-lang.org/std/string/struct.String.html#method.push)  
[push_str](https://doc.rust-lang.org/std/string/struct.String.html#method.push_str)  
and all the methods listed below in _str_ 

### Struct [Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html) in std::vec module
[push](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.push)  
[pop](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.pop)  
[slice](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.splice)  
[insert](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.insert)  
[remove](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.remove)  
[splice](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.splice)  
[dedup](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.dedup)  
and all the methods listed below in _slice_ 



### Primitive Type [str](https://doc.rust-lang.org/std/primitive.str.html)
[as_bytes](https://doc.rust-lang.org/std/primitive.str.html#method.as_bytes)  
[bytes](https://doc.rust-lang.org/std/primitive.str.html#method.bytes)  
[chars](https://doc.rust-lang.org/std/primitive.str.html#method.chars)  
[len](https://doc.rust-lang.org/std/primitive.str.html#method.len)  



### Primitive Type [Slice](https://doc.rust-lang.org/std/primitive.slice.html)
[contains](https://doc.rust-lang.org/std/primitive.slice.html#method.contains)  
[concat](https://doc.rust-lang.org/std/primitive.slice.html#method.concat)  
[join](https://doc.rust-lang.org/std/primitive.slice.html#method.join)  
[first](https://doc.rust-lang.org/std/primitive.slice.html#method.first)  
[last](https://doc.rust-lang.org/std/primitive.slice.html#method.last)  
[get](https://doc.rust-lang.org/std/primitive.slice.html#method.get)  
[len](https://doc.rust-lang.org/std/primitive.slice.html#method.len)  
[iter](https://doc.rust-lang.org/std/primitive.slice.html#method.iter)  
[iter_mut](https://doc.rust-lang.org/std/primitive.slice.html#method.iter_mut)  
[split](https://doc.rust-lang.org/std/primitive.slice.html#method.split)  
[rsplit](https://doc.rust-lang.org/std/primitive.slice.html#method.rsplit)  
[sort](https://doc.rust-lang.org/std/primitive.slice.html#method.sort)  
[sort_by](https://doc.rust-lang.org/std/primitive.slice.html#method.sort_by)  
[reverse](https://doc.rust-lang.org/std/primitive.slice.html#method.reverse)  
[windows](https://doc.rust-lang.org/std/primitive.slice.html#method.windows)  
[to_vec](https://doc.rust-lang.org/std/primitive.slice.html#method.to_vec)  
*[into_vec](https://doc.rust-lang.org/std/primitive.slice.html#method.into_vec)  




## std::collections
### Struct [VecDeque](https://doc.rust-lang.org/std/collections/struct.VecDeque.html)
[push_back](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.push_back)  
[pop_back](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.pop_back)  
[back](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.back)  
[back_mut](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.back_mut)  
[push_front](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.push_front)  
[pop_front](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.pop_front)  
[front](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.front)  
[front_mut](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.front_mut)  
[insert](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.insert)  
[get](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.get)  
[remove](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.remove)  
[append](https://doc.rust-lang.org/std/collections/struct.VecDeque.html#method.append)  


### Struct [HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html)  
[entry](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.entry)  
>[and_motify](https://doc.rust-lang.org/std/collections/hash_map/enum.Entry.html#method.and_modify)  
>[insert_entry](https://doc.rust-lang.org/std/collections/hash_map/enum.Entry.html#method.insert_entry)  
>[or_insert](https://doc.rust-lang.org/std/collections/hash_map/enum.Entry.html#method.or_insert)  

[insert](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.insert)  
[get](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.get)  
[keys](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.keys)  
[values](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.values)  
[iter](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.iter)  
[len](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.len)

### Struct [HashSet](https://doc.rust-lang.org/std/collections/struct.HashSet.html)
[insert](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.insert)  
[get](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.get)  
[get_or_insert](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.get_or_insert)  
[remove](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.remove)  
[take](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.take)  
[iter](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.iter)  
[len](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.len)  


## std::iter
### Trait [Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html)
[collect](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect)  
[collect_into](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect_into)  
[flatten](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.flatten)  
[fold](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.fold)  
[map](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map)  
[max](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.max)  
[max_by](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.max_by)  
[min](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.min)  
[min_by](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.min_by)  
[nth](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.nth)  
[reduce](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.reduce) && [fold](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.fold)  
[rev](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.rev)  
[skip](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.skip)  
[take](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.take)  
[zip](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.zip)

