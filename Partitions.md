# Partitions

- Tracking Issue: https://github.com/kdr-aus/ogma/issues/88

> _partition_:
>
> one of the parts or sections of a whole.

---

## Goals

Partitions are designed primarily as an _encapsulation_ boundary.
The _path name_ actually does not contribute much to the requirement of partitions,
for a command or def found under a particular _path_, the path could be emulated in the name (a
quirk of allowing loose syntax in identifiers).
Partitions deal primarily with **privacy**, with encapsulation and segregation a by-product.

The specific features of partitions are:

- [x] Ability to mark items as **private**,
- [ ] Provide syntax for accessing partition items,
- [x] Nesting partitions,
- [ ] Privacy scopes can view all parent partitions, but not children,
- [x] Import a set of partitions,
- [x] Ability to document on partitions, including root

---

## Syntax

### Accessing

https://github.com/kdr-aus/ogma/issues/92

- [ ] `path/to/item`
- [ ] `path.to.item`
- [ ] `path,to,item`
- [ ] `path::to::item`
- [ ] `path:to:item`
- [ ] `path>to>item`
- [ ] `path;to;item`
- [ ] `path~to~item`
- [ ] `path^to^item`
- [ ] `path'to'item`

### Defining

- Defined in _files_.
- Define _across files_, but also within a single file.
- How to handle name clashes???


### Importing

- Aliasing path to name??

- [x] `use <path>`
- [x] `with <path>`
- [x] `import <path>`

> [!info]
> A `import` directive was implemented.


---

## Characteristics

### Privacy Boundaries

Privacy boundaries are the main feature of partitions. They allow items defined within a partition
to scoped to within itself, or be exposed for external use. The scoping should also be
_hierarchical_. Items defined in child partitions would have complete access to items defined in
parent partitions.
Items are **public by default**, and only private if marked so.

---

## Usage

### Running a file

- When running a file, the file is **considered the root**, so imports and defs are local to the file.
- The filesystem is mapped from the file (`PathBuf->Vec<File>`)
- The fs map is turned into a partitions graph
	- This first constructs the graph using `PathBuf` as the boundary and inserts the defs,
	- It then resolves the imports/exports _from the root file_
	- This requires that the `File` is mapped somewhere, since it contains the directives
		- might not have to be the `File`, the directives definitely


### Shell

---

## Definition Usage

### API Mapping

> - Existing Usage Counter: `call LanguageClient#textDocument_references()` over item
>     - Only done in `ogma` crate, other crates can be worked in
> - New API is fully under `Definitions`
>     - Could use a wrapper type to split impls and types, but they will effectively just wrap `Definitions`

| Old | Usage | New | Note |
| --- | ----- | --- | ---- |
| `Definitions::new` | 244 | `new` |
| `Definitions::add_from_file` | 0 | _ignore_ |
| `Definitions::add_from_str` | 1 |  |
| `Definitions::clear` | 1 | _ignore_ |
| `Definitions::impls` | 8 | _ignore_ |
| `Definitions::types` | 17 | _ignore_ |
| `Implementations::contains_op` | 5 | `DefItems::contains` |
| `Implementations::get_help` | 0 | `DefItems::help` |
| `Implementations::get_help_all` | 1 |  |
| `Implementations::get_help_with_err` | 2 | `DefItems::help` |
| `Implementations::get_impl` | 1 | `DefItems::get` |
| `Implementations::get_impl_with_err` | 1 | `DefItems::get` |
| `Implementations::insert_intrinsic` | 1 |  | used in a macro |
| `Implementations::insert_impl` | 2 |  |
| `Implementations::iter` | 1 | `DefItems::iter` |
| `Implementations::iter_op` | 5 |  |
| `Implementations::clear` | 1 |  |
| `Types::init_std` | 1 |  |
| `Types::get_using_tag` | 7 | `DefItems::get` |
| `Types::get_using_str` | 3 | `DefItems::get` |
| `Types::contains_type` | 1 | `DefItems::contains` |
| `Types::insert` | 1 |  |
| `Types::clear` | 1 |  |
| `Types::help_iter` | 0 |  |
| `Types::iter` | 1 | `DefItems::iter` |

> [!note]
> There are a couple of common patterns.
> #### Gets
> - The name could be with `&str` or `&Tag`
> - A `&Tag` results with an informative error message
> Much of this can be reduced into single `get_{op,ty}` with some way to convey a return type of `Option` or `Result` (see [[#Polymorphic Return Type]]).
> #### Contains
> - Both impls and types utilise a `contains` method
> #### Iteration
> - Both impls and types have iterators
> - Impls have a specific iterator over the impls of an op str
> #### Help Fetching
> - Similar with gets in the return type
> #### Insertion
> - Both types and impls support insertions
> - This would need **mutable** access
> - Might be better on root

#### To Wrap or Not To Wrap

- Split `impl Definition` where each function would be suffixed with a `_{op, ty}`
    - ➕ Concrete implementations
    - ➕ Requires less indirection
    - ➕ Flexible on return types
    - ➖ Inconsistent API (see list above)
    - ➖ Messy code with lots of functions
- Use a trait
    - ➕ Consistent API
    - ➕ Changes to trait enforces changes
    - ➖ Trait import required (prelude?)
    - ➖ Inflexible to slight differences
	    - This can be beneficial for a consistent API

#### Polymorphic Return Type

This is specific to a _fetching_ pattern, where the _key_ could be a `&str` or `&Tag`, and the return type is dependent on the argument's type, with an option being returned for a `&str` and a result for `&Tag`. There are a couple of observations:
- The goal is to avoid the creation of `Error` if the error is not being used.
- `Error`s are **contextual on the function**, not the _argument_
- This might be transitive to  the partitions tree
- `&str` an `&Tag` are reduced to a common key type
	- probably `&str`

This can be achieved like so:

```rust
/// The `R` specifies the wrapped success value.
trait PolyGet<R>: ?Sized {
    // The output type (ie Option<R> or Result<R, Error>).
    type Output;
    // There needs to be a common key which is used.
	fn key(&self) -> &str;
    // Wrap a successful get.
	fn success(r: R) -> Self::Output;
	// On unsuccessful get, if a `&Tag` can be provided, an `Error`
	// can be built. Since the `Error` is contextual from the function,
	// a closure is supplied as the builder.
	// The implementor decides whether to invoke the function or not.
	fn fail<E>(e: E) -> Self::Output
	where
		E: FnOnce(&Tag) -> Error;
}

impl Definitions {
	fn get_op<K>(&self, key: &K) -> K::Output 
	where
	    K: PolyGet<&Impl>
	{ .. }

	fn get_ty<K>(&self, key: &K) -> K::Output 
	where
	    K: PolyGet<&Type>
	{ .. }
}
```

>[!note]
> We could use GATs, but it is not in stable yet.

#### Wrap Trait

```rust
trait DefItems {
    type Item;
	
	fn contains // .. etc
    
    fn get<K>(&self, key: &K) -> K::Output
    where
        K: PolyGet<&Self::Item>;

	fn help<K> // .. etc

    fn iter // ..
}
```

- The impls wrapper might need to use a key: `(&Tag, &Type)`
- The iterator return type would need to be associated.