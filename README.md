# JuByteCase-WikiJuByte Case API
The JuByte Case API is a library for managing player cases and logging case openings.

Installation
Maven
Add the following repository and dependency to your pom.xml:

java
<repositories>
  <repository>
    <id>jubyte</id>
    <name>JuByte</name>
    <url>https://repo.jubyte.com/repository/jubyte/</url>
  </repository>
</repositories>

<dependencies>
  <dependency>
    <groupId>com.jubyte.caseopening</groupId>
    <artifactId>JuByteCaseAPI</artifactId>
    <version>1.2.5-RELEASE</version>
  </dependency>
</dependencies>
Gradle
Add the following repository and dependency to your build.gradle:

java
repositories {
  maven {
    url 'https://repo.jubyte.com/repository/jubyte/'
    name 'JuByte'
    artifactUrls = ['https://repo.jubyte.com/repository/jubyte/']
  }
}

dependencies {
  implementation 'com.jubyte.caseopening:JuByteCaseAPI:1.2.5-RELEASE'
}
Usage
The JuByte Case API provides two interfaces for managing player cases and logging case openings: CasePlayerAPI and CaseLogAPI, respectively. To use these interfaces, first obtain an instance of CaseAPI using the static method getInstance(). Then, use the methods provided by the CasePlayerAPI and CaseLogAPI interfaces to manage player cases and log case openings, respectively.

java
import com.jubyte.caseopening.api.CaseAPI;
import com.jubyte.caseopening.api.log.CaseLogAPI;
import com.jubyte.caseopening.api.log.CaseLogEntry;
import com.jubyte.caseopening.api.player.CasePlayerAPI;

import java.util.List;
import java.util.UUID;

public class Example {

  public static void main(String[] args) {
    // Get an instance of CaseAPI
    CaseAPI caseAPI = CaseAPI.getInstance();

    // Use CasePlayerAPI to manage player cases
    UUID playerUUID = UUID.randomUUID();
    int currentCaseAmount = caseAPI.getCasePlayerApi().getCaseAmount(playerUUID, "example_case");
    caseAPI.getCasePlayerApi().addCaseAmount(playerUUID, 1, "example_case");
    int updatedCaseAmount = caseAPI.getCasePlayerApi().getCaseAmount(playerUUID, "example_case");

    // Use CaseLogAPI to log case openings
    List<CaseLogEntry> caseLogEntries = caseAPI.getCaseLogApi().getCaseLog(playerUUID);
    System.out.println("Player " + playerUUID + " has opened " + caseLogEntries.size() + " cases.");

    caseAPI.getCaseLogApi().getOpenedChests("example_case", amount -> {
      System.out.println("Player " + playerUUID + " has opened " + amount + " example_case cases.");
    });
  }

}

License
The JuByte Case API is licensed under the MIT License. See LICENSE for more information.
