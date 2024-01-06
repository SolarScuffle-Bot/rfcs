# Typepack Builder

## Summary

We propose a means of extracting type parameters from type aliases to build typepacks of forms similar to `<T...>(...F<T>) -> G<T...>`

## Motivation

Starting with type-safety and comprehensive type representation, we present a minified version of two libraries that requires these capabilities:

```lua
type SerDes<T> = {
  ser: (T) -> buffer,
  des: (buffer) -> T
}

Squash.number: SerDes<number> = { ... }
Squash.string: SerDes<string> = { ... }
Squash.boolean: SerDes<boolean> = { ... }
```

This is enough to implement standard types for both Luau and Roblox, but the type system is incapable of representing functions of this nature:

```lua
type SerDesVar<T...> = {
	ser: (T...) -> buffer,
	des: (buffer) -> T,
}

-- Notice the `any`
function Squash.variadic<T...>(...SerDes<any>): SerDesVar<T...>
	...
end
```

There are no mechanisms in Luau to infer this typepack. To get type-safety the user will have to explicitly type the output at each and every call site, allowing for mismatches and filling a codebase with more non-code bloat.


Or from the minified API of a different library in a similar situation, though with much more nuance that will be covered later:

```lua
type Factory<C> = {
	get: (unknown) -> C?,
	add: (unknown) -> C,
	remove: (unknown) -> (),
}

-- Notice the `any`s
function World.query({Factory<any>}): { [Factory<any>]: any? }
	...
end
```

There are again no mechanisms in Luau to infer this typepack, and a user can't even explicitly type the typepack because it can't be represented in the return type. There just isn't a syntax for it!

## Design

We propose taking advantage of the current redundancy in typepack syntax, always needing to include a trailing `...`, and using it towards a distinction in this syntax. When a typepack `<T...>` is present, each instance of the type `T` alone contributes towards "building" up the typepack, one type at a time.

```lua
-- It is now magically inferrable :party:
function Squash.variadic<T...>(...: SerDes<T>): SerDesVar<T...>
	...
end
```

It's an arguably elegant solution. For the other api:

```lua
-- Getting fancy now!
function World.query<C...>({Factory<C>}): { [Factory<C>]: C? }
	...
end
```

Generalizing this to cover the quirks and requirements of each syntax, we start with what can exist now and move on to what needs prerequisite RFCs to implement:

1. No Prerequisite RFCs:
	```lua
	-- Builds a typepack with 4 elements in sequence.
	type A = <T...>(T, F<T>, G<T>, T) -> T...

	-- Builds two typepacks with 2 elements interlaced. Vanilla Luau
	type B = <T..., U...>(T, F<U>, G<T>, U) -> V<T..., U...>
	```

2. The following all require property-definition-order to be maintained in table types, from first -> last defined.
	```lua
	-- Builds a typepack from elements of a table's values
	type C = <T...>({ x: T, y: T, z: T, [number]: T }) -> T...
	```

3. Requires tables to allow multiple indexers such as `{ [number]: any, [string]: any, [boolean]: any }`:
	```lua
	-- Builds a typepack from elements of a table's keys
	type D = <T...>({ [T]: any }) -> T...
	```

4. Requires array literal types such as `{number, number, string, boolean}`:
	```lua
	-- Builds a typepack from elements in an array
	type E = <T...>({ T }) -> T...
	```

## Drawbacks

Why should we *not* do this?

## Alternatives

What other designs have been considered? What is the impact of not doing this?
