---
name: ontology
description: Stanford's 7-step ontology methodology for TypeScript domain modeling — systematic process from competency questions to typed data architectures
version: 1.0.0
author: OpenClaw
triggers:
  - design a data model
  - define types for
  - build a taxonomy
  - classification system
  - refactor god interface
  - domain modeling
  - type hierarchy
  - knowledge graph
  - competency questions
  - ontology
---

# Ontology Engineering for TypeScript

> "An ontology is an explicit specification of a conceptualization."
> — Thomas R. Gruber, Stanford KSL, 1993

## Overview

This skill teaches Stanford's 7-step ontology development methodology adapted for TypeScript + SQL domain modeling. It transforms unstructured domain knowledge into rigorous, typed data architectures.

**Three theoretical pillars:**
1. **Gruber 1993** — Defined ontology, Ontolingua translation architecture, 5 design criteria
2. **Noy & McGuinness 2001** — Systematized into 7-step Ontology 101 methodology
3. **Shen Xin 2025** — Applied to enterprise systems (Digital Intelligence Triad, ITPAO fractal)

**Three fundamental rules:**
1. There is no single correct way to model a domain — the best solution depends on the application
2. Ontology development is necessarily iterative
3. Concepts should be close to real objects and relationships in your domain

## When to Use

**Use when:**
- Designing a new domain data model (types, interfaces, database tables)
- Building a classification or taxonomy (skill categories, user levels, content types)
- Creating a knowledge base with typed entities and relationships
- Designing a multi-dimensional scoring or matching system
- Refactoring a growing type system that has become inconsistent
- Evaluating whether to use inheritance, composition, or discriminated unions
- Building enterprise AI systems that need structured domain knowledge

**Don't use when:**
- Simple CRUD with obvious entity shapes (just write the interface)
- Single-file utility functions with no domain modeling
- Performance optimization of existing models
- The domain is already well-modeled and stable

---

## The 7-Step Process

### Step 1: Determine Domain and Scope

**Ontology concept:** Competency questions define the boundary of what your model must represent.

**Code equivalent:** Write a list of questions your type system must answer. These become acceptance criteria.

**Process:**
1. Define the domain in one sentence
2. Write 5-10 competency questions the system must answer
3. Identify what is explicitly OUT of scope
4. Identify who uses and maintains this model

**Example (Jyotish system):**
```
Domain: Vedic astrology chart interpretation

Competency questions:
- Which planet rules a given zodiac sign?     → Rashi.ruler: GrahaId
- Is Mars friendly to Jupiter?                 → Graha.friendlyPlanets: GrahaId[]
- What yogas are present with Mars in kendra?  → YogaCondition with condition types
- What is compatibility score between charts?  → Multi-dimensional Kuta scoring

Out of scope: Western astrology, daily horoscope text generation
```

**Decision checkpoint:** If you cannot write 5-10 competency questions, the domain is not well-enough understood. Brainstorm first.

**Template:**
```
Domain name: ___________
One-sentence purpose: ___________
Primary users: ___________

Competency Questions:
1. ___________  → Data requirement: ___________
2. ___________  → Data requirement: ___________
...

In Scope: ___________
Out of Scope: ___________
Might Add Later (v2): ___________
```

### Step 2: Consider Reusing Existing Ontologies

**Ontology concept:** Check existing knowledge bases before building from scratch.

**Checklist:**
- [ ] Check `src/types/` for existing type definitions
- [ ] Check `src/data/` for existing knowledge base files
- [ ] Check database schema for existing tables
- [ ] Check npm ecosystem for domain-specific type packages
- [ ] Check open-source ontologies (schema.org, Dublin Core, etc.)

**Rule:** Reuse > Extend > Create from scratch.

### Step 3: Enumerate Important Terms

**Ontology concept:** Brainstorm all domain concepts without organizing.

**Process:**
1. List every domain term — don't organize yet
2. Group loosely into nouns (entities), adjectives (properties), verbs (relationships)

**Example:**
```
Nouns (entities):     User, Post, Comment, Draft, Tag, Community
Adjectives (props):   published, archived, pinned, featured, deleted
Verbs (relations):    publishes, comments-on, belongs-to, tags, votes-on
```

### Step 4: Define Classes and Class Hierarchy

**The is-a litmus test:**
```
"Every [Child] is a [Parent]" must be TRUE
"Every [Parent] is a [Child]" must be FALSE
```

**TypeScript mapping:**

| Ontology Pattern | TypeScript Pattern | When to Use |
|-----------------|-------------------|-------------|
| Class hierarchy (is-a) | `interface D extends B` | Shared structure + specialization |
| Disjoint siblings | Discriminated union | Mutually exclusive variants |
| Abstract class | `interface` (not instantiated) | Shared contract only |
| Multiple inheritance | `type C = A & B` (intersection) | Rare; prefer composition |

**Critical rules:**
1. **is-a only**: Post is NOT a subclass of User (that's has-a)
2. **No cycles**: If A extends B and B extends A → same thing, merge them
3. **Disjointness**: Mutually exclusive siblings → discriminated union
4. **Sibling consistency**: All siblings at same level of generality
5. **Minimum subclasses**: 1 subclass = modeling problem; >12 = need intermediate categories

**Decision tree — Union vs Extends vs Composition:**
```
Are variants mutually exclusive?
  YES → Use discriminated union: type T = A | B | C
  NO  → Do variants share most fields?
    YES → Use extends: interface B extends A
    NO  → Use composition: type C = A & B
```

**Example:**
```typescript
// Discriminated union (mutually exclusive content types)
type Content =
  | { kind: 'post'; title: string; body: string }
  | { kind: 'moment'; media: string; caption?: string }
  | { kind: 'draft'; title: string; status: DraftStatus }

// Extension (shared base + specialization)
interface BaseEntity { id: string; createdAt: string; updatedAt: string }
interface Post extends BaseEntity { title: string; body: string; authorId: string }

// Composition (independent aspects)
type Timestamped = { createdAt: string; updatedAt: string }
type Authored = { authorId: string }
type Post = { id: string; title: string; body: string } & Timestamped & Authored
```

### Step 5: Define Properties (Slots)

**Key distinction:**
- **Intrinsic properties**: Belong to the entity itself (title, name, status)
- **Extrinsic properties**: Come from relationships (authorId, communityId)

**Rules:**
1. A property attaches to the MOST GENERAL class that has it
2. If ALL instances share a property → it's on the base class
3. If only SOME instances → it's on a subclass or variant
4. Relationships are modeled as ID references, not nested objects

**Example:**
```typescript
interface Post {
  // Intrinsic
  id: string
  title: string
  body: string
  status: 'draft' | 'published' | 'archived'

  // Extrinsic (relationships via ID)
  authorId: string       // → User
  communityId: string    // → Community
  tagIds: string[]       // → Tag[]
}
```

### Step 6: Define Constraints (Facets)

**Three-layer constraint alignment:**

```typescript
// Layer 1: TypeScript (compile time)
interface CreditAccount {
  balance: number         // must exist (not optional)
  currency: Currency      // type-constrained
}

// Layer 2: Zod (runtime validation)
const CreditSchema = z.object({
  balance: z.number().min(0).max(999999),    // range
  currency: z.enum(['CNY', 'USD']),          // domain
})

// Layer 3: SQL (storage)
// balance REAL NOT NULL DEFAULT 0 CHECK(balance >= 0)
// currency TEXT NOT NULL CHECK(currency IN ('CNY','USD'))
```

**Constraint types:**
| Type | TypeScript | Zod | SQL |
|------|-----------|-----|-----|
| Required | Non-optional field | `.min(1)` / no `.optional()` | `NOT NULL` |
| Optional | `field?: Type` | `.optional()` | nullable column |
| Cardinality | `T[]` / `T \| null` | `.array().min(1)` | separate table |
| Value range | Branded type | `.min()` / `.max()` | `CHECK(...)` |
| Value domain | Union literal | `.enum([...])` | `CHECK(... IN ...)` |

### Step 7: Create Instances

**Ontology concept:** Populate the model with concrete data. Instances validate your type system.

**Code equivalent:** `const` data arrays and seed data.

**Rule:** If real data doesn't fit your types, the types need revision — not the data.

**Example:**
```typescript
const GRAHAS: readonly Graha[] = [
  { id: 'surya',   name: 'Sun',   element: 'fire',  nature: 'malefic' },
  { id: 'chandra', name: 'Moon',  element: 'water', nature: 'benefic' },
  { id: 'mangal',  name: 'Mars',  element: 'fire',  nature: 'malefic' },
] as const satisfies readonly Graha[]

// Validation: does every instance satisfy the interface?
// If not → revise the interface, not the data
```

---

## Gruber's 5 Design Criteria

From Gruber, "A Translation Approach to Portable Ontology Specifications" (Stanford KSL-92-71, 1993):

### 1. Clarity
**Intent:** Definitions should be objective and formal, not depend on social context.
- ✅ Do: Use branded types for distinct concepts: `type UserId = string & { __brand: 'UserId' }`
- ❌ Don't: Use bare `string` for everything

### 2. Coherence
**Intent:** Inferred conclusions must not contradict definitions.
- ✅ Do: Use exhaustive discriminated unions: `type Status = 'draft' | 'published' | 'archived'`
- ❌ Don't: Use `string` and hope runtime values match

### 3. Extendibility
**Intent:** New concepts can be defined using existing vocabulary without modifying existing definitions.
- ✅ Do: Add new union variant: `type Content = Post | Moment | Draft | Poll`
- ❌ Don't: Add `pollOptions?: PollOption[]` to Post interface

### 4. Minimal Encoding Bias
**Intent:** Conceptual definitions should not depend on encoding choices.
- ✅ Do: `role: 'admin' | 'member' | 'guest'`
- ❌ Don't: `role: 0 | 1 | 2  // what does 2 mean?`

### 5. Minimal Ontological Commitment
**Intent:** Define only what is needed. Make as few claims as possible about the world.
- ✅ Do: `interface Post { id: string; title: string; body: string }`
- ❌ Don't: `interface Post { /* 50 optional fields for every possible future use */ }`

---

## Ontology → TypeScript Mapping (Core)

| Ontology Concept | TypeScript Equivalent | Example |
|-----------------|----------------------|---------|
| Class | `interface` / `type` | `interface User { ... }` |
| Subclass (is-a) | Union variant | `{ kind: 'admin'; permissions: string[] }` |
| Property (slot) | Interface field | `title: string` |
| Facet (constraint) | Zod schema / branded type | `z.string().min(1).max(200)` |
| Instance | `const` data / seed row | `{ id: 'surya', name: 'Sun' }` |
| Enumeration | String literal union | `'draft' \| 'published' \| 'archived'` |
| Relation | Foreign key / ID reference | `authorId: UserId` |
| Cardinality | Array / optional / required | `tags: string[]` (0..N) |
| Domain | Interface that owns property | `Post.title` (not global) |
| Range | Property value type | `createdAt: ISO8601String` |
| Inverse relation | Bidirectional ID refs | `Post.authorId ↔ User.postIds` |
| Transitive relation | Recursive type / graph | `Category.parentId: CategoryId \| null` |
| Disjoint classes | Discriminated union | `type Mode = 'silicon' \| 'carbon'` |
| Covering axiom | Exhaustive switch | All variants handled in `switch` |
| Annotation | JSDoc / branded type name | `/** @semantic Revenue */` |

---

## Decision Trees

### 1. Union vs Extends vs Composition

```
Are the variants mutually exclusive?
├── YES → Discriminated union: type T = A | B | C
└── NO  → Do they share >50% of fields?
    ├── YES → extends: interface B extends A
    └── NO  → Composition: type C = A & B
```

### 2. Enum vs String Literal vs Const Object

```
Does each value need associated data?
├── YES → Const object array: const ITEMS = [{ id, name, ... }] as const
└── NO  → Are values known at compile time?
    ├── YES → String literal union: type T = 'a' | 'b' | 'c'
    └── NO  → Branded string: type T = string & { __brand: 'T' }
```

### 3. Intrinsic vs Extrinsic Property

```
Does the property make sense without any other entity?
├── YES → Intrinsic: put it on the interface directly
└── NO  → It's a relationship
    → Does it reference a single entity?
    ├── YES → Foreign key: entityId: EntityId
    └── NO  → Junction table / array of IDs
```

---

## Auto-Trigger Keywords

This skill activates automatically when it detects these patterns:

- "design a data model" / "define types for" / "build a taxonomy"
- "classification system" / "type hierarchy" / "knowledge graph"
- "refactor god interface" / "too many optional fields"
- "discriminated union or extends" / "interface vs type"
- "competency questions" / "domain modeling" / "ontology"
- "本体论" / "领域建模" / "类型层级" / "分类体系"

---

## Pre-Implementation Checklist

Before implementing any domain model, verify:

- [ ] **Step 1**: 5+ competency questions written and answered
- [ ] **Step 2**: Searched existing codebase for reusable types
- [ ] **Step 3**: All domain terms enumerated (nouns/adjectives/verbs)
- [ ] **Step 4**: Class hierarchy passes is-a litmus test
- [ ] **Step 4**: Decision tree applied (union vs extends vs composition)
- [ ] **Step 5**: Properties attached to most general applicable class
- [ ] **Step 5**: Intrinsic vs extrinsic distinction clear
- [ ] **Step 6**: Three-layer constraints aligned (TS + Zod + SQL)
- [ ] **Step 7**: At least 3 realistic instances created and validated
- [ ] **Gruber**: All 5 design criteria checked

---

## References

- Gruber, T. R. (1993). "A Translation Approach to Portable Ontology Specifications." Stanford KSL-92-71.
- Noy, N. F. & McGuinness, D. L. (2001). "Ontology Development 101: A Guide to Creating Your First Ontology." Stanford SMI-2001-0880.
- Shen, X. (2025). "本体论下的企业级系统架构设计" + "基于数字员工的合馈制体系."
- Stanford Protégé: https://protege.stanford.edu/
