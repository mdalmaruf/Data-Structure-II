# Hashing & Dictionary Implementation

---

## 1) The Dictionary / Map ADT in 60 seconds
A **dictionary** (aka **map**, **associative array**) stores key–value pairs and supports:
- `insert(k, v)` / `put(k, v)`
- `get(k)` → value or “not found”
- `remove(k)`
- Optional: `contains(k)`, `size()`, `isEmpty()`, iteration over keys/values

**Goal:** near-constant-time average performance for `put/get/remove`.

---

## 2) Why Hashing?
Arrays give O(1) access by index, but keys are not indexes. **Hashing** converts an arbitrary key into an **array index** via a **hash function**:

```
index = h(key) mod m
```

- `h(key)`: integer hash code (possibly unbounded)
- `m`: table capacity (array length)

**Design target:** Spread keys **uniformly** over `0..m-1`.

### 2.1 Hash functions (practical notes)
- **Integers:** Often `h(k) = k` or mix bits (`(k ^ (k >>> 16))`) then `mod m`.
- **Strings:** Polynomial rolling hash is standard:  
  `h(s) = (s[0]*p^(L-1) + s[1]*p^(L-2) + ... + s[L-1]) mod M`  (choose prime `p`, large `M`).
- **Compound keys:** Hash components then mix (e.g., XOR/rotate/multiply).
- **Don’t reinvent** cryptographic hashes; they’re slow and unnecessary for normal maps.
- **Stability:** Equal keys must produce equal hash codes; unequal keys should *rarely* collide.

### 2.2 Load Factor (α)
```
α = n / m  (n = number of stored entries, m = table capacity)
```
- Higher α ⇒ more collisions for open addressing; acceptable for chaining up to ~1–2.
- We typically **resize/rehash** when α exceeds a threshold, e.g., 0.5 (open addressing) or 0.75 (chaining).

---

## 3) Collision Resolution: Four Main Families
Two classic families, plus a modern twist.

### A) Separate Chaining (Linked lists / dynamic buckets)
- Each slot holds a **bucket** (usually a linked list or small vector).
- On collision, append to the bucket.
- **Find:** hash to bucket, scan bucket for key.
- **Pros:** Simple, tolerant of α > 1, deletions easy, iteration natural.
- **Cons:** Extra pointers/allocations; cache locality worse than arrays; long buckets if hashing is poor.

### B) Open Addressing (Probing)
- All entries live **in the array**. On collision, **probe** alternative indexes.
- **Linear probing:** `i, i+1, i+2, ...` (mod m).  
  Pros: great cache locality.  Cons: **primary clustering**.
- **Quadratic probing:** `i + 1^2, i + 2^2, i + 3^2, ...` (mod m).  
  Reduces clustering; requires care with `m` and load factor.
- **Double hashing:** `i + j*h2(key)` (mod m), `j=0,1,2,...` with `h2(key)` odd wrt `m`.  
  Best distribution among probes; slightly more hashing work.

### C) Robin Hood Hashing (Open addressing variant)
- On insert, if the new key has probed farther than the current occupant, **steal its spot** ("rob from the rich").
- Tends to **equalize probe lengths**; reduces variance of lookups.

### D) Cuckoo Hashing (Two tables or two hash functions)
- Each key has two candidate locations. On collision, **evict** the resident and reinsert it at its alternate spot; may trigger a chain of evictions.
- Very fast lookups (almost always 1–2 probes); occasional costly relocations/rehashes.

**Rule of thumb:**
- **Beginner-friendly & flexible:** **Chaining**.
- **Memory-tight & cache-friendly:** **Open addressing** (Double hashing or Robin Hood).
- **Ultra-fast lookups with low α:** **Cuckoo**.

---

## 4) Efficiency of Hashing (big‑O and the levers that matter)

| Operation | Chaining (avg) | Open Addressing (avg) | Notes |
|---|---|---|---|
| `get`/`contains` | O(1 + α) | ≈ O(1) if α ≤ 0.7 | Probing cost grows rapidly past α≈0.75 |
| `put` (no resize) | O(1 + α) | ≈ O(1) | Amortized O(1) with resizing |
| `remove` | O(1 + α) | ≈ O(1) + tombstone mgmt | Open addressing needs **tombstones** or backshift |
| space overhead | pointers for buckets | very compact | |

**The big levers:**
1) **Hash quality** (uniform spread)  
2) **Table size** and **prime/odd choices** (for h2 in double hashing)  
3) **Load factor policy** (when to resize)  
4) **Collision strategy** (chaining vs probing flavor)

---

## 5) Rehashing: Keeping α and speed in check
When α crosses a threshold (e.g., 0.75 for chaining, 0.5–0.7 for probing), we **allocate a larger array** (often 2× or next prime) and **reinset all entries** with the new `m`.

**Amortized analysis:** Although a rehash takes O(n), it happens rarely ⇒ average `put` is still O(1).

**Tips:**
- For **chaining**, α can exceed 1 but keep expected bucket length small (≈ α).
- For **open addressing**, keep α ≤ ~0.7 to limit probe chains and clustering.

---

## 6) A Minimal Hash Map with Separate Chaining (Python)
Below is a pedagogical map to illustrate hashing, collision handling, and rehashing.

```python
class Entry:
    __slots__ = ("key", "value")
    def __init__(self, key, value):
        self.key = key
        self.value = value

class HashMap:
    def __init__(self, initial_capacity=8, load_factor=0.75):
        self._buckets = [[] for _ in range(initial_capacity)]
        self._size = 0
        self._load_factor = load_factor

    def _index(self, key):
        return hash(key) % len(self._buckets)

    def _rehash(self, new_capacity=None):
        old_items = []
        for bucket in self._buckets:
            old_items.extend(bucket)
        new_capacity = new_capacity or (len(self._buckets) * 2)
        self._buckets = [[] for _ in range(new_capacity)]
        self._size = 0
        for e in old_items:
            self.put(e.key, e.value)

    def put(self, key, value):
        if (self._size + 1) / len(self._buckets) > self._load_factor:
            self._rehash()
        idx = self._index(key)
        bucket = self._buckets[idx]
        for e in bucket:
            if e.key == key:
                e.value = value
                return
        bucket.append(Entry(key, value))
        self._size += 1

    def get(self, key, default=None):
        idx = self._index(key)
        for e in self._buckets[idx]:
            if e.key == key:
                return e.value
        return default

    def remove(self, key):
        idx = self._index(key)
        bucket = self._buckets[idx]
        for i, e in enumerate(bucket):
            if e.key == key:
                bucket.pop(i)
                self._size -= 1
                return True
        return False

    def __len__(self):
        return self._size

    def __iter.traverse__(self):
        for bucket in self._buckets:
            for e in bucket:
                yield (e.key, e.value)
```

**Sample usage & output**
```python
m = HashMap(initial_capacity=4)
m.put("apple", 3)
m.put("banana", 5)
m.put("pear", 7)
print(m.get("banana"))        # → 5
print(len(m))                  # → 3
m.put("banana", 6)
print(m.get("banana"))        # → 6
m.remove("apple")
print(m.get("apple"))         # → None
```

**What to notice:**
- Buckets are just lists: easy to teach and debug.
- `rehash()` triggers automatically when α > 0.75.

---

## 7) Open Addressing: Linear/Quadratic/Double Hashing (Python)
A compact open‑addressing map with **double hashing** and tombstones:

```python
class _Tomb: pass
_TOMB = _Tomb()

class OAHashMap:
    def __init__(self, capacity=8, load_factor=0.5):
        self._keys = [None] * capacity
        self._vals = [None] * capacity
        self._size = 0
        self._used = 0  # includes tombstones
        self._load_factor = load_factor

    def _h1(self, key):
        return hash(key) % len(self._keys)

    def _h2(self, key):
        # must be non-zero and relatively prime to capacity
        return 1 + (hash(key) % (len(self._keys) - 2))

    def _rehash(self, new_capacity=None):
        old = [(k, v) for k, v in zip(self._keys, self._vals) if k not in (None, _TOMB)]
        new_capacity = new_capacity or (len(self._keys) * 2)
        self._keys = [None] * new_capacity
        self._vals = [None] * new_capacity
        self._size = 0
        self._used = 0
        for k, v in old:
            self.put(k, v)

    def put(self, key, value):
        if (self._used + 1) / len(self._keys) > self._load_factor:
            self._rehash()
        i = self._h1(key)
        step = self._h2(key)
        first_tomb = None
        while True:
            k = self._keys[i]
            if k is None:
                insert_at = first_tomb if first_tomb is not None else i
                self._keys[insert_at] = key
                self._vals[insert_at] = value
                self._size += 1
                if first_tomb is None:
                    self._used += 1
                return
            elif k is _TOMB:
                if first_tomb is None:
                    first_tomb = i
            elif k == key:
                self._vals[i] = value
                return
            i = (i + step) % len(self._keys)

    def get(self, key, default=None):
        i = self._h1(key)
        step = self._h2(key)
        while True:
            k = self._keys[i]
            if k is None:
                return default
            if k != _TOMB and k == key:
                return self._vals[i]
            i = (i + step) % len(self._keys)

    def remove(self, key):
        i = self._h1(key)
        step = self._h2(key)
        while True:
            k = self._keys[i]
            if k is None:
                return False
            if k != _TOMB and k == key:
                self._keys[i] = _TOMB
                self._vals[i] = None
                self._size -= 1
                return True
            i = (i + step) % len(self._keys)

    def __len__(self):
        return self._size
```

**Key ideas:**
- **Double hashing** breaks up clusters better than linear probing.
- **Tombstones** preserve probe chains after deletions.
- Keep α low (≤ ~0.5–0.7) for predictable performance.

---

## 8) Comparing Collision Schemes — Practical Considerations

### 8.1 When is chaining better?
- Very large objects/values, variable bucket growth.
- Many deletions; less bookkeeping than tombstones.
- High load factors are acceptable (α > 1), though lookups become O(α).

### 8.2 When is open addressing better?
- Memory‑tight environments: no per‑node overhead.
- Hot loops tolerate fewer pointer chases; better cache locality.
- Double hashing or Robin Hood to minimize clustering.

### 8.3 Deletion complexity
- **Chaining:** remove from list; done.
- **Open addressing:** need **tombstones** (simple but can accumulate) or **back‑shift deletion** (more complex but cleaner arrays).

### 8.4 Iteration order
- Chaining: bucket order + within‑bucket order.
- Open addressing: array order; may include tombstones.

---

## 9) Java Notes: `hashCode` & `equals` Contract
In Java, correctness relies on:
- If `a.equals(b)` then **`a.hashCode() == b.hashCode()`** must hold.
- Unequal objects *should* have different hash codes (not guaranteed but recommended).

**Custom key example:**
```java
final class Point {
  final int x, y;
  Point(int x, int y) { this.x = x; this.y = y; }
  @Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Point p)) return false;
    return x == p.x && y == p.y;
  }
  @Override public int hashCode() {
    int h = 17;
    h = 31*h + x;
    h = 31*h + y;
    return h;
  }
}
```

---

## 10) A Dictionary Using Hashing — End‑to‑End Example
**Task:** Count word frequencies in a text using our chaining map (Python version).

```python
def word_count(text):
    m = HashMap()
    for raw in text.split():
        w = ''.join(ch for ch in raw.lower() if ch.isalnum())
        if not w:
            continue
        m.put(w, (m.get(w, 0) + 1))
    # collect results (for demo only)
    out = []
    for bucket in m._buckets:
        for e in bucket:
            out.append((e.key, e.value))
    return sorted(out)

print(word_count("To be, or not to be: that is the question"))
# Sample output (order may vary before sorting):
# [('be', 2), ('is', 1), ('not', 1), ('or', 1), ('question', 1), ('that', 1), ('the', 1), ('to', 2)]
```

---

## 11) Rehashing Demonstration
```python
m = HashMap(initial_capacity=2, load_factor=0.75)
for i in range(6):
    m.put(f"k{i}", i)
    # Observe when capacity doubles: 2 → 4 → 8
```

**What happens:** As `i` grows, α exceeds 0.75, triggering `rehash()` to keep operations near O(1).

---

## 12) Design Checklist (What to reason about in interviews/assignments)
1) Choice of collision policy and why (chaining vs double hashing vs Robin Hood vs cuckoo)
2) Target load factor and resize policy
3) Hash function quality and mixing (especially for strings & composites)
4) Deletion semantics (tombstones/back‑shift vs list removal)
5) Iteration guarantees (deterministic order or not)
6) Memory overhead vs speed trade‑offs

---

## 13) Quick Exercises (with answers)
**Q1.** Why might linear probing degrade as α→0.9?  
**A.** Primary clustering causes long runs; expected probes explode.

**Q2.** For double hashing, why must `h2(key)` be relatively prime to `m`?  
**A.** To ensure the probe sequence covers all slots before repeating; otherwise some slots are unreachable.

**Q3.** In chaining with α=2.0, what is the expected bucket length and expected lookup time?  
**A.** Expected list length ≈ 2; average lookup ≈ O(1 + α) ≈ O(3) constant‑ish.

**Q4.** When should you prefer cuckoo hashing?  
**A.** When lookups dominate and you can maintain low α; you want worst‑case O(1–2) probes.

---

