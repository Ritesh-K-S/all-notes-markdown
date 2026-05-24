# 19 - Cache System (LRU / LFU)

## 📋 Problem Statement

Design an in-memory cache system that:
- Supports get/put operations in O(1)
- Implements eviction policies (LRU, LFU)
- Has configurable capacity

---

## 📌 Requirements

### Functional Requirements
1. **put(key, value)** — insert or update
2. **get(key)** — retrieve value (or null if missing)
3. **Eviction** when capacity is reached
4. Support **LRU** (Least Recently Used) and **LFU** (Least Frequently Used)
5. **TTL** (Time To Live) support (optional)
6. All operations in **O(1)** time

---

## 💻 Code Implementation

### Cache Interface

```java
public interface Cache<K, V> {
    V get(K key);
    void put(K key, V value);
    void remove(K key);
    int size();
}
```

---

### LRU Cache (HashMap + Doubly Linked List)

```
Data Structure:
- HashMap<Key, Node>  → O(1) lookup
- Doubly Linked List  → O(1) add/remove, maintains access order

Most recently used → HEAD
Least recently used → TAIL (evict from here)
```

```java
public class LRUCache<K, V> implements Cache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head; // dummy head (most recent)
    private final Node<K, V> tail; // dummy tail (least recent)

    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;

        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }

    @Override
    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;

        // Move to front (most recently used)
        removeNode(node);
        addToFront(node);
        return node.value;
    }

    @Override
    public void put(K key, V value) {
        if (map.containsKey(key)) {
            // Update existing
            Node<K, V> node = map.get(key);
            node.value = value;
            removeNode(node);
            addToFront(node);
        } else {
            // Evict if at capacity
            if (map.size() >= capacity) {
                Node<K, V> lru = tail.prev; // least recently used
                removeNode(lru);
                map.remove(lru.key);
            }

            // Add new node
            Node<K, V> newNode = new Node<>(key, value);
            addToFront(newNode);
            map.put(key, newNode);
        }
    }

    @Override
    public void remove(K key) {
        Node<K, V> node = map.remove(key);
        if (node != null) removeNode(node);
    }

    @Override
    public int size() { return map.size(); }

    // ─── Helper methods ─────────────────────────
    private void addToFront(Node<K, V> node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }

    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
}
```

---

### LFU Cache (HashMap + Frequency Map + Doubly Linked List)

```
Data Structure:
- Map<Key, Node>           → O(1) lookup
- Map<Frequency, DLL>      → O(1) access to least frequent group
- minFrequency tracker     → O(1) eviction

When frequency tie → evict least recently used within that frequency
```

```java
public class LFUCache<K, V> implements Cache<K, V> {
    private final int capacity;
    private int minFrequency;
    private final Map<K, Node<K, V>> keyMap;
    private final Map<Integer, DoublyLinkedList<K, V>> frequencyMap;

    private static class Node<K, V> {
        K key;
        V value;
        int frequency;
        Node<K, V> prev, next;

        Node(K key, V value) {
            this.key = key;
            this.value = value;
            this.frequency = 1;
        }
    }

    private static class DoublyLinkedList<K, V> {
        Node<K, V> head, tail;
        int size;

        DoublyLinkedList() {
            head = new Node<>(null, null);
            tail = new Node<>(null, null);
            head.next = tail;
            tail.prev = head;
            size = 0;
        }

        void addToFront(Node<K, V> node) {
            node.next = head.next;
            node.prev = head;
            head.next.prev = node;
            head.next = node;
            size++;
        }

        void remove(Node<K, V> node) {
            node.prev.next = node.next;
            node.next.prev = node.prev;
            size--;
        }

        Node<K, V> removeLast() {
            if (size == 0) return null;
            Node<K, V> last = tail.prev;
            remove(last);
            return last;
        }

        boolean isEmpty() { return size == 0; }
    }

    public LFUCache(int capacity) {
        this.capacity = capacity;
        this.minFrequency = 0;
        this.keyMap = new HashMap<>();
        this.frequencyMap = new HashMap<>();
    }

    @Override
    public V get(K key) {
        Node<K, V> node = keyMap.get(key);
        if (node == null) return null;

        updateFrequency(node);
        return node.value;
    }

    @Override
    public void put(K key, V value) {
        if (capacity <= 0) return;

        if (keyMap.containsKey(key)) {
            Node<K, V> node = keyMap.get(key);
            node.value = value;
            updateFrequency(node);
        } else {
            if (keyMap.size() >= capacity) {
                // Evict least frequently used
                DoublyLinkedList<K, V> minFreqList = frequencyMap.get(minFrequency);
                Node<K, V> evicted = minFreqList.removeLast();
                keyMap.remove(evicted.key);
            }

            Node<K, V> newNode = new Node<>(key, value);
            keyMap.put(key, newNode);
            frequencyMap.computeIfAbsent(1, k -> new DoublyLinkedList<>()).addToFront(newNode);
            minFrequency = 1;
        }
    }

    private void updateFrequency(Node<K, V> node) {
        int oldFreq = node.frequency;
        DoublyLinkedList<K, V> oldList = frequencyMap.get(oldFreq);
        oldList.remove(node);

        // Update minFrequency if needed
        if (oldFreq == minFrequency && oldList.isEmpty()) {
            minFrequency++;
        }

        node.frequency++;
        frequencyMap.computeIfAbsent(node.frequency, k -> new DoublyLinkedList<>())
                    .addToFront(node);
    }

    @Override
    public void remove(K key) {
        Node<K, V> node = keyMap.remove(key);
        if (node != null) {
            frequencyMap.get(node.frequency).remove(node);
        }
    }

    @Override
    public int size() { return keyMap.size(); }
}
```

---

### LRU Cache with TTL

```java
public class LRUCacheWithTTL<K, V> extends LRUCache<K, V> {
    private final Map<K, Long> expiryMap;
    private final long defaultTTLMs;

    public LRUCacheWithTTL(int capacity, long defaultTTLMs) {
        super(capacity);
        this.expiryMap = new HashMap<>();
        this.defaultTTLMs = defaultTTLMs;
    }

    @Override
    public V get(K key) {
        if (isExpired(key)) {
            remove(key);
            expiryMap.remove(key);
            return null;
        }
        return super.get(key);
    }

    public void put(K key, V value, long ttlMs) {
        super.put(key, value);
        expiryMap.put(key, System.currentTimeMillis() + ttlMs);
    }

    @Override
    public void put(K key, V value) {
        put(key, value, defaultTTLMs);
    }

    private boolean isExpired(K key) {
        Long expiry = expiryMap.get(key);
        return expiry != null && System.currentTimeMillis() > expiry;
    }
}
```

---

## 🧪 Usage Example

```java
// LRU Cache
Cache<String, Integer> lru = new LRUCache<>(3);
lru.put("a", 1);
lru.put("b", 2);
lru.put("c", 3);
lru.get("a");       // access "a" → now most recent
lru.put("d", 4);    // evicts "b" (least recently used)
lru.get("b");       // → null (evicted)

// LFU Cache
Cache<String, Integer> lfu = new LFUCache<>(3);
lfu.put("a", 1);
lfu.get("a");        // freq("a") = 2
lfu.put("b", 2);
lfu.put("c", 3);
lfu.put("d", 4);     // evicts "b" or "c" (freq=1, least recent)
```

---

## 📊 Comparison

| Feature | LRU | LFU |
|---------|-----|-----|
| **Eviction** | Least recently used | Least frequently used |
| **Data Structure** | HashMap + DLL | HashMap + FreqMap + DLLs |
| **Time** | O(1) | O(1) |
| **Space** | O(N) | O(N) |
| **Best For** | Temporal locality | Frequency-based access |
| **Weakness** | Can evict frequently used items | Cache pollution from old popular items |

---

## 🔑 Key Takeaways

1. **LRU** = HashMap + Doubly Linked List → O(1) everything
2. **LFU** = HashMap + Frequency Map (freq → DLL) + minFrequency tracker
3. **Dummy head/tail** simplify edge cases in DLL
4. **TTL** adds expiration on top of eviction policy
5. Both are foundational data structures asked in DSA + LLD interviews
