## Red Flags — AI Code Generation Pitfalls

These are patterns that AI (including you) tends to generate that create silent bugs. **Never generate these patterns. Ever.**

### #1: Silent Null Guards (THE BIGGEST OFFENDER)

AI loves to "protect" code by adding null checks that silently return. This hides real bugs and breaks business logic.

```csharp
// BAD — AI-generated "safe" code. Silent failure. Game breaks with no error.
private void Start()
{
    m_gm = FindFirstObjectByType<GameManager>();
}

private void Update()
{
    if (m_gm == null) return;  // ← SILENT BUG. If GameManager is missing, 
                                //   spawner silently does nothing. No error.
                                //   Developer spends 2 hours wondering why 
                                //   monsters don't spawn.
    SpawnMonster();
}

// BAD — Same problem everywhere AI adds "safety"
public void TakeDamage(int amount)
{
    if (m_gm == null) return;   // ← Monster takes damage but score never updates.
    m_gm.AddScore(m_scoreValue); //   Player kills 50 monsters, score stays at 0.
}                                //   No error. No clue why.

// BAD — Null-conditional that hides broken references
public void PlayerDied()
{
    m_onGameOver?.Invoke();  // ← If UnityEvent is null because someone forgot
                              //   to wire it, game over never triggers. Silent.
}
```

**The rule: If something MUST exist for the game to work, NEVER null-guard it. Let it crash.**

```csharp
// GOOD — Crash loudly. Developer sees the error immediately.
private void Start()
{
    m_gm = FindFirstObjectByType<GameManager>();
    Debug.Assert(m_gm != null, $"[{GetType().Name}] GameManager not found in scene!");
}

private void Update()
{
    // No null check. If m_gm is null, NullReferenceException points 
    // directly to this line. Developer fixes it in 30 seconds.
    SpawnMonster();
}

// GOOD — Validate at init time, trust at runtime
public void TakeDamage(int amount)
{
    m_gm.AddScore(m_scoreValue);  // Crashes if broken. Good.
}
```

### When Null Checks ARE Appropriate

| Situation | Null Check OK? | Why |
|-----------|---------------|-----|
| Optional feature (aim assist, tutorial) | Yes | Game works without it |
| `FindFirstObjectByType` for optional system | Yes | Not every scene has every manager |
| Required dependency (GameManager, Player) | **NO** | Game is broken without it — crash loudly |
| `[SerializeField]` that must be assigned | **NO** | Use `Debug.Assert` in `Awake()` or `OnValidate` |
| Event callbacks (`OnDeath`, `OnHit`) | **NO for ?.**  | Use `Invoke()` not `?.Invoke()` for required events |
| Destroyed objects in same frame | Yes | `Destroy()` is end-of-frame, object still exists briefly |
| **`OnTriggerEnter2D` / `OnCollisionEnter2D` callbacks** | **YES — ALWAYS** | **Filter by type. Collisions fire for many objects you don't care about. This is CORRECT best practice, not a silent guard.** |
| **`GetComponent<T>()` results that may not be present** | **YES — use `TryGetComponent`** | **The component genuinely may not exist on the other object.** |

### Important: Collision/Trigger Callbacks Are NOT Silent Null Guards

This deserves its own section because it's commonly confused with Red Flag #1:

```csharp
// GOOD — This is CORRECT practice, not a silent null guard
private void OnTriggerEnter2D(Collider2D other)
{
    if (other == null) return;  // ← FINE. Rare but possible during scene teardown.
    
    if (!other.CompareTag("Enemy")) return;  // ← FINE. Filter by type.
    
    if (!other.TryGetComponent<Monster>(out var monster)) return;  // ← FINE.
                                                                    //   Monster may not exist.
    
    monster.TakeDamage(m_damage);
    Destroy(gameObject);
}
```

**Why this is correct, not a "silent guard":**

1. **Collisions fire for many different objects** — walls, items, monsters, other bullets, particles. You MUST filter to find the ones you care about.
2. **The "null" or "wrong type" case is expected, not broken** — a bullet hitting a wall is normal gameplay, not a bug. Returning silently is the correct response.
3. **You're not hiding a broken dependency** — you're handling the many valid cases where the collision isn't something you want to process.

The distinction:

| Pattern | Interpretation | Correct? |
|---------|---------------|----------|
| `if (m_gameManager == null) return;` in `Update()` | Broken required dependency, hidden | ❌ Silent guard |
| `if (other == null) return;` in `OnTriggerEnter2D` | Filtering unexpected collider cases | ✅ Correct |
| `if (!other.CompareTag("Enemy")) return;` | Filtering collisions by type | ✅ Correct |
| `if (!other.TryGetComponent<Monster>(out var m)) return;` | Checking if collision target has the component | ✅ Correct |
| `if (m_monster == null) return;` after `GetComponent<Monster>()` in `Awake()` | Required sibling component missing | ❌ Use `Debug.Assert` |

### Best Practice: Use `TryGetComponent`, Not `GetComponent` + Null Check

For collision callbacks, always prefer `TryGetComponent` over `GetComponent` + null check. It's faster (no exception allocation on failure) and reads cleaner:

```csharp
// GOOD — idiomatic Unity collision handling
private void OnTriggerEnter2D(Collider2D other)
{
    // Fast path: tag check first (cheaper than component lookup)
    if (!other.CompareTag("Enemy")) return;
    
    // Then get the component we need, if it exists
    if (!other.TryGetComponent<Monster>(out var monster)) return;
    
    monster.TakeDamage(m_damage);
    Destroy(gameObject);
}

// OK but less idiomatic
private void OnTriggerEnter2D(Collider2D other)
{
    var monster = other.GetComponent<Monster>();
    if (monster == null) return;  // Valid — bullet may hit non-monster
    
    monster.TakeDamage(m_damage);
    Destroy(gameObject);
}

// BAD — no filtering, crashes on non-Monster collisions
private void OnTriggerEnter2D(Collider2D other)
{
    other.GetComponent<Monster>().TakeDamage(m_damage);  // ← NullRef when bullet hits wall
}
```

### The Decision Framework

Before writing any null check, ask: **"If this is null, is the game broken?"**

- **Yes → Don't guard. Let it crash. Add `Debug.Assert` at init time.**
- **No, this is an expected case (collision filtering, optional lookup) → Guard normally.**
- **No, but this is a required dependency → Don't guard. Crash loudly.**
- **Never → Silent return on required dependencies with no log.** This is always wrong.

### #2: Defensive Returns That Break Game Flow

```csharp
// BAD — AI adds "safety" that breaks the death sequence
public void PlayerDied()
{
    if (!IsRunning) return;          // ← What if called twice? Fine to guard.
    if (m_state == GameState.GameOver) return;  // ← OK, prevents double-death.
    
    // But AI also adds these:
    if (m_onGameOver == null) return;  // ← BAD. If event is null, game over 
                                        //   screen never shows. Player is stuck.
    if (Score < 0) return;             // ← BAD. Why? Score can't be negative?
                                        //   Now PlayerDied silently fails for 
                                        //   some unknown "safety" reason.
}
```

**Rule:** Only guard against states that are logically possible and recoverable. Don't guard against states that "shouldn't happen" — if they happen, you want to know.

### #3: Try-Catch That Swallows Everything

```csharp
// BAD — AI wraps everything in try-catch "for safety"
public void SpawnWave()
{
    try
    {
        // ... spawn logic ...
    }
    catch (System.Exception)
    {
        // Silently swallowed. Wave never spawns. No error in console.
    }
}

// GOOD — Don't catch unless you can handle it. Let Unity show the error.
public void SpawnWave()
{
    // ... spawn logic ... 
    // If it throws, you see the red error in Console. Good.
}
```

### #4: Isolated Changes — Editing One System Without Updating Related Systems

This is the AI's blind spot. AI reads the one file you asked about, makes the change, and stops. It doesn't trace the ripple effects to every system that depends on the changed behavior.

```
Example: User asks "change terrain collision to use layers instead of tags"

What AI does:
  ✅ Updates TerrainCollider to use layer checks
  ❌ Doesn't update Bullet — still uses tag check, now phases through terrain
  ❌ Doesn't update Monster — still uses tag check, walks through walls
  ❌ Doesn't update Item — falls through floor
  ❌ Doesn't update physics matrix — new layer has no collision rules

Result: Terrain "works" for the player. Everything else is broken.
```

**This happens because AI treats each request as isolated.** It doesn't ask "what else touches this system?"

**The rule: Before making ANY change, trace the full impact chain.**

Every change must answer these questions:

1. **What other scripts reference the thing I'm changing?** (Search for class name, method name, enum value, layer name, tag name)
2. **What other prefabs use the same collision/trigger logic?** (If you change how Player detects terrain, every entity that touches terrain needs the same update)
3. **What physics layers / collision matrix entries need updating?** (Adding or changing a layer means updating the matrix)
4. **What managers coordinate this behavior?** (If Monster.TakeDamage changes, does GameManager.AddScore still get called correctly?)

```
CORRECT approach for "change terrain collision to layers":

1. List every script that references terrain:
   - PlayerController → OnCollisionEnter2D with terrain tag → UPDATE
   - Bullet → OnTriggerEnter2D destroy on terrain tag → UPDATE
   - Monster → ground check uses terrain tag → UPDATE
   - Item → falls until terrain contact → UPDATE

2. List physics layer changes:
   - Create Terrain layer (index 11)
   - Update collision matrix: Player↔Terrain, Monster↔Terrain, Item↔Terrain, Bullet↔Terrain

3. List prefab changes:
   - All terrain tilemap/collider objects → set layer to Terrain
   - Remove "Terrain" tag usage (or keep for non-physics purposes)

4. Deliver ALL changes together, not just the one file.
```

**How to enforce this when generating code:**

When the user asks to change something, ALWAYS output an **Impact Analysis** before writing code:

```
## Impact Analysis: [Change Description]

### Direct changes:
- TerrainCollider.cs — switch from CompareTag to layer check

### Ripple effects:
- Bullet.cs — line 34: OnTriggerEnter2D uses CompareTag("Terrain") → update to layer check
- Monster.cs — line 67: ground detection uses CompareTag("Terrain") → update to layer check  
- Item.cs — line 22: OnCollisionEnter2D uses CompareTag("Terrain") → update to layer check
- Physics Matrix — add Terrain layer (11), configure collision pairs
- Prefabs — all terrain objects need layer reassignment

### No impact:
- GameManager.cs — doesn't reference terrain directly
- MonsterSpawner.cs — spawns at transform positions, no terrain interaction
```

If you don't know what other scripts exist in the project, **ASK**. "What other scripts interact with terrain? Can you share them so I can update everything together?" is infinitely better than delivering a partial change that breaks 4 other systems.

### #5: Sunk Cost — Hacking Old Code Instead of Replacing It

AI treats existing code as sacred. When requirements change fundamentally, AI keeps patching the old solution with hacks instead of recognizing that the entire approach is wrong and needs to be replaced.

```csharp
// Original requirement: "I need a ground vehicle"
// AI builds this:
public class Car : MonoBehaviour
{
    [SerializeField] private float m_speed = 10f;
    [SerializeField] private float m_turnRate = 90f;
    private Rigidbody m_rb;

    private void Update()
    {
        float h = Input.GetAxis("Horizontal");
        float v = Input.GetAxis("Vertical");
        transform.Rotate(0f, h * m_turnRate * Time.deltaTime, 0f);
        m_rb.linearVelocity = transform.forward * v * m_speed;
    }
}

// New requirement: "Actually I need it to fly"
// BAD — AI hacks flying into the car:
public class Car : MonoBehaviour  // ← Still called "Car"
{
    [SerializeField] private float m_speed = 10f;
    [SerializeField] private float m_turnRate = 90f;
    [SerializeField] private bool m_canFly = false;       // ← bolt-on flag
    [SerializeField] private float m_flySpeed = 15f;      // ← bolt-on
    [SerializeField] private float m_liftForce = 5f;      // ← bolt-on
    [SerializeField] private float m_pitchRate = 45f;     // ← bolt-on

    private void Update()
    {
        if (m_canFly)                                      // ← branching hell
        {
            float pitch = Input.GetAxis("Vertical");
            float yaw = Input.GetAxis("Horizontal");
            float lift = Input.GetKey(KeyCode.Space) ? m_liftForce : -9.8f;
            transform.Rotate(pitch * m_pitchRate * Time.deltaTime, 
                           yaw * m_turnRate * Time.deltaTime, 0f);
            m_rb.linearVelocity = transform.forward * m_flySpeed 
                         + Vector3.up * lift;              // ← gravity hack
        }
        else
        {
            // original ground code, now dead weight when flying
            float h = Input.GetAxis("Horizontal");
            float v = Input.GetAxis("Vertical");
            transform.Rotate(0f, h * m_turnRate * Time.deltaTime, 0f);
            m_rb.linearVelocity = transform.forward * v * m_speed;
        }
    }
}
// Result: A "Car" that flies. Ground code is dead weight. Flight physics 
// are hacked in. Adding hover mode later = another if-branch. Adding boat 
// mode = another. Class becomes unmaintainable spaghetti.
```

**The correct response: Delete and rebuild.**

```csharp
// GOOD — Recognize the requirement changed. Delete Car. Build what's actually needed.
public class Aircraft : MonoBehaviour
{
    [Header("Flight")]
    [SerializeField, Range(5f, 50f), Tooltip("Forward speed in units/sec")]
    private float m_flySpeed = 15f;
    
    [SerializeField, Range(1f, 20f), Tooltip("Lift force when ascending")]
    private float m_liftForce = 5f;
    
    [SerializeField, Range(15f, 90f), Tooltip("Pitch rotation speed")]
    private float m_pitchRate = 45f;
    
    [SerializeField, Range(15f, 120f), Tooltip("Yaw rotation speed")]
    private float m_yawRate = 90f;

    private Rigidbody m_rb;

    private void Awake()
    {
        m_rb = GetComponent<Rigidbody>();
        m_rb.useGravity = false;  // Flight handles its own gravity
    }

    private void Update()
    {
        HandleRotation();
        HandleThrust();
    }

    private void HandleRotation() { /* clean flight rotation */ }
    private void HandleThrust() { /* clean flight physics */ }
}
// Clean. No dead code. No if-branches. Name matches what it is.
// If you later need BOTH car and aircraft, use composition: 
// GroundMovement component + FlightMovement component on the same prefab.
```

**Why AI does this:**
- AI sees existing code as "progress" and doesn't want to "lose" it
- AI tries to satisfy the new requirement with minimum diff to existing code
- AI doesn't step back and ask "is the current approach still the right foundation?"

**The signals that the approach needs replacing, not patching:**

| Signal | What AI Does (Wrong) | What It Should Do |
|--------|---------------------|-------------------|
| New feature needs `if (mode == X)` branches everywhere | Adds booleans and branches | Delete. Build for the new requirement. Compose if both are needed. |
| Class name no longer describes what it does | Renames but keeps the guts | Delete. New class, new name, new structure. |
| Original assumptions are invalidated | Wraps old logic in guards | Delete. Rebuild from the new assumptions. |
| "Make X work like Y" where X and Y are fundamentally different | Hacks Y's behavior into X | Delete X. Build Y. |
| Adding the feature requires touching 5+ methods | Surgically edits each method | Step back. The architecture doesn't support this feature. Redesign. |
| You're adding a `mode` or `type` enum to switch behavior | Adds enum + switch in Update | Separate components. One per behavior. Compose on the prefab. |

**The decision framework:**

Before modifying existing code for a new requirement, ask:

1. **"Does the existing class/system still represent what this thing IS?"** → If no, delete and rebuild.
2. **"Am I adding branches (`if/else`, `switch`) to handle the new case?"** → If it touches 3+ methods, the approach is wrong. Split into components or rebuild.
3. **"Would it be faster to write this from scratch than to modify what exists?"** → If yes, do that. Existing code has no inherent value.
4. **"Am I keeping old code alive that's now dead weight?"** → If any code path is only reached in the old mode, it's dead. Delete it.

**The rule: Code is disposable. Requirements are not. When requirements change, the code serves the requirement — not the other way around. Delete freely.**

### #6: Hard Manager Dependencies — Entities That Can't Work Alone

AI wires every entity directly to managers as if they're required. Drag a Player prefab into an empty test scene → `NullReferenceException` in `Start()` because `GameManager` doesn't exist. Drag a Monster prefab → crashes because it can't find `LevelManager`. Every prefab is broken without the full scene.

**This violates the core principle: a prefab must work when Instantiated.**

An artist should be able to drag a Monster into an empty scene, hit Play, and see it move left. A designer should be able to drag a Player into a test scene, hit Play, and control it. No managers needed. No full scene required.

```csharp
// BAD — Player crashes without GameManager
public class Player : MonoBehaviour
{
    private GameManager m_gm;

    private void Start()
    {
        m_gm = FindFirstObjectByType<GameManager>();
        m_gm.RegisterPlayer(this);  // ← NullRef in empty test scene. 
                                     //   Can't test player movement alone.
    }

    private void Update()
    {
        HandleMovement();             // ← This works alone...
        HandleShooting();             // ← This works alone...
        m_gm.UpdateComboMeter(0.1f);  // ← But this crashes without GameManager.
                                       //   Player is untestable in isolation.
    }

    private void OnTriggerEnter2D(Collider2D other)
    {
        if (other.CompareTag("Enemy"))
        {
            m_gm.PlayerDied();         // ← Crash. Can't test collision in 
                                        //   isolation either.
            gameObject.SetActive(false);
        }
    }
}
```

**The rule: Core entity behavior (movement, animation, physics) must NEVER depend on managers. Manager integration is optional — the entity works without it.**

```csharp
// GOOD — Player works completely alone. Manager integration is safe-optional.
public class Player : MonoBehaviour
{
    private GameManager m_gm;  // nullable — Player works without it

    private void Start()
    {
        // FindFirstObjectByType returns null if not found. That's fine.
        m_gm = FindFirstObjectByType<GameManager>();
    }

    private void Update()
    {
        HandleMovement();   // Works alone. No manager dependency.
        HandleShooting();   // Works alone. No manager dependency.
    }

    private void OnTriggerEnter2D(Collider2D other)
    {
        if (other.CompareTag("Enemy"))
        {
            // Manager integration — safe if manager doesn't exist
            m_gm?.PlayerDied();  // ← HERE ?.  is correct. Manager IS optional 
                                  //   for the entity to function. Player still 
                                  //   dies (SetActive false). Game over screen 
                                  //   just won't show in a test scene. That's fine.
            gameObject.SetActive(false);
        }
    }
}
```

**Wait — doesn't this contradict Red Flag #1 (no silent null guards)?**

No. The distinction is:

| Dependency Type | Null Guard? | Why |
|----------------|------------|-----|
| **Entity's OWN components** (Rigidbody, Collider, own config) | **NO** — `Debug.Assert` | Entity is broken without its own parts. Crash loudly. |
| **Cross-system managers** (GameManager, LevelManager, ScoreManager) | **YES** — `?.` is OK | Entity's core behavior (move, shoot, animate) must work without managers. Manager integration is a bonus. |
| **Required-at-game-level** (GameManager in the real game scene) | **YES for entity**, validated elsewhere | The SCENE validates that GameManager exists. The entity doesn't enforce it. |

**The test: Drag the prefab into an empty scene. Hit Play.**

| What Should Work | What's OK To Skip |
|-----------------|-------------------|
| Movement, input, physics | Score updates |
| Shooting, collision, animation | Wave progression |
| Visual feedback (flash on hit, death anim) | UI notifications |
| Sound effects (if AudioSource is on prefab) | Manager-driven events |
| AI behavior (patrol, chase, attack) | Global difficulty scaling |
| Self-destruction on death | Spawn chain reactions |

**How to structure this in code:**

```csharp
// Entity layer: works alone, zero external dependencies
public class Monster : MonoBehaviour
{
    // OWN components — required, Assert in Awake
    private Rigidbody2D m_rb;
    
    // OWN config — on the prefab, always present
    [SerializeField] private float m_moveSpeed = 3f;
    [SerializeField] private int m_hp = 10;
    
    private void Awake()
    {
        m_rb = GetComponent<Rigidbody2D>();
        Debug.Assert(m_rb != null, $"[{GetType().Name}] Missing Rigidbody2D!");
    }

    private void Start()
    {
        // Core behavior — no dependencies
        m_rb.linearVelocity = Vector2.left * m_moveSpeed;
    }

    public void TakeDamage(int amount)
    {
        m_hp -= amount;
        if (m_hp <= 0)
            Die();
    }

    private void Die()
    {
        // Core behavior: destroy self. Works in any scene.
        Destroy(gameObject);
    }
}

// Integration layer: connects entity to game systems (separate component or in-entity with null safety)
public class MonsterGameIntegration : MonoBehaviour
{
    private GameManager m_gm;
    private Monster m_monster;

    [SerializeField] private int m_scoreValue = 10;

    private void Awake()
    {
        m_monster = GetComponent<Monster>();
    }

    private void Start()
    {
        m_gm = FindFirstObjectByType<GameManager>();
    }

    private void OnDestroy()
    {
        // Only update score if GameManager exists
        m_gm?.AddScore(m_scoreValue);
    }
}
```

For small projects, the integration can live in the entity itself — just keep the **mental separation** clear: core behavior never touches managers, manager calls are always null-safe.

### #7: Responsibility Bloat — Expanding a Class Instead of Extracting a New One

When scope grows, AI bloats the existing class instead of extracting a new class with the right responsibility. The original class was built for ONE thing; when asked to handle MANY things, AI adds arrays, events, switching logic, and state management — all inside the original class. The class becomes a god object that violates single responsibility.

**The fundamental question AI fails to ask:** "Is this new requirement the SAME responsibility as the existing class, or a DIFFERENT responsibility?"

```csharp
// Original requirement: "I need a weapon"
// AI builds Weapon class holding weapon data + firing behavior:
public class Weapon : MonoBehaviour
{
    [SerializeField] private WeaponData m_data;  // damage, fireRate, ammo
    [SerializeField] private Transform m_firePoint;
    [SerializeField] private Bullet m_bulletPrefab;
    
    private float m_fireTimer;
    private int m_currentAmmo;

    public void Fire() { /* ... */ }
    public void Reload() { /* ... */ }
}

// New requirement: "Actually player needs multiple weapons, switchable"
//
// BAD — AI bloats Weapon class:
public class Weapon : MonoBehaviour
{
    [SerializeField] private WeaponData[] m_weaponDataArray;      // ← NEW: array
    [SerializeField] private Transform[] m_firePoints;             // ← NEW: array
    [SerializeField] private Bullet[] m_bulletPrefabs;             // ← NEW: array
    [SerializeField] private int m_currentWeaponIndex;             // ← NEW: state
    [SerializeField] private UnityEvent<int> m_onWeaponSwitched;   // ← NEW: event

    private float[] m_fireTimers;         // ← per-weapon timers now
    private int[] m_currentAmmoArray;     // ← per-weapon ammo now

    public void Fire() 
    {
        // BAD: Fire logic now indexes into arrays everywhere
        var data = m_weaponDataArray[m_currentWeaponIndex];
        var firePoint = m_firePoints[m_currentWeaponIndex];
        var prefab = m_bulletPrefabs[m_currentWeaponIndex];
        m_fireTimers[m_currentWeaponIndex] = data.fireRate;
        m_currentAmmoArray[m_currentWeaponIndex]--;
        // ...
    }

    public void SwitchWeapon(int index) { /* ... */ }  // ← NEW: switching logic
    public void AddWeapon(WeaponData data) { /* ... */ } // ← NEW: inventory logic
    public void RemoveWeapon(int index) { /* ... */ }    // ← NEW: inventory logic
    public int GetWeaponCount() => m_weaponDataArray.Length;  // ← NEW
    public WeaponData GetCurrentWeapon() => m_weaponDataArray[m_currentWeaponIndex];
    // ... 10 more methods ...
}
// Problems:
// - Can't test one weapon in isolation
// - "Weapon" class now manages inventory, which isn't its job
// - Every weapon behavior is coupled through array indexing
// - Adding a 3rd concept (e.g. dual-wield) = more arrays, more bloat
// - The class name "Weapon" no longer describes what it does
```

**The correct response: Extract a new class with the new responsibility.**

```csharp
// GOOD — Weapon stays simple. Extract WeaponInventory for the new responsibility.

// Weapon: represents ONE weapon. Same as before, testable in isolation.
public class Weapon : MonoBehaviour
{
    [SerializeField] private WeaponData m_data;
    [SerializeField] private Transform m_firePoint;
    [SerializeField] private Bullet m_bulletPrefab;
    
    private float m_fireTimer;
    private int m_currentAmmo;

    public WeaponData Data => m_data;
    public bool IsReady => m_fireTimer <= 0f;

    public void Fire() { /* unchanged */ }
    public void Reload() { /* unchanged */ }
}

// WeaponInventory: NEW class with the NEW responsibility — manage multiple weapons
public class WeaponInventory : MonoBehaviour
{
    [Header("Weapons")]
    [SerializeField, Tooltip("Weapon prefabs held in inventory")]
    private Weapon[] m_weaponPrefabs;

    [SerializeField, Tooltip("Transform where the active weapon is attached")]
    private Transform m_weaponHoldPoint;

    [Header("Events")]
    [SerializeField] private UnityEvent<Weapon> m_onWeaponSwitched;

    private Weapon[] m_weaponInstances;
    private int m_currentIndex;

    public Weapon CurrentWeapon => m_weaponInstances[m_currentIndex];
    public int WeaponCount => m_weaponInstances.Length;

    private void Awake()
    {
        // Instantiate all weapons, activate only the current one
        m_weaponInstances = new Weapon[m_weaponPrefabs.Length];
        for (int i = 0; i < m_weaponPrefabs.Length; i++)
        {
            m_weaponInstances[i] = Instantiate(m_weaponPrefabs[i], m_weaponHoldPoint);
            m_weaponInstances[i].gameObject.SetActive(i == m_currentIndex);
        }
    }

    public void SwitchToWeapon(int index)
    {
        if (index < 0 || index >= m_weaponInstances.Length) return;
        if (index == m_currentIndex) return;

        m_weaponInstances[m_currentIndex].gameObject.SetActive(false);
        m_currentIndex = index;
        m_weaponInstances[m_currentIndex].gameObject.SetActive(true);
        m_onWeaponSwitched?.Invoke(CurrentWeapon);
    }

    public void SwitchNext()
    {
        SwitchToWeapon((m_currentIndex + 1) % m_weaponInstances.Length);
    }
}

// Player uses WeaponInventory — doesn't need to know about individual weapons
public class Player : MonoBehaviour
{
    private WeaponInventory m_inventory;

    private void Awake()
    {
        m_inventory = GetComponentInChildren<WeaponInventory>();
    }

    private void Update()
    {
        if (Input.GetKey(KeyCode.Space) && m_inventory.CurrentWeapon.IsReady)
            m_inventory.CurrentWeapon.Fire();

        if (Input.GetKeyDown(KeyCode.Q))
            m_inventory.SwitchNext();
    }
}
```

**Benefits of extraction:**
- `Weapon` class unchanged — all existing tests still pass
- `Weapon` still testable in isolation — drag one into a scene, it works
- `WeaponInventory` has ONE responsibility: managing the weapon collection
- Adding dual-wield later = new class `DualWieldInventory` or composition — no bloat to existing classes
- Each class's name describes what it IS

### The Signals: You Should Extract, Not Expand

| Signal | AI's Wrong Response | Correct Response |
|--------|--------------------|--------------------|
| New requirement needs to manage MULTIPLE of the existing thing | Add arrays + index tracking to existing class | Extract `XyzInventory` / `XyzManager` / `XyzCollection` class |
| New requirement adds switching, selecting, or cycling logic | Add current-index field + switch method to existing class | Extract a coordinator class that owns the collection |
| New requirement adds lifecycle (add/remove/unlock) | Add CRUD methods to existing class | Extract an inventory/registry class |
| Existing class's name describes one thing, new requirement is about many | Keep name, add arrays | New class named for the new responsibility |
| New requirement needs state ABOUT the existing things (which is active, which is unlocked) | Add state fields inside existing class | Extract — state belongs in the coordinator |
| Methods are gaining index parameters (`Fire(int index)`, `Reload(int index)`) | Keep adding index params | Extract — the index means you're managing a collection |

### The Responsibility Test

Before adding to an existing class, ask:

1. **"What is this class's ONE responsibility?"** If you can't answer in one sentence, it already has too many.
2. **"Does the new requirement fit that same responsibility, or is it a different one?"**
   - Same responsibility → add to existing class
   - Different responsibility → extract a new class
3. **"If I add this feature, will the class name still accurately describe what it does?"** If no → extract.
4. **"Am I adding 'manage many of X' logic to the 'one X' class?"** → **Always extract.** The collection is a separate concept from the item.

### Naming Conventions for Extracted Classes

When extracting, name the new class for its responsibility, not its relationship:

| Existing (One Thing) | New Class (Many Things) | Why |
|---------------------|------------------------|-----|
| `Weapon` | `WeaponInventory` | Holds multiple weapons, tracks which is active |
| `Enemy` | `EnemySpawner` or `EnemyRegistry` | Spawns/tracks multiple enemies |
| `Item` | `ItemInventory` | Collection of items |
| `Skill` | `SkillLoadout` | Selected skills from a pool |
| `DialogueLine` | `DialogueSequence` | Ordered sequence of lines |
| `Sound` | `AudioManager` or `SoundBank` | Collection + playback logic |
| `Quest` | `QuestJournal` | Active + completed quest tracking |
| `Buff` | `BuffContainer` | Stack of active buffs with timing |

### The Rule

**When a class gains responsibility for managing MULTIPLE of itself — extract. When a class gains a NEW responsibility unrelated to its name — extract. Bloating is always the wrong answer.**

The first question when requirements grow is not "how do I add this?" — it's "does this belong in the existing class, or does it need its own?"

### Summary: The AI Red Flag Checklist

Before generating ANY code:

1. **Is this a required dependency?** → No null guard. `Debug.Assert` at init, trust at runtime.
2. **Is this genuinely optional?** → Guard + `Debug.LogWarning` explaining why it's missing.
3. **Am I adding a null check "just in case"?** → **STOP. Remove it.** "Just in case" = silent bug.
4. **Would the game still work if this fails silently?** → If no, don't let it fail silently.
5. **Does this change touch a system that other scripts depend on?** → **Trace the full impact chain. List every affected file. Deliver ALL changes together.**
6. **Am I only updating one file when the change affects many?** → **STOP. Do the impact analysis first.**
7. **Am I hacking a new requirement into code that was built for something else?** → **STOP. Ask: should I delete this and rebuild? If the class name, structure, or assumptions no longer fit — delete and rebuild.**
8. **Am I adding `if (mode)` branches to handle fundamentally different behaviors?** → **STOP. Use separate components. Compose on the prefab.**
9. **Can this entity work if I drag it into an empty scene?** → If no, the entity has hard manager dependencies. **Core behavior (movement, physics, animation, AI) must work alone. Manager integration (score, waves, UI) is optional.**
10. **Am I adding arrays, switching logic, or "manage many" logic to a class built for ONE thing?** → **STOP. Extract a new class for the collection. `Weapon` ≠ `WeaponInventory`.**