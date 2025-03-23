---
title: Simplify Java and SpringBoot migration with OpenRewrite
categories: [backend]
date: 2025-03-22 22:50:00
tags: [TIL, springboot, java]
image: "/assets/img/java_migration/java_migration.jpeg"
---
## Challenges of Migration
With older versions like Spring Boot 2.x reaching [end-of-life](https://endoflife.date/spring-boot) and no longer receiving support, migrating to newer versions is essential for security, compatibility, and performance improvements. However, the migration process comes with several challenges:

**1. Breaking Changes:** Major version upgrades often introduce breaking changes. For example, Spring Boot 3.x requires **Java 17** and migrates from `javax.*` to `jakarta.* packages`.

**2. Deprecated APIs:** Many commonly used APIs and patterns become deprecated and require replacements.

**3. Manual Updates:** Traditional migration requires manually updating dependencies, refactoring code, and fixing compatibility issues.

**4. Time-Consuming:** Large codebases may take weeks or months to migrate, increasing project costs and risks.

**5. Testing Burden:** Every change must be thoroughly tested to ensure functionality remains intact.

So, how can we simplify and accelerate the migration process? This is where OpenRewrite comes in handy.

## OpenRewrite
[OpenRewrite](https://docs.openrewrite.org/) is an open-source tool for automated code refactoring, helping developers reduce technical debt. It provides prebuilt refactoring recipes for framework migrations, security fixes, and code styling, cutting down effort from hours to minutes. 

Plugins for Gradle and Maven make it easy to apply these changes to repositories. Originally focused on Java, the OpenRewrite community is actively expanding support for more languages and frameworks.

Key Features
- **Automated Refactoring:** Automatically updates code syntax, dependencies, and patterns
- **Recipe-based:** Uses declarative recipes to define transformation rules
- **Style Preservation:** Maintains original code formatting and comments
- **Large-Scale Changes:** Can process entire codebases consistently
- **Extensible:** Supports custom recipes for specific migration needs

### How does it work?
- OpenRewrite modifies [Lossless Semantic Trees (LSTs)](https://docs.openrewrite.org/concepts-and-explanations/lossless-semantic-trees), which represent your source code, and converts them back into source code.

- You can review the changes and commit them as needed.

- Modifications are made using Visitors, which are grouped into **Recipes**.

- Recipes ensure changes are minimally invasive and maintain the original formatting.

## Practice
In this post, I'll demonstrate how to migrate a simple CRUD Spring Boot application built with Java 8, Spring Boot 2.x, and JUnit 4 to Java 21, Spring Boot 3.3, and JUnit 5 using OpenRewrite.

### Codebase
**Checkout here: [GitHub](https://github.com/hgky95/TIL/tree/main/java-springboot-migration)**

#### 1) pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.14</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo</name>
    <description>Demo project for Spring Boot Migration</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        // dependencies: starter web, data-jpa, etc
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project> 
```

#### 2) UserController.java
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private UserService userService;

    @RequestMapping(method = RequestMethod.GET)
    public List<User> getAllUsers() {
        return userRepository.findAll();
    }

    @RequestMapping(method = RequestMethod.POST)
    public ResponseEntity<?> createUser(@Valid @RequestBody User user) {
        User savedUser = userRepository.save(user);
        return ResponseEntity.ok().build();
    }

    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public ResponseEntity<User> getUserById(@PathVariable("id") Long id) {
        User user = userRepository.findById(id).orElse(null);
        return user != null ? ResponseEntity.ok(user) : ResponseEntity.notFound().build();
    }

    @RequestMapping(value = "/username")
    public ResponseEntity<User> getUserByUsername(@RequestParam String username) {
        User user = userService.findByUsername(username);
        return user != null ? ResponseEntity.ok(user) : ResponseEntity.notFound().build();
    }

} 
```

#### 3) UserService.java
```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public User findByUsername(String username) {
        return userRepository.findByUsernameNative(username);
    }
} 
```

#### 4) UserRepository.java
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query(value = "SELECT * FROM users WHERE username = ?1", nativeQuery = true)
    User findByUsernameNative(String username);

}
```

#### 5) User.java
```java
import javax.persistence.*;
import javax.validation.constraints.NotNull;

@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    @NotNull
    @Column(nullable = false)
    private String username;

    @Column
    private String email;

    // setter, getter
} 
```

#### 6) UserControllerTest.java
```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private UserRepository userRepository;

    @Before
    public void setup() {
        userRepository.deleteAll();
    }

    @Test
    public void testCreateUser() throws Exception {
        String userJson = "{\"username\":\"testuser\",\"email\":\"test@example.com\"}";

        mockMvc.perform(MockMvcRequestBuilders.post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(userJson))
                .andExpect(MockMvcResultMatchers.status().isOk());
    }

    @Test
    public void testGetUser() throws Exception {
        User user = new User();
        user.setUsername("testuser");
        user.setEmail("test@example.com");
        userRepository.save(user);

        mockMvc.perform(MockMvcRequestBuilders.get("/api/users/" + user.getId()))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.username").value("testuser"));
    }
} 
```
### Manually migration
Before using OpenRewrite, let's see what happens if we manually migrate to Java 21 and Spring Boot 3.3.

First, update `pom.xml`

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.10</version>
    <relativePath/>
</parent>

<properties>
    <java.version>21</java.version>
</properties>
```

Next, clean and build the project using Maven:
```mvn clean install```

There are several compilation errors

![compiled error](/assets/img/java_migration/manually_migrate.png)

Okay, let's revert the changes and migrate using OpenRewrite.

### Migration with OpenRewrite

#### Add OpenRewrite plugin
In `pom.xml`, add the OpenRewrite Maven plugin. If you're using Gradle, you can add the Gradle plugin.
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.openrewrite.maven</groupId>
            <artifactId>rewrite-maven-plugin</artifactId>
            <version>6.3.2</version>
        </plugin>
    </plugins>
</build>
```

#### Choose the Recipe for migration
To discover all the available recipes, you can check the [Recipe catalog](https://docs.openrewrite.org/recipes), which lists all the available recipes, including: Java, Spring Boot, Hibernate, Quarkus, Scala, .NET, Jenkins, etc.

In this showcase, I will use three recipes: Java 8 to 21, Spring 2 to 3, and JUnit 4 to 5.

Let’s add the recipes to the pom.xml; each recipe has its own dependency.

```xml
<plugin>
    <groupId>org.openrewrite.maven</groupId>
    <artifactId>rewrite-maven-plugin</artifactId>
    <version>6.3.2</version>
    <configuration>
        <exportDatatables>true</exportDatatables>
        <activeRecipes>
            <recipe>org.openrewrite.java.migrate.UpgradeToJava21</recipe>
            <recipe>org.openrewrite.java.spring.boot2.SpringBoot2JUnit4to5Migration</recipe>
            <recipe>org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_3</recipe>
        </activeRecipes>
    </configuration>
    <dependencies>
        <dependency>
            <groupId>org.openrewrite.recipe</groupId>
            <artifactId>rewrite-migrate-java</artifactId>
            <version>3.4.0</version>
        </dependency>
        <dependency>
            <groupId>org.openrewrite.recipe</groupId>
            <artifactId>rewrite-spring</artifactId>
            <version>6.3.0</version>
        </dependency>
    </dependencies>
</plugin>
```

Now, run Maven to install the OpenRewrite plugin and its recipes:

`mvn clean install`

#### Preview Migration
OpenRewrite provides the `dryRun` mode, which allows developers to preview the changes before actually applying them by using:

`mvn rewrite:dryRun`

You can see the changes that will be used for the actual migration.

![dry run mode](/assets/img/java_migration/dryRun.png)

#### Apply the migration
Now, let’s do the migration

`mvn rewrite:run`

Use your IDE or a diff checker tool to review the changes.

**1. Update pom.xml**: It automatically update the SpringBoot version to 3.3.10 and Java to 21, remove JUnit4.
![Update Pom.xml](/assets/img/java_migration/update_pom.png)

**2. Update Controller**:
Migrates from `javax.*` to `jakarta.* packages`, changing to use `@GetMapping`, `@PostMapping` for dedicated annotations.
![Update Controller](/assets/img/java_migration/update_controller.png)

**3. Update Unit Test**:
Replace `@Before` by `@BeforeEach`, update package name
![Update Unit Test](/assets/img/java_migration/update_test.png)

## Limitations
In this showcase, OpenRewrite successfully migrates to Java 21 and Spring Boot 3.3, but it still has some limitations.
Since OpenRewrite relies on predefined recipes, it supports many common frameworks but not all of them.

For example, if you need to migrate a third-party library like Ehcache2 to Ehcache3 (which is no longer supported in Spring 3.0), OpenRewrite does not provide a built-in recipe. In such cases, you must either write a custom recipe or perform the migration manually.

If you create a custom recipe, consider contributing it to the OpenRewrite community to help others with similar migrations.

## Summary
OpenRewrite significantly simplifies the Java and Spring Boot migration process by:

- Automating repetitive code changes
- Reducing migration time and effort
- Minimizing human errors
- Standardizing the migration approach

While OpenRewrite **doesn't eliminate the need for testing and validation**, it reduces the manual effort required for migrations. This allows developers to focus on more complex migration aspects and business logic updates.

## References
1. https://docs.openrewrite.org/
2. https://github.com/openrewrite 
