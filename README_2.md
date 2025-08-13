# AutoIncrementDict — Thread-Safe Label Interning

## Why We Don't Delete Labels

Our `label_dict` only stores **labels**, which are:

- **Few in number** compared to miners/data.
- **Reused often** — the same label appears for many miners and buckets.
- **Cheap to keep** in memory since they’re small strings.
- **Harmless to keep forever** — removing them risks breaking `get_by_id()` for old `labelId`s still in the DB.

**Conclusion:**  
Deletion gives almost no memory benefit but adds complexity and race-risk, while keeping them is simple and safe.

---

## Memory Footprint for 100 Million Labels

If we ballpark for **100 million labels**:

### 1. Python String Storage
- Average label length (guess): **20 bytes** (UTF-8 text like `"r/outfits"`).
- Python string object overhead: **~49–52 bytes** on 64-bit CPython (header, internals).
- Total per label: **~70–75 bytes**.

```text
100,000,000 × ~75 bytes ≈ 7.5 GB
```

---

### 2. Index Dictionary (`_id_by_key`)
- Dict entry overhead: **~72–80 bytes per entry** (pointer array, hash table slots).
- Adds **~7–8 GB**.

---

### 3. List of Keys (`_keys`)
- Stores references to strings: **~8 bytes per entry**.
- Total: **~0.8 GB**.

---

### **Total Rough Estimate**
```text
7.5 GB (strings) +
7.5 GB (dict) +
0.8 GB (list) ≈ 15.8–16 GB
```

---

✅ **Conclusion:**  
100M labels will fit in memory if you have **≥32 GB RAM** free for Python, but it’s still big enough that you’d want to avoid unbounded growth if label churn is huge.

💡 **Tip:** A memory-optimized `AutoIncrementDict` using `array`/`numpy` for IDs and interning strings could cut this almost in half (~8 GB for 100M labels).

---

## Thread-Safe, Append-Only `AutoIncrementDict`

This implementation is:

- **Thread-safe** for concurrent inserts and lookups.
- **Append-only** — IDs are never reused, keeping reads lock-free.
- **Optimized for speed** — lock is only taken on *new* keys, existing keys are read lock-free.
- **Safe** — no risk of breaking `get_by_id()` for old IDs still in use.

### Code

```python
import threading
from typing import Any, List, Dict

class AutoIncrementDict:
    """Thread-safe, append-only interner: stable IDs, no reuse.

    Fast path (existing keys) avoids locks; slow path (new keys) uses a small lock.
    """

    __slots__ = ("_id_by_key", "_keys", "_lock")

    def __init__(self):
        self._id_by_key: Dict[Any, int] = {}
        self._keys: List[Any] = []
        self._lock = threading.Lock()

    def get_or_insert(self, key: Any) -> int:
        # Fast path: no lock if already present.
        try:
            return self._id_by_key[key]
        except KeyError:
            pass

        # Slow path: insert under lock, re-check to avoid duplicate inserts.
        with self._lock:
            idx = self._id_by_key.get(key)
            if idx is not None:
                return idx
            idx = len(self._keys)
            self._keys.append(key)
            self._id_by_key[key] = idx
            return idx

    def get_by_id(self, id: int) -> Any:
        # Lock-free read: we never remove or reorder entries.
        return self._keys[id]

    def __len__(self) -> int:
        return len(self._keys)

    def __contains__(self, key: Any) -> bool:
        return key in self._id_by_key
```

---

## Notes
- This structure is ideal for scenarios where labels are **heavily reused** and rarely (or never) deleted.
- Avoids expensive locking on reads.
- IDs remain stable for the lifetime of the process.
