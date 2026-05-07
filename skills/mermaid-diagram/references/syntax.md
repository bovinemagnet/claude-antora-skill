# Mermaid syntax — extended reference

Load this when the inline cheatsheet in SKILL.md isn't enough — diagram types not covered there, or troubleshooting validation failures.

## Class diagram

```
classDiagram
  class Animal {
    +String name
    +int age
    +eat()
    +sleep()
  }
  class Dog {
    +String breed
    +bark()
  }
  Animal <|-- Dog : inherits
  Dog "1" --> "*" Toy : owns
```

- Visibility: `+` public, `-` private, `#` protected, `~` package.
- Relationships: `<|--` inheritance, `*--` composition, `o--` aggregation, `-->` association, `--` link, `..>` dependency, `..|>` realisation.
- Cardinality goes in quotes on either end.

## Entity-relationship diagram

```
erDiagram
  CUSTOMER ||--o{ ORDER : places
  ORDER ||--|{ LINE_ITEM : contains
  CUSTOMER {
    string name
    string email PK
  }
  ORDER {
    int id PK
    date placed_at
    string customer_email FK
  }
```

- Cardinality: `||` exactly one, `|o` zero or one, `}|` one or more, `}o` zero or more.
- Relationship line: `--` identifying, `..` non-identifying.
- Attributes: `string`, `int`, `date`, etc., with optional `PK`, `FK`, `UK` markers.

## Gantt chart

```
gantt
  title Project plan
  dateFormat YYYY-MM-DD
  section Discovery
    Interviews         :a1, 2026-01-10, 5d
    Synthesis          :after a1, 3d
  section Build
    Backend            :b1, 2026-01-20, 10d
    Frontend           :b2, after b1, 8d
    Integration        :crit, after b2, 4d
```

- Task statuses: `done`, `active`, `crit`. Combine: `crit, active`.
- Date formats follow `dateFormat`. Durations: `5d`, `1w`, `2h`.

## User journey

```
journey
  title User onboarding
  section Sign-up
    Visit landing page : 5: User
    Create account     : 3: User, System
    Verify email       : 2: User
  section First use
    Empty dashboard    : 1: User
    Add first item     : 4: User
    See result         : 5: User, System
```

- Score 1-5 (1 = bad, 5 = great). Comma-separated actors after the score.
- Sections group steps; one user journey per diagram.

## Git graph

```
gitGraph
  commit
  branch feature
  checkout feature
  commit
  commit
  checkout main
  merge feature
  commit tag: "v1.0"
```

- `commit`, `branch <name>`, `checkout <name>`, `merge <name>`.
- Tag a commit with `commit tag: "label"`.

## Pie chart

```
pie title Browser market share
  "Chrome" : 65
  "Safari" : 18
  "Firefox" : 9
  "Other"  : 8
```

Numeric weights; Mermaid renders the percentages.

## Validation troubleshooting

### "Parse error on line N"

Almost always a label with special characters. Mermaid treats `(`, `)`, `&`, `:`, `<`, `>` as syntax in unquoted contexts. Fix: wrap the label in quotes.

```
B[Build & deploy]      %% breaks
B["Build & deploy"]    %% works
```

### "Cannot read properties of undefined"

A reference to a node/participant/state that wasn't declared. In strict diagram types (sequence, state, ER) declare all entities up front before drawing relationships.

### Class diagram weirdness with generics

```
class List~T~ {
  +add(item T)
  +get(index int) T
}
```

Generics use `~T~` not `<T>` (angle brackets clash with HTML).

### Sequence diagram "participant order"

Participants render in the order they're declared. If the diagram looks tangled, declare them in the order you want them shown.

### State diagram nested states

```
stateDiagram-v2
  [*] --> Active
  state Active {
    [*] --> Running
    Running --> Paused: pause
    Paused --> Running: resume
  }
  Active --> [*]
```

Nested state IDs need to be unique across the whole diagram, not just the parent state.

### mmdc CLI common flags

```
mmdc -i input.mmd -o output.svg          # SVG (recommended for docs)
mmdc -i input.mmd -o output.png          # PNG raster
mmdc -i input.mmd -o output.svg -t dark  # dark theme
mmdc -i input.mmd -o output.svg -b transparent  # transparent background
```

For Antora docs the standard combo is `-o <name>.svg` with default theme. The `images/` directory in the module is the destination.

### Theme attributes inside the diagram

```
%%{init: {'theme':'default', 'themeVariables': {'primaryColor':'#0077b6'}}}%%
flowchart TD
  A --> B
```

Embedded theme overrides survive validation but should be used sparingly — most diagrams don't need theming, and theme drift across pages looks worse than uniform defaults.
