# JuByte CaseAPI

JuByte CaseAPI is the official Java library for programmatically extending the [JuByte CaseOpening](https://www.jubyte.com/) ecosystem. It exposes type-safe access to player inventories, cases, reward logs, and integration points of the Spigot/Paper plugin. This documentation is derived from the public CaseOpening source code and summarizes its architecture as well as every relevant API extension point.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Installation](#installation)
   - [Maven](#maven)
   - [Gradle (KTS/Groovy)](#gradle-ktsgroovy)
4. [Quick Start](#quick-start)
5. [API Reference](#api-reference)
   - [CaseAPI – Entry Point](#caseapi--entry-point)
   - [CasePlayerAPI – Player Inventories](#caseplayerapi--player-inventories)
   - [CaseLogAPI – Reward and Activity Logs](#caselogapi--reward-and-activity-logs)
   - [Additional Service Interfaces](#additional-service-interfaces)
6. [Data Models](#data-models)
7. [Events & Listener Hooks](#events--listener-hooks)
8. [Asynchronous Processing & Threading](#asynchronous-processing--threading)
9. [Persistence & Storage Backends](#persistence--storage-backends)
10. [Best Practices](#best-practices)
11. [Troubleshooting & FAQ](#troubleshooting--faq)
12. [License](#license)

---

## Overview

CaseOpening manages virtual cases, keys, and loot tables for Minecraft servers. The plugin tracks available cases for each player, creates case logs when openings occur, and offers extensive configuration options for rewards. The CaseAPI abstracts these features so you can manage cases, build custom UIs, or connect external systems (such as web shops).

Key characteristics:

- **Singleton entry point:** `CaseAPI#getInstance()` guarantees plugin-wide access.
- **Service split:** Player-specific actions live in `CasePlayerAPI`, logging features in `CaseLogAPI`, and additional service classes cover cases, keys, animations, drops, and configuration.
- **Return types:** The API mostly uses synchronous return values, but I/O-heavy operations rely on `CompletableFuture`.
- **Null safety:** Missing players or cases are represented via `Optional<T>` or empty collections.
- **Events:** Bukkit/Paper events announce case openings, drops, animations, and log updates.

## Architecture

The CaseOpening code base follows a modular layered structure:

1. **API module (`JuByteCaseApi`):** Contains interfaces, data objects, and utility classes documented here.
2. **Core module (`CaseOpening`):** Implements the API, manages persistence (SQL, MongoDB, or flat file), executes commands, and registers events.
3. **Integrations:** Additional modules (webhooks, PlaceholderAPI, multi-server) consume the API and demonstrate advanced usage.

The API implementation follows a service-locator pattern: `CaseAPI` manages internal service instances. Each service implements an interface in the API module, ensuring backwards compatibility across plugin updates. Data objects are immutable value types (builder pattern), while mutating operations are exposed through services only.

## Installation

### Maven

```xml
<repositories>
  <repository>
    <id>jubyte</id>
    <name>JuByte</name>
    <url>https://repo.jubyte.com/artifactory/jubyte/</url>
  </repository>
</repositories>
```

Add the dependency:

```xml
<dependency>
    <groupId>com.jubyte.caseopening</groupId>
    <artifactId>JuByteCaseApi</artifactId>
    <version>1.9.2-RELEASE</version>
    <scope>provided</scope>
</dependency>
```

> **Note:** The API is already loaded on the server by the CaseOpening plugin. Declare it as `provided` in your build so it is not bundled twice.

### Gradle (KTS/Groovy)

```kotlin
repositories {
    maven("https://repo.jubyte.com/artifactory/jubyte/")
}

dependencies {
    compileOnly("com.jubyte.caseopening:JuByteCaseApi:1.9.2-RELEASE")
}
```

The Groovy DSL uses the same entries.

## Quick Start

```java
import com.jubyte.caseopening.api.CaseAPI;
import com.jubyte.caseopening.api.log.CaseLogAPI;
import com.jubyte.caseopening.api.log.CaseLogEntry;
import com.jubyte.caseopening.api.player.CasePlayerAPI;

import java.util.List;
import java.util.UUID;

public final class CaseExample {

    public void runDemo(UUID playerUuid) {
        CaseAPI caseAPI = CaseAPI.getInstance();
        CasePlayerAPI playerApi = caseAPI.getCasePlayerApi();
        CaseLogAPI logApi = caseAPI.getCaseLogApi();

        String caseName = "starter";

        int current = playerApi.getCaseAmount(playerUuid, caseName);
        playerApi.addCaseAmount(playerUuid, 5, caseName);
        playerApi.removeCaseAmount(playerUuid, 1, caseName);

        List<CaseLogEntry> recent = logApi.getCaseLog(playerUuid, caseName);
        recent.stream()
              .findFirst()
              .ifPresent(entry -> getLogger().info(() ->
                      "Last drop: " + entry.getRewardName() + " (" + entry.getRarity() + ")"));
    }
}
```

### Plugin lifecycle initialization

1. **`onLoad`:** Ensure CaseOpening is declared as a dependency (`Bukkit.getPluginManager().getPlugin("CaseOpening")`).
2. **`onEnable`:** After the plugin started successfully, obtain the API (`CaseAPI.getInstance()`). Use dependency injection or a service locator to share the instance across your plugin.
3. **`onDisable`:** Cancel running asynchronous tasks (for example log lookups) and unregister listeners.

### Error handling

- Unknown cases throw a `CaseNotFoundException`. Handle it directly or check availability with `caseAPI.getCaseDefinition(caseName).isPresent()` first.
- Network or database failures propagate as `CompletionException` in futures. Use `.exceptionally()` handlers to log them.

## API Reference

### CaseAPI – Entry Point

`CaseAPI` is a singleton and internally uses thread-safe lazy initialization. Important methods:

| Method | Description |
| --- | --- |
| `CaseAPI#getInstance()` | Returns the active API instance or throws `IllegalStateException` when CaseOpening has not finished initializing. |
| `CaseAPI#getCasePlayerApi()` | Access to all player inventory operations. |
| `CaseAPI#getCaseLogApi()` | Access to log data (openings, rewards). |
| `CaseAPI#getCaseDefinitionApi()` | Provides the management interface for cases (load, create, update definitions). |
| `CaseAPI#getCaseKeyApi()` | Offers key management (key balances, case bindings). |
| `CaseAPI#getScheduler()` | Abstraction over the Bukkit scheduler for internal async tasks. |
| `CaseAPI#getSerializer()` | Exposes YAML/JSON serializers for exporting case data. |

> **Compatibility:** New API methods are added via default interfaces. Always check for additional services after updating the plugin.

### CasePlayerAPI – Player Inventories

`CasePlayerAPI` encapsulates all operations around a player's case balances. The source code exposes the following core methods:

| Method | Description |
| --- | --- |
| `int getCaseAmount(UUID player, String caseName)` | Returns the number of available cases. Missing entries default to `0`. |
| `void setCaseAmount(UUID player, int amount, String caseName)` | Sets the exact balance. Negative values are clamped to `0`. |
| `void addCaseAmount(UUID player, int amount, String caseName)` | Increases the balance by `amount`. |
| `void removeCaseAmount(UUID player, int amount, String caseName)` | Decreases the balance by `amount` without dropping below `0`. |
| `CompletableFuture<Map<String, Integer>> getCaseAmountsAsync(UUID player)` | Provides all case balances asynchronously. |
| `boolean hasCase(UUID player, String caseName)` | Checks whether the player owns at least one instance of a case. |
| `void clearCases(UUID player)` | Removes all cases (useful for resets). |

#### Example: Web shop integration

```java
public void grantPurchase(UUID playerUuid, String caseName, int amount) {
    CasePlayerAPI playerApi = CaseAPI.getInstance().getCasePlayerApi();

    playerApi.addCaseAmount(playerUuid, amount, caseName);
    Bukkit.getScheduler().runTask(CaseOpening.getInstance(), () ->
        Bukkit.getPlayer(playerUuid)
              .ifPresent(player -> player.sendMessage("§aYou received " + amount + "x " + caseName + "!")));
}
```

### CaseLogAPI – Reward and Activity Logs

`CaseLogAPI` manages historic case openings of a player. Data is delivered chronologically by default.

| Method | Description |
| --- | --- |
| `List<CaseLogEntry> getCaseLog(UUID player)` | All log entries of a player. |
| `List<CaseLogEntry> getCaseLog(UUID player, String caseName)` | Log entries filtered by case. |
| `CompletableFuture<List<CaseLogEntry>> getCaseLogAsync(UUID player)` | Async variant useful for UI rendering. |
| `void appendLogEntry(CaseLogEntry entry)` | Appends a manual log entry (for example for special events). |
| `void deleteLogEntries(UUID player)` | Removes logs of a player. |

`CaseLogEntry` exposes getters for the player UUID, case name, reward identifier, rarity (`CaseRarity`), timestamp (`Instant`), and metadata such as amount or trigger.

### Additional Service Interfaces

Besides the APIs above the CaseOpening project offers additional interfaces:

- **`CaseDefinitionAPI`** – Create, update, and list cases including loot tables (`CaseReward`, `CaseRewardPool`).
- **`CaseKeyAPI`** – Manage physical or virtual keys. Includes methods like `getKeyAmount`, `addKeyAmount`, `consumeKey`.
- **`CaseRewardAPI`** – Directly manipulate individual rewards, e.g., adjust drop rates or temporarily disable them.
- **`AnimationAPI`** – Access animation profiles (spins, roulette, instant drop), including `playAnimation(Player, CaseDefinition)`.
- **`EconomyAPI`** – Integrate external economy systems (Vault, TokenManager). Provides helper methods to charge currency before opening.

Service implementations may vary depending on the installed modules. Use `instanceof` checks or feature flags (`CaseAPI#getCapabilities()`) to safely interact with optional services.

## Data Models

The API module defines several central immutable types:

- **`CaseDefinition`** – Contains metadata (ID, display name, icon, description), drop configuration, and requirements (key type, required permission).
- **`CaseReward`** – Describes individual rewards. Attributes: identifier, item stack/command, weight, broadcast flag, minimum/maximum amount.
- **`CaseRewardPool`** – Groups rewards by rarity or category. Supports weighted selection and guaranteed drops.
- **`CaseKey`** – Represents keys (ID, display, material, consumption rules).
- **`CaseLogEntry`** – Documents openings (`player`, `caseName`, `rewardId`, `rarity`, `timestamp`, `source`).

Builders and factory methods (`CaseDefinition.builder()`, `CaseReward.ofItem(...)`) simplify creating new content. All models override `equals`/`hashCode`, making them suitable for caches or collections.

## Events & Listener Hooks

CaseOpening fires Bukkit/Paper events whenever relevant actions happen:

- `CaseOpenEvent` – Triggered before a case opens. Listeners can cancel (`setCancelled(true)`) or adjust costs.
- `CaseRewardEvent` – Fired after the reward is selected. Contains reward data, allowing external systems to augment rewards (for example statistics).
- `CaseAnimationStartEvent` / `CaseAnimationEndEvent` – Signal the start or end of an animation.
- `CaseLogCreateEvent` – Fired once a new log entry was persisted.

Register listeners inside your plugin (`PluginManager#registerEvents`). Watch the thread context: all events fire synchronously on the main thread.

## Asynchronous Processing & Threading

- Read-heavy operations (`getCaseLogAsync`, `getCaseAmountsAsync`) run on a dedicated thread pool. Use `thenAcceptAsync` to switch UI updates back to the main thread.
- Write operations (for example `setCaseAmount`) run synchronously to guarantee data consistency. Schedule them via Bukkit tasks if they are triggered off the main thread.
- Use `CaseAPI#getScheduler()` or `Bukkit.getScheduler()` for safe context switching.

## Persistence & Storage Backends

The core implementation supports multiple storage strategies:

| Backend | Description |
| --- | --- |
| MySQL/MariaDB | Enabled by default. Tables: `co_player_cases`, `co_case_logs`, `co_case_definitions`. |
| MongoDB | Optional module for document-based storage. Suitable for large log volumes. |
| YAML/JSON | Flat-file adapters for development or single-server setups. |

The API fully abstracts the storage layer. Developers do not execute SQL directly; all operations run through service interfaces.

## Best Practices

1. **Validation:** Call `caseDefinitionApi.exists(caseName)` before changing player balances.
2. **Consistent naming:** Use the case ID (lower case, underscores) for API calls rather than the display name.
3. **Internationalisation:** Reward texts originate from the case configuration. Use `CaseDefinition#getDisplayName(Locale)` for localized output.
4. **Caching:** Cache frequently requested case definitions locally (for example with `LoadingCache`). Refresh via `CaseDefinitionApi#reload(caseName)`.
5. **Transactions:** For coupled operations (such as currency deduction + case booking) use atomic sequences—verify balance, add the case, then record the log.
6. **Testing:** Mock services via the API interfaces to run unit tests without a live server.

## Troubleshooting & FAQ

**Q:** `IllegalStateException: CaseAPI has not been initialised yet`

> **A:** Make sure your plugin loads after CaseOpening (`depend: [CaseOpening]` in `plugin.yml`).

**Q:** `CaseNotFoundException`

> **A:** The case ID is unknown. Use `caseDefinitionApi.getDefinitions()` to inspect definitions at runtime or synchronise your configuration with the server.

**Q:** Logs stay empty.

> **A:** Enable log storage in the CaseOpening configuration (`logging.enabled: true`) and verify the database connection.

**Q:** How do I create cases programmatically?

> **A:** Use the `CaseDefinitionAPI` builder, populate rewards, and persist via `saveCaseDefinition`. Afterwards call `reload` to update caches.

## License

JuByte CaseAPI is released under the [MIT License](https://opensource.org/licenses/MIT).

```text
MIT License

Copyright (c) 2023 - 2025 JuByte

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
