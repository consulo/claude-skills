---
name: code-style
description: >
  Use this skill whenever Java code is written or reviewed in Consulo. Trigger on: any Java
  file creation or modification, "code style", "formatting", "null safety", "nullable",
  "NullMarked", "collection type", "module-info", "opens", or any request to write new Java
  code, "getInstance", "constructor injection", "field naming", "my prefix", "static field".
  "var keyword", "final parameter", "method parameter".
  MUST be used proactively for all Java code produced in this project.
---

# Consulo Java Code Style

---

## Formatting

- **Indent**: 4 spaces (no tabs)
- **Line length**: max 140 characters
- **Braces**: always required — never omit `{}` even for single-statement bodies
- **`else` / `catch` / `finally`**: always on a new line after the closing `}`

```java
// ✅ Correct
if (condition) {
    doSomething();
}
else {
    doOther();
}

try {
    riskyCall();
}
catch (IOException e) {
    handle(e);
}
finally {
    cleanup();
}

// ❌ Wrong — no braces
if (condition)
    doSomething();

// ❌ Wrong — else / catch on same line as }
if (condition) {
    doSomething();
} else {
    doOther();
}
```

---

## Null Safety

### `@NullMarked` — apply to every module

Every `module-info.java` must be annotated with `@NullMarked`. This makes **non-null the
default** for all types in the module — do not add `@Nonnull` / `@NotNull` anywhere.

```java
import org.jspecify.annotations.NullMarked;

@NullMarked
module consulo.my.module {
    requires consulo.some.api;
    exports consulo.my.module;
}
```

### `@Nullable` — mark exceptions only

Use `org.jspecify.annotations.Nullable` on return types, parameters, and fields that
genuinely can be `null`. Place the annotation directly before the type.

```java
import org.jspecify.annotations.Nullable;

// Return type
public @Nullable TextAttributes getAttributes(TextAttributesKey key) { ... }

// Parameter
public void process(Change change, @Nullable ChangeList changeList) { ... }

// Field
private @Nullable BuildProcessHandler myProcessHandler;
```

**Never use:** `@NotNull`, `@Nonnull` (from any package) — non-null is the default under
`@NullMarked`.

---

## Collections — Interface Type, Impl Value

Declare variables and fields using the **interface type**; assign the **concrete
implementation**.

```java
// ✅ Correct
List<String> names = new ArrayList<>();
Map<String, Integer> counts = new HashMap<>();
Map<String, Value> ordered = new LinkedHashMap<>();
Set<String> seen = new LinkedHashSet<>();
ConcurrentMap<Object, Object> cache = new ConcurrentHashMap<>();
Collection<LocalChangeList> pending = new HashSet<>();

// ❌ Wrong — concrete type on the left
ArrayList<String> names = new ArrayList<>();
HashMap<String, Integer> counts = new HashMap<>();
```

Applies to: fields, local variables, method parameters, and return types.

---

## `module-info.java` — No `open module`

Modules must **not** be declared `open`. Use qualified `opens ... to` to expose only the
specific packages that need reflective access, and only to the modules that need it.

```java
// ✅ Correct — selective opens
@NullMarked
module consulo.my.impl {
    requires consulo.util.xml.serializer;
    requires consulo.proxy;

    exports consulo.my.impl.api;

    opens consulo.my.impl.internal.config to consulo.util.xml.serializer;
    opens consulo.my.impl.internal.model to consulo.proxy;
}

// ❌ Wrong — entire module is open
open module consulo.my.impl {
    requires consulo.util.xml.serializer;
}
```

Common `opens` targets:

| Target module | Reason |
|---|---|
| `consulo.util.xml.serializer` | XML/state serialization of `@State` beans |
| `consulo.proxy` | Dynamic proxy generation |
| `consulo.component.impl` | Component wiring / injection |
| `args4j` | CLI argument parsing |
| `com.sun.jna` | Native library access |

---

## Field Naming

| Kind | Convention | Example |
|---|---|---|
| Instance field | `my` prefix + camelCase | `myName`, `myFileList`, `myProject` |
| Static constant | `UPPER_SNAKE_CASE` | `MAX_SIZE`, `DEFAULT_TIMEOUT` |
| Static non-constant | `our` prefix + camelCase (rare) | `ourInstance` |

```java
// ✅ Correct
private final List<String> myItems = new ArrayList<>();
private @Nullable Project myProject;
private static final int MAX_RETRIES = 3;
private static final Logger LOG = Logger.getInstance(MyClass.class);

// ❌ Wrong
private final List<String> items = new ArrayList<>();   // missing my prefix
private static final int maxRetries = 3;               // constant not UPPER_CASE
```

Constructor parameters must **not** carry the `my` prefix — so assignment never needs `this.`:

```java
// ✅ Correct — param is "project", field is "myProject", no this.
@Inject
public MyServiceImpl(Project project) {
    myProject = project;
}

// ❌ Wrong — param mirrors field name, forces this.
@Inject
public MyServiceImpl(Project myProject) {
    this.myProject = myProject;
}
```

---

## Dependency Injection — Constructor over `getInstance()`

Prefer **constructor injection** for dependencies. Avoid calling `ServiceManager.getService()`,
`ApplicationManager.getApplication()`, or any `#getInstance()` inside class logic — receive
dependencies as constructor parameters instead.

```java
// ✅ Correct — injected via constructor
public class MyServiceImpl {
    private final Project myProject;
    private final FileDocumentManager myFileDocumentManager;

    @Inject
    public MyServiceImpl(Project project, FileDocumentManager fileDocumentManager) {
        myProject = project;
        myFileDocumentManager = fileDocumentManager;
    }
}

// ❌ Wrong — pulling dependencies at call sites
public class MyServiceImpl {
    public void doWork() {
        Project project = ProjectManager.getInstance().getDefaultProject();
        FileDocumentManager fdm = FileDocumentManager.getInstance();
    }
}
```

`getInstance()` is acceptable only in static utility methods or in contexts where injection
is not available (e.g., `AnAction.actionPerformed` receiving an `AnActionEvent`).

**Never create static instances of services or components.** Static instances bypass the
injection framework, which means the container cannot manage their lifecycle or dispose them
correctly — leading to leaks and stale state.

```java
// ❌ Wrong — static instance, never disposed
private static final MyService ourService = new MyServiceImpl();

// ❌ Wrong — lazy static singleton, bypasses container
private static MyService ourInstance;
public static MyService getInstance() {
    if (ourInstance == null) {
        ourInstance = new MyServiceImpl();
    }
    return ourInstance;
}

// ✅ Correct — injected, container owns the lifecycle
@Inject
public MyConsumer(MyService service) {
    myService = service;
}
```

---

## No `var`

Always declare explicit types. `var` is forbidden — it hides the type and reduces readability.

```java
// ✅ Correct
List<String> names = new ArrayList<>();
FileEditorManager manager = FileEditorManager.getInstance(myProject);

// ❌ Wrong
var names = new ArrayList<String>();
var manager = FileEditorManager.getInstance(myProject);
```

---

## Method Parameters — No Noise, No Mutation

**No `final` on parameters or local variables** unless the value is genuinely shared across
a lambda/anonymous class boundary where the compiler requires it. Redundant `final` is noise; omit it.

**Do not reassign method parameters** — if a transformed value is needed, assign it to a
new local variable.

```java
// ✅ Correct
public void process(String name, int count) {
    String trimmed = name.trim();
    int adjusted = count > 0 ? count : 1;
}

// ❌ Wrong — final on parameters
public void process(final String name, final int count) { ... }

// ❌ Wrong — unnecessary final on local variable
public void process(String name) {
    final String trimmed = name.trim();
    final int length = trimmed.length();
}

// ❌ Wrong — parameter reassigned
public void process(String name, int count) {
    name = name.trim();
    count = count > 0 ? count : 1;
}
```
