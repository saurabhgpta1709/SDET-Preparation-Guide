# TypeScript Mastery Guide: Basic to Advanced Interview Questions

**Audience:** Automation engineers, SDETs, and developers preparing for
technical interviews\
**Goal:** Build a strong TypeScript foundation, understand the type
system, write maintainable code, and answer common interview questions
with confidence.

> **How to use this guide:** Read a topic, explain the answer aloud
> without looking, then run the example in a TypeScript project.
> Interviewers value the reasoning behind a choice---not only
> definitions.

## Table of Contents

1.  [Learning Roadmap](#1-learning-roadmap)
2.  [TypeScript Fundamentals](#2-typescript-fundamentals)
3.  [Types, Inference, and
    Assertions](#3-types-inference-and-assertions)
4.  [Functions and Function Types](#4-functions-and-function-types)
5.  [Objects, Interfaces, and Type
    Aliases](#5-objects-interfaces-and-type-aliases)
6.  [Unions, Intersections, and
    Narrowing](#6-unions-intersections-and-narrowing)
7.  [Arrays, Tuples, Enums, and
    Literals](#7-arrays-tuples-enums-and-literals)
8.  [Generics](#8-generics)
9.  [Classes and Object-Oriented
    TypeScript](#9-classes-and-object-oriented-typescript)
10. [Advanced Type System](#10-advanced-type-system)
11. [Modules, Compiler, and
    Configuration](#11-modules-compiler-and-configuration)
12. [Async TypeScript and Error
    Handling](#12-async-typescript-and-error-handling)
13. [TypeScript for Test Automation and
    Playwright](#13-typescript-for-test-automation-and-playwright)
14. [Scenario-Based Interview
    Questions](#14-scenario-based-interview-questions)
15. [Coding Exercises](#15-coding-exercises)
16. [Common Mistakes and Best
    Practices](#16-common-mistakes-and-best-practices)
17. [Revision Checklist](#17-revision-checklist)

------------------------------------------------------------------------

## 1. Learning Roadmap

### Suggested 4-week plan

  -----------------------------------------------------------------------
  Week                    Focus                   Practice outcome
  ----------------------- ----------------------- -----------------------
  1                       Basics, inference,      Type small utilities
                          functions, objects,     without using `any`
                          arrays                  

  2                       Unions, narrowing,      Model real API
                          interfaces, classes,    request/response data
                          generics                

  3                       Utility types,          Build reusable typed
                          conditional/mapped      helpers
                          types, `keyof`,         
                          `infer`, modules        

  4                       Async code, strict      Explain trade-offs and
                          configuration,          solve scenario
                          Playwright scenarios,   questions
                          mock interviews         
  -----------------------------------------------------------------------

### Set up a project

Install Node.js, then create a small project:

``` bash
mkdir typescript-practice
cd typescript-practice
npm init -y
npm install --save-dev typescript tsx @types/node
npx tsc --init
```

For strict checking, review `tsconfig.json` and enable settings such as:

``` json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "outDir": "dist"
  },
  "include": ["src/**/*.ts"]
}
```

Configuration depends on your Node.js version and module setup. For a
simple exercise, save code in `src/index.ts` and run:

``` bash
npx tsx src/index.ts
npx tsc --noEmit
```

`tsx` executes TypeScript conveniently; `tsc --noEmit` checks types
without generating JavaScript.

------------------------------------------------------------------------

## 2. TypeScript Fundamentals

### Q1. What is TypeScript?

**Answer:** TypeScript is a statically checked programming language
built on JavaScript. It adds a type system, interfaces, generics, and
compile-time checks. TypeScript is normally transpiled to JavaScript,
which runs in browsers or JavaScript runtimes.

``` ts
const userName: string = "Saurabh";
const retries: number = 3;
```

**Why it matters:** Types catch many mistakes during development,
improve editor autocomplete, document contracts, and make large
codebases easier to refactor.

**Important:** TypeScript types are mostly erased during compilation.
They do not automatically validate external JSON or API responses at
runtime.

### Q2. How is TypeScript different from JavaScript?

  -----------------------------------------------------------------------
  JavaScript                          TypeScript
  ----------------------------------- -----------------------------------
  Dynamic runtime language            JavaScript plus static type
                                      checking

  Type mistakes often appear at       Many type mistakes can be caught
  runtime                             before runtime

  No built-in compile-time interfaces Supports interfaces, generics,
  or generics                         unions, and more

  Runs directly in compatible         Usually compiled/transformed to
  runtimes                            JavaScript
  -----------------------------------------------------------------------

TypeScript does not replace JavaScript knowledge. You still need to
understand closures, promises, objects, prototypes, modules, and the
event loop.

### Q3. Is TypeScript statically typed or dynamically typed?

TypeScript provides **static type checking** at development/compile
time, while JavaScript values still have runtime types. Type annotations
do not normally add runtime enforcement.

``` ts
let count: number = 1;
// count = "two"; // Compile-time error
```

If untyped external data returns `"two"` at runtime, TypeScript cannot
protect you unless that data is validated.

### Q4. What are the advantages of TypeScript?

-   Finds many defects before execution.
-   Improves autocomplete and navigation.
-   Makes API contracts and domain models explicit.
-   Supports safer refactoring.
-   Helps teams maintain large automation frameworks.
-   Generics allow reusable utilities without losing type information.

Trade-off: configuration and type design add complexity, and types
cannot guarantee runtime data is valid.

### Q5. What is type inference?

Type inference means TypeScript determines a type from context or an
assigned value.

``` ts
let attempts = 3;       // inferred as number
const label = "Passed"; // inferred as the literal type "Passed"
```

Use annotations when they clarify a contract or prevent an overly
narrow/wide inference. Do not annotate every obvious local variable just
for the sake of it.

### Q6. What is the difference between `let` and `const` in type inference?

A `let` variable usually widens to a general type because it can be
reassigned; a `const` binding can preserve a literal type when
appropriate.

``` ts
let status = "passed";       // string
const fixedStatus = "passed"; // "passed"
```

`const` prevents reassignment of the binding, not mutation of an object
referenced by it.

### Q7. What is the difference between compile-time and runtime errors?

-   **Compile-time/type-checking error:** TypeScript detects a problem
    before the program runs.
-   **Runtime error:** The executing JavaScript encounters a failure,
    such as reading a property from `undefined`.

TypeScript reduces certain runtime errors but does not eliminate all
runtime failures.

### Q8. Does TypeScript guarantee that a program is correct?

No. It checks consistency with the type model. Incorrect assumptions,
unsafe assertions, `any`, race conditions, business-rule bugs, invalid
external data, and environment problems can still cause failures.

------------------------------------------------------------------------

## 3. Types, Inference, and Assertions

### Q9. Explain `any`, `unknown`, and `never`.

-   **`any`:** Turns off most type checking for that value. It is an
    escape hatch.
-   **`unknown`:** Can hold any value, but must be narrowed or checked
    before use.
-   **`never`:** Represents a value that cannot occur, such as a
    function that always throws or an impossible branch.

``` ts
function parseExternal(value: unknown): string {
  if (typeof value === "string") return value;
  throw new Error("Expected a string");
}

function fail(message: string): never {
  throw new Error(message);
}
```

Prefer `unknown` for untrusted input. Avoid spreading `any` through a
codebase.

### Q10. What is the difference between `null` and `undefined`?

Both can represent missing values, but they often express different
intent:

-   `undefined`: a value is absent or not assigned.
-   `null`: an explicit empty value.

With `strictNullChecks`, a `string` cannot be assigned `null` or
`undefined` unless the type includes them.

``` ts
let token: string | undefined;
let middleName: string | null = null;
```

### Q11. What is a type assertion?

A type assertion tells the compiler to treat an expression as a
specified type. It does **not** convert or validate the value at
runtime.

``` ts
const input = document.querySelector("#email") as HTMLInputElement;
```

This is only safe if the element really is an input. A safer version
checks:

``` ts
const element = document.querySelector("#email");
if (!(element instanceof HTMLInputElement)) {
  throw new Error("Email input was not found");
}
element.value = "qa@example.com";
```

Avoid using `as SomeType` to silence errors you have not understood.

### Q12. What is the non-null assertion operator `!`?

It tells TypeScript to treat a value as non-nullish at that point.

``` ts
const element = document.querySelector("#submit")!;
```

It does not add a runtime check. If the element is missing, the program
may still fail. Prefer explicit guards where possible.

### Q13. What are literal types?

Literal types restrict values to exact strings, numbers, booleans, or
other supported literal values.

``` ts
type TestStatus = "passed" | "failed" | "skipped";

function report(status: TestStatus): void {
  console.log(status);
}
```

This is useful for statuses, environment names, HTTP methods, and
configuration options.

### Q14. What does `as const` do?

It preserves literal types and makes object properties/array elements
readonly in the inferred type.

``` ts
const config = {
  environment: "qa",
  retries: 2
} as const;

// environment is "qa"; retries is 2
```

`as const` is a compile-time feature, not a deep runtime freeze.

### Q15. What is structural typing?

TypeScript generally checks compatibility by comparing the members/types
an object has, rather than requiring the same declared class or
interface name.

``` ts
type HasId = { id: number };
const user = { id: 7, name: "Asha" };

const entity: HasId = user; // Valid
```

This makes TypeScript flexible when working with plain objects and API
data.

------------------------------------------------------------------------

## 4. Functions and Function Types

### Q16. How do you type a function?

``` ts
function add(a: number, b: number): number {
  return a + b;
}

const multiply = (a: number, b: number): number => a * b;
```

Parameter types describe accepted inputs; the return type describes
output. TypeScript can often infer return types, but explicit return
types are helpful at public boundaries.

### Q17. What are optional and default parameters?

``` ts
function searchUser(query: string, limit?: number): void {
  console.log(query, limit);
}

function retry(testName: string, attempts = 2): void {
  console.log(testName, attempts);
}
```

An optional parameter may be omitted and is effectively possibly
`undefined`. A default parameter uses the default value when the
argument is `undefined`.

### Q18. What are rest parameters?

Rest parameters collect zero or more arguments into an array.

``` ts
function sum(...values: number[]): number {
  return values.reduce((total, value) => total + value, 0);
}
```

### Q19. What is function overloading?

Overloads expose multiple call signatures for one implementation.

``` ts
function format(value: string): string;
function format(value: number): string;
function format(value: string | number): string {
  return String(value);
}
```

Use overloads when input forms have meaningfully different call
signatures or return types. The implementation signature is not directly
visible to callers.

### Q20. What is a callback type?

A callback type defines the signature of a function passed into another
function.

``` ts
type CompletionHandler = (passed: boolean, message: string) => void;

function runCheck(handler: CompletionHandler): void {
  handler(true, "Health check passed");
}
```

### Q21. What does `void` mean for a function?

It means callers should not rely on a returned value. A function typed
`void` can still return a value internally in some assignability
contexts, but the caller treats the result as unusable.

### Q22. What is the difference between `void` and `never`?

-   `void`: the function completes, but provides no useful return value.
-   `never`: the function cannot complete normally, for example because
    it always throws or loops forever.

### Q23. How do you type `this` in a function?

TypeScript allows an explicit `this` parameter for compile-time
checking. It is not a real runtime argument.

``` ts
interface Counter {
  value: number;
  increment(this: Counter): void;
}

const counter: Counter = {
  value: 0,
  increment() {
    this.value++;
  }
};
```

Arrow functions capture lexical `this`; ordinary functions get `this`
based on how they are called.

------------------------------------------------------------------------

## 5. Objects, Interfaces, and Type Aliases

### Q24. What is an interface?

An interface describes the shape of an object or class contract.

``` ts
interface User {
  id: number;
  name: string;
  email?: string;
  readonly createdAt: Date;
}
```

`email?` is optional. `readonly` prevents reassignment through this
type, but does not deep-freeze the object at runtime.

### Q25. What is a type alias?

A type alias gives a name to a type, including primitives, unions,
tuples, function types, and object shapes.

``` ts
type UserId = string;
type Result = { ok: true; data: User } | { ok: false; error: string };
```

### Q26. Interface vs type alias---what is the difference?

Both can describe object shapes and often overlap in use.

-   Interfaces support declaration merging and are commonly used for
    extensible object/class contracts.
-   Type aliases can name unions, intersections, primitives, tuples, and
    conditional types.
-   Prefer the form that communicates the design best; consistency
    matters more than a rigid rule.

### Q27. What is `readonly`?

It prevents reassignment through a particular typed reference.

``` ts
interface TestCase {
  readonly id: string;
  title: string;
}
```

It is a compile-time restriction, not runtime immutability. Another
mutable reference may still change the same object.

### Q28. What is an index signature?

It describes objects whose property names follow a key type and whose
values share a type.

``` ts
interface HeadersMap {
  [headerName: string]: string;
}

const headers: HeadersMap = {
  Authorization: "Bearer token",
  "Content-Type": "application/json"
};
```

Use `Record<Keys, Value>` for many dictionary-like cases. Avoid broad
index signatures if they make every property too permissive.

### Q29. What is excess property checking?

When a fresh object literal is assigned to a target type, TypeScript
checks for unexpected properties in many contexts.

``` ts
interface Credentials {
  username: string;
}

// Error on a fresh literal:
// const c: Credentials = { username: "qa", password: "secret" };
```

This is not exact-object typing in every situation. Structural typing
still allows a variable with extra properties to be assigned when the
required members match.

### Q30. What are intersection types?

An intersection `A & B` combines requirements from both types.

``` ts
type Identified = { id: string };
type Timestamped = { createdAt: Date };
type AuditRecord = Identified & Timestamped;
```

A value of `AuditRecord` must satisfy both shapes. Conflicting
properties can create an impossible or awkward type.

------------------------------------------------------------------------

## 6. Unions, Intersections, and Narrowing

### Q31. What is a union type?

A union `A | B` means a value may be one of the listed alternatives. You
can only safely access members common to all alternatives until you
narrow it.

``` ts
function printId(id: string | number): void {
  console.log(id);
}
```

### Q32. What is type narrowing?

Narrowing uses runtime checks and control flow to reduce a broad type to
a more specific one.

``` ts
function normalize(value: string | number): string {
  if (typeof value === "number") return value.toFixed(2);
  return value.trim();
}
```

Common techniques include `typeof`, `instanceof`, equality checks, the
`in` operator, discriminant properties, and user-defined type
predicates.

### Q33. What is a discriminated union?

A discriminated union uses a shared property with different literal
values to identify each variant.

``` ts
type ApiResult =
  | { kind: "success"; data: string[] }
  | { kind: "failure"; message: string };

function handle(result: ApiResult): string {
  switch (result.kind) {
    case "success":
      return result.data.join(",");
    case "failure":
      return result.message;
  }
}
```

This pattern is excellent for API results, workflow states, and test
outcomes.

### Q34. What is a user-defined type guard?

A function returning `value is SomeType` tells TypeScript that a
successful result narrows the value.

``` ts
interface ApiUser {
  id: number;
  name: string;
}

function isApiUser(value: unknown): value is ApiUser {
  if (typeof value !== "object" || value === null) return false;
  if (!("id" in value) || !("name" in value)) return false;

  const candidate = value as Record<string, unknown>;
  return typeof candidate.id === "number" &&
    typeof candidate.name === "string";
}
```

A type guard must perform real checks. Merely claiming a predicate
without validating the value creates unsound assumptions.

### Q35. How do you handle exhaustive checks?

Use `never` to make the compiler flag newly added union cases that are
not handled.

``` ts
type State = "queued" | "running" | "done";

function assertNever(value: never): never {
  throw new Error(`Unexpected state: ${value}`);
}

function describe(state: State): string {
  switch (state) {
    case "queued": return "Waiting";
    case "running": return "In progress";
    case "done": return "Finished";
    default: return assertNever(state);
  }
}
```

If another state is added and not handled, the call to `assertNever`
becomes a type error.

### Q36. What is the difference between `in` and `instanceof`?

-   `in` checks whether a property exists on an object or its prototype
    chain.
-   `instanceof` checks whether an object's prototype chain includes a
    constructor's prototype.

Neither alone validates an arbitrary JSON payload fully.

------------------------------------------------------------------------

## 7. Arrays, Tuples, Enums, and Literals

### Q37. How do you type arrays?

``` ts
const scores: number[] = [90, 85, 97];
const names: Array<string> = ["Asha", "Ravi"];
```

`number[]` and `Array<number>` are equivalent for ordinary arrays.

### Q38. What is a tuple?

A tuple represents an array with a known structure and positions.

``` ts
type HttpResponseSummary = [statusCode: number, body: string];
const summary: HttpResponseSummary = [200, "OK"];
```

Tuples can model fixed-position values. Named object properties are
often clearer for complex records.

### Q39. What are enums? Should you always use them?

Enums define a set of named constants.

``` ts
enum Environment {
  QA = "qa",
  Staging = "staging",
  Production = "production"
}
```

Enums have runtime behavior in many configurations. Alternatives include
string-literal unions or `as const` objects, which can be simpler for
interoperable data:

``` ts
const environments = ["qa", "staging", "production"] as const;
type EnvironmentName = typeof environments[number];
```

Choose based on project conventions and whether a runtime enum object is
useful.

### Q40. What does `typeof` do in a type position?

`typeof` can capture the type of an existing variable or value.

``` ts
const defaults = { retries: 2, browser: "chromium" };
type Defaults = typeof defaults;
```

This avoids duplicating a type that can be derived from a value.

### Q41. What does `keyof` do?

`keyof T` creates a union of the known property keys of `T`.

``` ts
type Person = { id: number; name: string };
type PersonKey = keyof Person; // "id" | "name"
```

### Q42. What does indexed access type `T[K]` mean?

It retrieves the type of a property or group of properties from a type.

``` ts
type Person = { id: number; name: string };
type PersonName = Person["name"]; // string
type PersonValue = Person[keyof Person]; // number | string
```

------------------------------------------------------------------------

## 8. Generics

### Q43. What are generics and why use them?

Generics allow types to be parameters, preserving the relationship
between inputs and outputs while making code reusable.

``` ts
function identity<T>(value: T): T {
  return value;
}

const count = identity(5);       // number
const label = identity("ready"); // string
```

Without generics, using `any` may discard the useful connection between
input and output types.

### Q44. What is a generic constraint?

A constraint limits the types that can be used as a generic argument.

``` ts
function getLength<T extends { length: number }>(value: T): number {
  return value.length;
}

getLength("TypeScript");
getLength([1, 2, 3]);
// getLength(42); // Error
```

### Q45. What are generic interfaces and classes?

``` ts
interface Repository<T, Id> {
  findById(id: Id): Promise<T | undefined>;
  save(entity: T): Promise<void>;
}

class InMemoryRepository<T extends { id: string }>
  implements Repository<T, string> {
  private readonly records = new Map<string, T>();

  async findById(id: string): Promise<T | undefined> {
    return this.records.get(id);
  }

  async save(entity: T): Promise<void> {
    this.records.set(entity.id, entity);
  }
}
```

The type parameters let the repository work with different entity types
while preserving safety.

### Q46. What is a generic API response type?

``` ts
type ApiResponse<T> =
  | { ok: true; data: T; status: number }
  | { ok: false; error: string; status: number };

function unwrap<T>(response: ApiResponse<T>): T {
  if (!response.ok) throw new Error(response.error);
  return response.data;
}
```

This models success and failure explicitly. It still does not validate a
network response at runtime.

### Q47. What is the difference between `T extends X` and `T = X`?

-   `T extends X` constrains `T` to types assignable to `X`.
-   `T = X` supplies a default type argument when the caller does not
    specify one and inference does not choose another.

``` ts
type Box<T extends object = { id: string }> = { value: T };
```

### Q48. What is variance, at a high level?

Variance describes how subtype relationships between types affect
generic types.

-   **Covariance:** output positions generally preserve the subtype
    relationship.
-   **Contravariance:** input positions generally reverse it.
-   **Invariance:** neither direction is safely assignable.

TypeScript's behavior depends on the type construct and compiler
settings. In interviews, explain why accepting a function that handles
only a narrow input can be unsafe when a caller may pass a broader
input.

### Q49. When can generics become over-engineering?

When they make a simple function harder to read, expose implementation
details, or add type parameters that cannot be inferred. Start with the
simplest accurate type and add generics when they preserve a real
relationship or remove meaningful duplication.

------------------------------------------------------------------------

## 9. Classes and Object-Oriented TypeScript

### Q50. What access modifiers does TypeScript support?

-   `public`: accessible from anywhere; this is the default.
-   `protected`: accessible in the class and subclasses.
-   `private`: restricted by TypeScript to the declaring class.
-   `readonly`: prevents reassignment after initialization, subject to
    the language rules.

TypeScript's `private` is generally compile-time privacy. JavaScript
`#privateField` provides runtime-enforced private fields.

### Q51. What are `implements` and `extends`?

-   `extends` derives from a class or extends an interface/type in
    supported contexts.
-   `implements` checks that a class satisfies an interface or object
    contract; it does not copy implementation code into the class.

``` ts
interface Logger {
  log(message: string): void;
}

class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(message);
  }
}
```

### Q52. What is an abstract class?

An abstract class can provide shared implementation while requiring
subclasses to implement abstract members. It cannot be instantiated
directly.

``` ts
abstract class BasePage {
  abstract open(): Promise<void>;

  async waitForReady(): Promise<void> {
    // Shared readiness logic could live here.
  }
}
```

Use an interface when you need a contract without shared implementation;
use an abstract class when shared behavior or protected state is useful.

### Q53. What is method overriding?

A subclass provides its own implementation of an inherited method. With
`noImplicitOverride`, use `override` to make that intent explicit.

``` ts
class BaseReporter {
  report(): string {
    return "base";
  }
}

class HtmlReporter extends BaseReporter {
  override report(): string {
    return "<p>report</p>";
  }
}
```

### Q54. What are static members?

Static fields and methods belong to the class itself rather than each
instance.

``` ts
class IdGenerator {
  private static nextId = 1;

  static generate(): number {
    return this.nextId++;
  }
}
```

### Q55. Does TypeScript support multiple inheritance?

Classes cannot extend multiple classes. A class can implement multiple
interfaces, and object types can use intersections. Composition is often
preferable to deep inheritance.

### Q56. What is a parameter property?

A constructor parameter with an access modifier or `readonly` can
declare and initialize a property concisely.

``` ts
class UserService {
  constructor(
    private readonly baseUrl: string,
    private readonly timeoutMs = 5000
  ) {}
}
```

### Q57. How would you use dependency injection in a test framework?

Pass dependencies into a class rather than constructing every dependency
internally.

``` ts
interface HttpClient {
  get<T>(url: string): Promise<T>;
}

class UserApi {
  constructor(private readonly client: HttpClient) {}

  getUser(id: string): Promise<unknown> {
    return this.client.get(`/users/${id}`);
  }
}
```

This improves testability because a test can supply a fake client. Keep
the real runtime validation of the response separate from its
compile-time type.

------------------------------------------------------------------------

## 10. Advanced Type System

### Q58. What are utility types?

TypeScript provides built-in helpers to transform types.

``` ts
interface User {
  id: string;
  name: string;
  email: string;
  active: boolean;
}

type NewUser = Omit<User, "id">;
type UserPreview = Pick<User, "id" | "name">;
type PartialUser = Partial<User>;
type CompleteUser = Required<User>;
type ReadonlyUser = Readonly<User>;
type UserById = Record<string, User>;
```

Common utilities:

-   `Partial<T>`: makes properties optional.
-   `Required<T>`: makes properties required.
-   `Readonly<T>`: makes properties readonly.
-   `Pick<T, K>`: selects properties.
-   `Omit<T, K>`: removes properties.
-   `Record<K, V>`: maps keys to values.
-   `Exclude<T, U>`: removes union members assignable to `U`.
-   `Extract<T, U>`: keeps union members assignable to `U`.
-   `NonNullable<T>`: removes `null` and `undefined`.
-   `ReturnType<F>`: obtains a function's return type.
-   `Parameters<F>`: obtains a function's parameter tuple.
-   `Awaited<T>`: recursively models the resolved type of a promise-like
    value.

### Q59. What are mapped types?

Mapped types iterate over keys in a type to construct another type.

``` ts
type Optional<T> = {
  [K in keyof T]?: T[K];
};

type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};
```

Mapped types are useful for building reusable type transformations.

### Q60. What are conditional types?

Conditional types choose a type based on an assignability check.

``` ts
type IsString<T> = T extends string ? true : false;

type A = IsString<"hello">; // true
type B = IsString<number>;  // false
```

### Q61. What is a distributive conditional type?

A conditional type distributes over union members when the checked type
is a naked type parameter.

``` ts
type ToArray<T> = T extends unknown ? T[] : never;
type Result = ToArray<string | number>; // string[] | number[]
```

To prevent distribution, wrap the checked type in a tuple:
`[T] extends [unknown] ? ... : ...`.

### Q62. What is the `infer` keyword?

`infer` introduces a type variable inside a conditional type so
TypeScript can infer part of a type.

``` ts
type PromiseValue<T> = T extends Promise<infer U> ? U : T;

type A = PromiseValue<Promise<number>>; // number
type B = PromiseValue<string>;          // string
```

### Q63. What are template literal types?

They build string literal types from other types.

``` ts
type Method = "get" | "post";
type Endpoint = "/users" | "/payments";
type Route = `${Uppercase<Method>} ${Endpoint}`;
// "GET /users" | "GET /payments" | "POST /users" | "POST /payments"
```

### Q64. What is the `satisfies` operator?

It checks that an expression satisfies a target type while generally
preserving the expression's more specific inferred type.

``` ts
type Browser = "chromium" | "firefox" | "webkit";

const projects = {
  smoke: "chromium",
  crossBrowser: "webkit"
} satisfies Record<string, Browser>;
```

Use it when you want validation against a contract without unnecessarily
widening the expression's type.

### Q65. What is declaration merging?

Some declarations, notably interfaces and namespaces, can merge when
they share a name in the same declaration scope. Type aliases and
classes do not generally merge in the same way. Merging can support
library extension but should be used deliberately because it can obscure
where a type comes from.

### Q66. What are ambient declarations and `.d.ts` files?

Declaration files describe types for JavaScript code or libraries
without providing implementation. `declare` tells TypeScript that a
value exists elsewhere.

``` ts
declare const BUILD_VERSION: string;
```

A declaration is a promise to the compiler, not proof that the runtime
value exists.

### Q67. What is type erasure?

Most TypeScript-only constructs---interfaces, type aliases, and
annotations---are removed during compilation. Runtime constructs such as
ordinary classes and JavaScript expressions remain, depending on the
emitted target and compiler transformations.

### Q68. How do you model mutually exclusive object properties?

Use a discriminated union or `never` properties to prevent incompatible
combinations.

``` ts
type Auth =
  | { kind: "basic"; username: string; password: string; token?: never }
  | { kind: "bearer"; token: string; username?: never; password?: never };
```

This improves compile-time validation, but external input still needs
runtime checks.

### Q69. How would you type a function that accepts only keys of an object?

``` ts
function getProperty<T, K extends keyof T>(object: T, key: K): T[K] {
  return object[key];
}

const user = { id: 1, name: "Asha" };
const name = getProperty(user, "name"); // string
```

This is a classic generic relationship: the allowed key depends on the
supplied object, and the return type depends on the selected key.

### Q70. Why can overly complex types be a problem?

They increase compile time, make error messages difficult, and raise
maintenance costs. Prefer readable types, explicit intermediate aliases,
and runtime schemas at trust boundaries over clever type-level
programming.

------------------------------------------------------------------------

## 11. Modules, Compiler, and Configuration

### Q71. What is the difference between `import` and `export`?

`export` exposes declarations from a module; `import` brings exported
declarations into another module.

``` ts
// user.ts
export interface User {
  id: string;
}

export function getDisplayName(user: User): string {
  return user.id;
}

// app.ts
import { getDisplayName, type User } from "./user.js";
```

The `.js` specifier shown is common in NodeNext ESM projects; actual
module specifiers depend on the build/runtime setup.

### Q72. What does `tsconfig.json` do?

It defines compiler options and which files belong to the project. It
controls strictness, module behavior, target JavaScript, output, path
settings, and more.

### Q73. Explain `strict`.

`strict: true` enables a family of stricter checks, including
`noImplicitAny` and `strictNullChecks`. Other related options may be
enabled or configured separately. Strict mode helps surface problems
early and is a strong default for new projects.

### Q74. What is `noImplicitAny`?

It reports cases where TypeScript would otherwise infer `any`
implicitly. Explicit `any` can still be written, so lint rules and code
review may be needed to prevent unsafe use.

### Q75. What is `strictNullChecks`?

It treats `null` and `undefined` as distinct types, so code must account
for them instead of silently treating them as every type.

### Q76. What is the difference between `tsc` and a runtime such as `tsx`?

-   `tsc` is the TypeScript compiler/type checker; it can emit
    JavaScript or run checks with `--noEmit`.
-   `tsx` is a tool for running TypeScript files conveniently in
    supported Node.js workflows.
-   Running/transpiling code does not always mean full type checking
    occurred. Run `tsc --noEmit` in CI when you need a type-checking
    step.

### Q77. What are path aliases?

Path aliases give shorter import names through compiler configuration.
They may require matching support from the bundler, test runner, or
runtime; `paths` in `tsconfig` does not automatically rewrite every
runtime import.

------------------------------------------------------------------------

## 12. Async TypeScript and Error Handling

### Q78. How do you type a Promise?

``` ts
function fetchCount(): Promise<number> {
  return Promise.resolve(10);
}
```

`Promise<T>` represents a promise that resolves to a value of type `T`.
It may still reject at runtime.

### Q79. How do you type an async function?

An `async` function returns a Promise. The annotation describes the
resolved value inside `Promise<T>`.

``` ts
async function getUserName(): Promise<string> {
  return "Asha";
}
```

### Q80. How should you type `catch` errors?

With strict settings, catch variables are commonly treated as `unknown`.
Narrow them before accessing properties.

``` ts
try {
  await doWork();
} catch (error: unknown) {
  if (error instanceof Error) {
    console.error(error.message);
  } else {
    console.error("Unknown failure", error);
  }
}
```

Do not assume every thrown value is an `Error`.

### Q81. What is the difference between `Promise.all` and `Promise.allSettled`?

-   `Promise.all` resolves when all promises resolve and rejects as soon
    as one rejects.
-   `Promise.allSettled` waits for every promise and returns each
    outcome.

Use `Promise.all` when every result is required; use `allSettled` when
you need a complete picture of independent operations.

### Q82. How do you prevent race conditions in asynchronous tests?

Use isolated test data, await all asynchronous work, avoid shared
mutable state, and use unique IDs or per-test fixtures. If operations
must happen in order, sequence them explicitly. TypeScript can describe
the contracts but cannot itself prevent concurrency bugs.

------------------------------------------------------------------------

## 13. TypeScript for Test Automation and Playwright

### Q83. Why is TypeScript useful for Playwright automation?

It provides typed locators, fixtures, page objects, test data,
configuration, and API response models. It improves refactoring and
catches many incorrect method calls before tests run.

### Q84. How would you type a Page Object?

``` ts
import { Page, Locator } from "@playwright/test";

export class LoginPage {
  private readonly username: Locator;
  private readonly password: Locator;
  private readonly submitButton: Locator;

  constructor(private readonly page: Page) {
    this.username = page.getByLabel("Username");
    this.password = page.getByLabel("Password");
    this.submitButton = page.getByRole("button", { name: "Sign in" });
  }

  async login(username: string, password: string): Promise<void> {
    await this.username.fill(username);
    await this.password.fill(password);
    await this.submitButton.click();
  }
}
```

Keep locators and page-specific actions inside the page object.
Assertions about user-visible outcomes often belong in tests, unless the
action is specifically a reusable business assertion.

### Q85. How would you type test data?

``` ts
type LoginCase = {
  title: string;
  username: string;
  password: string;
  expectedMessage: string;
};

const cases: LoginCase[] = [
  {
    title: "valid user",
    username: "qa_user",
    password: "example-only",
    expectedMessage: "Welcome"
  }
];
```

Avoid real secrets in source control. Use fixtures or secret stores for
credentials.

### Q86. How do you type API response data safely?

Define a type for the expected structure, but remember that a type
annotation does not validate the response body.

``` ts
type UserDto = {
  id: number;
  name: string;
};

function isUserDto(value: unknown): value is UserDto {
  if (typeof value !== "object" || value === null) return false;
  const candidate = value as Record<string, unknown>;
  return typeof candidate.id === "number" &&
    typeof candidate.name === "string";
}
```

For large schemas, consider runtime validation with a schema library
such as Zod or an API schema/JSON Schema validator. Validate at the
boundary, then pass the validated type through the application.

### Q87. How would you design reusable typed API helpers?

``` ts
type ApiResult<T> =
  | { ok: true; status: number; data: T }
  | { ok: false; status: number; error: string };

async function expectSuccess<T>(result: ApiResult<T>): Promise<T> {
  if (!result.ok) {
    throw new Error(`API failed (${result.status}): ${result.error}`);
  }
  return result.data;
}
```

A real API client should also handle timeouts, authentication, retries
where appropriate, observability, and runtime validation. Avoid retries
for non-idempotent operations unless the API contract supports safe
retry behavior.

### Q88. How do you avoid `any` in a test framework?

-   Use `unknown` for external or unvalidated data.
-   Define DTOs and discriminated unions.
-   Use generics for reusable clients and fixtures.
-   Type fixture parameters and return values.
-   Narrow caught errors.
-   Add `noImplicitAny`, lint rules, and CI type checking.

### Q89. What is a good way to type environment configuration?

Parse environment variables at startup because environment values are
strings and may be missing.

``` ts
type Config = {
  baseUrl: string;
  timeoutMs: number;
};

function loadConfig(env: NodeJS.ProcessEnv): Config {
  const baseUrl = env.BASE_URL;
  const timeoutRaw = env.TIMEOUT_MS ?? "5000";
  const timeoutMs = Number(timeoutRaw);

  if (!baseUrl) throw new Error("BASE_URL is required");
  if (!Number.isFinite(timeoutMs) || timeoutMs <= 0) {
    throw new Error("TIMEOUT_MS must be a positive number");
  }

  return { baseUrl, timeoutMs };
}
```

Keep environment-specific secrets outside the repository and fail fast
for invalid configuration.

------------------------------------------------------------------------

## 14. Scenario-Based Interview Questions

### Q90. Your API returns JSON, but TypeScript says the response is `User`. Is it safe?

**Answer:** Not just because of the annotation. TypeScript does not
validate network payloads. Treat the response as `unknown`, validate its
shape with a runtime schema or type guard, and only then use it as
`User`. Also test missing fields, wrong types, `null`, unexpected enum
values, and malformed JSON.

### Q91. A developer used `any` to fix multiple compiler errors. What would you do?

**Answer:** First understand the errors and intended contract. Replace
`any` with a precise type or `unknown` plus narrowing. If a third-party
boundary requires an escape hatch, isolate it in one adapter, document
the reason, and validate the data before it enters core code.

### Q92. How would you type a payment workflow with multiple states?

**Answer:** Use a discriminated union so each state carries only valid
fields.

``` ts
type Payment =
  | { status: "created"; paymentId: string }
  | { status: "processing"; paymentId: string; startedAt: Date }
  | { status: "success"; paymentId: string; reference: string }
  | { status: "failed"; paymentId: string; reason: string };
```

Then switch on `status` and add an exhaustive `never` check. This
prevents code from accidentally accessing `reference` on a failed
payment.

### Q93. A test fixture can return either a browser page or an API client. How should you model it?

**Answer:** Prefer separate fixtures or a discriminated union if the
alternatives truly form one result. Avoid a loosely typed object with
many optional properties because callers can accidentally use a missing
property.

### Q94. Your generic helper has five type parameters and is hard to understand. What would you do?

**Answer:** Simplify the API. Infer types where possible, introduce
named type aliases for meaningful concepts, split responsibilities, and
keep generic parameters only where they preserve an important
relationship. Type-level sophistication is not a goal by itself.

### Q95. Tests pass locally but fail in CI due to missing environment variables. Can TypeScript solve it?

**Answer:** Not by itself. Define a typed configuration loader that
validates required variables at startup, report clear errors, and make
CI supply the correct variables. Types document the validated
configuration after parsing; they do not ensure the deployment
environment contains it.

### Q96. A type says `user.email` is a string, but the API sometimes sends `null`. How do you fix it?

**Answer:** First confirm the API contract. If `null` is valid, model it
explicitly as `string | null`; if not, treat it as a contract defect and
validate/reject it. Do not hide the mismatch with an assertion.

### Q97. How would you safely handle a dynamic object key?

**Answer:** Use `keyof` and a generic helper when keys come from a known
object. If keys come from external input, validate that the key is
allowed before indexing. Consider `noUncheckedIndexedAccess` so
uncertain lookups reflect possible `undefined`.

### Q98. How would you migrate a JavaScript automation framework to TypeScript?

**Answer:** Migrate incrementally: add `tsconfig`, enable `allowJs` if
useful, type shared utilities and configuration first, introduce strict
checks in manageable stages, add types to page objects and fixtures,
validate external data, then tighten linting and CI. Avoid converting
everything in one risky rewrite.

### Q99. How do you distinguish compile-time types from runtime validation?

**Answer:** Types guide the compiler and developers; runtime validators
inspect actual values. Use types for internal contracts and runtime
validation at trust boundaries such as HTTP responses, environment
variables, files, queues, and user input.

### Q100. How would you make a TypeScript automation framework maintainable for a team?

**Answer:** Use strict compiler settings, clear module boundaries, typed
fixtures and domain models, minimal unsafe assertions, runtime
validation for external data, consistent naming, code review, linting,
and CI type checks. Prefer composition and focused abstractions over a
huge base class. Keep tests independent and isolate data because types
do not solve concurrency or test-design problems.

------------------------------------------------------------------------

## 15. Coding Exercises

Try solving these before checking the sample approaches.

### Exercise 1: Safely format an identifier

**Task:** Accept a `string | number`; trim strings and return all
identifiers as strings.

``` ts
function formatId(id: string | number): string {
  if (typeof id === "number") return String(id);
  return id.trim();
}
```

### Exercise 2: Create a typed result

**Task:** Model success and failure without unrelated optional
properties.

``` ts
type Result<T> =
  | { ok: true; value: T }
  | { ok: false; error: string };

function mapResult<T, U>(
  result: Result<T>,
  transform: (value: T) => U
): Result<U> {
  if (!result.ok) return result;
  return { ok: true, value: transform(result.value) };
}
```

### Exercise 3: Pick selected properties

**Task:** Write a generic function that returns a property by key. Use
the answer from Q69 as a starting point. Then test valid and invalid
keys.

### Exercise 4: Validate an API response

**Task:** Write a guard for:

``` ts
type PaymentDto = {
  id: string;
  amount: number;
  status: "pending" | "success" | "failed";
};
```

Check the input is a non-null object, confirm each field has the
expected runtime type, and confirm `status` is one of the allowed
literals. Test malformed and valid values.

### Exercise 5: Build a retry configuration type

**Task:** Model `attempts`, `delayMs`, and an optional list of retryable
HTTP statuses. Decide which fields must be positive integers and enforce
those rules at runtime as well as in types where possible.

### Exercise 6: Make a function exhaustive

**Task:** Create a union for `queued`, `running`, `passed`, and `failed`
test states. Write a formatter using `switch` and `never`. Add another
state and confirm the compiler identifies the missing branch.

### Exercise 7: Type a reusable API client

**Task:** Create an `HttpClient` interface with typed `get<T>()` and
`post<TBody, TResponse>()` methods. Explain why generic return types do
not validate the server's actual response.

### Exercise 8: Type a Playwright page object

**Task:** Build a `CheckoutPage` with typed `Page` and `Locator` fields,
methods to fill shipping details, submit the order, and return a
meaningful result. Keep assertions in the test unless there is a clear
reason to encapsulate a domain assertion.

------------------------------------------------------------------------

## 16. Common Mistakes and Best Practices

1.  **Using `any` everywhere:** it defeats much of the type system.
    Prefer a real type or `unknown` with narrowing.
2.  **Assuming an assertion validates data:** `value as User` changes
    the compiler's belief, not the runtime value.
3.  **Forgetting nullability:** enable `strictNullChecks` and handle
    absent values intentionally.
4.  **Overusing enums:** consider literal unions or `as const` objects
    when they fit project conventions better.
5.  **Creating giant base classes:** prefer composition, interfaces, and
    focused services.
6.  **Using optional properties for mutually exclusive states:** use
    discriminated unions.
7.  **Ignoring runtime input:** validate JSON, environment variables,
    files, and message payloads at boundaries.
8.  **Confusing transpilation with type checking:** run `tsc --noEmit`
    or an equivalent checker in CI.
9.  **Overengineering types:** readability and maintainability matter
    more than cleverness.
10. **Ignoring module/runtime configuration:** align `module`,
    `moduleResolution`, package type, and execution tool.
11. **Sharing mutable test data:** types cannot prevent race conditions;
    isolate fixtures and records.
12. **Treating `readonly` as deep immutability:** it is a type-level
    restriction on a reference, not a runtime freeze.

### Practical project quality checklist

-   [ ] `strict: true` or a documented plan to enable it.
-   [ ] `noImplicitAny` and `strictNullChecks` active.
-   [ ] External inputs typed as `unknown` until validated.
-   [ ] No unnecessary `any` or non-null assertions.
-   [ ] Public functions have clear contracts.
-   [ ] Domain states use discriminated unions where appropriate.
-   [ ] CI runs tests and a TypeScript type check.
-   [ ] Environment configuration is parsed and validated.
-   [ ] Page objects and API clients use explicit interfaces.
-   [ ] Test data is isolated and secrets are not committed.

------------------------------------------------------------------------

## 17. Revision Checklist

Before an interview, make sure you can explain and demonstrate:

-   [ ] TypeScript vs JavaScript, compile-time vs runtime.
-   [ ] Inference, `any`, `unknown`, `never`, `null`, `undefined`.
-   [ ] Type annotations, assertions, `as const`, `satisfies`.
-   [ ] Functions, optional parameters, overloads, callbacks.
-   [ ] Interfaces vs type aliases, structural typing, `readonly`.
-   [ ] Unions, intersections, narrowing, guards, exhaustive checks.
-   [ ] Arrays, tuples, enums, `keyof`, `typeof`, indexed access.
-   [ ] Generics, constraints, defaults, and basic variance.
-   [ ] Classes, access modifiers, abstract classes, `implements`,
    `override`.
-   [ ] Utility types, mapped types, conditional types, `infer`,
    template literals.
-   [ ] Modules, `tsconfig`, strict mode, `tsc --noEmit`.
-   [ ] Promises, `async/await`, error narrowing, concurrency.
-   [ ] Runtime validation of API responses and environment variables.
-   [ ] Typed Playwright fixtures, page objects, and test data.
-   [ ] Scenario answers involving migration, maintainability, and CI.

## Recommended practice routine

For each interview question:

1.  Give a 30--60 second answer in your own words.
2.  Write a small example without copying.
3.  Explain one real-world use case in API or UI automation.
4.  Mention a trade-off or limitation.
5.  Run `npx tsc --noEmit` and fix the errors rather than suppressing
    them.

**Mastery target:** Be able to explain not just what a TypeScript
feature does, but why you would use it, what it cannot guarantee, and
how it improves the safety and maintainability of a real project.
