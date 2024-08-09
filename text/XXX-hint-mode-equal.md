- Title: Fine-grained `Typeclasses Strict Resolution`: `Hint Mode =`

- Drivers: Janno

----

# Summary

Coq lacks the ability to designate individual typeclass arguments as "invariant"
(i.e. frozen/unchangeable) in the search process. `Strict Resolution` can
sometimes be used to work around this omission but only if the class generates
no outputs. This CEP proposes a new `Hint Mode`, `=`, which provides the same
semantics as `Strict Resolution` on a per-argument level.

# Motivation

It is difficult in Coq to write typeclass instances that are robust w.r.t. the
presence of evars in the query. Let's consider a hypothetical class `C : T ->
Prop` that is used in user-facing proof automation. (The concrete type `T` does
not matter.) Without extra settings, an instance of type `forall x, C x -> C (M
x)` will trigger on `C (?y)` and lead to an endless loop.

There are two available settings that mitigate this issue. The first option is
to restrict the classes `Hint Mode`s. For `C`, the only modes that will help are
`+` and `!`. `+` defeats the point of our hypothetical example: the class should
support queries with evars in them. This leaves only `!`. However, a very simple
alteration to our example shows that `!` is not sufficient. An instance of type
`forall x, C (M1 x) -> C (M1 (M2 x))` will loop on the query `C (M1 ?y)`.

At the time of writing, the only solution to this problem is to introduce an
auxiliary class (again with `Hint Mode !`) to match applications of `M2`. This
workaround is used in production. For example, Iris employs such auxiliary classes:
https://gitlab.mpi-sws.org/iris/iris/-/blob/0da37f400b47ff58f8b57003aed398f6d4a2eb41/iris/proofmode/classes.v#L354.

There is a second option which is only applicable when the typeclass has no
outputs: Enabling `Set Typeclasses Strict Resolution` will freeze all evars
before applying hints. With this setting our instance no longer loops.
Unfortunately, examples such as those in Iris do not fulfill the requirement. The
`Frame` class in which the auxiliary classes are required has outputs.

Even in cases where `Strict Resolution` is applicable, it leads to arguably
confusing `Hint Mode` settings. Consider a typeclass that is polymorphic in some
structure with a carrier type `Class P : forall (s : S), car s -> Prop`. We
assume that `P` is purely a property of a term, i.e. it has no outputs and is
thus amenable to `Strict Resolution`. Assume further that there exists an
operator `F : forall (s : S), car s -> car s` with an instance `forall s x, P s
(F s x)`. It should be possible for the query `P ?s (F ?s ?x)` to succeed. We
thus set the modes to: `Hint Mode P - !`. But, following Coq's documentation,
this means that `s` is an output of the class. Clearly that is not the case here.

In summary, while workarounds do exist, they are too restrictive, create
confusing `Hint Mode` settings, and lead to undesirable boilerplate.


# Detailed design

The design is conceptually simple: `=` is added as a fourth `Hint Mode` option. The
preliminary name for this mode is "invariant", signaling that the term is not
going to be changed by typeclass search.

`=` is different from the existing modes in that it is not a new prefilter for
hints. In fact, it selects exactly the same hints as `Hint Mode -`. Unlike `-`,
it affects the behavior of hints once they are applied. This distinction makes
the semantics of `=` a little more complex when multiple modes for a single
class in a database match the current query. The current logic only needs to
figure out if there is a single matching mode (or no modes at all) and then
applies all hints. With this proposal, if one of the modes is `=`, we have to
perform a whole new set of hint applications for that particular mode line.

To keep the order of hint applications consistent with their cost, every mode
line that contains `=` will add one additional attempt of any given hint. That
attempt will have its `allowed_evars` set restricted in accordance with the `=`
modes of the classes arguments. The order between attempts of the same hint that
belong to different mode lines should be left unspecified.

# Drawbacks

The fact that multiple mode lines can now lead to additional hint applications
is unfortunate but seems unavoidable. Fortunately, overlapping mode lines are
rare in practice and there should not be many use cases for overlapping mode
lines that involve `=`. In case of non-overlapping mode lines, it is likely that
`=` will simply replace many uses of `+` and !` in which case no additional work
is performed.

A drawback that is shared with `Strict Resolution` is that `=` cannot control
the behavior of `Hint Extern` which are free to instantiate any and all evars. I
don't think this issue can be addressed in this scope of this CEP.

Another drawback is that the already extremely confusing language around `Hint
Mode` becomes even more complicated. Arguable, the newly proposed `=` mode is
the true "input" mode: the term is provided and unchanged by the search
procedure. The current "input" mode `+` should be called "ground" instead. The
SWI-Prolog documentation is bit more careful about terminology and correctly
avoids calling `+` (or `++`) an input mode:
https://www.swi-prolog.org/pldoc/man?section=modes. Unfortunately it still calls
`-` an output mode despite going on a lengthy explanation about how such an
argument can be used to provide input. Perhaps `-` should be called
"unrestricted" instead. (Interestingly, the documentation also lists a true input
mode, `--`. I have not found a use for this yet in Coq outside of debugging.)

# Alternatives

Syntax alternatives: `@` is used in the Prolog world but I do not know why. `=`
is arguably more intuitive.

Other systems:
- Lean 4 has `+` (the default) and `-` (by wrapping the argument type in
  `outParams`). See
  https://lean-lang.org/lean4/doc/typeclass.html#output-parameters. It also has
  `semiOutParam` which could be related but I do not understand the documentation:
  https://github.com/leanprover/lean4/blob/ea43ebd/src/Init/Prelude.lean#L641


# Unresolved questions
- `=` versus `@` (or maybe something else entirely?)
- Should the system warn about mode lines such as the one below? I do not
  understand enough about how the unification algorithms deal with frozen evars
  to say if the first mode line is truly redundant.

```coq
Hint Mode C =.
Hint Mode C -.
```
