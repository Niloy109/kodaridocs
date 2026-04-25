# Phoenix Crates API Reference

Premium crate plugin by Phoenix Plugins. The API lets plugins query crate types and keys, open crates programmatically (random or forced reward), fetch player data, override reward selection, mutate winning items, and register custom animations. Add `PhoenixCrates` to `depend` or `softdepend` in plugin.yml.

## Code Examples

### Getting the API

```java
import com.phoenixplugins.phoenixcrates.api.PhoenixCratesAPI;
import com.phoenixplugins.phoenixcrates.api.managers.CratesManager;
import com.phoenixplugins.phoenixcrates.api.managers.KeysManager;
import com.phoenixplugins.phoenixcrates.api.managers.PlayersManager;

CratesManager crates = PhoenixCratesAPI.getCratesManager();
KeysManager keys = PhoenixCratesAPI.getKeysManager();
PlayersManager players = PhoenixCratesAPI.getPlayersManager();
```

### Look Up Crates and Keys

```java
import com.phoenixplugins.phoenixcrates.api.PhoenixCratesAPI;
import com.phoenixplugins.phoenixcrates.api.crate.CrateInstance;
import com.phoenixplugins.phoenixcrates.api.crate.CrateType;
import com.phoenixplugins.phoenixcrates.api.key.Key;
import org.bukkit.block.Block;

// Get a crate definition by ID
CrateType type = PhoenixCratesAPI.getCratesManager().getTypeByIdentifier("mythical");

// Get a placed crate at a block location
CrateInstance crate = PhoenixCratesAPI.getCratesManager().getCrateByBlock(block);

// Get a key by ID
Key key = PhoenixCratesAPI.getKeysManager().getKeyByIdentifier("mythical_key");
if (key != null) {
    org.bukkit.inventory.ItemStack keyItem = key.getPlainItem().getItemStack();
}
```

### Open a Crate (Random Reward, Key From Player)

```java
import com.phoenixplugins.phoenixcrates.api.PhoenixCratesAPI;
import com.phoenixplugins.phoenixcrates.api.crate.CrateInstance;
import com.phoenixplugins.phoenixcrates.api.session.OpeningContext;
import com.phoenixplugins.phoenixcrates.api.exceptions.CrateOpeningException;
import org.bukkit.block.Block;
import org.bukkit.entity.Player;

CrateInstance crate = PhoenixCratesAPI.getCratesManager().getCrateByBlock(block);
if (crate != null) {
    OpeningContext ctx = OpeningContext.random(
            player,
            true, // requireKeyInMainHand
            1     // openingQuantity
    );

    try {
        crate.openCrate(ctx);
    } catch (CrateOpeningException e) {
        player.sendMessage("Couldn't open: " + e.getMessage());
    }
}
```

### Force a Specific Reward

```java
import com.phoenixplugins.phoenixcrates.api.PhoenixCratesAPI;
import com.phoenixplugins.phoenixcrates.api.crate.CrateInstance;
import com.phoenixplugins.phoenixcrates.api.crate.CrateType;
import com.phoenixplugins.phoenixcrates.api.reward.Reward;
import com.phoenixplugins.phoenixcrates.api.session.OpeningContext;
import com.phoenixplugins.phoenixcrates.api.exceptions.CrateOpeningException;

CrateInstance crate = PhoenixCratesAPI.getCratesManager().getCrateByBlock(block);
if (crate != null) {
    Reward reward = crate.getType().getRewardByIdentifier("legendary_drop");
    if (reward != null) {
        OpeningContext ctx = OpeningContext.selective(player, reward);
        try {
            crate.openCrate(ctx);
        } catch (CrateOpeningException e) {
            // handle
        }
    }
}
```

### Query Crate Properties

```java
import com.phoenixplugins.phoenixcrates.api.crate.CrateType;
import com.phoenixplugins.phoenixcrates.api.reward.Reward;

import java.util.List;
import java.util.Set;

CrateType type = /* ... */;

String id = type.getIdentifier();
String displayName = type.getDisplayName();
String permission = type.getPermission();
boolean enabled = type.isEnabled();
boolean keyRequired = type.isKeyRequired();
int cooldownSeconds = type.getOpenCooldownSeconds();
double moneyCost = type.getOpenMoneyCost();

Set<String> linkedKeys = type.getLinkedKeysIds();
List<? extends Reward> allRewards = type.getRegisteredRewards();
List<? extends Reward> available = type.getAvailableRewards();
```

### Fetch Player Data Async

```java
import com.phoenixplugins.phoenixcrates.api.PhoenixCratesAPI;
import com.phoenixplugins.phoenixcrates.api.player.PlayerData;
import org.bukkit.entity.Player;

PhoenixCratesAPI.getPlayersManager()
    .fetchPlayerDataAsync(player)
    .thenAccept(data -> {
        // PlayerData is @ApiStatus.Experimental — API may change
    });

// Synchronous cached lookup
PlayerData cached = PhoenixCratesAPI.getPlayersManager().getCachedDataNow(player);
boolean isCached = PhoenixCratesAPI.getPlayersManager().isDataCached(player);
```

### Listen for Crate Events

```java
import com.phoenixplugins.phoenixcrates.api.crate.events.CratePreOpenEvent;
import com.phoenixplugins.phoenixcrates.api.crate.events.CrateOpenEvent;
import com.phoenixplugins.phoenixcrates.api.crate.events.CratePlaceEvent;
import com.phoenixplugins.phoenixcrates.api.crate.events.CratePreviewOpenEvent;
import com.phoenixplugins.phoenixcrates.api.crate.events.CrateRewardSelectionEvent;
import com.phoenixplugins.phoenixcrates.api.crate.events.CrateRewardPlayerEvent;
import com.phoenixplugins.phoenixcrates.api.crate.events.CrateRewardWinItemsEvent;
import com.phoenixplugins.phoenixcrates.api.crate.events.CratePreRerollEvent;
import com.phoenixplugins.phoenixcrates.api.reward.BaseReward;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;

import java.util.Comparator;

public class CrateListener implements Listener {

    @EventHandler
    public void onPreOpen(CratePreOpenEvent event) {
        // Cancellable
        if (!event.getPlayer().hasPermission("myplugin.opencrates")) {
            event.setCancelled(true);
        }
    }

    @EventHandler
    public void onOpen(CrateOpenEvent event) {
        // NOT cancellable
    }

    @EventHandler
    public void onPlace(CratePlaceEvent event) {
        // Cancellable
        event.setCancelled(true); // block placement
    }

    @EventHandler
    public void onPreviewOpen(CratePreviewOpenEvent event) {
        // Cancellable
    }

    @EventHandler
    public void onRewardSelection(CrateRewardSelectionEvent event) {
        // Override the reward selection
        BaseReward best = event.getCandidates().stream()
                .max(Comparator.comparingDouble(BaseReward::getWeight))
                .orElse(null);

        if (best != null) {
            event.setSelectedReward(best);
            event.setHandled(true);
        }
    }

    @EventHandler
    public void onRewardPlayer(CrateRewardPlayerEvent event) {
        // NOT cancellable — fired when reward is given
    }

    @EventHandler
    public void onWinItems(CrateRewardWinItemsEvent event) {
        // Mutate the items the player physically receives
        event.getWinItems().add(new org.bukkit.inventory.ItemStack(org.bukkit.Material.DIAMOND));
    }

    @EventHandler
    public void onPreReroll(CratePreRerollEvent event) {
        // Cancellable
        event.setCancelled(true);
    }
}
```

### Pending Item Delivery (Offline Rewards)

```java
import com.phoenixplugins.phoenixcrates.api.events.items.ItemDeliveryEvent;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;

public class DeliveryListener implements Listener {

    @EventHandler
    public void onDelivery(ItemDeliveryEvent event) {
        // Mutate or consume items before delivery to the player
        event.getRemainingItems().clear();
        event.consumeAll(); // mark all as delivered
    }
}
```

## API Reference (Trimmed)

### `com.phoenixplugins.phoenixcrates.api.PhoenixCratesAPI` (all static)

| Return | Method | Description |
|---|---|---|
| `CratesManager` | `getCratesManager()` | Crate types and instances |
| `KeysManager` | `getKeysManager()` | Key definitions |
| `PlayersManager` | `getPlayersManager()` | Player data |
| `AnimationsRegistry` | `getAnimationsRegistry()` | Custom animations |

### `com.phoenixplugins.phoenixcrates.api.managers.CratesManager`

| Return | Method | Description |
|---|---|---|
| `List<? extends CrateType>` | `getRegisteredTypes()` | All crate types |
| `int` | `countRegisteredTypes()` | Type count |
| `Set<String>` | `getCratesIdentifier()` | All crate IDs |
| `CrateType` | `getTypeByIdentifier(String)` | Type by ID |
| `CrateInstance` | `getCrateByBlock(Block)` | Placed crate at block |
| `List<? extends OpeningSession>` | `getPlayerOpeningSessions(Player)` | Active sessions |

### `com.phoenixplugins.phoenixcrates.api.managers.KeysManager`

| Return | Method | Description |
|---|---|---|
| `List<? extends Key>` | `getRegisteredKeys()` | All keys |
| `Set<String>` | `getKeysIdentifier()` | All key IDs |
| `Key` | `getKeyByIdentifier(String)` | Key by ID |

### `com.phoenixplugins.phoenixcrates.api.managers.PlayersManager`

| Return | Method | Description |
|---|---|---|
| `CompletableFuture<? extends PlayerData>` | `fetchPlayerDataAsync(OfflinePlayer)` | Async fetch |
| `PlayerData` | `getCachedDataNow(Player)` | Cached lookup |
| `PlayerData` | `getPlayerIfCached(OfflinePlayer, boolean requestCache)` | Optional cache |
| `boolean` | `isDataCached(OfflinePlayer)` | Cache check |

### `com.phoenixplugins.phoenixcrates.api.crate.CrateInstance`

| Return | Method | Description |
|---|---|---|
| `CrateType` | `getType()` | Crate type |
| `Location` | `getLocation()` | Block location |
| `void` | `openCrate(OpeningContext)` | Open (throws `CrateOpeningException`) |

### `com.phoenixplugins.phoenixcrates.api.crate.CrateType`

| Return | Method | Description |
|---|---|---|
| `String` | `getIdentifier()` | Crate ID |
| `String` | `getDisplayName()` | Display name |
| `String` | `getPermission()` | Permission node |
| `boolean` | `isEnabled()` | Is enabled |
| `boolean` | `isKeyRequired()` | Requires key |
| `boolean` | `isPermissionRequired()` | Requires permission |
| `int` | `getOpenCooldownSeconds()` | Cooldown |
| `double` | `getOpenMoneyCost()` | Money cost |
| `Set<String>` | `getLinkedKeysIds()` | Compatible key IDs |
| `List<? extends Reward>` | `getRegisteredRewards()` | All rewards |
| `List<? extends Reward>` | `getAvailableRewards()` | Currently available |
| `Reward` | `getRewardByIdentifier(String)` | Reward by ID |

### `com.phoenixplugins.phoenixcrates.api.session.OpeningContext`

| Return | Method | Description |
|---|---|---|
| `static OpeningContext` | `random(Player, boolean requireKeyInMainHand, int quantity)` | Random open with key consumption |
| `static OpeningContext` | `selective(Player, Reward)` | Forced reward |
| `Player` | `getPlayer()` | Player |
| `int` | `getRequiredKeys()` | Keys needed |

### `com.phoenixplugins.phoenixcrates.api.key.Key`

| Return | Method | Description |
|---|---|---|
| `String` | `getDisplayName()` | Display name |
| `boolean` | `isEnabled()` | Enabled |
| `boolean` | `isVirtual()` | Virtual key (no item) |
| `boolean` | `isGlowing()` | Glow effect |
| `Item` | `getPlainItem()` | Key item wrapper |

### `com.phoenixplugins.phoenixcrates.api.reward.Reward`

| Return | Method | Description |
|---|---|---|
| `String` | `getIdentifier()` | Reward ID |
| `Item` | `getDisplayItem()` | Display item |
| `List<? extends Item>` | `getWinItems()` | Items given on win |
| `List<String>` | `getWinCommands()` | Commands run on win |
| `double` | `getWeight()` | Selection weight |
| `double` | `getPercentage()` | Win percentage |
| `int` | `getRequiredKeys()` | Keys needed |
| `int` | `getWinLimits()` | Max wins |
| `boolean` | `isBroadcast()` | Broadcasts on win |
| `boolean` | `isAlternative()` | Is fallback reward |
| `AlternativeReward` | `getAlternative()` | Fallback reward |
| `List<String>` | `getRestrictedPermissions()` | Restricted to permissions |

### Events (`com.phoenixplugins.phoenixcrates.api.crate.events`)

| Event | Cancellable | Key Methods |
|---|---|---|
| `CratePreOpenEvent` | Yes | `getType()`, `getPlayer()` |
| `CrateOpenEvent` | No | `getType()`, `getCrate()`, `getPlayer()` |
| `CratePlaceEvent` | Yes | `getWhoPlaced()`, `getLocation()` |
| `CratePreviewOpenEvent` | Yes | `getCrateType()`, `getPlayer()` |
| `CrateRewardSelectionEvent` | No (use `setHandled`) | `getCandidates()`, `getSelectedReward()`, `setSelectedReward()`, `setHandled(boolean)` |
| `CrateRewardPlayerEvent` | No | `getCrate()`, `getReward()`, `getPlayer()` |
| `CrateRewardWinItemsEvent` | No (mutate list) | `getReward()`, `getPlayer()`, `getWinItems()` (mutable) |
| `CratePreRerollEvent` | Yes | (constructor takes `OpeningSession`) |
| `CrateRerollEvent` | No | `getCrate()`, `getReward()`, `getPlayer()` |
| `ItemDeliveryEvent` | (mutate items) | `getRemainingItems()`, `setRemainingItems()`, `consumeAll()` |

> **Note:** `PlayerData` is `@ApiStatus.Experimental` — methods may change between releases.
