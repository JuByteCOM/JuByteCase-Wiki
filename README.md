```
# JuByte CaseAPI

JuByte CaseAPI is a Java library that provides an API to interact with the JuByte CaseOpening system. 

## Installation

### Maven

Add the following repository to your `pom.xml`:

```xml
<repositories>
  <repository>
    <id>jubyte</id>
    <name>JuByte</name>
    <url>https://repo.jubyte.com/repository/jubyte/</url>
  </repository>
</repositories>
```

And then add the following dependency:

```xml
<dependency>
  <groupId>com.jubyte.caseopening</groupId>
  <artifactId>JuByteCaseAPI</artifactId>
  <version>1.2.5-RELEASE</version>
</dependency>
```

### Gradle

Add the following repository to your `build.gradle` file:

```groovy
repositories {
    maven {
        url "https://repo.jubyte.com/repository/jubyte/"
    }
}
```

And then add the following dependency:

```groovy
dependencies {
    implementation 'com.jubyte.caseopening:JuByteCaseAPI:1.2.5-RELEASE'
}
```

## Usage

```java
import com.jubyte.caseopening.api.CaseAPI;
import com.jubyte.caseopening.api.log.CaseLogAPI;
import com.jubyte.caseopening.api.player.CasePlayerAPI;

import java.util.UUID;

public class Example {

    public static void main(String[] args) {
        CaseAPI caseAPI = CaseAPI.getInstance();
        CasePlayerAPI casePlayerAPI = caseAPI.getCasePlayerApi();
        CaseLogAPI caseLogAPI = caseAPI.getCaseLogApi();

        UUID playerUUID = UUID.randomUUID();
        String caseName = "my_case";

        // Get the amount of cases the player has for a specific case
        int amount = casePlayerAPI.getCaseAmount(playerUUID, caseName);

        // Set the amount of cases the player has for a specific case
        casePlayerAPI.setCaseAmount(playerUUID, 10, caseName);

        // Add cases to the player's inventory for a specific case
        casePlayerAPI.addCaseAmount(playerUUID, 5, caseName);

        // Remove cases from the player's inventory for a specific case
        casePlayerAPI.removeCaseAmount(playerUUID, 3, caseName);

        // Get the case log for a specific player
        caseLogAPI.getCaseLog(playerUUID);

        // Get the case log for a specific player and case name
        caseLogAPI.getCaseLog(playerUUID, caseName);
    }
}
```

## License

JuByte CaseAPI is released under the [MIT License](https://opensource.org/licenses/MIT).
```
