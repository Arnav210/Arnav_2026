---
toc: true
layout: post
title: DS2 Objectives Evidence
permalink: /cs113_evidence/
comments: true
---

## 1. Data Structures

| Concept | File(s) | What It Shows |
|---------|---------|---------------|
| Collections | [`GameControl.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/GameControl.js) | `Set` deduplicates interaction handlers |
| Lists | [`Game.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/Game.js) · [`DonationService.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/DonationService.java) | Array snapshot for level reset; `ArrayList` for sorted donations |
| Stacks / Queues | [`DonationService.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/DonationService.java) | `ArrayDeque` as LIFO status-undo stack per donation |
| Trees | [`CategoryTree.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/CategoryTree.java) · [`CategoryTreeTest.java`](https://github.com/Arnav210/hh_spring/blob/master/src/test/java/com/open/spring/mvc/donation/CategoryTreeTest.java) | N-ary food category hierarchy; DFS traversal, search, path-to-root |
| Sets | [`GameControl.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/GameControl.js) · [`DonationGraph.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/DonationGraph.java) | Handler uniqueness; `HashSet` as adjacency neighbor container |
| Dictionaries / Maps | [`donationApi.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/api/donationApi.js) · [`DonationService.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/DonationService.java) | Status/error maps; `HashMap`, `LinkedHashMap` for donor counts |
| Graphs | [`DonationGraph.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/DonationGraph.java) · [`DonationGraphTest.java`](https://github.com/Arnav210/hh_spring/blob/master/src/test/java/com/open/spring/mvc/donation/DonationGraphTest.java) | Adjacency-list directed graph; BFS recommendations, connected components |

### Collections

`GameControl.js` uses a `Set` — a collection that automatically deduplicates — to track interaction handlers so the same handler can never be registered twice.

<details markdown="1">
<summary>hunger_heroes — assets/js/GameEnginev1.1/essentials/GameControl.js</summary>

```js
// hunger_heroes — assets/js/GameEnginev1.1/essentials/GameControl.js
this.globalInteractionHandlers = new Set();
this.savedInteractionHandlers = new Set();
```

</details>

---

### Lists (Arrays)

`Game.js` snapshots the level queue into an array for reset capability. `donationApi.js` returns sorted arrays of donations.

<details markdown="1">
<summary>hunger_heroes — assets/js/GameEnginev1.1/essentials/Game.js</summary>

```js
// hunger_heroes — assets/js/GameEnginev1.1/essentials/Game.js
this.initialLevelClasses = [...(environment.gameLevelClasses || [])];
```

</details>

`DonationService.java` uses `ArrayList` as the primary working collection for sorted donation lists:

<details markdown="1">
<summary>hh_spring — DonationService.java (Javadoc comment)</summary>

```java
// hh_spring — DonationService.java (Javadoc comment)
// * {@link ArrayList} — sorted working copies of donation lists
List<Donation> donations = new ArrayList<>(donationRepo.findAll());
Collections.sort(donations, Comparator.comparing(Donation::getExpiryDate));
```

</details>

---

### Stacks / Queues

`DonationService.java` implements a **status-undo stack** per donation using `ArrayDeque` (LIFO). Every status change pushes the old status; `undoStatusChange()` pops and restores it.

<details markdown="1">
<summary>hh_spring — DonationService.java</summary>

```java
// hh_spring — DonationService.java
private final Map<String, Deque<String>> undoStacks = new ConcurrentHashMap<>();

/**
 * Pushes the current status onto the undo stack before a status change.
 * Demonstrates Stack (LIFO) data structure using ArrayDeque.
 */
private void pushUndo(String donationId, String oldStatus) {
    undoStacks.computeIfAbsent(donationId, k -> new ArrayDeque<>()).push(oldStatus);
}

public Donation acceptDonation(String donationId, String acceptedBy) {
    Donation d = donationRepo.findByDonationId(donationId).orElseThrow(...);
    pushUndo(donationId, d.getStatus());   // ← stack push before state change
    d.setStatus("accepted");
    ...
}
```

</details>

---

### Trees

`CategoryTree.java` is an **N-ary tree** that classifies food categories hierarchically. The root is "All Food", branching into Perishable / Non-Perishable / Prepared, then into 12 leaf categories (dairy, meat-protein, canned, etc.) that match the donation form's allowed values.

<details markdown="1">
<summary>hh_spring — CategoryTree.java</summary>

```java
// hh_spring — CategoryTree.java

/**
 * Tree-based hierarchical classification of food categories.
 * Implements a general tree (N-ary) where each node = a food category.
 *
 * Tree Operations:
 *  - DFS Traversal  — pre-order, O(n)
 *  - Search         — DFS by name, O(n), case-insensitive
 *  - Path to Root   — O(h) where h = tree height
 *  - Count Leaves   — O(n)
 */
public class CategoryTree {

    public static class CategoryNode {
        private final String name;
        private final String description;
        private final List<CategoryNode> children = new ArrayList<>();
        private CategoryNode parent;

        public CategoryNode addChild(CategoryNode child) {
            child.parent = this;
            children.add(child);
            return child;
        }

        public int getDepth() {
            int depth = 0;
            CategoryNode current = this;
            while (current.parent != null) { depth++; current = current.parent; }
            return depth;
        }
    }

    public CategoryTree() {
        root = new CategoryNode("All Food", "Root category for all food donations");

        CategoryNode perishable    = root.addChild(new CategoryNode("Perishable", "..."));
        CategoryNode nonPerishable = root.addChild(new CategoryNode("Non-Perishable", "..."));
        CategoryNode prepared      = root.addChild(new CategoryNode("Prepared & Specialty", "..."));

        perishable.addChild(new CategoryNode("dairy",        "Milk, cheese, yogurt..."));
        perishable.addChild(new CategoryNode("meat-protein", "Beef, chicken, fish..."));
        // ... 10 more leaf nodes
    }
}
```

</details>

`CategoryTreeTest.java` tests DFS traversal, path-to-root, and leaf counting:

<details markdown="1">
<summary>hh_spring — CategoryTreeTest.java</summary>

```java
// hh_spring — CategoryTreeTest.java
@Test void testLeafCount()  { assertEquals(12, tree.countLeaves()); }
@Test void testRootDepth()  { assertEquals(0,  tree.getRoot().getDepth()); }
@Test void testLeafDepth()  { assertEquals(2,  tree.search("dairy").getDepth()); }

@Test void testLeafPath() {
    List<String> path = tree.pathToRoot("dairy");
    assertEquals(List.of("dairy", "Perishable", "All Food"), path);
}

@Test void testPreOrderProperty() {
    List<String> t = tree.preOrderTraversal();
    assertTrue(t.indexOf("Perishable") < t.indexOf("dairy")); // parent before child
}
```

</details>

---

### Sets

`GameControl.js` uses `Set` to ensure global interaction handlers are unique — adding the same function twice has no effect.

<details markdown="1">
<summary>hunger_heroes — assets/js/GameEnginev1.1/essentials/GameControl.js</summary>

```js
// hunger_heroes — assets/js/GameEnginev1.1/essentials/GameControl.js
this.globalInteractionHandlers = new Set();
// Snapshot before loading a nested game:
this.savedInteractionHandlers = new Set(this.globalInteractionHandlers);
```

</details>

`DonationGraph.java` uses `HashSet` as the adjacency list's neighbor container (O(1) contains check):

<details markdown="1">
<summary>hh_spring — DonationGraph.java</summary>

```java
// hh_spring — DonationGraph.java
private final Map<String, Set<String>> adjacency = new HashMap<>();
private final Set<String> allNodes = new HashSet<>();

public void addEdge(String donor, String receiver) {
    adjacency.computeIfAbsent(donor, k -> new HashSet<>()).add(receiver);
    allNodes.add(donor);
    allNodes.add(receiver);
}
```

</details>

---

### Dictionaries / Maps

`donationApi.js` maps raw status strings to display strings and maps HTTP error codes to user-friendly messages:

<details markdown="1">
<summary>hunger_heroes — assets/js/api/donationApi.js</summary>

```js
// hunger_heroes — assets/js/api/donationApi.js
const STATUS_MAP = {
    'active':   'Available',
    'matched':  'Matched',
    'archived': 'Archived',
};

const ERROR_MESSAGES = {
    404: 'Donation not found.',
    403: 'Permission denied.',
    500: 'Server error — please try again later.',
};

normalizeStatus(raw) {
    return STATUS_MAP[raw?.toLowerCase()] ?? raw;
}
```

</details>

`DonationService.java` uses `HashMap` for per-donor donation counts and `LinkedHashMap` for insertion-ordered results:

<details markdown="1">
<summary>hh_spring — DonationService.java</summary>

```java
// hh_spring — DonationService.java
Map<String, Long> counts = new HashMap<>();
for (Donation d : donationRepo.findAll()) {
    counts.merge(d.getDonorName(), 1L, Long::sum);  // O(1) per update
}
// LinkedHashMap preserves ordered output
Map<String, Long> result = new LinkedHashMap<>();
```

</details>

---

### Graphs

`DonationGraph.java` models donor→receiver relationships as a **directed graph** with an adjacency list. It implements BFS, connected-component detection, and influence ranking.

<details markdown="1">
<summary>hh_spring — DonationGraph.java</summary>

```java
// hh_spring — DonationGraph.java

/**
 * Graph-based model of donor→receiver relationships.
 * Uses adjacency list (HashMap<String, Set<String>>) for a directed graph
 * where each edge connects a donor to a receiver who accepted a donation.
 */
public class DonationGraph {
    private final Map<String, Set<String>> adjacency = new HashMap<>();

    // BFS — finds all donors reachable within N hops (for recommendations)
    // Time: O(V + E)   Space: O(V)
    public Set<String> bfsReachable(String start, int maxDepth) {
        Set<String> visited = new HashSet<>();
        Deque<String> queue = new ArrayDeque<>();
        queue.add(start);
        int depth = 0;
        while (!queue.isEmpty() && depth < maxDepth) {
            int size = queue.size();
            for (int i = 0; i < size; i++) {
                String node = queue.poll();
                for (String neighbor : adjacency.getOrDefault(node, Set.of())) {
                    if (visited.add(neighbor)) queue.add(neighbor);
                }
            }
            depth++;
        }
        visited.remove(start);
        return visited;
    }
}
```

</details>

The test suite verifies BFS and connected components:

<details markdown="1">
<summary>hh_spring — DonationGraphTest.java</summary>

```java
// hh_spring — DonationGraphTest.java
@Test
void testBfsReachableFromAlice() {
    // alice→bob, alice→charlie, bob→dave
    Set<String> reached = graph.bfsReachable("alice", 1);
    assertTrue(reached.contains("bob"));
    assertTrue(reached.contains("charlie"));
    assertFalse(reached.contains("dave")); // depth 2, not reached at depth 1
}
```

</details>

---

## 2. Algorithms

| Algorithm | File(s) | Complexity |
|-----------|---------|------------|
| Searching (binary) | [`DonationService.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/DonationService.java) | O(log n) on sorted expiry list |
| Searching (linear) | [`InteractionManager.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/InteractionManager.js) | O(n) nearest-NPC scan |
| Sorting | [`DonationService.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/DonationService.java) · [`donationApi.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/api/donationApi.js) | O(n log k) min-heap top-N; O(n log n) multi-key JS sort |
| Hashing | [`donation.py`](https://github.com/Arnav210/hunger_heroes_backend/blob/main/model/donation.py) · [`DonationService.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/DonationService.java) · [`DonationGraph.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/DonationGraph.java) | SHA-256 ID; SecureRandom ID; `HashMap`/`HashSet` hash-based O(1) lookup |
| Big-O Analysis | [`DonationService.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/DonationService.java) | Explicit O(·) annotations in Javadoc throughout |

### Searching

`DonationService.java` performs a **binary search** on a sorted list of donations to find the first one expiring on or after a given date (O(log n)):

<details markdown="1">
<summary>hh_spring — DonationService.java</summary>

```java
// hh_spring — DonationService.java
// Binary search by expiry — O(log n) after O(n log n) sort
List<Donation> sorted = new ArrayList<>(donationRepo.findAll());
sorted.sort(Comparator.comparing(Donation::getExpiryDate));
int lo = 0, hi = sorted.size() - 1;
while (lo <= hi) {
    int mid = (lo + hi) >>> 1;
    if (!sorted.get(mid).getExpiryDate().isBefore(targetDate)) hi = mid - 1;
    else lo = mid + 1;
}
```

</details>

`InteractionManager.js` performs a **linear scan** to find the nearest NPC (O(n)):

<details markdown="1">
<summary>hunger_heroes — assets/js/GameEnginev1.1/essentials/InteractionManager.js</summary>

```js
// hunger_heroes — assets/js/GameEnginev1.1/essentials/InteractionManager.js
updateInteractButtonByDistance(player, npcs) {
    let nearest = null, minDist = Infinity;
    for (const npc of npcs) {
        const dx = player.x - npc.x, dy = player.y - npc.y;
        const dist = Math.sqrt(dx * dx + dy * dy);
        if (dist < minDist) { minDist = dist; nearest = npc; }
    }
    return nearest;
}
```

</details>

---

### Sorting

`DonationService.java` uses a **min-heap** (`PriorityQueue`) to extract the top-N donors in O(n log k):

<details markdown="1">
<summary>hh_spring — DonationService.java  getDonorLeaderboard()</summary>

```java
// hh_spring — DonationService.java  getDonorLeaderboard()
PriorityQueue<Map.Entry<String, Long>> heap =
    new PriorityQueue<>(Comparator.comparingLong(Map.Entry::getValue));

for (Map.Entry<String, Long> e : counts.entrySet()) {
    heap.offer(e);
    if (heap.size() > limit) heap.poll();  // evict smallest  O(log k)
}
```

</details>

`donationApi.js` implements O(n log n) multi-key sorting with a custom comparator:

<details markdown="1">
<summary>hunger_heroes — assets/js/api/donationApi.js</summary>

```js
// hunger_heroes — assets/js/api/donationApi.js
sortByUrgency(donations) {
    return [...donations].sort((a, b) => {
        const urgencyOrder = { critical: 0, urgent: 1, standard: 2 };
        const urgencyDiff = (urgencyOrder[a.urgency] ?? 3) - (urgencyOrder[b.urgency] ?? 3);
        if (urgencyDiff !== 0) return urgencyDiff;
        return new Date(a.expiry_date) - new Date(b.expiry_date); // tiebreaker
    });
}
```

</details>

---

### Hashing

`generate_donation_id()` uses SHA-256 hashing to produce a compact, collision-resistant donation ID:

<details markdown="1">
<summary>Arnav_hunger_heroes_backend — model/donation.py</summary>

```python
# Arnav_hunger_heroes_backend — model/donation.py
def generate_donation_id():
    import hashlib, time, random
    raw = f"{time.time()}-{random.random()}"
    h = hashlib.sha256(raw.encode()).hexdigest()
    return int(h[:8], 16)   # first 8 hex chars → integer hash
```

</details>

`DonationService.java` uses `generateDonationId()` with `SecureRandom` and base-36 encoding on the Spring side:

<details markdown="1">
<summary>hh_spring — DonationService.java</summary>

```java
// hh_spring — DonationService.java
private static final SecureRandom RANDOM = new SecureRandom();
private static final String ID_CHARS = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";

private String generateDonationId() {
    StringBuilder sb = new StringBuilder("HH-");
    for (int i = 0; i < 6; i++) sb.append(ID_CHARS.charAt(RANDOM.nextInt(ID_CHARS.length())));
    sb.append('-');
    for (int i = 0; i < 4; i++) sb.append(ID_CHARS.charAt(RANDOM.nextInt(ID_CHARS.length())));
    return sb.toString();   // e.g. "HH-M3X7K9-AB2F"
}
```

</details>

`HashMap` and `HashSet` in Java rely on `Object.hashCode()` for O(1) average-case put/get/contains. `DonationGraph` and `DonationService` use `HashMap<String, Set<String>>` and `HashSet<String>` throughout, so every `addEdge()` and `bfsReachable()` call is backed by hash-based lookup:

<details markdown="1">
<summary>hh_spring — DonationGraph.java</summary>

```java
// hh_spring — DonationGraph.java
private final Map<String, Set<String>> adjacency = new HashMap<>();  // hash-based O(1) lookup
private final Set<String> allNodes = new HashSet<>();                 // hash-based O(1) contains
```

</details>

---

### Algorithm Analysis (Big-O)

`DonationService.java` includes explicit Big-O annotations in Javadoc:

<details markdown="1">
<summary>hh_spring — DonationService.java</summary>

```java
// hh_spring — DonationService.java

/**
 * Priority-score matching — ranks available donations for a receiver.
 * Time complexity: O(n · m) where n = donations, m = dietary filters.
 */

/**
 * Binary search by expiry — after sorting donations by expiry date,
 * performs an O(log n) binary search to find the first donation
 * expiring on or after a given date.
 */

// getDonorLeaderboard — O(n log k) via PriorityQueue min-heap
// Count per donor  O(n)
// Sort by count descending using a PriorityQueue (min-heap)  O(n log k)
```

</details>

---

## 3. Object-Oriented Programming

| Concept | File(s) | What It Shows |
|---------|---------|---------------|
| Abstraction | [`GameObject.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/GameObject.js) · [`Donation.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/Donation.java) / [`FoodItem.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/FoodItem.java) | Runtime-enforced abstract base; mapped superclass |
| Encapsulation | [`Transform.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/Transform.js) · [`HungerHeroScore.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/hungerheroes/HungerHeroScore.java) | Private JS fields `#x #y`; Lombok `@Data` generated getters/setters |
| Inheritance | [`Character.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/Character.js) → [`Player.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/Player.js) / [`Npc.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/Npc.js) · `Donation extends FoodItem` | Three-level JS hierarchy; Java entity inheritance |
| Polymorphism | [`GameLevel.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/GameLevel.js) · [`HungerHeroScoreController.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/hungerheroes/HungerHeroScoreController.java) | Runtime dispatch of `update()`; `ResponseEntity<?>` for mixed return types |
| Design Patterns | [`GameEnvScore.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/GameEnvScore.js) · [`GameLevelHungerHeroes.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/projects/hunger-heroes-game/levels/GameLevelHungerHeroes.js) | Singleton score instance; Factory sprite creation |

### Abstraction

`GameObject.js` is an abstract base class enforced at runtime — attempting to instantiate it directly throws a `TypeError`.

<details markdown="1">
<summary>hunger_heroes — assets/js/GameEnginev1.1/essentials/GameObject.js</summary>

```js
// hunger_heroes — assets/js/GameEnginev1.1/essentials/GameObject.js
class GameObject {
    constructor(gameEnv) {
        if (new.target === GameObject) {
            throw new TypeError("Cannot construct GameObject instances directly");
        }
        this.gameEnv = gameEnv;
    }
    draw()    { throw new Error("draw() must be implemented"); }
    update()  { throw new Error("update() must be implemented"); }
    resize()  { throw new Error("resize() must be implemented"); }
    destroy() { throw new Error("destroy() must be implemented"); }
}
```

</details>

`FoodItem.java` in hh_spring is a mapped superclass that `Donation` extends, abstracting shared food properties:

<details markdown="1">
<summary>hh_spring — Donation.java (Javadoc)</summary>

```java
// hh_spring — Donation.java (Javadoc)
/**
 * Extends {@link FoodItem} (demonstrating inheritance)
 * and adds donor-specific information, lifecycle tracking,
 * and volunteer assignment capabilities.
 */
@Entity
@DiscriminatorValue("DONATION")
public class Donation extends FoodItem { ... }
```

</details>

---

### Encapsulation

`Transform.js` wraps position and velocity in private fields, validating inputs before setting them:

<details markdown="1">
<summary>hunger_heroes — assets/js/GameEnginev1.1/essentials/Transform.js</summary>

```js
// hunger_heroes — assets/js/GameEnginev1.1/essentials/Transform.js
class TransformState {
    #x; #y; #vx; #vy;   // private fields

    setPosition(x, y) {
        TransformValidation.number(x, "x");
        TransformValidation.number(y, "y");
        this.#x = x;
        this.#y = y;
    }
    getPosition() { return { x: this.#x, y: this.#y }; }
}
```

</details>

`HungerHeroScore.java` uses Lombok `@Data` to encapsulate all fields behind generated getters/setters:

<details markdown="1">
<summary>hh_spring — HungerHeroScore.java</summary>

```java
// hh_spring — HungerHeroScore.java
@Data           // generates getters, setters, equals, hashCode, toString
@NoArgsConstructor
@AllArgsConstructor
@Entity
@Table(name = "hunger_heroes_leaderboard")
public class HungerHeroScore {
    private Integer npcsVisited;
    private Integer dialoguesCompleted;
    private Integer score;            // never set by client — computeScore() only
    private Integer timePlayedSeconds;
}
```

</details>

---

### Inheritance

Three-level JS class hierarchy: `GameObject → Character → Player / Npc`.

<details markdown="1">
<summary>hunger_heroes</summary>

```js
// hunger_heroes
class Character extends GameObject {
    constructor(data, gameEnv) { super(gameEnv); /* sprite/animation */ }
}

class Player extends Character {
    update() {
        super.update();
        if (!this.moved && this.gravity) {
            this.velocity.y += 0.5 + this.acceleration * this.time;
        }
    }
}

class Npc extends Character {
    update() {
        if (this.walkingArea) { this.patrol(); }
        this.draw();
    }
}
```

</details>

`Donation extends FoodItem` on the Java side:

<details markdown="1">
<summary>hh_spring — Donation.java</summary>

```java
// hh_spring — Donation.java
@Entity
@DiscriminatorValue("DONATION")
public class Donation extends FoodItem {
    private String donorName;
    private String donorEmail;
    private String status = "active";
    // ... donor-specific fields
}
```

</details>

---

### Polymorphism

`GameLevel.js` calls `update()` on every object in `this.gameObjects` without knowing the concrete type — each subclass dispatches its own implementation.

<details markdown="1">
<summary>hunger_heroes — assets/js/GameEnginev1.1/essentials/GameLevel.js</summary>

```js
// hunger_heroes — assets/js/GameEnginev1.1/essentials/GameLevel.js
for (const obj of this.gameObjects) {
    obj.update();   // → Player.update(), Npc.update(), etc. at runtime
}
```

</details>

`HungerHeroScoreController.java` demonstrates polymorphism via `ResponseEntity<?>` — the same return type handles both success objects and error `Map` bodies:

<details markdown="1">
<summary>hh_spring — HungerHeroScoreController.java</summary>

```java
// hh_spring — HungerHeroScoreController.java
@PostMapping("/leaderboard")
public ResponseEntity<?> submitScore(@RequestBody HungerHeroScore score) {
    if (score.getNpcsVisited() == null)
        return ResponseEntity.badRequest().body(Map.of("message", "npcsVisited is required"));
    score.computeScore();
    return ResponseEntity.ok(repo.save(score));   // same method, different runtime type
}
```

</details>

---

### Design Patterns

**Singleton** — `GameEnvScore.js` ensures exactly one score instance per game environment:

<details markdown="1">
<summary>hunger_heroes — assets/js/GameEnginev1.1/essentials/GameEnvScore.js</summary>

```js
// hunger_heroes — assets/js/GameEnginev1.1/essentials/GameEnvScore.js
class GameEnvScore {
    static #instance = null;
    static getInstance() {
        if (!GameEnvScore.#instance) {
            GameEnvScore.#instance = new GameEnvScore();
        }
        return GameEnvScore.#instance;
    }
}
```

</details>

**Factory** — `GameLevelHungerHeroes.js` delegates all sprite/NPC creation to a `SpriteGenerator` factory:

<details markdown="1">
<summary>hunger_heroes — assets/js/projects/hunger-heroes-game/levels/GameLevelHungerHeroes.js</summary>

```js
// hunger_heroes — assets/js/projects/hunger-heroes-game/levels/GameLevelHungerHeroes.js
SpriteGenerator.createBackground('foodbank');
SpriteGenerator.createNpcSprite('📦', '#3b82f6', 'CREATE');
SpriteGenerator.createPlayerSprite('default');
```

</details>

---

## 4. Software Development

| Topic | Tool / Approach | File(s) | Evidence |
|-------|----------------|---------|----------|
| Version Control | Git / GitHub | [`.github/workflows/jekyll-gh-pages.yml`](https://github.com/Ahaanv19/hunger_heroes/blob/main/.github/workflows/jekyll-gh-pages.yml) | Branch-triggered CI/CD; `main` branch protection |
| Testing | JUnit 5 + pytest | [`CategoryTreeTest.java`](https://github.com/Arnav210/hh_spring/blob/master/src/test/java/com/open/spring/mvc/donation/CategoryTreeTest.java) · [`DonationGraphTest.java`](https://github.com/Arnav210/hh_spring/blob/master/src/test/java/com/open/spring/mvc/donation/DonationGraphTest.java) · [`test_donation.py`](https://github.com/Arnav210/hunger_heroes_backend/blob/main/testing/test_donation.py) · [`test_models.py`](https://github.com/Arnav210/hunger_heroes_backend/blob/main/testing/test_models.py) | Unit tests with `@Nested`, fixtures, in-memory DB |
| Build Tools | Maven · Make · pip | [`pom.xml`](https://github.com/Arnav210/hh_spring/blob/master/pom.xml) · [`Makefile`](https://github.com/Ahaanv19/hunger_heroes/blob/main/Makefile) · `Dockerfile` | Spring Boot 3.5.0 / Java 21 deps; Jekyll build targets; gunicorn Flask build |
| Debugging | SLF4J · `console.log` | [`BankService.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/bank/BankService.java) · [`GameControl.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/GameControl.js) · [`donationApi.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/api/donationApi.js) | Structured log levels; fallback tracing |
| API Development | Spring REST · Flask Blueprint · JS client | [`HungerHeroScoreController.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/hungerheroes/HungerHeroScoreController.java) · [`game_leaderboard.py`](https://github.com/Arnav210/hunger_heroes_backend/blob/main/api/game_leaderboard.py) · [`donationApi.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/api/donationApi.js) | CRUD endpoints, input validation, Spring/Flask fallback |
| Database | JPA/Hibernate · SQLAlchemy | [`Donation.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/Donation.java) · [`model/donation.py`](https://github.com/Arnav210/hunger_heroes_backend/blob/main/model/donation.py) | Entity with one-to-many audit log; ORM with FK and JSON field |
| I/O & Control Flow | JS keyboard input | [`Player.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/GameEnginev1.1/essentials/Player.js) | Keyboard branch, gravity loop, respawn |

### Version Control (Git)

The project uses GitHub with feature branches and pull requests. Every merge to `main` triggers the CI/CD pipeline (see below). The `.github/workflows/jekyll-gh-pages.yml` workflow is the result of this version-control discipline — the pipeline only runs because there is a protected `main` branch being merged into.

<details markdown="1">
<summary>hunger_heroes — .github/workflows/jekyll-gh-pages.yml</summary>

```yaml
# hunger_heroes — .github/workflows/jekyll-gh-pages.yml
on:
  push:
    branches: ["main"]
  workflow_dispatch:
```

</details>

---

### Testing

**JUnit 5 (hh_spring):** `CategoryTreeTest.java` tests every tree operation — structure, DFS search, pre-order traversal, and path-to-root — using `@Nested` test groups for clarity.

<details markdown="1">
<summary>hh_spring — CategoryTreeTest.java</summary>

```java
// hh_spring — CategoryTreeTest.java
@Nested
@DisplayName("DFS Search")
class SearchTests {

    @Test @DisplayName("Search finds existing category")
    void testSearchFound() {
        CategoryTree.CategoryNode result = tree.search("frozen");
        assertNotNull(result);
        assertEquals("frozen", result.getName());
    }

    @Test @DisplayName("Search is case-insensitive")
    void testSearchCaseInsensitive() {
        assertNotNull(tree.search("DAIRY"));
        assertNotNull(tree.search("Bakery"));
    }

    @Test @DisplayName("Search returns null for non-existent category")
    void testSearchNotFound() {
        assertNull(tree.search("pizza"));
    }
}
```

</details>

`DonationGraphTest.java` verifies BFS reachability and connected components on a live `DonationGraph` instance:

<details markdown="1">
<summary>hh_spring — DonationGraphTest.java</summary>

```java
// hh_spring — DonationGraphTest.java
@BeforeEach
void setUp() {
    graph = new DonationGraph();
    graph.addEdge("alice", "bob");
    graph.addEdge("alice", "charlie");
    graph.addEdge("bob", "dave");
    graph.addEdge("eve", "frank");
}

@Test void testNodeCount() { assertEquals(6, graph.nodeCount()); }
@Test void testEdgeCount() { assertEquals(4, graph.edgeCount()); }

@Test void testGetNeighbors() {
    Set<String> neighbors = graph.getNeighbors("alice");
    assertEquals(2, neighbors.size());
    assertTrue(neighbors.contains("bob") && neighbors.contains("charlie"));
}
```

</details>

**pytest (Arnav_hunger_heroes_backend):** `test_donation.py` creates a fresh in-memory SQLite database per test and verifies full Donation CRUD:

<details markdown="1">
<summary>Arnav_hunger_heroes_backend — testing/test_donation.py</summary>

```python
# Arnav_hunger_heroes_backend — testing/test_donation.py
@pytest.fixture(autouse=True)
def setup_db():
    """Create a fresh in-memory database for each test."""
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///:memory:'
    app.config['TESTING'] = True
    with app.app_context():
        db.create_all()
        yield
        db.session.rollback()
        db.drop_all()

def test_donation_creation_full_fields():
    with app.app_context():
        d = Donation(
            id=generate_donation_id(),
            food_name='Veggie Tray',
            quantity=10,
            expiry_date=date.today() + timedelta(days=2),
            allergens=['nuts'],
            status='active',
        )
        db.session.add(d)
        db.session.commit()
        fetched = Donation.query.get(d.id)
        assert fetched.food_name == 'Veggie Tray'
        assert fetched.status == 'active'
```

</details>

`test_models.py` uses `unittest.TestCase` with `setUp`/`tearDown` lifecycle:

<details markdown="1">
<summary>Arnav_hunger_heroes_backend — testing/test_models.py</summary>

```python
# Arnav_hunger_heroes_backend — testing/test_models.py
class TestUserModel(unittest.TestCase):
    def setUp(self):
        self.app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///:memory:'
        db.create_all()

    def tearDown(self):
        db.session.remove()
        db.drop_all()

    def test_user_creation_with_all_fields(self):
        user = User(name='John Donor', uid='johndoner', role='Donor', email='john@example.com')
        result = user.create()
        self.assertIsNotNone(result)
```

</details>

---

### Build Tools

**Maven (hh_spring):** `pom.xml` declares Spring Boot 3.5.0 on Java 21 with all dependencies:

<details markdown="1">
<summary>hh_spring — pom.xml</summary>

```xml
<!-- hh_spring — pom.xml -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.0</version>
</parent>
<groupId>com.open</groupId>
<artifactId>spring</artifactId>
<properties>
    <java.version>21</java.version>
</properties>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
    <!-- + spring-security, validation, Thymeleaf, AWS SDK, JUnit 5 -->
</dependencies>
```

</details>

**Make (hunger_heroes):** `Makefile` automates Jekyll builds and project asset compilation:

<details markdown="1">
<summary>hunger_heroes — Makefile</summary>

```makefile
# hunger_heroes — Makefile
build-registered-projects:
    cd _projects && make

build:
    bundle exec jekyll build
```

</details>

**pip (Arnav_hunger_heroes_backend):** The `Dockerfile` installs all Flask dependencies from `requirements.txt`:

<details markdown="1">
<summary>Arnav_hunger_heroes_backend — Dockerfile</summary>

```dockerfile
# Arnav_hunger_heroes_backend — Dockerfile
FROM docker.io/python:3.12
COPY . /
RUN pip install --no-cache-dir -r requirements.txt
RUN pip install gunicorn
ENV FLASK_ENV=production
CMD ["gunicorn", "main:app"]
```

</details>

---

### Debugging

**JavaScript console logging (hunger_heroes):** `GameControl.js` uses structured `console.log` / `console.warn` to trace canvas click events and handler lifecycle:

<details markdown="1">
<summary>hunger_heroes — assets/js/GameEnginev1.1/essentials/GameControl.js</summary>

```js
// hunger_heroes — assets/js/GameEnginev1.1/essentials/GameControl.js
console.log('[CanvasClickHandler] constructor:', { gameContainer });
console.warn('[CanvasClickHandler] No gameContainer to bind click listener');
console.log('[CanvasClickHandler] handleCanvasClick fired', { event });
console.log('Saved interaction handlers:', this.savedInteractionHandlers.size);
console.warn('Could not snapshot parent control state for nested game:', e);
```

</details>

`donationApi.js` logs fallback behavior when backends are unreachable:

<details markdown="1">
<summary>hunger_heroes — assets/js/api/donationApi.js</summary>

```js
// hunger_heroes — assets/js/api/donationApi.js
console.log('Spring unavailable, trying Flask…', springErr.message);
console.log('✅ Spring POST:', endpoint, resultId);
console.log('Flask label fetch failed', e.message);
```

</details>

**SLF4J logging (hh_spring):** `BankService.java` uses SLF4J with structured log levels:

<details markdown="1">
<summary>hh_spring — src/main/java/com/open/spring/mvc/bank/BankService.java</summary>

```java
// hh_spring — src/main/java/com/open/spring/mvc/bank/BankService.java
private static final Logger logger = LoggerFactory.getLogger(BankService.class);

public Bank findByPersonId(Long personId) {
    Bank bank = bankRepository.findByPersonId(personId);
    if (bank == null) {
        logger.error("No bank account found for Person ID: {}", personId);
        throw new RuntimeException("Bank account not found for Person ID: " + personId);
    }
    return bank;
}

public void clearAllBanks() {
    logger.info("Starting to clear all bank records...");
    logger.warn("Some records were not deleted. Attempting alternative deletion method...");
    logger.info("Final bank record count after deletion: {}", finalCount);
}
```

</details>

---

### API Development

**Spring Boot REST API (hh_spring):** `HungerHeroScoreController.java` exposes the leaderboard API.

<details markdown="1">
<summary>hh_spring — HungerHeroScoreController.java</summary>

```java
// hh_spring — HungerHeroScoreController.java
@RestController
@RequestMapping("/api/hunger-heroes")
@CrossOrigin(origins = "*", methods = { GET, POST, OPTIONS })
public class HungerHeroScoreController {

    @GetMapping("/leaderboard")
    public ResponseEntity<List<HungerHeroScore>> getLeaderboard(
            @RequestParam(required = false) String levelId,
            @RequestParam(defaultValue = "50") int limit) {
        List<HungerHeroScore> scores = (levelId != null)
            ? repo.findByLevelOrderByScoreDesc(levelId).stream().limit(limit).toList()
            : repo.findAllOrderByScoreDesc().stream().limit(limit).toList();
        return ResponseEntity.ok(scores);
    }

    @PostMapping("/leaderboard")
    public ResponseEntity<?> submitScore(@RequestBody HungerHeroScore score) {
        if (score.getNpcsVisited() == null)
            return ResponseEntity.badRequest().body(Map.of("message", "npcsVisited is required"));
        score.setNpcsVisited(Math.min(5, Math.max(0, score.getNpcsVisited())));
        score.computeScore();           // server-side — never trust the client
        score.setCreatedAt(LocalDateTime.now());
        return ResponseEntity.ok(repo.save(score));
    }
}
```

</details>

**Flask REST API (Arnav_hunger_heroes_backend):** The donation API is exposed via Flask Blueprints:

<details markdown="1">
<summary>Arnav_hunger_heroes_backend — api/game_leaderboard.py</summary>

```python
# Arnav_hunger_heroes_backend — api/game_leaderboard.py
hunger_heroes_api = Blueprint('hunger_heroes_api', __name__, url_prefix='/api/hunger-heroes')

@hunger_heroes_api.route('/', methods=['GET'])
def get_scores():
    scores = HungerHeroScore.query.order_by(HungerHeroScore.score.desc()).all()
    return jsonify([s.to_dict() for s in scores])

@hunger_heroes_api.route('/', methods=['POST'])
def submit_score():
    data = request.get_json()
    score = HungerHeroScore(uid=data['uid'], score=data['score'])
    db.session.add(score)
    db.session.commit()
    return jsonify(score.to_dict()), 201
```

</details>

**Frontend API client (hunger_heroes):** `donationApi.js` calls Spring first, falls back to Flask, with timeout and CORS credentials:

<details markdown="1">
<summary>hunger_heroes — assets/js/api/donationApi.js</summary>

```js
// hunger_heroes — assets/js/api/donationApi.js
async springFetch(endpoint, options = {}) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 5000);
    try {
        const res = await fetch(`${SPRING_URI}${endpoint}`, {
            ...options, signal: controller.signal, credentials: 'include'
        });
        if (!res.ok) throw new Error(ERROR_MESSAGES[res.status] ?? 'Unknown error');
        return await res.json();
    } catch (e) {
        console.log('Spring unavailable, trying Flask…', e.message);
        return this.flaskFetch(endpoint, options);   // fallback
    } finally {
        clearTimeout(timeout);
    }
}
```

</details>

---

### Database Integration

**JPA/Hibernate (hh_spring):** `Donation.java` is a full JPA entity with one-to-many audit logs and JSON columns:

<details markdown="1">
<summary>hh_spring — Donation.java</summary>

```java
// hh_spring — Donation.java
@Data @NoArgsConstructor @AllArgsConstructor
@Entity @DiscriminatorValue("DONATION")
public class Donation extends FoodItem {

    @Column(unique = true, nullable = false, length = 50)
    private String donationId;      // e.g. "HH-M3X7K9-AB2F"

    @Column(columnDefinition = "TEXT")
    private String allergens;       // stored as comma-separated

    @Column(nullable = false, length = 20)
    private String status = "active";

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt = LocalDateTime.now();

    @OneToMany(mappedBy = "donation", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<DonationStatusLog> statusLogs = new ArrayList<>();
}
```

</details>

**SQLAlchemy (Arnav_hunger_heroes_backend):** The `Donation` Python model uses SQLAlchemy ORM with foreign keys and JSON fields:

<details markdown="1">
<summary>Arnav_hunger_heroes_backend — model/donation.py</summary>

```python
# Arnav_hunger_heroes_backend — model/donation.py
class Donation(db.Model):
    __tablename__ = 'donations'
    id = db.Column(db.String(20), primary_key=True, default=generate_donation_id)
    food_name = db.Column(db.String(200), nullable=False)
    allergens = db.Column(db.JSON, default=list)
    status = db.Column(db.String(20), default='active')
    donor_id = db.Column(db.Integer, db.ForeignKey('users.id'))
    donor = db.relationship('User', backref='donations')
```

</details>

---

### Input / Output & Control Structures

`Player.js` reads keyboard/touch input and applies physics with conditional branches and loops:

<details markdown="1">
<summary>hunger_heroes — assets/js/GameEnginev1.1/essentials/Player.js</summary>

```js
// hunger_heroes — assets/js/GameEnginev1.1/essentials/Player.js
update() {
    super.update();
    if (this.keys['ArrowLeft'])       { this.velocity.x = -this.speed; this.moved = true; }
    else if (this.keys['ArrowRight']) { this.velocity.x =  this.speed; this.moved = true; }
    else                              { this.velocity.x = 0; }

    if (!this.moved && this.gravity) {
        this.velocity.y += 0.5 + this.acceleration * this.time;   // gravity
    }
    if (this.position.y > this.gameEnv.innerHeight) {
        this.position.y = 0;    // respawn at top
    }
}
```

</details>

---

## 5. Deployment

| Topic | Technology | File(s) | Evidence |
|-------|-----------|---------|----------|
| Docker | `eclipse-temurin:21-jdk-alpine` · `python:3.12` | [`hh_spring/Dockerfile`](https://github.com/Arnav210/hh_spring/blob/master/Dockerfile) · [`Arnav_hunger_heroes_backend/Dockerfile`](https://github.com/Arnav210/hunger_heroes_backend/blob/main/Dockerfile) | Maven Wrapper build; gunicorn Flask server |
| Docker Compose | Compose v3 | [`hh_spring/docker-compose.yml`](https://github.com/Arnav210/hh_spring/blob/master/docker-compose.yml) | Port 8286, volume mounts, prod profile, auto-restart |
| DNS | A records to subdomains | [`assets/js/api/config.js`](https://github.com/Ahaanv19/hunger_heroes/blob/main/assets/js/api/config.js) | Public URLs hiding internal ports |
| nginx | Reverse proxy + CORS | [`hh_spring/nginx_for_flask_8585`](https://github.com/Arnav210/hh_spring/blob/master/nginx_for_flask_8585) | Subdomain routing to port 8286; allowlist for `*.opencodingsociety.com` |
| CI/CD | GitHub Actions + GitHub Pages | [`.github/workflows/jekyll-gh-pages.yml`](https://github.com/Ahaanv19/hunger_heroes/blob/main/.github/workflows/jekyll-gh-pages.yml) | Jekyll build + deploy on every `main` push |

### Docker

**hh_spring:** The Spring Boot backend is containerized using `eclipse-temurin:21-jdk-alpine` and built with Maven Wrapper inside the container:

<details markdown="1">
<summary>hh_spring — Dockerfile</summary>

```dockerfile
# hh_spring — Dockerfile
FROM eclipse-temurin:21-jdk-alpine
WORKDIR /app
RUN apk update && apk upgrade && apk add --no-cache git
COPY . /app
RUN ./mvnw package
CMD ["java", "-jar", "target/spring-0.0.1-SNAPSHOT.jar"]
EXPOSE 8286
```

</details>

`docker-compose.yml` orchestrates the container with volume mounts and auto-restart:

<details markdown="1">
<summary>hh_spring — docker-compose.yml</summary>

```yaml
# hh_spring — docker-compose.yml
version: '3'
services:
  web:
    image: java_springv1
    build: .
    ports:
      - "8286:8286"
    volumes:
      - ./volumes:/app/volumes
    environment:
      - SPRING_PROFILES_ACTIVE=prod
    restart: unless-stopped
```

</details>

**Arnav_hunger_heroes_backend (Flask):**

<details markdown="1">
<summary>Arnav_hunger_heroes_backend — Dockerfile</summary>

```dockerfile
# Arnav_hunger_heroes_backend — Dockerfile
FROM docker.io/python:3.12
COPY . /
RUN pip install --no-cache-dir -r requirements.txt
RUN pip install gunicorn
ENV FLASK_ENV=production
ENV GUNICORN_CMD_ARGS="--workers=1 --bind=0.0.0.0:8288"
EXPOSE 8288
CMD ["gunicorn", "main:app"]
```

</details>

---

### DNS

Both backends are mapped to public domain names via DNS `A` records. Users never see a port number.

| Service | Public URL | Internal Port |
|---------|-----------|---------------|
| Flask backend | `https://hungerheros.opencodingsociety.com` | `8288` |
| Spring backend | `https://hungerherosspring.opencodingsociety.com` | `8286` |

These URIs are configured in the frontend:

<details markdown="1">
<summary>hunger_heroes — assets/js/api/config.js</summary>

```js
// hunger_heroes — assets/js/api/config.js
const SPRING_URI = 'https://hungerherosspring.opencodingsociety.com';
const FLASK_URI  = 'https://hungerheros.opencodingsociety.com';
```

</details>

---

### nginx (Reverse Proxy)

nginx routes HTTPS traffic on port 443 to the internal Spring/Flask ports and handles CORS allowlisting for `opencodingsociety.com` subdomains:

<details markdown="1">
<summary>hh_spring — nginx_for_flask_8585</summary>

```nginx
# hh_spring — nginx_for_flask_8585
server {
    server_name hungerherosspring.opencodingsociety.com;

    location / {
        proxy_pass http://localhost:8286;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # CORS — only allow opencodingsociety.com subdomains
        set $cors_origin "";
        if ($http_origin ~* "^https://(.*\.)?opencodingsociety\.com$") {
            set $cors_origin $http_origin;
        }
        if ($request_method = OPTIONS) {
            add_header "Access-Control-Allow-Credentials" "true" always;
            add_header "Access-Control-Allow-Origin" $cors_origin always;
            add_header "Access-Control-Allow-Methods" "GET, POST, PUT, OPTIONS, HEAD" always;
            return 204;
        }
    }
}
```

</details>

---

### CI/CD

The Jekyll frontend auto-builds and deploys to GitHub Pages on every push to `main`:

<details markdown="1">
<summary>hunger_heroes — .github/workflows/jekyll-gh-pages.yml</summary>

```yaml
# hunger_heroes — .github/workflows/jekyll-gh-pages.yml
name: Deploy Jekyll with GitHub Pages dependencies preinstalled
on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.1'
          bundler-cache: true
      - name: Install Jekyll and dependencies
        run: gem install bundler && bundle install
      - name: Install Python dependencies
        run: |
          python -m venv venv
          source venv/bin/activate
          pip install -r requirements.txt
      - name: Execute conversion script
        run: source venv/bin/activate && python scripts/convert_notebooks.py
      - name: Build registered project assets
        run: make build-registered-projects
      - name: Build with Jekyll
        run: bundle exec jekyll build
      - uses: actions/upload-pages-artifact@v3

  deploy:
    environment:
      name: github-pages
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/deploy-pages@v4
```

</details>

---

## 6. Documentation

| Topic | File(s) | What It Shows |
|-------|---------|---------------|
| Code Comments (Javadoc) | [`CategoryTree.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/CategoryTree.java) · [`DonationService.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/donation/DonationService.java) · [`HungerHeroScore.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/hungerheroes/HungerHeroScore.java) | Class-level + method Javadoc with `@param`, `@return`, `@throws`, Big-O |
| API Documentation | [`HungerHeroScoreController.java`](https://github.com/Arnav210/hh_spring/blob/master/src/main/java/com/open/spring/mvc/hungerheroes/HungerHeroScoreController.java) · [`BACKEND_GAME_SPRING_PROMPT.md`](https://github.com/Ahaanv19/hunger_heroes/blob/main/BACKEND_GAME_SPRING_PROMPT.md) | Endpoint table in Javadoc; full request/response schemas |
| Help System | Core feature page | In-game NPC dialogue guides users through donation flow |
| Blog Portfolio | GitHub Issues | Planning, proposals, task divisions documented per sprint |

### Code Comments (JavaDoc)

`CategoryTree.java` includes class-level and method-level Javadoc documenting complexity, data structures, and parameter contracts:

<details markdown="1">
<summary>hh_spring — CategoryTree.java</summary>

```java
// hh_spring — CategoryTree.java

/**
 * Tree-based hierarchical classification of food categories.
 *
 * Tree Operations:
 *  - DFS Traversal  — pre-order traversal to list all categories. Time: O(n).
 *  - Search         — finds a node by name using DFS. Time: O(n).
 *  - Path to Root   — traces ancestry from leaf to root. Time: O(h).
 *  - Count Leaves   — counts terminal categories. Time: O(n).
 *
 * @author Ahaan
 * @version 1.0
 */
public class CategoryTree {

    /**
     * Adds a child node to this category.
     *
     * @param child the child category node
     * @return the child node (for chaining)
     */
    public CategoryNode addChild(CategoryNode child) {
        child.parent = this;
        children.add(child);
        return child;
    }

    /**
     * Returns the depth of this node in the tree (root = 0).
     *
     * @return the depth level
     */
    public int getDepth() { ... }
}
```

</details>

`DonationService.java` documents every public method with `@param`, `@return`, and `@throws`:

<details markdown="1">
<summary>hh_spring — DonationService.java</summary>

```java
// hh_spring — DonationService.java

/**
 * Creates a new donation, generating a unique ID and logging the initial status.
 *
 * @param donation the donation entity to persist
 * @return the saved donation with generated ID
 * @throws IllegalArgumentException if validation fails
 */
public Donation createDonation(Donation donation) { ... }

/**
 * Marks a donation as "accepted".
 *
 * @param donationId the external donation ID
 * @param acceptedBy who is accepting the donation
 * @return the updated donation
 * @throws NoSuchElementException  if not found
 * @throws IllegalStateException   if status transition is invalid
 */
public Donation acceptDonation(String donationId, String acceptedBy) { ... }
```

</details>

`HungerHeroScore.java` documents the server-side score formula:

<details markdown="1">
<summary>hh_spring — HungerHeroScore.java</summary>

```java
// hh_spring — HungerHeroScore.java

/**
 * Compute score server-side. Call this before persisting.
 *
 * Formula:
 *   score = (npcsVisited * 100) + (dialoguesCompleted * 25) + timeBonus
 *   timeBonus = max(0, 300 - timePlayedSeconds)
 */
public void computeScore() {
    int npcPoints      = (npcsVisited != null ? npcsVisited : 0) * 100;
    int dialoguePoints = (dialoguesCompleted != null ? dialoguesCompleted : 0) * 25;
    int timeBonus      = Math.max(0, 300 - (timePlayedSeconds != null ? timePlayedSeconds : 0));
    this.score = npcPoints + dialoguePoints + timeBonus;
}
```

</details>

---

### API Documentation

`HungerHeroScoreController.java` documents all endpoints in its class-level Javadoc:

<details markdown="1">
<summary>hh_spring — HungerHeroScoreController.java</summary>

```java
// hh_spring — HungerHeroScoreController.java

/**
 * REST Controller for Hunger Heroes Game Leaderboard.
 *
 * Endpoints:
 *   GET  /api/hunger-heroes/leaderboard                → all scores (?levelId=&limit=)
 *   GET  /api/hunger-heroes/leaderboard/top/{limit}    → top N scores
 *   POST /api/hunger-heroes/leaderboard                → submit score
 *   GET  /api/hunger-heroes/leaderboard/user/{personId}→ user's scores
 *
 * CORS: Allow all origins for GET, require auth for POST.
 */
```

</details>

The `BACKEND_GAME_SPRING_PROMPT.md` and `BACKEND_TRAINING_HUB_SPRING_PROMPT.md` files in the `hunger_heroes` repo serve as full API reference documents, specifying request/response schemas for every endpoint.

---

### Help System

The core feature layout has a built in help system designed to guide users through the steps of creating a donation, finding food, and volunteering. By converting this process into a small game, it makes it easier for all users to navigate with ease. [Core Feature Page](https://ahaanv19.github.io/hunger_heroes/donate/)

---

### Blog Portfolio

All of our planning is thoroughly documented through blogs and issues in our repository, as shown below.

| Post | Topic |
|------|-------|
| [Team Research Plan](https://github.com/Ahaanv19/hunger_heroes/issues/9) | Guidelines for structuring research |
| [Project Proposal and More](https://github.com/Ahaanv19/hunger_heroes/issues/13) | Idea, Feedback, Prototype |
| [Task Divisions](https://github.com/Ahaanv19/hunger_heroes/issues/27/) | Delegations of tasks to each group member |

---

## 7. Project Impact & Ethics

### Project Impact

Hunger Heroes addresses the fact that **30–40% of food in the US ends up as waste** while millions face food insecurity. The platform:

- Connects surplus-food donors with nearby shelters matched by ZIP code
- Gamifies participation (leaderboard, training hub) to drive ongoing engagement
- Tracks donation lifecycle from creation → accepted → delivered with full audit logs (`DonationStatusLog`)

---

### Ethical Considerations

- **Food safety:** `Donation.java` tracks `allergens`, `dietaryTags`, `expiryDate`, and `specialInstructions` to protect recipients.
- **Allergen transparency:** Required allergen disclosure in both the Spring entity and Flask model.
- **Security:** nginx enforces CORS allowlists; `HungerHeroScoreController.java` clamps and validates all inputs server-side — never trusting the client.
- **Privacy:** `credentials: 'include'` with httpOnly cookies for auth; tokens never stored in `localStorage`.
- **Data integrity:** `DonationService.java` uses `@Transactional` and the undo-stack pattern to prevent corrupt lifecycle transitions.
