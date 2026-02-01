# JuByte CaseAPI

JuByte CaseAPI is the official Java library for programmatically extending the [JuByte CaseOpening](https://www.jubyte.com/) ecosystem. It exposes type-safe access to player case balances and case opening logs. This wiki is derived from the public CaseOpening source code and documents every API entry point that actually exists in the plugin.

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
   - [CasePlayerAPI – Player Case Balances](#caseplayerapi--player-case-balances)
   - [CaseLogAPI – Case Opening Logs](#caselogapi--case-opening-logs)
6. [Data Model](#data-model)
7. [Best Practices](#best-practices)
8. [Troubleshooting & FAQ](#troubleshooting--faq)
9. [License](#license)

---

## Overview

CaseOpening manages virtual cases and their rewards for Minecraft servers. The CaseAPI gives you a small, focused surface so you can:

- Read or change how many cases a player owns.
- Read log entries of previous case openings.
- Ask how many chests have been opened (globally or for a specific case).

Everything is intentionally simple. There are only three main types in the API: `CaseAPI`, `CasePlayerAPI`, and `CaseLogAPI`.

## Architecture

The API follows a service-locator pattern:

1. **`CaseAPI`** is the single entry point. You always call `CaseAPI.getInstance()` first.
2. From there, you can ask for the **player service** (`CasePlayerAPI`) and the **log service** (`CaseLogAPI`).
3. Data from logs is represented by the **`CaseLogEntry`** model.

The API is synchronous and returns simple values like `int`, `List`, or callbacks via `Consumer`.

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
    <version>1.10.2-RELEASE</version>
    <scope>provided</scope>
</dependency>
```

> **Note:** The API is already loaded on the server by the CaseOpening plugin. Declare it as `provided` so it is not bundled twice.

### Gradle (KTS/Groovy)

```kotlin
repositories {
    maven("https://repo.jubyte.com/artifactory/jubyte/")
}

dependencies {
    compileOnly("com.jubyte.caseopening:JuByteCaseApi:1.10.2-RELEASE")
}
```

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
        if (!recent.isEmpty()) {
            CaseLogEntry entry = recent.get(0);
            getLogger().info("Last drop: " + entry.getAwardName());
        }
    }
}
```

### Plugin lifecycle initialization

1. **`onLoad`:** Ensure CaseOpening is declared as a dependency in `plugin.yml` (`depend: [CaseOpening]`).
2. **`onEnable`:** After CaseOpening starts, call `CaseAPI.getInstance()`.
3. **`onDisable`:** Cancel any tasks you started yourself.

## API Reference

### CaseAPI – Entry Point

`CaseAPI` is the global entry point. You use it to access the player and log services.

| Method | Description |
| --- | --- |
| `CaseAPI#getInstance()` | Returns the current API instance (or `null` if CaseOpening is not ready). |
| `CaseAPI#getCasePlayerApi()` | Access to player case balances. |
| `CaseAPI#getCaseLogApi()` | Access to case opening logs. |

### CasePlayerAPI – Player Case Balances

`CasePlayerAPI` manages how many cases a player owns for each case name.

| Method | Description |
| --- | --- |
| `int getCaseAmount(UUID uuid, String caseName)` | Returns how many cases the player has for that case. |
| `void setCaseAmount(UUID uuid, int amount, String caseName)` | Sets the exact number of cases. |
| `void addCaseAmount(UUID uuid, int amount, String caseName)` | Adds cases to the current amount. |
| `void removeCaseAmount(UUID uuid, int amount, String caseName)` | Removes cases from the current amount. |

### CaseLogAPI – Case Opening Logs

`CaseLogAPI` lets you read log entries and get total open counts.

| Method | Description |
| --- | --- |
| `List<CaseLogEntry> getCaseLog(UUID uuid)` | Returns all log entries for a player. |
| `List<CaseLogEntry> getCaseLog(UUID uuid, String caseName)` | Returns log entries for a player filtered by case name. |
| `void getOpenedChests(Consumer<Long> whenReceived)` | Calls your callback with the total number of opened chests. |
| `void getOpenedChests(String caseName, Consumer<Long> whenReceived)` | Calls your callback with the total for a specific case name. |

## Data Model

### CaseLogEntry

A `CaseLogEntry` represents one opening in the log. It stores:

- `id`: Numeric log entry ID.
- `uuid`: UUID of the player.
- `chestName`: Name of the opened case.
- `awardName`: Name of the reward.
- `time`: Unix timestamp in milliseconds.

## Best Practices

1. **Keep case names consistent.** Always use the same internal case name when reading and writing balances.
2. **Use `addCaseAmount` and `removeCaseAmount` for adjustments.** It keeps your code readable.
3. **Handle missing API gracefully.** `CaseAPI.getInstance()` can return `null` if CaseOpening is not ready.
4. **Log totals with callbacks.** `getOpenedChests` delivers data via a `Consumer<Long>` so you can keep the main thread free.

## Troubleshooting & FAQ

**Q:** `CaseAPI.getInstance()` returns `null`

> **A:** Ensure your plugin loads after CaseOpening (`depend: [CaseOpening]` in `plugin.yml`) and call it in `onEnable`.

**Q:** I get zero results from `getCaseLog`

> **A:** Check if the server is actually recording logs and that the player has opened cases.

**Q:** How do I count total openings?

> **A:** Use `getOpenedChests(Consumer<Long>)` or the case-specific variant.

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
