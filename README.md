# Java Collections: Never Confuse the Methods Again

## The Golden Rule (memorize this first)

Java doesn't pick verbs randomly. Each family of data structure has **one** verb for "add stuff in" because of *what kind of thing* you're adding.

| Verb | Used by | Why this verb |
|---|---|---|
| **add()** | List, Set, Queue (basic) | You're adding ONE element to a group |
| **put()** | Map | You're placing a **key → value pair** (put the value *at* the key) |
| **push()** | Stack, Deque | You're pushing onto a pile (top of stack) |
| **offer()** | Queue, Deque | You're "offering" an element — used when the structure might *reject* it (e.g. bounded queue), so it returns `false` instead of throwing |
| **insert()** | ❌ Not a real Java method | Java never uses `insert()`. (This word exists in SQL/JS/other languages — that's likely where your confusion comes from.) |
| **delete()** | ❌ Not a real Java method | Java never uses `delete()` either. Java always uses **remove()**. |

**So: "insert" and "delete" are NOT Java Collection method names at all — that's the #1 source of your confusion. Java only ever uses `add`/`put`/`push` to insert, and `remove` (never `delete`) to take out.**

---

## 1. List (ArrayList, LinkedList) — an ordered, index-based bucket

Think of a List like a **row of numbered lockers**. You mostly work with positions (index) or the object itself.

| Method | What it does | Memory trick |
|---|---|---|
| `add(x)` | appends x to the end | "add to the pile" |
| `add(i, x)` | inserts x at index i | "add, but AT this spot" |
| `get(i)` | returns element at index i | "get = read only" |
| `set(i, x)` | replaces element at index i with x | "set = overwrite" |
| `remove(i)` | removes element AT index i (int arg) | index in → element out |
| `remove(Object o)` | removes the first occurrence of o | object in → object out |
| `contains(x)` | true/false if x exists | — |
| `indexOf(x)` | position of x, or -1 | — |
| `size()` | number of elements | — |
| `isEmpty()` | true if size == 0 | — |
| `clear()` | wipes everything | — |

⚠️ **Classic trap:** `list.remove(2)` on a `List<Integer>` removes the element **at index 2**, not the value 2! To remove the *value* 2, you must write `list.remove(Integer.valueOf(2))` — because `remove(int)` and `remove(Object)` are overloaded, and an `int` always matches the index version first.

---

## 2. Set (HashSet, LinkedHashSet, TreeSet) — no duplicates, no index

You can't "get element at position 3" — there is no position. So notice: **no `get()`, no `set()`, no index-based `remove()`.**

| Method | What it does |
|---|---|
| `add(x)` | adds x (returns `false` if already present — this is your duplicate check!) |
| `remove(x)` | removes x by value |
| `contains(x)` | true/false |
| `size()` / `isEmpty()` / `clear()` | same as List |

Memory trick: **Set = "a bag with no seats/slots"** → nothing about position exists, so anything index-related (get/set/insert-at) is gone.

---

## 3. Map (HashMap, LinkedHashMap, TreeMap) — key → value pairs

This is where **put/get** takes over from **add/remove**, because you're never handling one lonely value — always a *pair*.

| Method | What it does | Memory trick |
|---|---|---|
| `put(k, v)` | stores value v under key k (overwrites if k exists) | "PUT the value AT this key" |
| `get(k)` | returns the value for key k, or `null` if absent | "GET what's stored at this key" |
| `remove(k)` | removes the pair with key k | remove by KEY, not value |
| `containsKey(k)` | true/false | — |
| `containsValue(v)` | true/false (slower, scans all values) | — |
| `getOrDefault(k, def)` | returns value, or `def` if key missing | avoids null checks |
| `putIfAbsent(k, v)` | only puts if k is NOT already there | safe insert |
| `keySet()` | returns all keys as a Set | — |
| `values()` | returns all values as a Collection | — |
| `entrySet()` | returns all key-value pairs — best for loops | — |
| `size()` / `isEmpty()` / `clear()` | same as always | — |

Memory trick: **"Map = post office."** You `put` a letter at an address (key), and you `get` it back by giving the same address. There's no "add" because a lonely value with no address makes no sense in a map.

---

## 4. Queue (LinkedList, ArrayDeque, PriorityQueue) — FIFO line

Queue methods come in **two flavors**: ones that **throw exceptions** on failure, and ones that **return special values** (null/false) on failure. Memorize the pairs, not each word individually.

| Action | Throws exception | Returns special value instead |
|---|---|---|
| Insert | `add(x)` | `offer(x)` → false if it fails |
| Remove & return front | `remove()` | `poll()` → null if empty |
| Look at front, don't remove | `element()` | `peek()` → null if empty |

Memory trick: **"offer/poll/peek = the polite, safe versions."** They never throw — they hand you back a signal (`null`/`false`) instead of crashing. Use these in almost all real code; use `add/remove/element` only when an empty queue truly should be a bug.

---

## 5. Deque (ArrayDeque, LinkedList) — double-ended queue (both ends!)

Everything above, but now you must say **which end**: `First` or `Last`.

| End | Insert | Remove | Peek |
|---|---|---|---|
| Front | `addFirst(x)` / `offerFirst(x)` | `removeFirst()` / `pollFirst()` | `peekFirst()` |
| Back | `addLast(x)` / `offerLast(x)` | `removeLast()` / `pollLast()` | `peekLast()` |

Memory trick: it's literally `[add/offer/remove/poll/peek]` + `[First/Last]` — mix and match, the pattern never breaks.

---

## 6. Stack — LIFO pile (top only)

Two ways Java gives you a stack:

**A) The old `Stack` class** (legacy, but you'll see it in code/interviews):

| Method | What it does |
|---|---|
| `push(x)` | places x on top |
| `pop()` | removes AND returns the top element |
| `peek()` | looks at top without removing |
| `isEmpty()` | true if empty |

**B) The modern, recommended way — `Deque` used as a stack:**
```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(x);   // same as addFirst(x)
stack.pop();     // same as removeFirst()
stack.peek();    // same as peekFirst()
```
Memory trick: **push/pop/peek = stack vocabulary, always.** Whenever you see these three words together, think "top of a pile," never "front of a line."

---

## The One Picture to Keep in Your Head

```
                 ADD family                    PUT family
              (single elements)              (key-value pairs)
                     |                              |
        ┌────────────┼────────────┐                 |
        │             │            │                 |
      List           Set        Queue/Deque         Map
   add(i,x)/get     add/remove   add/offer      put/get/remove
   set/remove(i)    (no index!)  push/pop           (by KEY)
                                 peek/poll

     REMOVE = Java's only "take it out" word. DELETE never appears.
     INSERT never appears either — it's always ADD, PUT, or PUSH.
```

## Quick Self-Test (say these out loud)

1. Storing a username → password pair? → **put()** (it's a pair → Map)
2. Adding a task to a to-do list in order? → **add()** (List)
3. Adding a card to the top of a card pile? → **push()** (Stack)
4. Taking the next person out of a line, safely? → **poll()** (Queue)
5. Removing a friend's name (no duplicates allowed) from your contacts Set? → **remove()**
6. "Insert a customer record" — which Java method? → **There isn't one called insert() — it's `add()` for a List or `put()` for a Map.**

If you can answer these without thinking, the confusion is gone for good.
