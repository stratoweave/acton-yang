# Acton YANG
[![REUSE Compliance Check](https://github.com/stratoweave/acton-yang/actions/workflows/reuse-compliance.yml/badge.svg)](https://github.com/stratoweave/acton-yang/actions/workflows/reuse-compliance.yml)

The schema.act module, which implements the core of the YANG schema in, is
generated from the YANG RFC 7950 by rfcgen.act. The output is concatenated with
a manually written top half, in schema-manual.act

## Compile

```shell
make
```

# Things we should do (TO DO) [2/26]
- [X] TODO: drop y_ prefix
- [X] TODO: group multiple YANG modules into a tree so that the generated output has a common root for multiple modules
- [x] TODO: use DNode
- [ ] TODO: support all leaf types
  - [ ] actually support all leaf types
  - [ ] write comprehensive tests for input YANG
- [ ] TODO: populate namespace / prefix in stmt_to_snode
- [x] TODO: support schema expansion of uses / groupings
- [x] TODO: support expanding of types to the base type
- [x] TODO: support YANG parsing of multiple modules, like for imports and augment
- [x] TODO: support schema expansion of augments
- [x] TODO: rewrite - to _ in names
- [x] TODO: resolve union of various int types to largest (u)int
- [ ] TODO: detect conflicts between a_b and a-b, and name in some deterministic fashion
- [x] TODO: add simple compile() function to compile a bundle of YANG models and get a DNode tree
- [x] TODO: fix bug, duplicating content when revision statement is present
- [x] TODO: correctly map union base type, currently getting "string" and not "str" as it should be
- [ ] TODO: add test for expansion of uses/grouping, where same grouping is used in two places but in one place it is modified - we need to get copies of the content of the grouping, not mutate the grouping by "moving" it (rewriting parent etc) into uses location
- [ ] TODO: compilation must consider import dependencies when compiling modules, compile in order of dependencies!
- [ ] TODO: compilation of augments must be ordered, in a single module we can have a container /c1, then first augment which augments target path /c1 and adds in container c2, e.g. /c1/c2, then a second augment with target path /c1/c2 and adds in a third container c3 in there. Now consider the order of the augments in the YANG module, they could be in arbitrary order. We must compile the augments in order - simplest approach is to order augments by target path length, starting with shortest?
- [x] TODO: support submodules
- [ ] TODO: support submodules with different prefix than parent module or some advanced setup like that? not sure exactly what the test case should look like...
- [ ] TODO: when printing data classes, do not expand groupings into the specific path but use single Acton class that maps to grouping and reuse that in multiple places - only works when the uses is unmodified, i.e. no refines or whatnot
- [ ] TODO: check conflicts when merging submodule content into main module
- [ ] TODO: add conflicting node names, like two children called "c1" under the same container
- [ ] TODO: propagate namespace & prefix through SchemaNode tree
- [ ] TODO: augment should use prefix when resolving nodes so we can augment correct node based on namespace
  - prefix are resolved from the perspective of the module that contains the augment statement
    - but nodes are placed in target path module
      - thus all nodes in target module need to have namespace so we can correct

# Documentation
## Identities
YANG identities define globally unique constants organized into a hierarchical structure. An identity defined in a YANG module belongs to the namespace of that YANG module. Each identity may be derived from one or more base identities.

Identities are then used in leafs and leaf-lists of type *identityref*. The *identityref* type may then restrict the set of valid values to one or more base identities. The global nature of identities and the possibility to create cross-module relationships influenced the decision to compose the list of identities in the global context. In acton-yang they are available in the *DRoot.identities* attribute (*DRoot* is the output of *yang.compile(list_of_yang_modules)*).

### User Interface (adata)
A leaf of YANG type identityref with a restricted set of bases only accepts a limited set of valid values. We enforce that in the generated Acton module (adata), partially also at compile time. In adata we use the type *yang.identityref.Identityref* for a list. This type holds information that maps to a single identity - its name and namespace qualifiers. For example, the following YANG module with local identities and a leaf:

```yang
module foo {
  namespace "uri:example:foo";
  prefix "f";

  identity name;
  identity bob {
    base name;
  }
  identity rick {
    base name;
  }

  leaf name {
    type identityref {
      base name;
    }
  }
}
```

Results in the following abbreviated Acton module

```acton
# ...
# Identityref constants
foo_name = Identityref('name', ns='uri:example:foo', mod='foo', pfx='f')
foo_bob = Identityref('bob', ns='uri:example:foo', mod='foo', pfx='f')
foo_rick = Identityref('rick', ns='uri:example:foo', mod='foo', pfx='f')

# ...

class root(yang.adata.MNode):
    name: Identityref
# ...
```

The Identityref module constants at the top of the generated module are available as a convenience to users. In the transform code they may reference these constants for assignment or equality checks:

```acton
if root.name == foo.foo_bob:
    print("Hello Bob!")

new = root(foo.foo_rick)
```

The generated module constants do not (yet) encode the identity hierarchy. This means that it is possible to assign an invalid identity - one that is not derived from any of the allowed bases and that will not result in a compilation error. When the adata object graph is converted to the internal gdata representation, identityref leaf values are validated though and an invalid identity assignment will result in a runtime error.

### Internal implementation
#### Compiling the modules
*yang.schema.DRoot* joins the top-level data nodes from all compiled modules under a single root node. At this point we also collect the identities defined in all modules and compose a global list of identities. These are represented as *yang.schema.DIdentity*. We also validate the existence of all referenced base identities, including cross-module dependencies, and also check that the identity hierarchy is acyclic. The resulting *list[DIdentity]* is the global registry of identities with the base identities resolved as references to objects in the same list.

The global identity list is also printed in the generated adata module and made available as the *_identities* module constant for internal use.

Note: the decision to use a list instead of a dictionary was influenced by the requirements for identity lookup from partial data. In XML and JSON data formats the namespace qualifiers for an identity (namespace and module name respectivelly) are optional if the identity is defined in the same namespace.

#### Converting input data (XML, JSON) to gdata
The *yang.gdata* functions that "take" an XML node or JSON dict and return a value, return a *yang.identityref.PartialIdentityref* value. The attributes of this type are the same as *Identityref* which is user-facing, with the exception that everything but the identity name is optional. This allows the generated gdata conversion free functions to look up an identity (*yang.schema.DIdentity*) from the global identity registry and then also validate whether this identity conforms to the type restrictions (is derived from listed bases). As part of this the input *PartialIdentityref* is converted to a complete *Identityref* with all namespace qualifiers set. That *Identityref* is then also used as the value for a *yang.gdata.Leaf*.

#### Converting gdata to output (XML, JSON)
The guiding principle for gdata has always been that it should contain just enough information necessary to convert the internal gdata representation to a user-facing format: XML or JSON. The *Identityref* values could be seen as an exception to this, in that they always include the namespace qualifiers, even in cases where they are optional because the namespace is implicitly set to that of the containing data node. As a result of this design decision, the identityref leaf values output as XML always include the namespace attribute and when output to JSON they always include the module prefix.
