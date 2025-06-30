- Title: Multiple Default Typeclass Databases

- Drivers: @Janno, @mattam82

----

# Summary

The hint databases `typeclass_instances` is special in ways that cannot be
emulated by using other databases. The user experience of building up class
hierarchies and proof automation outside the default database is practically
prohibitive. This forces most Rocq developments to share a single database, and,
thus, makes it all but impossible to fine-tune settings such as opacity to the
needs of any individual project unless the project is never combined with other
projects that use typeclasses.

We propose to allow projects to register new "default TC databases", i.e.
databases that are used by default in tactics such as `apply` and `rewrite`. By
disentangling the identity of the database(s) used from their status as a
default TC database, we allow projects to regain full control over all hint database
settings.

# Motivation
At the time of writing, `typeclass_instances` is the only database that is used
seriously for developing typeclass hierarchies and proof automation in almost
all Rocq developments. This is owed to its special status in tactics and certain
vernacular commands:

1. At least `apply _`, `typeclasses eauto` (without arguments), and
   `(setoid_)rewrite` hardcode `typeclass_instances` as the (only) source of instances.
2. `typeclass_instance` is the default database for opacity and hint mode information for instances found in the local context.
3. `Instance` is restricted to `typeclass_instances` (`Hint Resolve` exists as a partial workaround).
4. `Typeclasses Opaque/Transparent` is restricted to `typeclass_instances` (`Hint Opaque/Transparent` exists as a workaround).

All these restrictions, but mostly (1), severely harm modularity: by forcing all
Rocq developments to go through a single database, Rocq makes it impossible to
enforce project-specific database settings that make sense for one development
but not for others.

This is most visible in opacity settings: for example, two developments that
make use of `List.map` in their typeclass searches might have conflicting needs
about its matching and reduction behavior during the search. One client might
need to have the function reduce and thus set it to be transparent, while
another client might rely on it being opaque for performance reasons. This is
currently impossible if these two developments want to be able to co-exist,
possibly in a third development that includes them both.

The key observation that motivates this CEP is that the problem is almost
entirely trivial to solve in many cases. In particular, many developments
already work on entirely or almost entirely disjoint typeclasses. The only
reason they all end up in the same database is because the system is set up in a
way that makes that choice the past of least resistance (and sometimes the only
path, e.g. if clients are expected to benefit from the typeclasses in `apply _`
and the other tactics listed above).

We propose to address the problem by introducing the notion of "default
typeclass databases". Any database can be made a default TC database by
registering it with a new vernacular command:
```rocq
Add Default Typeclass Database my_db.
```
This command changes the behavior of all tactics that hardcode
`typeclass_instances` to also consider `my_db` as an alternative but *equal*
source of hints. In particular, there will be no behavior specific to
`typeclass_instances` that will not also extend to `my_db`.

The dual command,
```rocq
Remove Default Typeclass Database my_db.
```
undoes the change and reverts to the original list of databases. We expect this
command to be used mostly for debugging.

We expect most developments that develop hierarchies of classes and/or proof
automation to create and register one such database, ideally using the name of
the project to work around the lack of namespacing in hint databases.

This proposal improves the current state in the following ways:
1. Disjoint TC hierarchies can live in entirely disjoint databases, with no
   possible incompatibilities in opacity settings for shared terms.
2. Even non-disjoint hierarchies extended by distinct developments can disagree
   on the opacity of shared terms as long each development's respective
   instances work in isolation.

# Detailed design

## Core Search
The implementation of typeclass search already supports searching multiple
databases. There are two technical issues that need to be addressed:

### Duplicate Hints

With multiple default TC databases it could very well be the case that some
hints are duplicated across these databases. While it might be tempting to try
to deduplicate them, this is not actually possible in the general case and
probably should not be attempted. The reason is that the priority, opacity
settings, and hint modes of the databases involved can differ in incomparable
ways. This makes the semantics of the hints incomparable, i.e. there will likely
be no unifying relaxation of the priority, transparency, and the modes that
covers both sources of the hint faithfully. For this reason, duplicate hints
will simply be applied multiple times at exactly their specified priority,
opacity, and modes.

### Local Instances

Instances in the local context are currently interpreted in two possible ways:
- For `typeclasses eauto with <db1> [db2 .. dbN]` (i.e. one or more explicit
  database(s)), the instance's opacity and modes are taken from `db1`.
- In all other cases, the instance's opacity and modes are taken from
  `typeclass_instances`.

This behavior will be changed to "duplicate" the local instance across all
databases (explicitly specified, if applicable, or default TC databases,
otherwise) that have either (1) instances, or (2) modes for the class in
question.

The duplicated instances will use all settings from their respective
database. This ensures that, if the instance matches, it matches in accordance
with one coherent setting, i.e. we pretend as if the instance had temporarily
been added to the database.

## Vernacular Commands

### Default Typeclass Database
`Add Default Typeclass Database` and `Remove Default Typeclass Database` are
similar to existing tables but take identifiers instead of references. The exact
implementation is to be determined but should be trivial. The tables should
follow the (recently fixed) locality rules of the existing tables.

### Existing Commands
All commands that are specific to `typeclass_instances` will be generalized
using attributes that allow specifying the target database. This includes
`Instance`, `Typeclass Transparent/Opaque`. The exact attribute name needs
bikeshedding.

For consistency, this attribute should eventually be available in all `Hint`
related vernacular commands. But this can be done in a separate step.

### Possibly: Select Default Typeclass Database
The overhead of requiring annotations for every use of a new default TC database
might be too high. We could add a setting that picks a "default" default TC
database (needs a better name) and interprets `Instance` and `Typeclasses
Transparent/Opaque` to take effect in that database.

# Drawbacks

The new system requires database annotations on many commands unless we add a
way to select a "default" default TC database.

# Alternatives

I am not aware of other systems that suffer from these particular problems (yet)
and I am not aware of alternative proposals that try to disentangle transparency
state and modes on shared definitions.

# Unresolved questions

None as of now.


