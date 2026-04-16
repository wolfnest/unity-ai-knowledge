# Unity Development Standards (Solo / Duo Team)

## Before You Start — Choose Your Architecture

Ask the developer two questions before writing any code:

### 1. Data Architecture

| Approach | Best For | How It Works |
|----------|----------|-------------|
| **Prefab Only** | Small games, fast prototypes, solo dev, game jams | All config lives on prefabs as `[SerializeField]`. No SOs for entity config. Fastest to iterate — select prefab, tweak values, done. |
| **Prefab + SO** | Mid-size games, shared configs, team with designers | SOs hold pure data (stats, tuning, collection lists). Prefabs own visuals and behavior. SOs never bind to prefabs at runtime. |

**If unsure, start with Prefab Only.** You can always extract shared config into SOs later. You can never easily merge scattered SOs back into prefabs.

### 2. Communication & Wiring Architecture

| Approach | Best For | How It Works |
|----------|----------|-------------|
| **Unity Native** | Small games, prototypes, fast iteration | `[SerializeField]`, `FindFirstObjectByType`, direct method calls, `UnityEvent`. No frameworks. Fastest to get running. |
| **VContainer + EventBus** | Production scale, large games, 3+ devs | VContainer for DI wiring. MessagePipe or similar for decoupled events. Testable, scalable, but slower to set up. |

**If unsure, start with Unity Native.** Add VContainer when `FindFirstObjectByType` calls get messy or you need unit testing. Never hand-roll DI or event systems.

### Migration Path
```
Prototype:     Prefab Only + Unity Native
Growing:       Prefab + SO + Unity Native (extract shared data into SOs)
Production:    Prefab + SO + VContainer + EventBus (add DI and decoupled events)
```

These choices affect every section below — SO rules, prefab structure, communication patterns, and how managers are wired.

---

## The One Rule

**If a developer can't read the code and understand what it does, where it goes, and why — it's bad code.** No exceptions.

It doesn't matter if the code follows Clean Architecture, SOLID, DDD, or any other practice. If a developer opens a file and feels lost — the code is wrong, not the developer. Patterns exist to serve readability, not the other way around.

Good code reads like a story. One feature = traceable in one or two files. No jumping across 6 files to understand "what happens when a monster dies."

Everything below serves this principle.

---

Apply these standards when writing, reviewing, or suggesting Unity C# code. These rules are designed for small teams (1-2 developers) who need to move fast without creating architectural debt.

---

## 1. ScriptableObject = Pure Data Only

SOs are lightweight data containers. They load into memory as data — keep them that way.

**SO holds:**
- Numbers, strings, enums, booleans (stats, tuning, IDs, thresholds)
- Sprites ONLY for UI purposes (card art, banners, icons — not gameplay entities)
- Lists of prefab references for composition (shop items, wave monster lists, area configs)

**SO does NOT hold:**
- Prefab references that require instantiation to preview (forces artist to enter Play mode)
- Animation clips, particle systems, materials (heavyweight assets that bloat SO memory)
- Anything that belongs on the prefab itself (colliders, renderers, component config)
- Runtime mutable state (current HP, score, position)

**SO never binds TO a prefab at runtime.** If a spawner needs to create an entity, either:
- The SO contains a prefab reference in a list (LevelConfig → wave monster prefabs) — SO owns the reference
- Or skip the SO entirely and put config on the prefab as `[SerializeField]`

Never: spawn prefab → then attach SO to it. This breaks Inspector workflows and makes artist testing impossible without Play mode.

**Test:** "Is this pure data a designer could edit in a spreadsheet?" → SO. "Does this need instantiation to see the result?" → Prefab.

### When to Use SO vs Prefab Config
| Use SO | Use Prefab [SerializeField] |
|--------|---------------------------|
| Global tuning (DifficultyConfig) | Per-entity stats (Monster HP, Drone fire rate) |
| Collection lists (shop items, wave configs) | Visual setup (sprites, animators, colliders) |
| Shared data across multiple prefabs | Config unique to one prefab |
| UI data (card sprites, banner text) | Anything requiring Instantiate to test |

---

## 2. Prefab = Visual + Behavior (Self-Contained)

A prefab must work when Instantiated — no external `Initialize(8 params)` required.

### Structure Convention
```
EntityPrefab (Root)
├── MainComponent (Monster, Drone, Bullet)
│   └── [SerializeField] config / data fields
├── Rigidbody2D / Colliders (if needed)
├── Sprite (child)
│   ├── SpriteRenderer
│   └── SpriteAnimator (if animated, with serialized animation data)
└── Attachment points (child Transforms: FirePoint, SpawnPoint, etc.)
```

### Rules
- One root MonoBehaviour per prefab — it IS the entity
- Config SO is `[SerializeField]` on the root component, set in the prefab Inspector
- Visual setup (sprites, animation clips, materials) lives on the prefab, NOT in the SO
- Child structure: Root (logic + physics) > Sprite (visual) > attachment points
- Use `[RequireComponent]` when the root MonoBehaviour depends on sibling components:
  ```csharp
  [RequireComponent(typeof(Rigidbody2D))]
  [RequireComponent(typeof(BoxCollider2D))]
  public class Monster : MonoBehaviour { ... }
  ```
  This auto-adds the components when the script is attached and prevents accidental removal in Inspector.

---

## 3. MonoBehaviours Do the Work

No plain C# "service" classes that need manual constructor wiring. If logic needs Unity lifecycle → it's a MonoBehaviour.

### Self-Init Pattern
```csharp
// GOOD: Self-initializes, finds what it needs
public class MonsterSpawner : MonoBehaviour
{
    [SerializeField] private LevelConfig m_levelConfig;  // assigned in Inspector
    private GameManager m_gm;

    private void Start()
    {
        m_gm = FindFirstObjectByType<GameManager>();
    }
}

// BAD: Requires external wiring
public class MonsterSpawnService  // plain C# class
{
    public MonsterSpawnService(SignalBus bus, MonsterConfig[] configs,
        DifficultyService diff, GameSessionData data, ObjectPool pool,
        float spawnX, float yMin, float yMax, float leftEdgeX) { ... }
}
```

### Reference Strategy
| Need | Method | When |
|------|--------|------|
| Own children / config | `[SerializeField]` | Always first choice |
| Scene singletons | `FindFirstObjectByType<T>()` in `Start()` | Prototype / small projects |
| Cross-cutting dependencies | VContainer `[Inject]` | When project grows (50+ scripts, 3+ devs) |
| Never | `Initialize(param, param, param, ...)` chains | Creates hidden dependency order |

---

## 4. Communication Patterns

Choose based on the relationship:

| Pattern | When | Example |
|---------|------|---------|
| **Direct method call** | 1 sender → 1 known receiver | `weapon.Fire()`, `weapon.LevelUp()` |
| **UnityEvent** (serialized) | 1 sender → N receivers, wired in Inspector | `OnGameOver` → stop spawners, show UI, play SFX |
| **C# Action/event** | 1 sender → N receivers, wired in code | `Monster.OnHitPlayer += HandleDeath` |
| **EventBus** | Many senders → many receivers, fully decoupled | Only with a framework (VContainer's MessagePipe). Never hand-roll |

### What NOT to do
- Don't hand-roll a SignalBus with `Dictionary<Type, List<Delegate>>` — it's untraceable
- Don't create service classes just to forward calls between two objects
- Don't use events when a direct call works — events add indirection with no benefit for 1:1 communication

---

## 5. Managers = MonoBehaviours, Each Owns Their Domain

Each manager is a MonoBehaviour in the scene, responsible for one clear domain. Not plain C# services — real scene objects that can have `[SerializeField]` references and UnityEvents.

### Manager Roles
| Manager | Responsibility | Owns |
|---------|---------------|------|
| **GameManager** | Game state, flow, score | IsRunning, Score, DifficultyLevel, OnGameStarted/OnGameOver UnityEvents |
| **LevelManager** | Wave progression, spawn points | LevelConfig, spawnEdgeRight, yMin/yMax, current wave tracking |
| **ItemManager** | Item spawning, item effects | ItemConfigs (from waves), spawn timing, item collection logic |
| **PlayerManager** | Player control, weapon, drones | Input, movement, weapon holding, drone formation, animation |

### Rules
- **Each manager owns its domain** — no manager reaches into another's internals
- **Managers talk via public methods or UnityEvents** — `GameManager.PlayerDied()`, not direct field access
- **For small games**, GameManager can absorb score, difficulty, and simple state — don't split prematurely
- **Never create plain C# service classes** (CombatService, ScoreService, etc.) — if it needs scene access, it's a MonoBehaviour

### Example
```csharp
// GameManager — game state + flow
public class GameManager : MonoBehaviour
{
    public int Score { get; private set; }
    public bool IsRunning { get; private set; }

    [SerializeField] private UnityEvent m_onGameStarted;
    [SerializeField] private UnityEvent m_onGameOver;

    public void AddScore(int value) { ... }
    public void PlayerDied() { ... }
}

// LevelManager — wave progression + spawn config
public class LevelManager : MonoBehaviour
{
    [SerializeField] private LevelConfig m_levelConfig;
    [SerializeField] private Transform m_spawnEdgeRight;
    // tracks current wave, provides spawn points to spawners
}

// PlayerManager — player behavior
public class PlayerManager : MonoBehaviour
{
    [SerializeField] private Drone m_defaultDronePrefab;
    // handles input, weapon, drones, animation
}
```

Other objects find managers: `FindFirstObjectByType<GameManager>()?.AddScore(10)`

### When to split vs merge
- **< 5 systems**: GameManager can hold everything (score, difficulty, spawning)
- **5-15 systems**: Split into domain managers (Game, Level, Item, Player)
- **15+ systems**: Consider VContainer to wire managers together

---

## 6. VContainer (When You Need DI)

Use VContainer when the project outgrows `FindFirstObjectByType` — typically 50+ scripts, 3+ devs, or when you need testability.

### When to Use
- **Skip DI** (solo/duo, < 20 scripts): `[SerializeField]` + `FindFirstObjectByType`
- **Use VContainer** (3+ devs or 50+ scripts): Automates what GameManager was hand-doing

### Guidelines
- **One root LifetimeScope per scene** — child scopes only for additive scenes or UI panels with different lifetimes
- **RegisterComponent** for scene MonoBehaviours — don't `new` them
- **Register plain C# as `ITickable`** only if it genuinely needs per-frame update
- **Don't register everything** — if a MonoBehaviour only talks to its own children, `[SerializeField]` is fine
- VContainer handles **cross-cutting wiring**. Local references stay local.

### Concrete Registration Example
```csharp
public class GameLifetimeScope : LifetimeScope
{
    [SerializeField] private GameManager m_gameManager;
    [SerializeField] private LevelManager m_levelManager;

    protected override void Configure(IContainerBuilder builder)
    {
        // Register scene MonoBehaviours (already in scene, not new'd)
        builder.RegisterComponent(m_gameManager);
        builder.RegisterComponent(m_levelManager);

        // Register plain C# as ITickable for per-frame update
        builder.Register<DifficultyScaler>(Lifetime.Singleton)
               .As<ITickable>();
    }
}
```

### ITickable Example
```csharp
// Plain C# class that needs per-frame logic without being a MonoBehaviour
public class DifficultyScaler : ITickable
{
    private readonly GameManager m_gameManager;

    [Inject]
    public DifficultyScaler(GameManager gameManager)
    {
        m_gameManager = gameManager;
    }

    public void Tick()
    {
        // Called every frame by VContainer
        if (m_gameManager.IsRunning)
            m_gameManager.DifficultyLevel += Time.deltaTime * 0.01f;
    }
}
```

### Migration from FindFirstObjectByType to [Inject]
```csharp
// BEFORE: Manual lookup
public class MonsterSpawner : MonoBehaviour
{
    private GameManager m_gm;

    private void Start()
    {
        m_gm = FindFirstObjectByType<GameManager>();
    }
}

// AFTER: VContainer injection
public class MonsterSpawner : MonoBehaviour
{
    [Inject] private readonly GameManager m_gm;

    // No Start() needed for wiring — VContainer handles it
}
```

### Migration Path
```
Prototype:     FindFirstObjectByType + [SerializeField]
Growing:       Add VContainer, register GameManager + shared systems
Production:    Full DI with interfaces, ITickable, child scopes
```

---

## 7. Project Folder Convention

```
Assets/_Project/
├── Prefabs/
│   ├── Monsters/       # Monster prefabs
│   ├── Weapons/        # Weapon prefabs
│   ├── Bullets/        # Bullet prefabs
│   ├── Items/          # Collectible item prefabs
│   └── Drones/         # Drone prefabs
├── ScriptableObjects/
│   ├── Monsters/       # MonsterConfig assets (mirrors Prefabs/)
│   ├── Weapons/        # WeaponConfig assets
│   ├── Items/          # ItemConfig assets
│   ├── Levels/         # LevelConfig assets
│   └── Difficulty/     # DifficultyConfig assets
├── Scripts/
│   ├── Editor/         # Editor-only tools, wizards, validators
│   ├── Config/         # ScriptableObject class definitions
│   ├── Enums/          # All enums
│   ├── Core/           # Infrastructure (ILogger, etc.)
│   ├── Enemies/        # Monster
│   ├── Items/          # Item
│   ├── Player/         # Player, Drone
│   ├── Projectiles/    # Bullet
│   ├── Spawning/       # MonsterSpawner, ItemSpawner
│   └── UI/             # Hud, GameOver
├── Scenes/
└── Art/
```

`Prefabs/` mirrors `ScriptableObjects/` — same subfolder names.

### Assembly Definitions

As the project grows, add Assembly Definition files (`.asmdef`) to major folders (`Scripts/`, `Scripts/Editor/`, third-party packages). Benefits:
- **Faster compile times** — Unity only recompiles the assembly that changed, not the entire project
- **Enforced dependency boundaries** — assemblies must explicitly reference each other, preventing accidental coupling
- **Required for `Editor/` folders** — keeps editor-only code out of builds

Start with one `.asmdef` for `Scripts/Runtime/` and one for `Scripts/Editor/`. Split further when compile times exceed 5 seconds.

---

## 8. Unity Lifecycle & Gotchas

### Execution Order
```
Awake()      → Called once, before Start. Use for self-init (GetComponent, set defaults)
OnEnable()   → Called each time object is enabled. Use for subscribing to events
Start()      → Called once, after all Awake. Use for cross-references (FindFirstObjectByType)
Update()     → Every frame. Game logic here
LateUpdate() → After all Update. Camera follow, UI sync
OnDisable()  → When disabled. Unsubscribe events
OnDestroy()  → When destroyed. Final cleanup
```

**Rule:** `Awake` = init yourself. `Start` = find others. Never find other objects in `Awake` — they may not exist yet.

### Common Gotchas
- **`Destroy()` is end-of-frame** — the object still exists until frame ends. Use `gameObject == null` carefully; Unity overrides `==` operator for destroyed objects
- **`GetComponent` is expensive** — never call in `Update()`. Cache in `Awake()`:
  ```csharp
  // BAD
  void Update() { GetComponent<SpriteRenderer>().color = Color.red; }

  // GOOD
  private SpriteRenderer m_sr;
  void Awake() { m_sr = GetComponent<SpriteRenderer>(); }
  void Update() { m_sr.color = Color.red; }
  ```
- **`FindFirstObjectByType` is slow** — cache in `Start()`, never in `Update()`
- **String comparison on tags** — use `CompareTag("Enemy")` not `tag == "Enemy"` (avoids GC alloc)
- **Coroutine on disabled object** — `StartCoroutine` throws if the MonoBehaviour is disabled. Check `enabled` first

---

## 9. State Machines

Use for anything with distinct modes of behavior.

### Simple Enum State Machine (good for small scope)
```csharp
public enum GameState { Menu, Playing, Paused, GameOver }

public class GameManager : MonoBehaviour
{
    public GameState State { get; private set; }

    public void StartGame()
    {
        State = GameState.Playing;
        // enable spawners, player input, etc.
    }

    public void PauseGame()
    {
        State = GameState.Paused;
        Time.timeScale = 0f;
    }
}
```

### When to Use What
| Complexity | Pattern | Example |
|-----------|---------|---------|
| 2-4 states, simple transitions | Enum + switch | Game flow (Menu/Play/Pause/GameOver) |
| 5+ states, complex transitions | State classes with Enter/Exit/Update | Enemy AI (Idle/Patrol/Chase/Attack/Flee/Die) |
| Designer-configurable states | SO-based state machine | Boss phases, dialogue trees |

### Enemy AI Pattern
```csharp
// Simple: enum + switch in Update
private void Update()
{
    switch (m_state)
    {
        case EnemyState.Moving:
            MoveLeft();
            if (PlayerInRange()) m_state = EnemyState.Chasing;
            break;
        case EnemyState.Chasing:
            MoveTowardPlayer();
            if (InAttackRange()) m_state = EnemyState.Attacking;
            break;
        case EnemyState.Attacking:
            Attack();
            break;
    }
}
```

---

## 10. Physics Layer Strategy

### Layer Naming Convention
| Layer | Index | Used By |
|-------|-------|---------|
| Player | 6 | Player prefab |
| Monster | 7 | All monster prefabs |
| Bullet | 8 | All bullet prefabs |
| Item | 9 | All item prefabs |
| Drone | 10 | Drone prefabs |

### Collision Matrix Rules
- **Bullet ↔ Monster**: Yes (damage)
- **Bullet ↔ Item**: Yes (collect)
- **Bullet ↔ Player**: No (friendly fire off)
- **Monster ↔ Player**: Yes (game over)
- **Monster ↔ Monster**: No (don't block each other)
- **Item ↔ Player**: No (items are shot, not touched)

### Layer vs Tag vs GetComponent
| Method | Speed | Use When |
|--------|-------|----------|
| Layer + collision matrix | Fastest | Filtering what CAN collide at all |
| `CompareTag()` | Fast | Quick type check in `OnTriggerEnter2D` |
| `GetComponent<T>()` | Slowest | Need to access the component's data |

**Best practice:** Use layers to prevent unnecessary collisions, then `GetComponent` only on the collisions that actually happen.

---

## 11. Component Composition

### Favor Composition Over Inheritance

```csharp
// BAD: Deep inheritance
class Entity : MonoBehaviour { }
class MovingEntity : Entity { }
class ShootingMovingEntity : MovingEntity { }
class FlyingShootingMovingEntity : ShootingMovingEntity { }  // nightmare

// GOOD: Composition — add behaviors as components
// Monster handles HP + collision
// SpriteAnimator handles animation
// A "movement" component handles movement
// Mix and match on the prefab
```

### When Inheritance IS OK
- **Shared base with 2-3 children max** — e.g. `ProjectileBase` with `Bullet`, `Beam`
- **Abstract template pattern** — base defines the flow, children override specifics
- **Keep it to one level deep** — if you need grandchildren, switch to composition

### Rule of Thumb
> "Can I describe this as 'X **has** Y behavior'?" → Composition (add component)
> "Can I describe this as 'X **is a** Y'?" → Maybe inheritance (but question it first)

---

## 12. Save/Load Patterns

### Prototype: PlayerPrefs
```csharp
// Quick and dirty — fine for prototype
PlayerPrefs.SetInt("HighScore", score);
PlayerPrefs.SetString("LastWeapon", weaponId);
```

### Production: JSON Serialization
```csharp
[Serializable]
public class SaveData
{
    public int highScore;
    public string lastWeaponId;
    public int lastWeaponLevel;
}

// Save
var json = JsonUtility.ToJson(saveData);
File.WriteAllText(Application.persistentDataPath + "/save.json", json);

// Load
var saveJson = File.ReadAllText(Application.persistentDataPath + "/save.json");
var loaded = JsonUtility.FromJson<SaveData>(saveJson);
```

**Note:** `JsonUtility` cannot serialize `Dictionary`, polymorphic types, or `null` values. For those cases, use **Newtonsoft.Json** (available via Unity's `com.unity.nuget.newtonsoft-json` package). Prefer `JsonUtility` for simple flat data and Newtonsoft for anything complex.

### What to Save vs Recompute
| Save | Recompute |
|------|-----------|
| High score, currency | Current HP (reset each run) |
| Unlocked weapons/levels | Difficulty multipliers (from level) |
| Player preferences (volume, controls) | UI state (menu open/closed) |
| Progression (stars, achievements) | Runtime object positions |

---

## 13. Debug Tooling

### Inspector as Documentation
```csharp
[Header("Movement")]
[SerializeField, Range(1f, 20f), Tooltip("Units per second")]
private float m_moveSpeed = 5f;

[Header("Combat")]
[SerializeField, Min(1), Tooltip("HP at spawn, before difficulty scaling")]
private int m_baseHp = 30;

[Header("Detection")]
[SerializeField, Min(0f), Tooltip("0 = always move left, >0 = chase player within range")]
private float m_detectRange = 0f;
```

Use `[Header]`, `[Tooltip]`, `[Range]`, `[Min]` on every serialized field — the Inspector IS the documentation.

### Gizmos for Spatial Data
```csharp
private void OnDrawGizmosSelected()
{
    // Visualize detect range
    Gizmos.color = new Color(1f, 0f, 0f, 0.3f);
    Gizmos.DrawWireSphere(transform.position, m_detectRange);

    // Visualize spawn area
    Gizmos.color = Color.green;
    Gizmos.DrawLine(
        new Vector3(m_spawnX, m_yMin, 0f),
        new Vector3(m_spawnX, m_yMax, 0f));
}
```

### Debug.Log Strategy
- **Prefix with class name**: `Debug.Log($"[MonsterSpawner] Wave {index} loaded")`
- **Use LogWarning for recoverable issues**: missing config, empty arrays
- **Use LogError for broken state**: null reference that shouldn't be null
- **Strip logs in production**: use `#if UNITY_EDITOR` or Conditional attribute

---

## 14. SO as Enum Replacement

Instead of hardcoded enums, use SO instances as type-safe keys.

### When Enum is Fine
```csharp
// Few values, never changes, no data attached
public enum BulletType { SemiAuto, Spread, Beam, Bomb }
```

### When SO is Better
```csharp
// Many types, designers add new ones, each has unique data
[CreateAssetMenu(menuName = "Game/StatusEffect")]
public class StatusEffectSO : ScriptableObject
{
    // [field: SerializeField] requires Unity 2022.3+
    [field: SerializeField] public string EffectName { get; private set; }
    [field: SerializeField] public float Duration { get; private set; }
    [field: SerializeField] public Sprite Icon { get; private set; }
    [field: SerializeField] public Color TintColor { get; private set; }
}

// Reference in code:
[SerializeField] private StatusEffectSO m_poisonEffect;
[SerializeField] private StatusEffectSO m_burnEffect;
```

### Benefits
- Designers add new types without code changes or recompile
- Each SO instance can carry its own data (icon, color, values)
- References are drag-and-drop in Inspector
- Extensible: add a new StatusEffect SO asset, it works everywhere

### Keep Enum When
- Values are purely code-driven (state machine states)
- There are < 5 values and they'll never change
- No data needs to be attached per value

---

## 15. Git + Unity

### Must-Have .gitignore Rules
```
/[Ll]ibrary/
/[Tt]emp/
/[Oo]bj/
/[Bb]uild/
/[Bb]uilds/
/[Ll]ogs/
*.csproj
*.sln
```

### .meta Files
- **Always commit .meta files** — they contain GUIDs that link assets to references
- If you delete a .meta → all references to that asset break across all scenes/prefabs
- If two people create the same file → merge conflict on .meta (resolve by keeping one GUID)

### Avoiding Merge Conflicts
| Asset Type | Conflict Risk | Strategy |
|-----------|--------------|----------|
| Scenes | High | One person per scene. Use additive scenes to split work |
| Prefabs | Medium | Avoid editing the same prefab simultaneously. Use prefab variants |
| ScriptableObjects | Medium | Each person works on different SO assets |
| Scripts | Low | Standard merge. One class per file helps |
| Materials/Shaders | Low | Rarely edited by multiple people |

### Git LFS
Use Git LFS for binary assets:
```
# .gitattributes
*.png filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.psd filter=lfs diff=lfs merge=lfs -text
*.fbx filter=lfs diff=lfs merge=lfs -text
*.wav filter=lfs diff=lfs merge=lfs -text
*.mp3 filter=lfs diff=lfs merge=lfs -text
*.anim filter=lfs diff=lfs merge=lfs -text
*.controller filter=lfs diff=lfs merge=lfs -text
*.asset filter=lfs diff=lfs merge=lfs -text
```

### Unity Project Settings
- **Version Control Mode**: Visible Meta Files
- **Asset Serialization Mode**: Force Text (enables diffing scenes/prefabs)

---

## 16. Performance Patterns (Prototype → Production)

### Prototype (Don't Worry Yet)
- `Instantiate`/`Destroy` freely
- `FindFirstObjectByType` in `Start()`
- `GetComponent` cached in `Awake()`
- Focus on gameplay, not performance

### When Profiler Says Optimize
| Problem | Solution |
|---------|----------|
| GC spikes from Instantiate/Destroy | Add object pooling for bullets, VFX |
| `FindFirstObjectByType` too slow | Cache reference or use singleton |
| Too many `GetComponent` calls | Cache all in `Awake()` |
| String allocs from Debug.Log | Wrap in `#if UNITY_EDITOR` |
| Physics too many collisions | Tighten collision matrix layers |
| Too many Update() calls | Merge into manager-driven tick, or use `UpdateManager` pattern |

### Asset Loading
For large asset catalogs (100+ prefabs), consider Unity Addressables for async loading and memory management — outside this doc's scope. Never use `Resources.Load` in production.

### Never Premature
- **Profile first, optimize second** — Unity Profiler is your evidence
- Don't pool 10 bullets "just in case" — pool when you see GC spikes on 1000 bullets
- Don't cache every reference — cache when the profiler shows `FindFirstObjectByType` as a hotspot

---

## 17. Prototype-First Rules

- Start with `Instantiate`/`Destroy`. Add pooling when the profiler says so.
- Start with `FindFirstObjectByType`. Cache in `Start()`. Add singleton or DI when it's too slow.
- Start with `public` fields on GameManager. Add encapsulation when the team grows.
- Start with `UnityEvent` in Inspector. Add code-driven events when you need runtime subscription.
- **Don't architect for 100 developers when there are 2 of you.**

---

## 18. Naming & Readability — Code Explains Itself

Code should read like English. If you need a comment to explain what a variable or function does, the name is wrong.

### Naming Rules
```csharp
// BAD: What is this?
float t = 0.5f;
int n = 4;
void Process() { }
void DoStuff() { }
List<GameObject> list1;

// GOOD: Name tells you everything
float fireTimer = 0.5f;
int hordeBurstSize = 4;
void SpawnHorde() { }
void ApplyDamage(int amount) { }
List<GameObject> waveNormalPrefabs;
```

### Function Names = Verb + Context
```csharp
// BAD                          // GOOD
void Handle() { }              void HandleShooting() { }
void Check() { }               void CheckWaveAdvance() { }
void Do() { }                  void SpawnMonsterAtPosition() { }
void Run() { }                 void RecalculateDronePositions() { }
bool Get() { }                 bool CanLevelUp { get; }
```

### Boolean Names = Question
```csharp
// BAD                          // GOOD
bool flag;                      bool isRunning;
bool check;                     bool isPlayerAlive;
bool status;                    bool canRecruit;
bool active;                    bool isEliteUnlocked;
```

### No Inline Comments for Obvious Code
```csharp
// BAD: Comment restates the code
this.m_currentHp -= amount;  // subtract amount from current hp
if (this.m_currentHp <= 0)   // check if hp is zero or less
{
    Destroy(this.gameObject);  // destroy the game object
}

// GOOD: Code speaks for itself. Comment only when WHY is unclear
this.m_currentHp -= amount;
if (this.m_currentHp <= 0)
{
    Destroy(this.gameObject);
}

// GOOD: Comment explains WHY, not WHAT
// Clamp to yMin+halfSpread so the formation doesn't spawn partially off-screen
var centerY = Random.Range(yMin + halfSpread, yMax - halfSpread);
```

### When Comments ARE Useful
- **Why** something non-obvious is done: `// Beam fires with speed=0 because it stays at fire point`
- **Gotchas**: `// Destroy is end-of-frame, so isDead flag prevents double-processing`
- **Public API contracts**: XML doc on public methods other devs will call
- **TODO/HACK markers**: `// TODO: add pooling when profiler shows GC spikes`

### When Comments Are Code Smell
- Explaining WHAT the code does → rename the variable/function instead
- Commenting every line → the code is too complex, refactor it
- Commented-out code → delete it, git has history
- `// this function does X, Y, and Z` → function does too much, split it

### Class & File Naming
| Entity | Convention | Example |
|--------|-----------|---------|
| MonoBehaviour on prefab root | Entity name (no suffix) | `Monster`, `Drone`, `Bullet`, `Item` |
| ScriptableObject config | `EntityConfig` | `MonsterConfig`, `LevelConfig`, `DifficultyConfig` |
| Enum | Singular noun | `ItemType`, `MonsterTier`, `BulletType` |
| Manager MonoBehaviour | `DomainManager` | `GameManager`, `LevelManager`, `ItemManager` |
| Spawner MonoBehaviour | `EntitySpawner` | `MonsterSpawner`, `ItemSpawner` |
| Serializable data class | `EntityData` | `WeaponLevelData`, `HordeSpawnData` |
| File name | = Class name | `Monster.cs` contains `class Monster` |

### One Class Per File, Always
- File name must match class name exactly
- No helper classes stuffed at the bottom of another file
- Exception: small `[Serializable]` data classes can live with their parent SO (e.g. `WeaponLevelData` in `WeaponConfig.cs`)

---

## 19. Prefab References Use Component Type, Not GameObject

When a field references a prefab, use the component type — not `GameObject`. This gives type safety in the Inspector (can't assign wrong prefab).

```csharp
// BAD: Accepts any GameObject — no type safety
[SerializeField] private GameObject bulletPrefab;
[SerializeField] private GameObject monsterPrefab;

// GOOD: Inspector only accepts prefabs with this component
[SerializeField] private Bullet bulletPrefab;
[SerializeField] private Monster monsterPrefab;
[SerializeField] private Drone dronePrefab;
```

When you `Instantiate`, the return type matches the component:
```csharp
// With component type — clean, returns Bullet directly
var bullet = Instantiate(bulletPrefab, position, Quaternion.identity);
bullet.Activate(speed, lifetime, damage);

// If you need the GameObject for something:
Destroy(bullet.gameObject);
```

Apply everywhere: SO fields, MonoBehaviour fields, method parameters, local variables.

---

## 20. No `this.` Prefix

In C#, `this.` is unnecessary for accessing instance members. It adds visual noise without clarity.

```csharp
// BAD: Cluttered with this.
void HandleShooting()
{
    this.m_fireTimer -= Time.deltaTime;
    if (this.m_fireTimer > 0f) return;
    this.m_fireTimer = this.m_currentStats.FireRate;
    this.Fire(this.m_currentStats);
}

// GOOD: Clean, reads naturally
void HandleShooting()
{
    m_fireTimer -= Time.deltaTime;
    if (m_fireTimer > 0f) return;
    m_fireTimer = m_currentStats.FireRate;
    Fire(m_currentStats);
}
```

**Keep `this` only when:**
- Passing self as argument: `OnHitPlayer?.Invoke(this)`
- Extension methods (required by C# syntax)

**Private fields use `m_` prefix (C# standard). Parameters use plain names. No shadowing possible:**
```csharp
// BAD: no prefix — parameter shadows field
private float speed;
public void Activate(float speed)
{
    speed = speed;   // assigns to itself, field never set!
}

// GOOD: m_ prefix on private fields
private float m_speed;
public void Activate(float speed)
{
    m_speed = speed;  // clear, no ambiguity
}
```

This applies to all private fields:
```csharp
public class Monster : MonoBehaviour
{
    [SerializeField] private int m_hp = 30;
    [SerializeField] private float m_moveSpeed = 3f;

    private bool m_isDead;
    private Transform? m_playerTransform;

    public void Activate(float hpMultiplier, float speedMultiplier)
    {
        m_hp = Mathf.RoundToInt(m_hp * hpMultiplier);
        m_moveSpeed *= speedMultiplier;
    }
}
```

---

## 21. Async & Coroutines

### Coroutines — Simple Delays & Sequences
Use coroutines for straightforward time-based logic:
```csharp
private IEnumerator SpawnWaveWithDelay(float delay)
{
    yield return new WaitForSeconds(delay);
    SpawnWave();
}
```

### UniTask — Production Async
For scene loading, web requests, or complex async chains, use [UniTask](https://github.com/Cysharp/UniTask):
```csharp
private async UniTask LoadSceneAsync(string sceneName)
{
    await SceneManager.LoadSceneAsync(sceneName);
    Debug.Log($"[LevelManager] Scene {sceneName} loaded");
}
```

### Rules
- **Never use `async void`** in Unity — exceptions are silently swallowed and crash reporting breaks. Use `async UniTaskVoid` (fire-and-forget with proper error handling) or return `IEnumerator` / `UniTask`
- **Coroutines** are fine for simple delays, spawn sequences, and visual effects
- **UniTask** when you need cancellation tokens, `await` composition, or non-MonoBehaviour async
- Coroutines die when the MonoBehaviour is disabled — plan for that or use UniTask with `CancellationToken`

---

## 22. Testing

### Edit Mode Tests (Pure Logic)
Test ScriptableObject data, utility functions, math helpers, and state machines without entering Play Mode:
```csharp
[Test]
public void DamageCalculation_AppliesMultiplier()
{
    int result = CombatUtils.CalculateDamage(baseDamage: 10, multiplier: 1.5f);
    Assert.AreEqual(15, result);
}
```

### Play Mode Tests (MonoBehaviour Behavior)
Test MonoBehaviour lifecycle, physics interactions, and integration:
```csharp
private Monster m_monsterPrefab;

[SetUp]
public void SetUp()
{
    // Load from test resources (Assets/Resources/TestMonster.prefab)
    m_monsterPrefab = Resources.Load<Monster>("TestMonster");
}

[UnityTest]
public IEnumerator Monster_DiesWhenHpReachesZero()
{
    var monster = Object.Instantiate(m_monsterPrefab);
    monster.TakeDamage(999);
    yield return null; // wait one frame for Destroy
    Assert.IsTrue(monster == null); // Unity's == override
}
```

### ConfigValidator Pattern
Use `OnValidate` or editor scripts to catch config errors at edit time, not runtime. `OnValidate` is editor-only — Unity strips it from builds, so wrap it in `#if UNITY_EDITOR` to be explicit:
```csharp
#if UNITY_EDITOR
private void OnValidate()
{
    if (m_moveSpeed <= 0f)
        Debug.LogError($"[{name}] moveSpeed must be > 0", this);
}
#endif
```

### VContainer Enables Testability
With VContainer, dependencies are constructor-injected into plain C# classes — making them testable without a scene:
```csharp
[Test]
public void DifficultyScaler_IncreasesOverTime()
{
    var mockGm = new GameManager(); // or a mock
    var scaler = new DifficultyScaler(mockGm);
    // test Tick() behavior directly
}
```

---

## 23. Silent Error Prevention — No Defensive Null Guards on Required Dependencies

This is the most common source of invisible bugs, especially in AI-generated code. A null check that silently returns looks "safe" — but it hides broken references and corrupts game logic.

### The Problem

```csharp
// BAD: "Safe" code that creates invisible bugs
private void Start()
{
    m_gm = FindFirstObjectByType<GameManager>();
}

private void Update()
{
    if (m_gm == null) return;  // Spawner silently stops working.
                                // No error. No clue why monsters don't spawn.
    SpawnMonster();
}

public void TakeDamage(int amount)
{
    if (m_gm == null) return;   // Monster dies but score never updates.
    m_gm.AddScore(m_scoreValue); // Player sees score stuck at 0. No error.
}
```

The developer spends hours debugging because the console is clean — the null guard hid the real problem.

### The Rule

**If something MUST exist for the game to work, never null-guard it. Let it crash.**

A `NullReferenceException` pointing to the exact line is infinitely more useful than silence.

### The Pattern: Assert at Init, Trust at Runtime

```csharp
// GOOD: Validate once at initialization, crash loudly if broken
private void Start()
{
    m_gm = FindFirstObjectByType<GameManager>();
    Debug.Assert(m_gm != null, $"[{GetType().Name}] GameManager not found in scene!");
}

private void Update()
{
    // No null check. If m_gm is null, the Assert already fired in Start.
    // If it somehow becomes null later, NullReferenceException points here.
    SpawnMonster();
}

public void TakeDamage(int amount)
{
    m_gm.AddScore(m_scoreValue);  // No guard. Crashes if broken. Developer sees it immediately.
}
```

For `[SerializeField]` fields that must be assigned in Inspector:
```csharp
#if UNITY_EDITOR
private void OnValidate()
{
    if (m_bulletPrefab == null)
        Debug.LogError($"[{name}] bulletPrefab is not assigned!", this);
    if (m_firePoint == null)
        Debug.LogError($"[{name}] firePoint is not assigned!", this);
}
#endif

private void Awake()
{
    Debug.Assert(m_bulletPrefab != null, $"[{GetType().Name}] bulletPrefab not assigned in Inspector!");
    Debug.Assert(m_firePoint != null, $"[{GetType().Name}] firePoint not assigned in Inspector!");
}
```

### When Null Checks ARE Correct

| Situation | Null Check? | Example |
|-----------|------------|---------|
| Required dependency | **NO** — Assert at init, trust at runtime | GameManager, Player, LevelConfig |
| `[SerializeField]` must be assigned | **NO** — `OnValidate` + `Debug.Assert` in `Awake` | bulletPrefab, firePoint |
| Optional feature | **YES** — guard + log | Aim assist, tutorial overlay, analytics |
| Optional system not in every scene | **YES** — guard + log | MusicManager in a test scene |
| Destroyed object same frame | **YES** — Destroy is end-of-frame | Checking if target was just killed |
| **Collision / trigger callbacks** | **YES — ALWAYS** | `OnTriggerEnter2D`, `OnCollisionEnter2D` — filter by type |
| **`GetComponent<T>()` on collision target** | **YES — use `TryGetComponent`** | Bullet checking if collided object has Monster component |

### Collision/Trigger Callbacks: Always Guard, Always Filter

Collision and trigger callbacks are a distinct case. They fire for every object that touches the collider — walls, particles, other bullets, the player, monsters, items, pickups. You MUST filter, because most of the collisions aren't what you want to process.

This is NOT a "silent null guard" violation — it's correct Unity practice:

```csharp
// CORRECT: Standard collision handling pattern
private void OnTriggerEnter2D(Collider2D other)
{
    // Defensive null check — rare but possible during scene teardown
    if (other == null) return;

    // Type filter — collisions fire for many different objects
    if (!other.CompareTag("Enemy")) return;

    // Component lookup — use TryGetComponent, not GetComponent + null check
    if (!other.TryGetComponent<Monster>(out var monster)) return;

    monster.TakeDamage(m_damage);
    Destroy(gameObject);
}
```

**Why each guard is correct:**

1. **`other == null` check** — rare, but during scene unload or object destruction, you can get a callback with a null collider. Defensive but valid.
2. **`CompareTag` filter** — a bullet doesn't care about hitting walls, other bullets, or particles. Filtering is the whole point.
3. **`TryGetComponent` check** — the other object tagged "Enemy" may have a different component than `Monster` (e.g. a boss with `BossMonster`). Checking is correct, not silent failure.

### `TryGetComponent` vs `GetComponent` — Always Prefer `TryGetComponent`

For collision callbacks and runtime lookups where the component may not exist:

```csharp
// GOOD — idiomatic, fast, no exception on failure
if (!other.TryGetComponent<Monster>(out var monster)) return;
monster.TakeDamage(m_damage);

// OK but older style
var monster = other.GetComponent<Monster>();
if (monster == null) return;
monster.TakeDamage(m_damage);

// BAD — crashes on non-Monster collisions
other.GetComponent<Monster>().TakeDamage(m_damage);
```

`TryGetComponent` was introduced specifically to avoid the null check pattern and is faster because it doesn't allocate an exception internally when the component is missing.

### The Difference Between "Silent Guard" and "Collision Filter"

| Pattern | Correct? | Reason |
|---------|---------|--------|
| `if (m_gameManager == null) return;` in `Update()` | ❌ Bad | Required dependency is broken — should crash |
| `if (m_rigidbody == null) return;` after `GetComponent` in `Awake` | ❌ Bad | Missing required sibling component — Assert instead |
| `if (other == null) return;` in `OnTriggerEnter2D` | ✅ Good | Defensive filter on engine callback |
| `if (!other.CompareTag("Enemy")) return;` | ✅ Good | Filtering collision types |
| `if (!other.TryGetComponent<Monster>(out var m)) return;` | ✅ Good | Target may legitimately not have the component |
| `m_onAchievement?.Invoke();` for optional event | ✅ Good | Event is genuinely optional |
| `m_onGameOver?.Invoke();` for required event | ❌ Bad | Required event — should crash if unwired |

The rule: **null guards are correct when the null case is an expected, valid gameplay state — not when they hide broken dependencies.**

### Null-Conditional Operator (`?.`) — Use With Caution

```csharp
// BAD: Hides missing required event
m_onGameOver?.Invoke();  // If event is null, game over screen never shows.
                          // Player is stuck. No error.

// GOOD: Required event — crash if not wired
m_onGameOver.Invoke();   // NullRef if not set up. Developer fixes it.

// OK: Genuinely optional — game works without it
m_onAchievementUnlocked?.Invoke();  // Analytics/cosmetic, game runs fine without
```

**Rule of thumb for `?.`:** If removing the entire line would break the game → don't use `?.`. If the game still plays fine without it → `?.` is OK.

### Common AI-Generated Anti-Patterns

```csharp
// ANTI-PATTERN 1: Guard every FindFirstObjectByType
m_gm = FindFirstObjectByType<GameManager>();
if (m_gm == null) return;  // ← If GameManager is missing, the SCENE is broken.
                             //   Returning silently doesn't help anyone.

// ANTI-PATTERN 2: Guard before every method call
if (m_gm != null && m_gm.IsRunning)  // ← m_gm should never be null.
                                       //   If it is, Assert caught it in Start.

// ANTI-PATTERN 3: Try-catch that swallows errors
try { SpawnWave(); }
catch (System.Exception) { }  // ← Wave silently fails. Console is clean.
                                //   Developer has no idea spawning is broken.

// ANTI-PATTERN 4: "Just in case" guards with no logic
if (m_hp < 0) return;            // ← Why would HP be negative? If it is,
                                   //   something else is broken. Don't hide it.
if (m_bulletPrefab == null) return; // ← Should have been caught by OnValidate.
                                     //   Silently not firing = invisible bug.
```

### Decision Flowchart

Before writing any null check:

1. **"Is this required for the game to function?"** → YES → No guard. `Debug.Assert` at init.
2. **"Is this genuinely optional?"** → YES → Guard + `Debug.LogWarning` with context.
3. **"Am I adding this just in case / for safety?"** → **REMOVE IT.** This is how silent bugs are born.
4. **"Would the game still work if this silently fails?"** → NO → Don't let it silently fail.

---

## 24. No Isolated Changes — Trace the Full Impact Chain

When modifying a system, every script that depends on the changed behavior must be updated in the same pass. Partial changes are worse than no change — they create inconsistency where some systems work and others silently break.

### The Problem

```
Developer asks: "Change terrain collision from tags to layers"

Incomplete change (what typically happens):
  ✅ TerrainCollider.cs — updated to layer check
  ❌ Bullet.cs — still uses CompareTag("Terrain"), now phases through terrain
  ❌ Monster.cs — still uses CompareTag("Terrain"), walks through walls
  ❌ Item.cs — still uses CompareTag("Terrain"), falls through floor
  ❌ Physics collision matrix — new Terrain layer not configured

The ONE file that was changed works. The four files that weren't changed are broken.
No errors in console. Bugs appear as mysterious physics failures.
```

### Why This Happens

Each change request gets treated as isolated — "fix this one file." But game systems are interconnected. A collision check, a layer assignment, a tag name, a public method signature, an enum value — these are shared contracts. Changing one side of the contract without updating the other side breaks the contract silently.

### The Rule: Impact Analysis Before Code

Before writing any change, list every file and system affected:

```
## Impact Analysis: Change terrain from tag-based to layer-based collision

### Direct changes:
- TerrainCollider.cs → remove CompareTag, use layer mask

### Ripple effects (THESE ARE THE ONES THAT GET MISSED):
- Bullet.cs (line 34) → OnTriggerEnter2D uses CompareTag("Terrain") → update
- Monster.cs (line 67) → ground check uses CompareTag("Terrain") → update
- Item.cs (line 22) → OnCollisionEnter2D uses CompareTag("Terrain") → update
- Physics Matrix → create Terrain layer (11), set collision pairs
- All terrain prefabs → change layer from Default to Terrain

### Not affected:
- GameManager.cs → no terrain references
- MonsterSpawner.cs → spawn positions only, no terrain interaction
```

### Common Incomplete Change Patterns

| Change | What Gets Updated | What Gets Missed |
|--------|------------------|-----------------|
| Rename a tag | The one script that triggered the request | Every other script using `CompareTag` with the old name |
| Change physics layer | The one prefab you're working on | All other prefabs on the same layer + collision matrix |
| Modify a public method signature | The method itself | Every call site — other scripts calling that method |
| Change an enum value | The enum definition | Every `switch` statement, serialized fields storing the old value |
| Rename a SerializeField | The field in the script | The serialized value in every prefab/scene using the script (Unity loses the reference) |
| Change damage calculation | The damage method | Health display, death logic, score, sound effects, VFX triggers |
| Modify spawner timing | The spawner | Difficulty curve, wave balance, item drop rates tied to spawn count |

### How to Trace Impact

For any change, search for these connections:

1. **Method calls** — Who calls the method you're changing? (`Ctrl+Shift+F` for method name)
2. **Shared data** — Who else reads/writes the field, tag, layer, or enum you're changing?
3. **Physics relationships** — What other objects collide/trigger with the changed object?
4. **Event subscribers** — Who listens to events fired by the changed system?
5. **Prefab references** — What prefabs serialize a reference to the changed script/component?
6. **Manager coordination** — Which manager orchestrates the behavior you're changing?

### When You Don't Know the Full Codebase

If you can't see all the project files, **ask before delivering partial changes**:

> "This change affects terrain collision. To make sure nothing breaks, I need to see every script that references terrain — probably your Bullet, Monster, and Item scripts. Can you share those so I can update everything together?"

A complete change delivered once is always better than a partial change that requires 4 follow-up bug fixes.

---

## 25. Delete and Rebuild — Don't Hack New Requirements Into Old Code

When requirements change fundamentally, the existing code may no longer be the right foundation. Patching it with flags, branches, and mode switches creates spaghetti. Deleting and rebuilding creates clean code that matches what the thing actually IS now.

### The Problem

Requirements evolve. A ground vehicle becomes a flying vehicle. A melee system needs ranged attacks. A single-player game gets multiplayer. The existing code was built on assumptions that are no longer true.

The wrong response: keep the old code and bolt on the new behavior with `if/else` branches, boolean flags, and mode enums. Every method gains a fork. The class name no longer describes what it does. Dead code paths accumulate. Each new requirement adds another layer of hacks.

The right response: recognize that the foundation has changed, delete what no longer fits, and build fresh from the new requirements.

### Signs the Approach Needs Replacing

| Signal | Wrong Response | Right Response |
|--------|---------------|----------------|
| New feature needs `if (mode)` in 3+ methods | Add boolean + branches everywhere | Delete. Separate components or new class. |
| Class name doesn't describe what it does anymore | Rename but keep internals | Delete. New class, new structure. |
| Original assumptions are invalidated | Wrap old logic in guards and conditions | Delete. Rebuild from new assumptions. |
| "Make X behave like Y" where X ≠ Y | Hack Y's behavior into X's structure | Delete X. Build Y. |
| Adding a feature touches 5+ existing methods | Surgical edits across the class | Architecture doesn't support it. Redesign. |
| Adding a `type` or `mode` enum to switch behavior | `switch` in every method | One component per behavior. Compose on prefab. |

### The Decision Framework

Before modifying existing code for a new requirement:

1. **"Does this class still represent what the thing IS?"** → If no → delete and rebuild.
2. **"Am I adding `if/else` branches in 3+ methods for the new behavior?"** → Yes → wrong approach. Split or rebuild.
3. **"Would writing from scratch be faster and cleaner than modifying?"** → Yes → write from scratch. Existing code has no inherent value.
4. **"Is there dead code that only runs in the old mode?"** → Yes → the old mode is dead weight. Delete it.

### When Modification IS Correct

Not every change requires a rebuild. Modification is fine when:
- The new feature fits naturally into the existing structure (adding a new weapon type to a weapon system built for multiple types)
- The class name, responsibilities, and assumptions still hold
- The change is additive, not transformative (new behavior alongside old, not replacing old)
- No branching needed — the existing abstraction already handles the variation

### Composition Over Hacking

When both the old and new behaviors are needed (vehicle that can drive AND fly):

```csharp
// GOOD: Composition — separate components, compose on prefab
public class GroundMovement : MonoBehaviour { /* wheels, steering */ }
public class FlightMovement : MonoBehaviour { /* lift, pitch, yaw */ }
public class VehicleController : MonoBehaviour
{
    [SerializeField] private GroundMovement m_ground;
    [SerializeField] private FlightMovement m_flight;
    
    // Switch between modes cleanly — each component handles its own physics
    public void SwitchToFlight() { m_ground.enabled = false; m_flight.enabled = true; }
    public void SwitchToGround() { m_flight.enabled = false; m_ground.enabled = true; }
}

// BAD: One class with mode branches
public class Vehicle : MonoBehaviour
{
    [SerializeField] private bool m_isFlying;  // branch in every method
    void Update()
    {
        if (m_isFlying) { /* flight code */ }
        else { /* ground code */ }
    }
}
```

### The Rule

**Code is disposable. Requirements are not.** When requirements change, the code serves the requirement — not the other way around. Delete freely. Rebuild confidently. The old code is in git if you ever need it back.

---

## 26. Module Independence — Every Entity Works In an Empty Scene

This builds on Section 2 (Prefab = Self-Contained) but goes further: an entity's **core behavior** must never depend on managers or other scene objects.

The test is simple: **Drag any prefab into an empty scene. Hit Play. It works.**

- Player → moves, shoots, animates. No GameManager needed.
- Monster → moves left, takes damage, dies. No LevelManager needed.
- Bullet → flies forward, destroys on hit or timeout. No ScoreManager needed.
- Item → drifts, has a visual. No ItemManager needed.

### What "Works" Means

| Must Work In Empty Scene | OK To Skip Without Managers |
|--------------------------|---------------------------|
| Movement, input response | Score updates |
| Physics, collision, triggers | Wave progression |
| Animation, visual feedback | UI notifications |
| AI behavior (patrol, chase) | Global difficulty scaling |
| Sound (if AudioSource on prefab) | Analytics, achievements |
| Self-destruction on death | Spawn chain reactions, drops |
| Damage dealing/receiving | Combo meters, style ranks |

### The Two-Layer Pattern

Mentally (and optionally physically) separate each entity into two layers:

**Entity Layer** — core behavior, zero external dependencies:
```csharp
public class Monster : MonoBehaviour
{
    // OWN components — required, Assert in Awake
    private Rigidbody2D m_rb;
    
    // OWN config — lives on the prefab
    [SerializeField] private float m_moveSpeed = 3f;
    [SerializeField] private int m_hp = 10;
    
    private void Awake()
    {
        m_rb = GetComponent<Rigidbody2D>();
        Debug.Assert(m_rb != null, $"[{GetType().Name}] Missing Rigidbody2D!");
    }

    private void Start()
    {
        m_rb.linearVelocity = Vector2.left * m_moveSpeed;  // Works alone.
    }

    public void TakeDamage(int amount)
    {
        m_hp -= amount;
        if (m_hp <= 0)
            Destroy(gameObject);  // Works alone. No manager needed to die.
    }
}
```

**Integration Layer** — connects to game systems, tolerates their absence:
```csharp
// Option A: Separate component (cleanest, best for large projects)
public class MonsterGameIntegration : MonoBehaviour
{
    [SerializeField] private int m_scoreValue = 10;
    private GameManager m_gm;

    private void Start()
    {
        m_gm = FindFirstObjectByType<GameManager>();
    }

    private void OnDestroy()
    {
        m_gm?.AddScore(m_scoreValue);  // ?.  is correct here — manager IS optional
    }
}

// Option B: In the entity itself (simpler, fine for small projects)
// Keep the mental separation: core methods never touch managers,
// manager calls are always null-safe with ?.
```

### Null-Guard Clarity With Module Independence

This refines the rules from Section 23:

| What You're Accessing | Null Guard? | Example |
|----------------------|------------|---------|
| Entity's own components | **NO** — `Debug.Assert` in `Awake` | `GetComponent<Rigidbody2D>()` |
| Entity's own `[SerializeField]` config | **NO** — `OnValidate` + `Assert` | `m_bulletPrefab`, `m_firePoint` |
| Cross-system managers | **YES** — `?.` is correct | `m_gm?.AddScore(10)` |
| Other entities in the scene | **YES** — they may not exist | `FindFirstObjectByType<Player>()?.transform` |

The key insight: `?.` on a manager is NOT a "silent null guard" (Red Flag #1). It's correct **because the manager is genuinely optional for the entity's core behavior.** The distinction is whether the entity is broken without it (bad null guard) or works fine without it (correct optionality).

### Why This Matters

- **Testing:** Artist drags Monster into test scene, tweaks movement speed, sees result instantly. No need to build the full game scene.
- **Iteration:** Designer tests Player controls without waiting for GameManager, UI, spawners to be ready.
- **Debugging:** When something breaks, you can isolate the entity in an empty scene. If it still breaks → entity bug. If it works alone → integration bug.
- **Reuse:** Entities can be used across scenes, prototypes, and projects without dragging their manager dependencies along.

---

## 27. Extract, Don't Expand — New Responsibility Means New Class

When a class built for ONE thing gains a requirement to handle MANY things (or gains an unrelated responsibility), the correct response is to extract a new class — not to expand the existing one with arrays, state, and switching logic.

### The Core Question

Every time a new requirement lands on an existing class, ask:

**"Is this the SAME responsibility as the existing class, or a DIFFERENT responsibility?"**

- Same responsibility → modify the class
- Different responsibility → extract a new class

"Managing a single weapon" and "managing a collection of weapons with switching logic" are different responsibilities. The first is `Weapon`. The second is `WeaponInventory`.

### Signs You Should Extract

| Signal | Meaning |
|--------|---------|
| You're adding arrays of the existing type inside the class | You're adding a collection → extract the collection |
| You're adding a "current index" or "active one" field | You're adding selection logic → extract the coordinator |
| Methods are gaining `int index` parameters | You're indexing into a collection → extract it |
| You're adding CRUD (Add/Remove/Unlock/Switch) methods | You're adding inventory semantics → extract an inventory |
| The class name no longer describes what the class does | The class has grown beyond its responsibility → extract |
| Testing one instance in isolation is no longer possible | Coupling through shared state → extract |

### Extraction Naming Convention

Name the extracted class for its new responsibility:

| One of X | Many of X / Coordinator for X |
|----------|------------------------------|
| `Weapon` | `WeaponInventory` |
| `Enemy` | `EnemySpawner`, `EnemyRegistry` |
| `Item` | `ItemInventory` |
| `Skill` | `SkillLoadout` |
| `Quest` | `QuestJournal` |
| `Buff` | `BuffContainer` |
| `DialogueLine` | `DialogueSequence` |
| `Sound` | `SoundBank`, `AudioManager` |
| `Card` | `Deck`, `Hand` |
| `Ability` | `AbilityBook` |

The pattern: the single class keeps its identity and stays testable in isolation. The new class owns the "manage many" concerns.

### Example: Weapon + WeaponInventory

```csharp
// Weapon: ONE responsibility — represent a single weapon, fire it
public class Weapon : MonoBehaviour
{
    [SerializeField] private WeaponData m_data;
    [SerializeField] private Transform m_firePoint;
    [SerializeField] private Bullet m_bulletPrefab;
    
    private float m_fireTimer;

    public WeaponData Data => m_data;
    public bool IsReady => m_fireTimer <= 0f;

    public void Fire()
    {
        if (!IsReady) return;
        m_fireTimer = m_data.fireRate;
        Instantiate(m_bulletPrefab, m_firePoint.position, m_firePoint.rotation);
    }

    private void Update()
    {
        m_fireTimer = Mathf.Max(0f, m_fireTimer - Time.deltaTime);
    }
}

// WeaponInventory: ONE responsibility — manage the collection of weapons
public class WeaponInventory : MonoBehaviour
{
    [SerializeField] private Weapon[] m_weaponPrefabs;
    [SerializeField] private Transform m_weaponHoldPoint;
    [SerializeField] private UnityEvent<Weapon> m_onWeaponSwitched;

    private Weapon[] m_instances;
    private int m_currentIndex;

    public Weapon CurrentWeapon => m_instances[m_currentIndex];

    private void Awake()
    {
        m_instances = new Weapon[m_weaponPrefabs.Length];
        for (int i = 0; i < m_weaponPrefabs.Length; i++)
        {
            m_instances[i] = Instantiate(m_weaponPrefabs[i], m_weaponHoldPoint);
            m_instances[i].gameObject.SetActive(i == m_currentIndex);
        }
    }

    public void SwitchToWeapon(int index)
    {
        if (index < 0 || index >= m_instances.Length || index == m_currentIndex) return;
        m_instances[m_currentIndex].gameObject.SetActive(false);
        m_currentIndex = index;
        m_instances[m_currentIndex].gameObject.SetActive(true);
        m_onWeaponSwitched?.Invoke(CurrentWeapon);
    }

    public void SwitchNext() => SwitchToWeapon((m_currentIndex + 1) % m_instances.Length);
}
```

### Testability Check

After extraction:
- ✅ Drag `Weapon` prefab into empty scene → fires bullets. Works alone.
- ✅ Drag `WeaponInventory` prefab with weapons configured → switches between weapons. Works alone.
- ✅ Each class has one clear responsibility visible from its name.
- ✅ Adding a 3rd concept (dual-wield, weapon upgrades) extracts yet another class — no existing class bloats.

### The Rule

**A class's name is a promise. When new requirements break the promise, extract a new class — don't expand the old one.**

When you catch yourself writing:
- `T[]` arrays of the existing type
- "current index" tracking
- Switch/cycle methods
- Add/Remove/Unlock methods

...inside a class built for ONE of that type — stop and extract. The collection is a separate concept from the item.

---

## Code Style Checklist

### Prototype (Must-Have)

When prototyping or in game jams, verify at minimum:

- [ ] Prefabs are self-contained — Instantiate and they work
- [ ] No `Initialize()` methods with 4+ parameters
- [ ] MonoBehaviours find their own dependencies in `Start()` or via `[SerializeField]`
- [ ] `GetComponent` results cached in `Awake()`, never in `Update()`
- [ ] One clear owner per behavior — no logic split across multiple classes
- [ ] Debug.Log prefixed with class name: `[ClassName] message`
- [ ] No `this.` prefix on instance members
- [ ] Private fields use `m_` prefix
- [ ] No silent null guards on required dependencies — `Debug.Assert` at init, crash at runtime
- [ ] No `?.` on required events or callbacks
- [ ] Every change includes impact analysis — no isolated single-file changes when multiple systems are affected
- [ ] No hacking new requirements into old code — if the class name or assumptions no longer fit, delete and rebuild
- [ ] Every entity works in an empty scene — core behavior has zero manager dependencies
- [ ] No responsibility bloat — when adding "manage many" logic, extract a new class (`WeaponInventory`) instead of expanding the existing one (`Weapon`)

### Production (Full Review)

When shipping or working in a team, verify everything above plus:

- [ ] SOs contain only data and asset references — no visual setup
- [ ] No plain C# service classes that need manual constructor wiring
- [ ] Communication uses the simplest pattern that works (direct call > UnityEvent > Action > EventBus)
- [ ] Folder structure mirrors between Prefabs/ and ScriptableObjects/
- [ ] All `[SerializeField]` fields have `[Header]` and `[Tooltip]`
- [ ] Physics layers configured — not everything collides with everything
- [ ] Composition over inheritance — components not class hierarchies
- [ ] Prefab references use component type (`Bullet`, `Monster`) not `GameObject`
- [ ] Assembly Definitions in place for `Scripts/Runtime/` and `Scripts/Editor/`
- [ ] ConfigValidator or `OnValidate` catches bad Inspector values at edit time
- [ ] No `async void` — use `UniTask` or coroutines
- [ ] No try-catch that swallows exceptions silently
- [ ] `OnValidate` validates all required `[SerializeField]` fields are assigned
- [ ] Key logic has Edit Mode tests; critical flows have Play Mode tests