# HMCCosmetics API Reference

Cosmetic plugin by HibiscusMC supporting hats, backpacks, balloons, dyeable armor, and custom slots. The API lets plugins equip/remove cosmetics, query equipped slots, manage visibility (hide/show), control wardrobe state, and listen for cosmetic events. Add `HMCCosmetics` to `depend` or `softdepend` in plugin.yml.

> **Important:** `HMCCosmeticsAPI.getUser(uuid)` returns null if the player is offline OR if the configured load delay hasn't elapsed yet. Always null-check, or use `PlayerLoadEvent` as the safe signal that the user is ready.

> **Important:** Custom slot/provider registration MUST happen in `onLoad()`, NOT `onEnable()`.

## Code Examples

### Getting a CosmeticUser

```java
import com.hibiscusmc.hmccosmetics.api.HMCCosmeticsAPI;
import com.hibiscusmc.hmccosmetics.user.CosmeticUser;
import org.bukkit.entity.Player;

CosmeticUser user = HMCCosmeticsAPI.getUser(player.getUniqueId());
if (user == null) return; // not loaded yet
```

### Equip a Cosmetic

```java
import com.hibiscusmc.hmccosmetics.api.HMCCosmeticsAPI;
import com.hibiscusmc.hmccosmetics.cosmetic.Cosmetic;
import com.hibiscusmc.hmccosmetics.user.CosmeticUser;
import org.bukkit.Color;

CosmeticUser user = HMCCosmeticsAPI.getUser(player.getUniqueId());
Cosmetic cosmetic = HMCCosmeticsAPI.getCosmetic("my_hat");

if (user != null && cosmetic != null) {
    // Without color
    HMCCosmeticsAPI.equipCosmetic(user, cosmetic);

    // With color (for dyeable cosmetics)
    HMCCosmeticsAPI.equipCosmetic(user, cosmetic, Color.RED);
}
```

### Unequip a Cosmetic

```java
import com.hibiscusmc.hmccosmetics.api.HMCCosmeticsAPI;
import com.hibiscusmc.hmccosmetics.cosmetic.CosmeticSlot;
import com.hibiscusmc.hmccosmetics.user.CosmeticUser;

CosmeticUser user = HMCCosmeticsAPI.getUser(player.getUniqueId());
if (user != null) {
    HMCCosmeticsAPI.unequipCosmetic(user, CosmeticSlot.valueOf("HELMET"));
}
```

### Query Equipped Cosmetics

```java
import com.hibiscusmc.hmccosmetics.cosmetic.Cosmetic;
import com.hibiscusmc.hmccosmetics.cosmetic.CosmeticSlot;
import com.hibiscusmc.hmccosmetics.user.CosmeticUser;
import org.bukkit.inventory.ItemStack;

import java.util.Set;

CosmeticUser user = /* ... */;

CosmeticSlot helmetSlot = CosmeticSlot.valueOf("HELMET");
boolean hasHelmet = user.hasCosmeticInSlot(helmetSlot);

if (hasHelmet) {
    Cosmetic helmet = user.getCosmetic(helmetSlot);
    ItemStack displayItem = user.getUserCosmeticItem(helmetSlot);
}

// All occupied slots
Set<CosmeticSlot> filledSlots = user.getSlotsWithCosmetics();

// Check if a player CAN equip something (permissions, conditions)
boolean canEquip = user.canEquipCosmetic(cosmetic, false);
```

### Hide/Show Cosmetics

Each `HiddenReason` is tracked separately. Cosmetics stay hidden as long as ANY reason is active.

```java
import com.hibiscusmc.hmccosmetics.user.CosmeticUser;
import com.hibiscusmc.hmccosmetics.user.CosmeticUser.HiddenReason;

CosmeticUser user = /* ... */;

// Hide for your plugin's reasons (use PLUGIN for general plugin-driven hides)
user.hideCosmetics(HiddenReason.PLUGIN);

// Show again — only removes YOUR reason, others may still keep them hidden
user.showCosmetics(HiddenReason.PLUGIN);

boolean isHidden = user.isHidden();
boolean hiddenByPlugin = user.isHidden(HiddenReason.PLUGIN);

// Suppress event fire if you don't want listeners to react
user.silentlyAddHideFlag(HiddenReason.PLUGIN);
```

`HiddenReason` values: `NONE`, `WORLDGUARD`, `PLUGIN`, `VANISH`, `POTION`, `ACTION`, `COMMAND`, `EMOTE`, `GAMEMODE`, `WORLD`, `DISABLED`

### Wardrobe Control

```java
import com.hibiscusmc.hmccosmetics.user.CosmeticUser;
import com.hibiscusmc.hmccosmetics.user.manager.UserWardrobeManager;
import com.hibiscusmc.hmccosmetics.config.Wardrobe;

CosmeticUser user = /* ... */;

if (user.isInWardrobe()) {
    user.leaveWardrobe(false); // ejected = false
}

// Enter (Wardrobe object obtained from config or another source)
// user.enterWardrobe(wardrobe, true); // ignoreDistance
```

### Backpacks and Balloons

```java
import com.hibiscusmc.hmccosmetics.cosmetic.types.CosmeticBackpackType;
import com.hibiscusmc.hmccosmetics.cosmetic.types.CosmeticBalloonType;
import com.hibiscusmc.hmccosmetics.user.CosmeticUser;

CosmeticUser user = /* ... */;

if (!user.isBackpackSpawned()) {
    // user.spawnBackpack(backpackType);
}
user.respawnBackpack();
user.despawnBackpack();

// Same pattern for balloons:
// user.spawnBalloon(balloonType);
user.respawnBalloon();
user.despawnBalloon();
```

### Listen for Cosmetic Events

```java
import com.hibiscusmc.hmccosmetics.api.events.PlayerLoadEvent;
import com.hibiscusmc.hmccosmetics.api.events.PlayerCosmeticEquipEvent;
import com.hibiscusmc.hmccosmetics.api.events.PlayerCosmeticPostEquipEvent;
import com.hibiscusmc.hmccosmetics.api.events.PlayerCosmeticRemoveEvent;
import com.hibiscusmc.hmccosmetics.api.events.PlayerCosmeticHideEvent;
import com.hibiscusmc.hmccosmetics.api.events.PlayerWardrobeEnterEvent;
import com.hibiscusmc.hmccosmetics.cosmetic.Cosmetic;
import com.hibiscusmc.hmccosmetics.user.CosmeticUser;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;

public class CosmeticListener implements Listener {

    @EventHandler
    public void onLoad(PlayerLoadEvent event) {
        // NOT cancellable — safe signal that user data is ready
        CosmeticUser user = event.getUser();
    }

    @EventHandler
    public void onEquip(PlayerCosmeticEquipEvent event) {
        // Cancellable
        CosmeticUser user = event.getUser();
        Cosmetic cosmetic = event.getCosmetic();

        // Optionally swap the cosmetic before equipping
        // event.setCosmetic(otherCosmetic);

        if (!user.getPlayer().hasPermission("cosmetics.use")) {
            event.setCancelled(true);
        }
    }

    @EventHandler
    public void onPostEquip(PlayerCosmeticPostEquipEvent event) {
        // NOT cancellable — fired after successful equip
    }

    @EventHandler
    public void onRemove(PlayerCosmeticRemoveEvent event) {
        // Cancellable
        Cosmetic cosmetic = event.getCosmetic();
    }

    @EventHandler
    public void onHide(PlayerCosmeticHideEvent event) {
        // Cancellable — fires only when a NEW HiddenReason is added
        CosmeticUser.HiddenReason reason = event.getReason();
    }

    @EventHandler
    public void onWardrobeEnter(PlayerWardrobeEnterEvent event) {
        // Cancellable
        // event.setWardrobe(otherWardrobe); // can swap wardrobe
    }
}
```

### Register a Custom Slot

```java
import com.hibiscusmc.hmccosmetics.api.HMCCosmeticsAPI;
import com.hibiscusmc.hmccosmetics.cosmetic.CosmeticSlot;
import org.bukkit.plugin.java.JavaPlugin;

public class MyPlugin extends JavaPlugin {

    @Override
    public void onLoad() {
        // MUST be called in onLoad(), NOT onEnable()
        CosmeticSlot mySlot = HMCCosmeticsAPI.registerCosmeticSlot("MY_SLOT");
    }
}
```

## API Reference (Trimmed)

### `com.hibiscusmc.hmccosmetics.api.HMCCosmeticsAPI` (all static)

| Return | Method | Description |
|---|---|---|
| `Cosmetic` | `getCosmetic(String id)` | Lookup by ID (null if missing) |
| `CosmeticUser` | `getUser(UUID)` | Get user (null if offline or not loaded) |
| `Menu` | `getMenu(String id)` | Lookup menu by ID |
| `void` | `equipCosmetic(CosmeticUser, Cosmetic)` | Equip without color |
| `void` | `equipCosmetic(CosmeticUser, Cosmetic, Color)` | Equip with color |
| `void` | `unequipCosmetic(CosmeticUser, CosmeticSlot)` | Remove from slot |
| `List<Cosmetic>` | `getAllCosmetics()` | All registered cosmetics |
| `List<CosmeticUser>` | `getAllCosmeticUsers()` | All online users |
| `Map<String, CosmeticSlot>` | `getAllCosmeticSlots()` | All slots |
| `CosmeticSlot` | `registerCosmeticSlot(String id)` | Register slot (`onLoad()` ONLY) |
| `void` | `registerCosmeticUserProvider(CosmeticUserProvider)` | Custom user factory (`onLoad()` ONLY) |
| `void` | `registerCosmeticProvider(CosmeticProvider)` | Custom cosmetic factory (`onLoad()` ONLY) |

### `com.hibiscusmc.hmccosmetics.user.CosmeticUser`

| Return | Method | Description |
|---|---|---|
| `Player` | `getPlayer()` | Bukkit player (may be null for NPC users) |
| `void` | `addCosmetic(Cosmetic, Color)` | Equip |
| `void` | `removeCosmeticSlot(CosmeticSlot)` | Remove |
| `Cosmetic` | `getCosmetic(CosmeticSlot)` | Get equipped |
| `ImmutableCollection<Cosmetic>` | `getCosmetics()` | All equipped |
| `boolean` | `hasCosmeticInSlot(CosmeticSlot)` | Check slot |
| `Set<CosmeticSlot>` | `getSlotsWithCosmetics()` | Occupied slots |
| `boolean` | `canEquipCosmetic(Cosmetic, boolean)` | Permission/condition check |
| `boolean` | `updateCosmetic(CosmeticSlot)` | Force refresh |
| `ItemStack` | `getUserCosmeticItem(CosmeticSlot)` | Display item |
| `Color` | `getCosmeticColor(CosmeticSlot)` | Applied color |
| `void` | `hideCosmetics(HiddenReason)` | Hide |
| `void` | `showCosmetics(HiddenReason)` | Show |
| `boolean` | `isHidden()` | Any reason active |
| `boolean` | `isHidden(HiddenReason)` | Specific reason |
| `void` | `silentlyAddHideFlag(HiddenReason)` | Hide without firing event |
| `void` | `enterWardrobe(Wardrobe, boolean)` | Enter wardrobe |
| `void` | `leaveWardrobe(boolean)` | Exit wardrobe |
| `boolean` | `isInWardrobe()` | In wardrobe |
| `void` | `spawnBackpack(CosmeticBackpackType)` | Spawn backpack |
| `void` | `despawnBackpack()` | Despawn |
| `void` | `respawnBackpack()` | Respawn |
| `boolean` | `isBackpackSpawned()` | Check |
| `void` | `spawnBalloon(CosmeticBalloonType)` | Spawn balloon |

### `com.hibiscusmc.hmccosmetics.cosmetic.Cosmetic` (abstract)

| Return | Method | Description |
|---|---|---|
| `String` | `getId()` | Cosmetic ID |
| `String` | `getPermission()` | Required permission (or null) |
| `boolean` | `requiresPermission()` | Has permission |
| `ItemStack` | `getItem()` | Display item (cloned) |
| `CosmeticSlot` | `getSlot()` | Target slot |
| `boolean` | `isDyeable()` | Supports color |

### `com.hibiscusmc.hmccosmetics.cosmetic.CosmeticSlot`

NOT an enum — registry-based.

| Return | Method | Description |
|---|---|---|
| `static CosmeticSlot` | `register(String id)` | Register new slot |
| `static Map<String, CosmeticSlot>` | `values()` | All slots |
| `static CosmeticSlot` | `valueOf(String name)` | Lookup (case-insensitive) |
| `static boolean` | `contains(String name)` | Existence check |

**Built-in slots:** `HELMET`, `CHESTPLATE`, `LEGGINGS`, `BOOTS`, `MAINHAND`, `OFFHAND`, `BACKPACK`, `BALLOON`

### Events (`com.hibiscusmc.hmccosmetics.api.events`)

| Event | Cancellable | Key Methods |
|---|---|---|
| `PlayerPreLoadEvent` | Yes | `getUniqueId()` |
| `PlayerLoadEvent` | No | `getUser()` |
| `PlayerPreUnloadEvent` | Yes | `getUser()` |
| `PlayerUnloadEvent` | No | `getUser()` |
| `PlayerCosmeticEquipEvent` | Yes | `getUser()`, `getCosmetic()`, `setCosmetic()` |
| `PlayerCosmeticPostEquipEvent` | No | `getUser()`, `getCosmetic()` |
| `PlayerCosmeticRemoveEvent` | Yes | `getUser()`, `getCosmetic()` |
| `PlayerCosmeticHideEvent` | Yes | `getUser()`, `getReason()` |
| `PlayerCosmeticShowEvent` | Yes | `getUser()`, `getReason()` |
| `PlayerWardrobeEnterEvent` | Yes | `getUser()`, `getWardrobe()`, `setWardrobe()` |
| `PlayerWardrobeLeaveEvent` | Yes | `getUser()`, `getWardrobeManager()` |
| `PlayerMenuOpenEvent` | Yes | `getUser()`, `getMenu()` |
| `PlayerMenuCloseEvent` | No | `getUser()`, `getMenu()`, `getReason()` |
| `HMCCosmeticSetupEvent` | No | (none — fires on plugin setup/reload) |
| `CosmeticTypeRegisterEvent` | No | `getId()`, `getConfig()` |
