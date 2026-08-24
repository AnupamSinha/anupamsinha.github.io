---
title: "Building a CLI with Spring Shell — Interactive Command-Line Apps"
date: 2026-08-22
categories: [Spring Boot, Tools]
tags: [spring-shell, cli, command-line, spring-boot, java-21, developer-tools]
description: "Build interactive command-line applications with Spring Shell. Covers @ShellComponent, @ShellMethod, parameter validation, tab completion, custom prompts, and packaging as a native executable."
mermaid: true
---

## The Problem

Not everything needs to be a web app. Sometimes you need:

- A DevOps tool that queries infrastructure
- A data migration script with interactive prompts
- An internal CLI for managing your application
- A developer utility that wraps complex workflows

You could use plain `args[]` parsing, but that means handling help text, tab completion, validation, and interactive mode yourself. Spring Shell gives you all of that for free.

## What is Spring Shell

Spring Shell is a framework for building interactive, production-grade CLI applications on top of Spring Boot. It provides:

- **Interactive REPL** — type commands, get results
- **Tab completion** — out of the box
- **Command grouping** — organize related commands
- **Parameter validation** — Bean Validation on command parameters
- **Custom prompts** — make it feel like your tool
- **Spring DI** — inject services, repositories, anything

```mermaid
graph LR
    A[User Input] --> B[Spring Shell]
    B --> C[Command Parser]
    C --> D[Parameter Resolver]
    D --> E["@ShellMethod"]
    E --> F[Your Business Logic]
    F --> G[Output]
```

## Setup

Add the Spring Shell starter to your `pom.xml`:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.shell</groupId>
        <artifactId>spring-shell-starter</artifactId>
        <version>3.4.0</version>
    </dependency>
</dependencies>
```

Enable interactive mode in `application.yml`:

```yaml
spring:
  shell:
    interactive:
      enabled: true
  main:
    banner-mode: off
```

## Writing Commands

### @ShellComponent and @ShellMethod

Commands are just Spring beans with annotated methods:

```java
@ShellComponent
public class GreetingCommands {

    @ShellMethod(key = "hello", value = "Say hello")
    public String hello(@ShellOption(defaultValue = "World") String name) {
        return "Hello, %s! Welcome to Spring Shell CLI.".formatted(name);
    }

    @ShellMethod(key = "date", value = "Display current date and time")
    public String date() {
        return LocalDateTime.now()
                .format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    }

    @ShellMethod(key = "version", value = "Display application version")
    public String version() {
        return """
                Spring Shell CLI v1.0.0
                Java %s
                OS: %s %s
                """.formatted(
                System.getProperty("java.version"),
                System.getProperty("os.name"),
                System.getProperty("os.arch"));
    }
}
```

Usage:
```
shell:> hello --name Anupam
Hello, Anupam! Welcome to Spring Shell CLI.

shell:> date
2026-08-22 14:30:45
```

### File Operations

A more practical example — file system commands:

```java
@ShellComponent
public class FileCommands {

    @ShellMethod(key = "list-files", value = "List files in a directory")
    public String listFiles(@ShellOption(defaultValue = ".") String directory)
            throws IOException {

        Path dir = Path.of(directory);
        if (!Files.isDirectory(dir)) {
            return "Not a directory: " + directory;
        }

        try (Stream<Path> paths = Files.list(dir)) {
            return paths
                    .map(path -> {
                        String type = Files.isDirectory(path) ? "[DIR]  " : "[FILE] ";
                        return type + path.getFileName();
                    })
                    .sorted()
                    .collect(Collectors.joining("\n"));
        }
    }

    @ShellMethod(key = "word-count", value = "Count words in a file")
    public String wordCount(String file) throws IOException {
        Path path = Path.of(file);
        if (!Files.exists(path)) {
            return "File not found: " + file;
        }

        String content = Files.readString(path);
        long lines = content.lines().count();
        long words = Stream.of(content.split("\\s+")).count();
        long chars = content.length();

        return "%d lines, %d words, %d characters".formatted(lines, words, chars);
    }
}
```

### HTTP Commands

Spring Shell isn't limited to local operations — you can call APIs too:

```java
@ShellComponent
public class HttpCommands {

    private final HttpClient httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10))
            .build();

    @ShellMethod(key = "http-get", value = "Perform an HTTP GET request")
    public String httpGet(String url) throws IOException, InterruptedException {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .GET()
                .timeout(Duration.ofSeconds(30))
                .build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        return "Status: %d\n\n%s".formatted(
                response.statusCode(),
                response.body().substring(0, Math.min(response.body().length(), 2000)));
    }

    @ShellMethod(key = "ping", value = "Ping a host")
    public String ping(String host) throws IOException {
        long start = System.currentTimeMillis();
        InetAddress address = InetAddress.getByName(host);
        boolean reachable = address.isReachable(5000);
        long elapsed = System.currentTimeMillis() - start;

        return reachable
                ? "Host %s (%s) is reachable — %d ms".formatted(host, address.getHostAddress(), elapsed)
                : "Host %s is not reachable".formatted(host);
    }
}
```

## Parameters and Validation

### @ShellOption

Control how parameters are parsed:

```java
@ShellMethod(key = "search", value = "Search files")
public String search(
        @ShellOption(value = "--path", defaultValue = ".") String path,
        @ShellOption(value = "--pattern") String pattern,
        @ShellOption(value = "--recursive", defaultValue = "false") boolean recursive) {
    // ...
}
```

Usage: `search --path /tmp --pattern "*.log" --recursive`

### Bean Validation

Add `jakarta.validation` and validate inputs:

```java
@ShellMethod(key = "create-user", value = "Create a new user")
public String createUser(
        @NotNull @Size(min = 3, max = 50) String username,
        @Email String email) {
    return "Created user: " + username + " (" + email + ")";
}
```

Spring Shell will display validation errors before your method even runs.

## Command Groups

Organize commands into logical groups using `@ShellCommandGroup`:

```java
@ShellComponent
@ShellCommandGroup("File Operations")
public class FileCommands {
    // Commands appear under "File Operations" in help output
}

@ShellComponent
@ShellCommandGroup("Network")
public class HttpCommands {
    // Commands appear under "Network" in help output
}
```

The `help` command automatically groups them:

```
File Operations
    list-files: List files in a directory
    file-info: Display file information
    word-count: Count words in a file

Network
    http-get: Perform an HTTP GET request
    ping: Ping a host
```

## Custom Prompt

Replace the default `shell:>` prompt:

```java
@Component
public class CustomPromptProvider implements PromptProvider {

    @Override
    public AttributedString getPrompt() {
        String cwd = Path.of("").toAbsolutePath().getFileName().toString();
        return new AttributedString(
                cwd + " ▸ ",
                AttributedStyle.DEFAULT.foreground(AttributedStyle.CYAN));
    }
}
```

Now your CLI shows: `my-project ▸`

## Tab Completion

Spring Shell provides tab completion for free. For custom completion, implement a `ValueProvider`:

```java
@Component
public class DirectoryValueProvider implements ValueProvider {

    @Override
    public List<CompletionProposal> complete(CompletionContext context) {
        String prefix = context.currentWordUpToCursor();
        try {
            Path dir = prefix.isEmpty() ? Path.of(".") : Path.of(prefix).getParent();
            if (dir == null) dir = Path.of(".");

            return Files.list(dir)
                    .filter(Files::isDirectory)
                    .map(p -> new CompletionProposal(p.toString()))
                    .toList();
        } catch (IOException e) {
            return List.of();
        }
    }
}
```

Wire it to a parameter:

```java
@ShellMethod(key = "cd", value = "Change directory")
public String cd(
        @ShellOption(valueProvider = DirectoryValueProvider.class) String path) {
    // ...
}
```

## Packaging

### Fat JAR

The standard Spring Boot fat JAR works out of the box:

```bash
./mvnw clean package
java -jar target/my-cli-0.0.1-SNAPSHOT.jar
```

### GraalVM Native Image

For instant startup (critical for CLIs), compile to a native executable:

```xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>
```

```bash
./mvnw -Pnative native:compile
./target/my-cli
```

Native images start in milliseconds instead of seconds — a huge difference for CLI tools that users run repeatedly.

## Common Problems

| Problem | Cause | Fix |
|---------|-------|-----|
| Application starts as web server | Missing shell starter or web starter present | Remove `spring-boot-starter-web`, keep only `spring-shell-starter` |
| Commands not discovered | Class not annotated with `@ShellComponent` | Add `@ShellComponent` annotation |
| Parameters not parsed | Method parameter names lost at compile time | Use `@ShellOption(value = "--name")` explicitly |
| Tab completion not working | Running from IDE without JLine terminal | Run from actual terminal, not IDE console |
| `NoSuchBeanDefinitionException` | Missing component scan | Ensure command classes are in a sub-package of main class |
| Non-interactive mode issues | `spring.shell.interactive.enabled` not set | Add to `application.yml` |

## Full Working Example

The complete implementation is available on GitHub:

- [spring-shell-cli](https://github.com/anupamsinha/spring-shell-cli) — File commands, HTTP commands, greeting commands with Spring Shell 3.4

Clone it, build with `./mvnw clean package`, and run the JAR to enter the interactive shell.

## References

- [Spring Shell Reference Documentation](https://docs.spring.io/spring-shell/reference/)
- [Spring Shell GitHub Repository](https://github.com/spring-projects/spring-shell)
- [GraalVM Native Image — Spring Boot](https://docs.spring.io/spring-boot/reference/native-image/index.html)
- [JLine 3 — Terminal Library](https://github.com/jline/jline3)
- [Spring Boot CLI Applications Guide](https://docs.spring.io/spring-boot/reference/using/command-line-runner.html)
