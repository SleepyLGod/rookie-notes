# 🧐 Bloom Filter

* A space-efficient probabilistic data structure used to check whether an element belongs to a set.
* For a membership query, a Bloom Filter returns one of two results: the element may exist, or it definitely does not exist.

When checking whether an element belongs to a set, a Bloom Filter may treat an element that is not in the set as if it were in the set. This is called a **false positive**. Therefore, Bloom Filters are not suitable for applications that require zero errors. In applications that can tolerate a low error rate, Bloom Filters trade a small number of mistakes for a large reduction in storage space.

## **BitMap & Bloom Filter**

If we only need to know whether an item exists, a bitmap can reduce memory overhead.

For example, suppose we need to determine whether a web page URL, which may occupy many bytes, is in a blacklist.

Each map value uses only 1 bit, which greatly reduces memory overhead. The concrete method is to use a hash function to map the URL into a bit array of size `n`, and then set the corresponding bit to `true`.

![](https://s2.loli.net/2022/07/24/5YhNiEVmXLu2Qvs.png)

This can determine whether the URL is in the blacklist in **O(1)** time with very low memory overhead.

However, one unavoidable problem is hash collision. Even with a good hash function, collisions can still happen. During query, a hash collision means **an incorrect query result**. To reduce collisions as much as possible, the bitmap needs to be much larger than the number of blacklisted URLs. Under a random-hashing assumption, the probability of collision during query is:

```text
number of blacklisted URLs / bitmap size
```

Therefore, higher query accuracy also brings higher memory overhead. A data structure that can improve this tradeoff effectively is the **Bloom Filter**.

A Bloom Filter uses **multiple** hash functions to hash the data. In the example below, two hash functions are used, so multiple positions are set to `true`. Compared with a bitmap, this reduces the probability of query failure as much as possible within limited space. From an information-entropy perspective, each position now carries more information, so the Bloom Filter uses space more efficiently than a plain bitmap.

![image-20210715164606827](https://s2.loli.net/2022/07/24/fPCbeqjgmAErS4V.png)

## **Set Representation and Element Query**

Now look more concretely at how a Bloom Filter represents a set with a bit array. Initially, a Bloom Filter is a bit array with `m` bits, and every bit is set to `0`.

![img](https://s2.loli.net/2022/07/12/pJo4ygWAaXqTvxR.jpg)

To represent a set `S={x1, x2, ..., xn}` containing `n` elements, a Bloom Filter uses `k` independent hash functions. Each hash function maps every element in the set into the range `{1, ..., m}`. For any element `x`, the position mapped by the `i`-th hash function, `hi(x)`, is set to `1` (`1 <= i <= k`). Note that if a position is set to `1` multiple times, only the **first** write has an effect; later writes do not change anything. In the figure below, _k=3_, and two hash functions select the same position, the fifth bit from the left.

![img](https://s2.loli.net/2022/07/24/NWEUT5amSiZdc9f.jpg)

When **inserting** an element, the Bloom Filter applies `k` hash functions to compute `k` positions in the bit array, and then sets all those bits to `1`.

When **checking** whether `y` belongs to the set, we apply the `k` hash functions to `y`. If **all positions `hi(y)` are `1` (`1 <= i <= k`)**, we consider `y` to be in the set. Otherwise, we consider `y` not to be in the set. In the figure below, `y1` is not in the set. **`y2` may belong to the set, or it may just be a false positive.**

![img](https://s2.loli.net/2022/07/24/7Cld2BObSRMIT5r.jpg)

Deletion is not allowed in a standard Bloom Filter. If we delete bits, an element that originally existed may later be reported as not existing, which is not acceptable.

The no-deletion property means invalid elements may accumulate over time. For example, an element may have already been deleted from disk, while the `bloomfilter` still reports that it may exist. This creates more and more false positives. In practice, systems often discard the old `BloomFilter` and rebuild a new one.

What if we need a Bloom Filter that supports deletion?

One option is the **Counting Bloom Filter**.

It replaces the bit array with an array of counters. Each original bit is expanded into a counter, so the structure pays several times more storage space in exchange for delete support.

There is also a better solution: the Cuckoo Filter.

## **Comparison**

Common data structures such as `hashmap`, `set`, and `bit array` can all be used to test whether an element exists in a set.

* For a `hashmap`, the underlying structure is essentially an **array of pointers**. One pointer costs `sizeof(void *)`; on a 64-bit system, that is **64 bits**. If separate chaining is used to handle collisions, extra pointer overhead is also needed. For a `BloomFilter`, if a 1% error rate is allowed when returning "may exist", each element needs roughly 10 bits of storage. The overall **storage-space cost** is about 15% of a `hashmap`.
* For a `set`:
  * If it is implemented with a `hashmap`, the situation is the same as above.
  * If it is implemented with a **balanced tree**, **each node needs one pointer to store the data position and two pointers to point to its children**, so its overhead is larger than a `hashmap`.
* For a `bit array`, to test whether an element exists, we first hash the element and take the modulo to locate a specific bit. If the bit is `1`, the element is reported as existing; if the bit is `0`, the element is reported as not existing. We can see that when a bit array reports that an element exists, it can also produce a **false positive**. To obtain the same false-positive rate as a Bloom Filter, a bit array needs **more storage space** than the Bloom Filter.

**Disadvantages**:

* Compared with `hashmap` and `set`, a `BloomFilter` has a false-positive rate when it returns "may exist". In that case, the caller may do unnecessary work because of the false positive. **For `hashmap` and `set`, there is no false-positive case.**
* Compared with a `bit array`, a `BloomFilter` needs to compute multiple hashes during insertion and lookup, while a bit array only needs one hash. In fact, a **bit array can be viewed as a special case of a `BloomFilter`**.

Use one concrete example to describe when to use a Bloom Filter, and what its advantages and disadvantages are in that scenario.

A group of elements is stored on disk. The data volume is very large, and the application wants to avoid reading from disk whenever the element does not exist.

* At this point, we can build a Bloom Filter in memory for the data on disk. For one read request, the cases are:
  * The requested element is not on disk. If the Bloom Filter returns "does not exist", the application does not need to execute the disk-read path. Let this probability be `P1`. **If the Bloom Filter returns "may exist", this is a false positive. Let this probability be `P2`.**
  * The requested element is on disk, and the Bloom Filter returns "exists". Let this probability be `P3`.
* If a `hashmap` or `set` is used instead, the cases are:
  * The requested data is not on disk, so the application does not execute the disk-read path. This probability is `P1 + P2`.
  * The requested element is on disk, so the application executes the disk-read path. This probability is `P3`.

Assume the cost of not reading from disk is `C1`, and the cost of reading from disk is `C2`. Then the costs of a Bloom Filter and a hashmap are:

```apl
Cost(BloomFilter) = P1 * C1 + (P2 + P3) * C2
Cost(HashMap) = (P1 + P2) * C1 + P3 * C2;

Delta = Cost(BloomFilter) - Cost(HashMap)
      = P2 * (C2 - C1)
```

Therefore, a Bloom Filter uses an additional time cost of `P2 * (C2 - C1)` to obtain lower space overhead than a hashmap.

Since **P2 is the main factor affecting Bloom Filter performance overhead**, how should a Bloom Filter be designed to reduce `P2`, namely the false-positive probability?

## **Parameter Selection**

In practice, applications care about false positives because they are related to extra cost. Usually, the application expects to specify a false-positive probability and the number of elements that will be inserted.

Using the same notation as above, `m`, `n`, and `k`, let the false-positive probability be `p`.

**=> :**

If we need to **minimize** the false-positive probability, the value of `k` should be as follows (①):

```apl
k = m * ln2 / n;
```

The value of **p** has the following relationship with `m` and `n` (②):

```apl
m = - n * lnp / (ln2) ^ 2
```

Substituting ① into ②, we get the value of `k` for a given `n` and `p`:

```apl
k = -lnp / ln2
```

Finally, `m` can also be computed in the same way.

## BloomFilter Implementation and Optimization

### **Basic version**

DS:

```cpp
template <typename T>
class BloomFilter {
    public:
    BloomFilter(const int32_t n, const double false_positive_p);
    void insert(const T &key);
    bool key_may_match(const T &key);

    private:
    std::vector<char> bits_; // Simulate a bit array.
    int32_t k_;
    int32_t m_;
    int32_t n_;
    double p_;
}

// init:
template <typename T>
BoomFilter<T>::BoomFilter(const int32_t n, const double false_positive_p)
    : bits_(), k_(0), m(0), n_(n), p_(false_positive_p) {
        k_ = static_cast<int32_t>(-std::log(p_) / std::log(2));
        m_ = static_cast<int32_t>(k_ * n * 1.0 / std::log(2));
        bits_.resize((m_ + 7) / 8, 0);
}

// insert:
// Set each bit computed by the hash functions to 1.
template<typename T>
void BloomFilter<T>::insert(const T &key) {
  uint32_t hash_val = 0xbc9f1d34;
  for (int i = 0; i < k_; ++i) {
    hash_val = key.hash(hash_val);
    const uint32_t bit_pos = hash_val % m_;
    bits_[bit_pos/8] |= 1 << (bit_pos % 8);
  }
}

// check
// Check the bit corresponding to each hash function. If all are 1, return "exists"; otherwise, return "does not exist".
template<typename T>
bool BloomFilter<T>::key_may_match(const T &key) {
  uint32_t hash_val = 0xbc9f1d34;
  for (int i = 0; i < k_; ++i) {
    hash_val = key.hash(hash_val);
    const uint32_t bit_pos = hash_val % m_;
    if ((bits_[bit_pos / 8] & (1 << (bit_pos % 8))) == 0) {
      return false;
    }
  }
  return true;
}
```

**A BloomFilter contains three operations:**

* Initialization: the constructor in the code above.
* Insertion: `insert` in the code above.
* Existence check: `key_may_match` in the code above.

### **Optimization**

The implementation calls `hash_func` multiple times, which is expensive when hashing long strings.

The implementation calls `hash_func` multiple times, which is expensive when hashing long strings.

We can use **two hash computations** instead of the repeated hash computations above.

For example, `insert_optimized`:

```cpp
template<typename T>
void BloomFilter<T>::insert2(const T &key) {
    uint32_t hash_val = key.hash(0xbc9f1d34); // 1st
    const uint32_t delta = (hash_val >> 17) | (hash_val << 15); // 2nd
    for (int i = 0; i < k; ++i) {
        const uint32_t bit_pos = hash_val % m_;
        bits_[bit_pos/8] |= 1 << (bit_pos % 8);
        hash_val += delta;
    }
}
```

First, compute the normal hash once. Then compute another value with bit shifts. Finally, during the `k` iterations, keep accumulating the two results.

After optimization, the maximum false-positive probability may increase. This can be compensated for by increasing `k`, because the optimized hash algorithm has relatively low overhead when `k` grows.

> P.S.:
>
> `int_t` is a structural type marker. It can be understood as an abbreviation for `type`/`typedef`, meaning that the type is defined through `typedef` rather than being a new data type. Because word length differs across platforms, using [preprocessing](https://so.csdn.net/so/search?q=%E9%A2%84%E7%BC%96%E8%AF%91\&spm=1001.2101.3001.7020) and `typedef` is an effective way to maintain cross-platform code.
>
> * `int8_t` : typedef signed char;
> * `uint8_t` : typedef unsigned char;
> * `int16_t` : typedef signed short ;
> * `uint16_t` : typedef unsigned short ;
> * `int32_t` : typedef signed int;
> * `uint32_t` : typedef unsigned int;
> * `int64_t` : typedef signed long long;
> * `uint64_t` : typedef unsigned long long;
>
> **`size_t` and `ssize_t`**
>
> `size_t` is mainly used for counting. For example, the return type of `sizeof()` is `size_t`. Its width also differs across machines.
>
> `size_t` is unsigned, while `ssize_t` is signed.
>
> On a 32-bit machine, it is defined as: `typedef unsigned int size_t;` (4 bytes). On a 64-bit machine, it is defined as: `typedef unsigned long size_t;` (8 bytes).
>
> Since `size_t` is unsigned, **when a variable may be negative, `ssize_t` must be used**.
>
> This is because when a signed integer and an unsigned integer participate in an operation, the signed integer is first automatically converted to an unsigned integer.
>
> **Four type-conversion operators**

| Keyword            | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `static_cast`      | Used for **benign** conversions. It generally does not cause surprises and has low risk. Examples include existing implicit conversions, conversion between `void` pointers and concrete typed pointers, and conversions between classes with converting constructors or conversion functions and other types. However, **`static_cast` cannot be used for conversions between unrelated types**, such as conversions between two concrete pointer types or between `int` and a pointer. If the conversion is invalid, compilation fails.                                                                                                                                                 |
| `const_cast`       | Used for conversions between **const and non-const**, or between **volatile and non-volatile**.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `reinterpret_cast` | A highly dangerous conversion. It only reinterprets binary bits and does not adjust the data using existing conversion rules, but it provides the most flexible C++ type conversion. It can be considered a supplement to `static_cast`. Some conversions that `static_cast` cannot perform can be done with `reinterpret_cast`, such as **conversions between two concrete pointer types or between `int` and a pointer**. Some compilers allow converting an `int` to a pointer but not the reverse.                                                                                                                                                                            |
| `dynamic_cast`     | Uses `RTTI` for type-safe downcasting. It is the counterpart of `static_cast`: `dynamic_cast` means "dynamic conversion", while `static_cast` means "static conversion". `dynamic_cast` performs type conversion at runtime with `RTTI`, which requires the base class to contain virtual functions. Upcasting is safe. Downcasting is unsafe without a runtime check: each class stores type information in memory, and the compiler links type information for classes with inheritance relationships into an inheritance chain. When `dynamic_cast` converts a pointer, the program first finds the object pointed to by the pointer, then finds the current class type information of that object, and traverses upward along the inheritance chain. If the target type is found, the conversion is safe and succeeds. If the target type is not found before reaching the top of the inheritance chain, the conversion is risky and fails. |

> The syntax of these four keywords is the same:
>
> `xxx_cast<newType>(data)`
>
> `newType` is the target type, and `data` is the data being converted.
