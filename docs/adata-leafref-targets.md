# Adata Leafref Targets

This document describes the current prototype for exposing YANG
`leafref` values in generated adata as reference objects with a resolved
`target`.

## Goal

The old adata model flattened a `leafref` to its raw scalar value during
gdata to adata conversion. For transforms that meant:

- only the key value was preserved
- following the reference required manual lookup code
- two references to the same target could materialize separate adata
  objects instead of sharing one object instance

The current model keeps the raw scalar value and also resolves the
reference to a typed adata target object.

## Supported subset

This is intentionally not a general runtime XPath evaluator. Code
generation only enables the ref-target behavior for a narrow subset of
leafrefs:

- the leafref path must resolve statically at code generation time
- the path must contain only simple location steps
- predicates are not supported
- the target must be the single key leaf of a list
- only containers may appear between the root and that target list

If a leafref falls outside this subset, adata continues to treat it like
an ordinary scalar leaf.

## Generated shape

For a supported leafref field, code generation emits a specialized ref
class for that exact schema location. For example, a leafref from
`/transforms/consumer/input` to `/inventory/producer/name` generates a
class like:

```acton
class refs__transforms__consumer__input_ref(yang.adata.Ref):
    target: ?refs__inventory__producer_entry

    def __init__(self, value: str, target: ?refs__inventory__producer_entry=None):
        self.value = value
        self.target = target
```

The public API of the generated field is:

- `value`: the raw leafref value, for example `"prod-a"`
- `target`: the resolved owning adata object, for example a
  `producer_entry`

The shared runtime base `yang.adata.Ref` only stores `value`. The typed
`target` attribute lives on each generated ref class.

## Resolution model

Resolution happens during generated `from_gdata()` conversion.

The top-level caller does not normally construct or pass a cache object.
What the caller provides is:

- `n`: the gdata node being deserialized into adata
- optionally `lookup_root`: a root-shaped gdata view used for ref lookup

When a subtree is being materialized, `n` is the subtree at the
transform point while `lookup_root` is expected to describe the same
conceptual data tree from the absolute schema root. In the full-tree
case, generated root `from_gdata()` simply defaults `lookup_root` to
`n`.

From those inputs, generated adata creates or reuses a
`yang.adata.RefContext`. This context contains:

- `lookup_root`: the gdata tree used for ref lookup
- `target_cache`: a dictionary that canonicalizes resolved target objects

`target_cache` is created internally on the first call to
`yang.adata.init_ref_ctx(...)`. It is then passed around internally as
`_ref_ctx` through all nested generated `from_gdata()` calls, so all
refs built during one conversion share one cache.

### Why `lookup_root` exists

Transforms are often materialized from a subtree rather than the schema
root. A transform point like `/transforms/consumer[name='c1']` does not
contain the rest of the tree needed to follow a leafref into
`/inventory/producer[...]`.

To handle that, generated `from_gdata()` has this shape:

```acton
from_gdata(n, lookup_root=None, _ref_ctx=None)
```

- `n` is the subtree being materialized into adata
- `lookup_root` is a larger root-shaped gdata view used only for ref
  resolution

If the generated root object is built from the full tree, `lookup_root`
defaults to `n`. For subtree materialization, the caller can pass a
separate root view that represents the same conceptual dataset from the
absolute root.

## How a lookup is made

For each supported leafref field, code generation emits a resolver
function specialized to that field. The generated resolver already knows:

- the target list path
- the target key leaf
- the generated adata entry type of the target list

So runtime lookup does not interpret XPath. It uses the compiled target
metadata baked into the generated code.

Given a raw ref value, the resolver:

1. creates the specialized ref object with `value=<raw>`
2. initializes or reuses the shared `RefContext`
3. computes a cache key from the target list path and raw key value
4. checks whether the target adata object is already cached
5. if not cached, walks `lookup_root` to the target list
6. scans the target list for an element whose key leaf matches the raw
   ref value
7. materializes that list entry as adata
8. stores the resulting target object in the cache
9. assigns `ref.target`

The actual tree walk is done by
`yang.adata.lookup_list_entry_by_key(...)`. It is deliberately small and
only supports the same narrow subset as code generation:

- walk container children down to the target list
- iterate list elements
- compare the target key leaf with the raw ref value

## Cache behavior and shared identity

The cache is per materialization, not global.

The cache key is:

```text
<target-list-path>::<repr(key-value)>
```

For example:

```text
/inventory/producer::'prod-a'
```

This cache is reused in two directions:

### Ref resolves to an already built target

If the target list entry was already materialized earlier in the same
conversion, the resolver finds it in `target_cache` and assigns it
directly to `ref.target`.

### Target list entry is first built through a ref

If the resolver materializes the target first, it stores that target in
the cache. Later, when generated list-entry `from_gdata()` encounters the
same list element through normal tree traversal, it also checks the same
cache key and reuses the existing object.

This is what gives pointer identity across references. If two consumers
both have `input = "prod-a"`, both refs will have the same
`producer_entry` object in `target`.

## What `.target` means

The useful object for transforms is usually not the leaf instance itself,
but the owning modeled node. In the current subset, the target is the
owning list entry because the leafref is required to point to the single
key leaf of a list.

So for a leafref to `/inventory/producer/name`, `.target` is a
`producer_entry`, not the raw `name` leaf.

## Serialization and validation

Ref objects still serialize as their raw leaf values.

That is handled by `yang.adata.try_validate_mnode_value(...)`, which
unwraps `Ref.value` before leaf validation. As a result:

- `to_gdata()` emits the raw scalar leafref value
- the resolved `target` is not serialized
- ordinary leaf validation still works

`pradata()` was also updated to emit ref-wrapper construction for
supported leafrefs, so generated source roundtrips preserve the new adata
shape.

## Example flow

Consider this YANG model:

```yang
container inventory {
  list producer {
    key "name";
    leaf name { type string; }
    container config {
      leaf interval { type uint32; }
    }
  }
}

container transforms {
  list consumer {
    key "name";
    leaf name { type string; }
    leaf input {
      type leafref {
        path "/inventory/producer/name";
      }
    }
  }
}
```

If a transform is materialized from `/transforms` with a separate
`lookup_root` that also contains `/inventory`, then:

1. `consumer.input` is read as `"prod-a"`
2. the generated resolver looks for `/inventory/producer[name='prod-a']`
3. the matching `producer` entry is built as adata
4. `consumer.input.target` points to that `producer_entry`
5. another consumer with the same `input` value reuses the same cached
   `producer_entry`

That is why mutating the target through one ref is observed through the
other in the current tests.

## Current limitations

This is still a phase-1 implementation. Notable limits are:

- no predicate support in leafref paths
- no support for targets that are not single-key list keys
- no support for intermediate lists in the target path
- no lazy resolution after materialization
- no global cache across separate `from_gdata()` calls

The design is still useful because it matches the common transform case:
compile a small, predictable subset into generated code and avoid a large
runtime resolver.
