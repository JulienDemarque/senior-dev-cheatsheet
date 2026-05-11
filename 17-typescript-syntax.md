# TypeScript syntax (cheatsheet)

Type-level and value-level patterns for application and library code. Assumes `strict` compiler options unless noted.

## Primitive and object types

```ts
let n: number = 1;
let s: string = "a";
let b: boolean = true;
let u: undefined = undefined;
let nl: null = null;
const sym = Symbol("k");
```

`undefined` and `null` are distinct; enable `strictNullChecks` so you must handle both.

## interface vs type

```ts
interface User {
  id: string;
  name: string;
}

type Point = { x: number; y: number };

// type can do unions — interfaces cannot
type Id = string | number;
```

`interface` merges across declarations (declaration merging); `type` does not. Prefer `interface` for object shapes you extend; `type` for unions, mapped types, and tuples.

## Union and intersection

```ts
type AorB = A | B;
type AandB = A & B;

type Result<T> = { ok: true; value: T } | { ok: false; error: string };
```

## Narrowing

```ts
function printId(id: number | string) {
  if (typeof id === "string") {
    console.log(id.toUpperCase());
  } else {
    console.log(id.toFixed(2));
  }
}

declare const r: { tag: "n"; v: number } | { tag: "s"; v: string };
switch (r.tag) {
  case "n":
    return r.v * 2;
  case "s":
    return r.v.length;
}
```

Discriminated unions (`tag` / `kind`) are the idiomatic alternative to class hierarchies for many APIs.

## Generics

```ts
function first<T>(xs: readonly T[]): T | undefined {
  return xs[0];
}

interface Box<T> {
  value: T;
}
```

## keyof and typeof

```ts
const config = { host: "localhost", port: 5432 } as const;
type Keys = keyof typeof config; // "host" | "port"

function get<K extends keyof typeof config>(k: K): (typeof config)[K] {
  return config[k];
}
```

## const assertion

```ts
const modes = ["read", "write"] as const;
type Mode = (typeof modes)[number]; // "read" | "write"
```

Freezes literal types for tuples and object literals—great for config and route tables.

## satisfies

```ts
const routes = {
  home: "/",
  user: (id: string) => `/users/${id}`,
} satisfies Record<string, string | ((id: string) => string)>;
```

Checks shape without widening inferred literals to `string`.

## readonly

```ts
type ROUser = Readonly<{ id: string; tags: readonly string[] }>;

function sum(xs: readonly number[]): number {
  return xs.reduce((a, b) => a + b, 0);
}
```

## Utility types

```ts
type T0 = Pick<User, "id" | "name">;
type T1 = Omit<User, "password">;
type T2 = Partial<User>;
type T3 = Required<Pick<User, "id">>;
type T4 = Readonly<User>;
type T5 = Record<string, number>;
```

## Template literal types

```ts
type Lang = "en" | "fr";
type Route = `/${Lang}/home`; // "/en/home" | "/fr/home"
```

## unknown vs any

```ts
function safeParse(json: string): unknown {
  return JSON.parse(json);
}

function use(x: unknown) {
  if (typeof x === "object" && x !== null && "id" in x) {
    // narrow before use
  }
}
```

Prefer `unknown` at boundaries; reserve `any` for incremental migration only.

## Optional chaining and nullish coalescing

```ts
const len = user?.profile?.bio?.length;
const port = options.port ?? 8080; // only null/undefined, not 0 or ""
```

## Type guards

```ts
function isString(x: unknown): x is string {
  return typeof x === "string";
}
```

## enum caution

`enum` emits runtime JS and has reverse mapping for numeric enums—many codebases prefer string unions + `as const` instead.

## satisfies vs as

- `satisfies` — value must match type; preserves narrow literal inference.
- `as` — assertion: tells the compiler to trust you; can hide bugs.

```ts
const bad = { a: 1 } as { a: string }; // compiles, wrong at runtime
```

## declare and ambient modules

```ts
declare global {
  interface Window {
    MY_FLAG?: boolean;
  }
}
```

For `.d.ts` shims when libraries lack types.

### See also

[Python syntax](./16-python-syntax.md#python-syntax-cheatsheet) for the parallel cheatsheet in another language.
