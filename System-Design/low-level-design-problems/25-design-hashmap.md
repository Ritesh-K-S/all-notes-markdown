# 25 - Design HashMap

## 📋 Problem Statement

Design a HashMap from scratch that:
- Supports put(key, value), get(key), remove(key) in O(1) average
- Handles hash collisions
- Supports dynamic resizing

---

## 📌 Requirements

### Functional Requirements
1. **put(key, value)** — insert or update
2. **get(key)** — retrieve value (or null)
3. **remove(key)** — delete entry
4. **containsKey(key)** — check existence
5. **size()** — number of entries
6. Handle **collisions** via chaining (linked list)
7. **Dynamic resizing** when load factor exceeds threshold

---

## 💻 Code Implementation

### Node (Entry)

```java
public class Entry<K, V> {
    K key;
    V value;
    Entry<K, V> next;

    public Entry(K key, V value) {
        this.key = key;
        this.value = value;
        this.next = null;
    }
}
```

### HashMap Implementation

```java
public class MyHashMap<K, V> {
    private static final int DEFAULT_CAPACITY = 16;
    private static final float LOAD_FACTOR = 0.75f;

    private Entry<K, V>[] buckets;
    private int size;
    private int capacity;

    @SuppressWarnings("unchecked")
    public MyHashMap() {
        this.capacity = DEFAULT_CAPACITY;
        this.buckets = new Entry[capacity];
        this.size = 0;
    }

    // ─── Hash Function ──────────────────────────
    private int hash(K key) {
        if (key == null) return 0;
        int h = key.hashCode();
        // Spread upper bits down (like Java's HashMap)
        h ^= (h >>> 16);
        return h & (capacity - 1); // equivalent to h % capacity when capacity is power of 2
    }

    // ─── PUT ────────────────────────────────────
    public void put(K key, V value) {
        int index = hash(key);
        Entry<K, V> head = buckets[index];

        // Check if key already exists → update
        Entry<K, V> current = head;
        while (current != null) {
            if (keysEqual(current.key, key)) {
                current.value = value; // update
                return;
            }
            current = current.next;
        }

        // Key doesn't exist → insert at head of chain
        Entry<K, V> newEntry = new Entry<>(key, value);
        newEntry.next = head;
        buckets[index] = newEntry;
        size++;

        // Check if resize needed
        if ((float) size / capacity >= LOAD_FACTOR) {
            resize();
        }
    }

    // ─── GET ────────────────────────────────────
    public V get(K key) {
        int index = hash(key);
        Entry<K, V> current = buckets[index];

        while (current != null) {
            if (keysEqual(current.key, key)) {
                return current.value;
            }
            current = current.next;
        }
        return null;
    }

    // ─── REMOVE ─────────────────────────────────
    public V remove(K key) {
        int index = hash(key);
        Entry<K, V> current = buckets[index];
        Entry<K, V> prev = null;

        while (current != null) {
            if (keysEqual(current.key, key)) {
                if (prev == null) {
                    buckets[index] = current.next; // remove head
                } else {
                    prev.next = current.next;
                }
                size--;
                return current.value;
            }
            prev = current;
            current = current.next;
        }
        return null;
    }

    // ─── CONTAINS KEY ───────────────────────────
    public boolean containsKey(K key) {
        return get(key) != null;
    }

    // ─── SIZE ───────────────────────────────────
    public int size() { return size; }

    public boolean isEmpty() { return size == 0; }

    // ─── RESIZE (Rehash) ────────────────────────
    @SuppressWarnings("unchecked")
    private void resize() {
        int newCapacity = capacity * 2;
        Entry<K, V>[] newBuckets = new Entry[newCapacity];
        int oldCapacity = capacity;
        Entry<K, V>[] oldBuckets = buckets;

        this.capacity = newCapacity;
        this.buckets = newBuckets;
        this.size = 0;

        // Rehash all entries
        for (int i = 0; i < oldCapacity; i++) {
            Entry<K, V> current = oldBuckets[i];
            while (current != null) {
                put(current.key, current.value);
                current = current.next;
            }
        }
    }

    // ─── Key Equality ───────────────────────────
    private boolean keysEqual(K k1, K k2) {
        if (k1 == null && k2 == null) return true;
        if (k1 == null || k2 == null) return false;
        return k1.equals(k2);
    }

    // ─── Display (Debug) ────────────────────────
    public void display() {
        for (int i = 0; i < capacity; i++) {
            System.out.print("Bucket " + i + ": ");
            Entry<K, V> current = buckets[i];
            while (current != null) {
                System.out.print("[" + current.key + "=" + current.value + "] → ");
                current = current.next;
            }
            System.out.println("null");
        }
    }
}
```

---

## 📊 How It Works

```
Initial State (capacity = 4):
┌─────────┐
│ Bucket 0 │ → null
│ Bucket 1 │ → null
│ Bucket 2 │ → null
│ Bucket 3 │ → null
└─────────┘

After put("a", 1), put("b", 2), put("c", 3):  (assume hash("a")=0, hash("b")=1, hash("c")=2)
┌─────────┐
│ Bucket 0 │ → [a=1] → null
│ Bucket 1 │ → [b=2] → null
│ Bucket 2 │ → [c=3] → null
│ Bucket 3 │ → null
└─────────┘

Collision: put("e", 5) where hash("e")=0:
┌─────────┐
│ Bucket 0 │ → [e=5] → [a=1] → null     ← chaining!
│ Bucket 1 │ → [b=2] → null
│ Bucket 2 │ → [c=3] → null
│ Bucket 3 │ → null
└─────────┘

After resize (size/capacity >= 0.75):
- New capacity = old × 2
- Rehash ALL entries with new capacity
```

---

## 🧪 Usage Example

```java
MyHashMap<String, Integer> map = new MyHashMap<>();

map.put("apple", 1);
map.put("banana", 2);
map.put("cherry", 3);

System.out.println(map.get("banana"));     // 2
System.out.println(map.containsKey("fig")); // false

map.put("banana", 20);                     // update
System.out.println(map.get("banana"));     // 20

map.remove("cherry");
System.out.println(map.size());            // 2

// Null key support
map.put(null, 99);
System.out.println(map.get(null));         // 99
```

---

## 📊 Time & Space Complexity

| Operation | Average | Worst Case (all collisions) |
|-----------|---------|-----------------------------|
| **put** | O(1) | O(N) |
| **get** | O(1) | O(N) |
| **remove** | O(1) | O(N) |
| **resize** | O(N) | O(N) |
| **Space** | O(N) | O(N) |

---

## 🔧 Collision Resolution Techniques

| Technique | Description |
|-----------|-------------|
| **Separate Chaining** | Linked list at each bucket (used above) |
| **Open Addressing** | Find next empty slot (linear/quadratic probing) |
| **Robin Hood Hashing** | Steal from rich buckets |
| **Cuckoo Hashing** | Two hash functions, guaranteed O(1) lookup |

### Java 8+ Optimization
When chain length > 8, Java converts linked list to **Red-Black Tree** → O(log N) worst case

---

## ⚠️ Edge Cases

- **Null key** → special handling (bucket 0)
- **Null value** → allowed
- **Same key, different value** → update
- **Hash collision** → chaining resolves it
- **Large number of entries** → resize maintains O(1) average
- **Non-uniform hash function** → many collisions, degrades to O(N)

---

## 🔑 Key Takeaways

1. **Array of buckets** + **chaining** (linked lists) is the foundation
2. **Hash function** must be **uniform** — spread keys evenly
3. **Load factor** (0.75) triggers **resize** (double capacity + rehash)
4. Capacity is always **power of 2** — enables bitwise modulo
5. `hashCode()` + `equals()` contract is critical — both must be consistent
6. Java 8+ converts long chains to **balanced trees** for O(log N)
