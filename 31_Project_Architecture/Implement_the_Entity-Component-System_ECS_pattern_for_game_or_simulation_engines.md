# Implement the Entity-Component-System (ECS) pattern for game or simulation engines

**Category:** Project Architecture

---

## Topic Overview

**ECS** separates identity (entities), data (components), and behavior (systems). Entities are just IDs. Components are plain data structs stored contiguously in memory. Systems iterate over components matching specific archetypes. This data-oriented design maximizes cache efficiency and enables massive parallelism, making it the dominant pattern in game engines and simulations.

### ECS vs Traditional OOP

| Aspect | OOP Inheritance | ECS |
| --- | --- | --- |
| Entity | Object with methods | Just an ID |
| Data | Mixed with behavior | Separate components (SOA) |
| Behavior | Virtual methods | Systems iterate components |
| Cache | Poor (scattered objects) | Excellent (contiguous arrays) |
| Composition | Inheritance hierarchy | Mix-and-match components |
| Parallelism | Difficult | Natural (systems on disjoint data) |

---

## Self-Assessment

### Q1: Implement a basic ECS framework

**Answer:**

```cpp

#include <cstdint>
#include <vector>
#include <unordered_map>
#include <bitset>
#include <memory>
#include <typeindex>
#include <cassert>
#include <algorithm>

// === Entity: just an ID ===
using Entity = uint32_t;
constexpr Entity NULL_ENTITY = 0;

// === Component ID system ===
inline uint32_t next_component_id() {
    static uint32_t id = 0;
    return id++;
}

template<typename T>
uint32_t component_id() {
    static uint32_t id = next_component_id();
    return id;
}

constexpr size_t MAX_COMPONENTS = 64;
using ComponentMask = std::bitset<MAX_COMPONENTS>;

// === Component pool: contiguous storage for one type ===
class IComponentPool {
public:
    virtual ~IComponentPool() = default;
    virtual void remove(Entity e) = 0;
};

template<typename T>
class ComponentPool : public IComponentPool {
public:
    T& add(Entity e, T component = {}) {
        assert(entity_to_index_.find(e) == entity_to_index_.end());
        size_t index = data_.size();
        data_.push_back(std::move(component));
        entities_.push_back(e);
        entity_to_index_[e] = index;
        return data_[index];
    }

    void remove(Entity e) override {
        auto it = entity_to_index_.find(e);
        if (it == entity_to_index_.end()) return;

        size_t index = it->second;
        size_t last = data_.size() - 1;

        if (index != last) {
            // Swap with last to maintain contiguity
            data_[index] = std::move(data_[last]);
            entities_[index] = entities_[last];
            entity_to_index_[entities_[index]] = index;
        }

        data_.pop_back();
        entities_.pop_back();
        entity_to_index_.erase(e);
    }

    T* get(Entity e) {
        auto it = entity_to_index_.find(e);
        return it != entity_to_index_.end() ? &data_[it->second] : nullptr;
    }

    bool has(Entity e) const {
        return entity_to_index_.count(e) > 0;
    }

    // Direct array access for systems
    std::vector<T>& data() { return data_; }
    const std::vector<Entity>& entities() const { return entities_; }
    size_t size() const { return data_.size(); }

private:
    std::vector<T> data_;           // Contiguous component data
    std::vector<Entity> entities_;  // Parallel entity array
    std::unordered_map<Entity, size_t> entity_to_index_;
};

// === World: manages entities and components ===
class World {
public:
    Entity create() {
        Entity e = next_entity_++;
        masks_[e] = {};
        return e;
    }

    void destroy(Entity e) {
        for (auto& [type, pool] : pools_) {
            pool->remove(e);
        }
        masks_.erase(e);
    }

    template<typename T>
    T& add(Entity e, T component = {}) {
        auto& pool = get_or_create_pool<T>();
        masks_[e].set(component_id<T>());
        return pool.add(e, std::move(component));
    }

    template<typename T>
    void remove(Entity e) {
        get_pool<T>().remove(e);
        masks_[e].reset(component_id<T>());
    }

    template<typename T>
    T* get(Entity e) {
        return get_pool<T>().get(e);
    }

    template<typename T>
    bool has(Entity e) const {
        return masks_.at(e).test(component_id<T>());
    }

    // Query: iterate entities with specific components
    template<typename... Ts>
    void each(std::function<void(Entity, Ts&...)> fn) {
        ComponentMask required;
        (required.set(component_id<Ts>()), ...);

        for (auto& [entity, mask] : masks_) {
            if ((mask & required) == required) {
                fn(entity, *get<Ts>(entity)...);
            }
        }
    }

private:
    template<typename T>
    ComponentPool<T>& get_or_create_pool() {
        auto id = component_id<T>();
        if (pools_.find(id) == pools_.end()) {
            pools_[id] = std::make_unique<ComponentPool<T>>();
        }
        return static_cast<ComponentPool<T>&>(*pools_[id]);
    }

    template<typename T>
    ComponentPool<T>& get_pool() {
        return static_cast<ComponentPool<T>&>(*pools_.at(component_id<T>()));
    }

    Entity next_entity_ = 1;
    std::unordered_map<Entity, ComponentMask> masks_;
    std::unordered_map<uint32_t, std::unique_ptr<IComponentPool>> pools_;
};

```

### Q2: Define components and systems

**Answer:**

```cpp

// === Components: plain data, no behavior ===
struct Position { float x, y, z; };
struct Velocity { float dx, dy, dz; };
struct Health { int current, max; };
struct Sprite { int texture_id; int width, height; };
struct Collider { float radius; };
struct AIController { float aggro_range; Entity target; };

// === Systems: pure functions operating on component sets ===

// Movement system: processes (Position, Velocity)
void movement_system(World& world, float dt) {
    world.each<Position, Velocity>(
        [dt](Entity e, Position& pos, Velocity& vel) {
            pos.x += vel.dx * dt;
            pos.y += vel.dy * dt;
            pos.z += vel.dz * dt;
        });
}

// Collision system: processes (Position, Collider)
void collision_system(World& world) {
    struct ColliderData {
        Entity entity;
        Position* pos;
        Collider* col;
    };
    std::vector<ColliderData> colliders;

    world.each<Position, Collider>(
        [&](Entity e, Position& pos, Collider& col) {
            colliders.push_back({e, &pos, &col});
        });

    // O(n^2) broad phase (use spatial hash in production)
    for (size_t i = 0; i < colliders.size(); ++i) {
        for (size_t j = i + 1; j < colliders.size(); ++j) {
            float dx = colliders[i].pos->x - colliders[j].pos->x;
            float dy = colliders[i].pos->y - colliders[j].pos->y;
            float dist_sq = dx * dx + dy * dy;
            float r = colliders[i].col->radius + colliders[j].col->radius;
            if (dist_sq < r * r) {
                handle_collision(colliders[i].entity, colliders[j].entity);
            }
        }
    }
}

// Render system: processes (Position, Sprite)
void render_system(World& world, Renderer& renderer) {
    world.each<Position, Sprite>(
        [&](Entity e, Position& pos, Sprite& sprite) {
            renderer.draw(sprite.texture_id,
                         pos.x, pos.y, sprite.width, sprite.height);
        });
}

// === Game loop ===
void game_loop(World& world, Renderer& renderer) {
    float dt = 1.0f / 60.0f;
    while (running) {
        movement_system(world, dt);
        collision_system(world);
        render_system(world, renderer);
    }
}

```

### Q3: Creating entities by composing components

**Answer:**

```cpp

// === Entity creation through composition (no inheritance!) ===

Entity create_player(World& world, float x, float y) {
    Entity e = world.create();
    world.add<Position>(e, {x, y, 0});
    world.add<Velocity>(e, {0, 0, 0});
    world.add<Health>(e, {100, 100});
    world.add<Sprite>(e, {TEXTURE_PLAYER, 32, 32});
    world.add<Collider>(e, {16.0f});
    // No AIController — player is human-controlled
    return e;
}

Entity create_enemy(World& world, float x, float y) {
    Entity e = world.create();
    world.add<Position>(e, {x, y, 0});
    world.add<Velocity>(e, {0, 0, 0});
    world.add<Health>(e, {50, 50});
    world.add<Sprite>(e, {TEXTURE_ENEMY, 32, 32});
    world.add<Collider>(e, {16.0f});
    world.add<AIController>(e, {200.0f, NULL_ENTITY});
    return e;
}

Entity create_projectile(World& world, float x, float y,
                          float dx, float dy) {
    Entity e = world.create();
    world.add<Position>(e, {x, y, 0});
    world.add<Velocity>(e, {dx, dy, 0});
    world.add<Sprite>(e, {TEXTURE_BULLET, 8, 8});
    world.add<Collider>(e, {4.0f});
    // No Health — projectiles are destroyed on collision
    return e;
}

// Static prop: only Position + Sprite (no physics)
Entity create_tree(World& world, float x, float y) {
    Entity e = world.create();
    world.add<Position>(e, {x, y, 0});
    world.add<Sprite>(e, {TEXTURE_TREE, 64, 96});
    return e;
}

// The movement_system automatically skips trees (no Velocity component)
// The AI system automatically skips players (no AIController component)
// Composition > inheritance!

```

---

## Notes

- **Components are data only** — no methods, no virtual functions, no inheritance
- **Systems are functions** — they iterate over matching component sets
- Contiguous component storage = excellent cache performance (data-oriented design)
- Popular C++ ECS libraries: **EnTT**, **flecs**, **entityx**
- Archetype-based ECS (like flecs) stores entities with the same component set together for even better cache performance
- Systems with disjoint component access can run in parallel (movement || render)
- ECS scales to millions of entities; OOP inheritance hierarchies don't
