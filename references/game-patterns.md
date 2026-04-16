## Common Game Patterns

When the user asks to build a specific game type, use these starting points. Each pattern lists the core systems needed — start with these, expand as gameplay demands.

### Horizontal Shmup / Side-Scroller
- PlayerManager (input, movement, weapon, drones)
- MonsterSpawner (wave-based, reads LevelConfig)
- Bullet (pooled projectiles)
- GameManager (score, state, difficulty scaling)

### Top-Down RPG
- PlayerController (grid or free movement)
- NPCManager (dialogue, quests)
- InventorySystem (items as SOs)
- CombatManager (turn-based or action)
- MapManager (tilemap, room transitions)

### Puzzle Game
- BoardManager (grid state, match logic)
- PieceFactory (spawn pieces from config)
- ScoreManager (combos, multipliers)
- TutorialManager (guided first play)

### Idle / Clicker
- ResourceManager (currencies, production rates)
- UpgradeSystem (costs, multipliers as SOs)
- PrestigeManager (reset loop)
- SaveManager (frequent auto-save critical)

### Platformer
- PlayerController (physics-based movement, ground check)
- LevelManager (checkpoints, scene transitions)
- EnemyAI (patrol, chase state machines)
- CollectibleManager (coins, power-ups)

### Open World Action RPG (Genshin-like)
Complex genre — scope aggressively for solo/duo. Build vertically (one area, one party, one element) before going wide.
- CharacterManager (party roster, active character, character switching with cooldown)
- CharacterController3D (movement, dash, jump, climb, swim — state machine with 6+ states)
- CombatManager (normal attack combos, elemental skill, elemental burst, hit detection, damage calculation)
- ElementalSystem (element types as SOs, elemental reaction matrix — Vaporize/Melt/Overload/etc., aura tracking per enemy)
- PartyManager (4-character party, formation, swap logic, off-field energy regen)
- EnemyAI (aggro range, attack patterns, elemental aura, stagger/poise system)
- WorldManager (region loading, teleport waypoints, interactable objects — use additive scenes per region)
- QuestManager (main quest chain, world quests, daily commissions — quest state as serialized data)
- InventoryManager (weapons, artifacts, materials, consumables — item data as SOs, inventory state as runtime data)
- GachaManager (banner config as SOs, pity counter, pull animation, reward resolution)
- UIManager (HUD, party HP bars, elemental burst gauge, minimap, dialogue panels)
- SaveManager (critical — party, inventory, world state, quest progress all need persistent save)

**Architecture note:** This genre demands Prefab + SO + VContainer even for solo dev. Character data, weapon stats, artifact sets, elemental reactions, enemy configs, and gacha banners are all shared data that belongs in SOs. The system count (15+) justifies VContainer for wiring.

**Scope warning:** A full Genshin clone is years of work for a large team. For solo/duo, pick ONE vertical slice: one playable character, one region, one element pair, basic combat. Expand from there.

### First Person Shooter (FPS)
- PlayerController (CharacterController or Rigidbody, mouse look, WASD movement, jump, crouch)
- WeaponManager (weapon inventory, equip/swap, current weapon state)
- Weapon (abstract base or composition — each weapon handles fire mode, recoil, spread, reload, ammo)
- ProjectileManager (hitscan via Raycast for bullets, physics projectiles for grenades/rockets — pool both)
- DamageSystem (hitbox zones — head/body/limb multipliers, armor, damage falloff by distance)
- EnemyAI (NavMeshAgent patrol, alert/search/combat states, cover seeking, line-of-sight checks)
- NetworkManager (if multiplayer — consider Mirror or Netcode for GameObjects, server-authoritative hit detection)
- AudioManager (spatial 3D audio, gunshot falloff, footstep surfaces, weapon switching SFX)
- UIManager (crosshair, ammo counter, health bar, hitmarker, killfeed, minimap)

**Key technical considerations:**
- Use `Physics.Raycast` for hitscan weapons, not projectile prefabs — instant, accurate, cheap
- Separate hitbox colliders (head, torso, limbs) as children of enemy rig — layer them on a Hitbox layer
- Mouse look: `Cursor.lockState = CursorLockMode.Locked`, split X rotation (body yaw) and Y rotation (camera pitch, clamped)
- Recoil: apply camera pitch offset over time, recover gradually — feels better than random spread

### Third Person Shooter (TPS)
- PlayerController (Rigidbody or CharacterController, camera-relative movement, sprint, dodge/roll)
- CameraManager (Cinemachine FreeLook or custom orbit cam, aim mode zoom, collision avoidance, shoulder swap)
- WeaponManager (weapon inventory, equip/holster, aim-down-sights toggle)
- Weapon (fire modes, spread, recoil — same as FPS but with over-shoulder aiming offset)
- CoverSystem (cover detection via raycasts, snap-to-cover, lean/peek, blind fire — state machine)
- DamageSystem (hitbox zones, armor, damage types)
- EnemyAI (NavMeshAgent, cover-to-cover movement, flanking, suppression behavior)
- AnimationManager (Animator with layers — locomotion, upper body aim, recoil, reload blend)
- UIManager (crosshair, health, ammo, cover indicator, interaction prompts)

**Key technical considerations:**
- Camera collision: use `Physics.SphereCast` from character to desired camera position, pull camera forward on hit
- Aim offset: IK or animation rigging to point weapon toward crosshair world position (center screen raycast → world point)
- Movement feels different from FPS: camera-relative input (`Transform.forward` from camera, projected onto ground plane)
- Dodge/roll: root motion animation or physics impulse + invincibility frames, state machine state

### Extraction Shooter (Tarkov / Dark and Darker-like)
High complexity — builds on TPS/FPS base with persistent loot economy and session-based raids.
- RaidManager (raid session lifecycle: deploy → in-raid → extract/death, raid timer, extraction points)
- LootManager (loot spawn tables as SOs, container contents generation, world loot placement per map)
- InventoryManager (grid-based inventory — Tetris-style slot system, container UI, drag-and-drop, weight/encumbrance)
- StashManager (persistent between-raid storage, sort, search, transfer to/from raid loadout)
- ExtractionManager (extraction zones, extraction conditions — keys, timers, cooperation requirements)
- HealthSystem (per-limb HP, bleeding, fractures, pain, medical items — each limb affects gameplay differently)
- WeaponManager (weapon modding — attachments as SOs, barrel/stock/sight/grip slots, stat aggregation)
- TraderManager (trader inventories, reputation, buy/sell/barter, restock timers, quest unlocks)
- InsuranceManager (insure items before raid, return timer, chance of return)
- MapManager (raid maps, spawn points, AI patrol routes, loot zones, extraction markers)
- QuestManager (trader quests, objectives tracked per-raid, persistent progress)
- NetworkManager (authoritative server — anti-cheat critical, server-side hit validation, loot state sync)
- AudioManager (directional footsteps by surface, gunshot distance attenuation, ambient — audio is gameplay in extraction shooters)

**Architecture note:** This genre requires Prefab + SO + VContainer minimum. Weapon attachments, loot tables, trader inventories, medical items, and map configs are all SO-heavy. The system count (15+) and the need for testability (economy balance, loot tables) demands proper DI.

**Scope warning:** Extraction shooters are among the hardest genres to build. For solo/duo, start with: one map, basic FPS movement, 3-5 loot items, one extraction point, AI scavs. The inventory grid and loot economy alone are weeks of work.

### Anime Action Game (Character Action / Stylish Combat)
- CharacterController (fast movement, air dash, double jump, wall run — state machine with 8+ states)
- ComboManager (input buffer, combo chains, cancel windows, hit confirms — frame-data driven)
- AttackData (SO per attack: damage, hitstun frames, knockback vector, hitbox timing, cancel list, animation clip reference)
- HitboxManager (activate/deactivate hitbox colliders per animation frame, multi-hit prevention per combo)
- EnemyAI (aggro, attack patterns, super armor states, stagger thresholds, launch/juggle vulnerability)
- StyleMeter (combo counter, style rank — D/C/B/A/S/SS/SSS, decay timer, rank-up thresholds)
- CameraManager (dynamic combat camera — zoom on finishers, slow-mo on parry, lock-on with target switching)
- AnimationManager (Animator state machine with sub-state machines per weapon, blend trees for locomotion, IK for targeting)
- VFXManager (slash trails, impact sparks, screen shake, hit flash — pooled particle systems)
- UIManager (health, style meter, combo counter, enemy HP bars, lock-on indicator, input prompts)
- CutsceneManager (in-engine cutscenes, dialogue with character portraits, camera transitions)

**Key technical considerations:**
- Frame data is king: each attack needs startup frames, active frames, recovery frames, cancel windows — store as SO or serialized data on the attack prefab
- Input buffering: queue inputs during recovery frames, execute on next cancel window — makes combat feel responsive
- Hitstop: freeze both attacker and target for 2-5 frames on hit — sells impact without slow-mo
- Launch + juggle: apply upward force on hit, track airborne state on enemy, allow air combos while enemy is in juggle state

### Visual Novel / Dialogue-Heavy Game
- DialogueManager (conversation flow, branching choices, text display with typewriter effect)
- DialogueData (SO per conversation: nodes with text, speaker, portrait, choices, branch conditions)
- CharacterDatabase (SO per character: name, portraits for each emotion, voice clips)
- ChoiceManager (player choices, flag tracking, choice history — affects story branches)
- StoryFlagManager (persistent flags/variables that track story state, checked by dialogue branches)
- SceneTransitionManager (background changes, character enter/exit, fade transitions)
- SaveManager (critical — save at any point, multiple save slots, auto-save on choices)
- AudioManager (BGM per scene, ambient, voice lines, SFX for UI)
- GalleryManager (CG unlock tracking, scene replay, music player)
- UIManager (text box, character portraits, choice buttons, backlog, settings, save/load menu)

**Architecture note:** Visual novels are SO-heavy by nature. Every conversation, character, and CG is data. Prefab + SO from the start. Communication can stay Unity Native — the systems are mostly linear (dialogue drives everything).

### Tower Defense
- TowerManager (tower placement, upgrade, sell, tower inventory)
- Tower (base class or composition — targeting logic, fire rate, range, damage type)
- PathManager (enemy path waypoints, multiple lanes, path visualization)
- WaveManager (wave config as SOs — enemy types, counts, spawn intervals, delays between waves)
- EnemyTD (follows path waypoints, has HP, speed, armor, elemental resistance, death drops currency)
- EconomyManager (currency for placing/upgrading towers, wave completion bonuses, interest)
- GridManager (placement grid, valid/invalid cells, tower footprints, snap-to-grid)
- UIManager (tower selection panel, upgrade panel, wave preview, currency display, speed controls)

### Card Game / Deckbuilder
- DeckManager (player deck, draw pile, discard pile, hand — shuffle, draw, discard logic)
- CardData (SO per card: name, cost, effect type, target type, art, description, rarity)
- CardUI (visual card in hand — drag to play, hover to preview, fan layout in hand)
- CombatManager (turn flow: player turn → enemy turn, energy/mana per turn, end turn)
- EffectSystem (card effects: damage, block, buff, debuff, draw, heal — each as a strategy or enum+switch)
- EnemyAI (intent system — telegraph next action, pattern-based or random from pool)
- RewardManager (post-combat rewards: card choices, relics, currency)
- MapManager (branching path map — nodes: combat, shop, rest, event, boss)
- RelicManager (passive modifiers as SOs, trigger conditions, stacking rules)