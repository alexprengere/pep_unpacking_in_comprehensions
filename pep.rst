PEP 9999: Allowing Iterable Unpacking in Comprehensions
=======================================================

| **PEP**: 9999 (draft)
| **Title**: Allowing Iterable Unpacking in Comprehensions
| **Author**: Alex Prengère alex.prengere@gmail.com
| **Status**: Draft
| **Type**: Standards Track
| **Python-Version**: 3.15
| **Created**: 29-Apr-2025
| **Discussions-To**: `Discuss Python Ideas – Topic
  35291 <https://discuss.python.org/t/using-unpacking-to-generalize-comprehensions-with-multiple-elements/35291>`__

Abstract
--------

This PEP proposes extending Python's comprehension syntax to allow the
use of iterable unpacking (``*`` and ``**`` operators) inside list, set,
and dictionary comprehensions, as well as generator expressions.
This change would enable a comprehension to yield multiple elements per
iteration (flattening an iterable of iterables) or merge multiple mappings
in a dict comprehension. The proposed syntax is a natural extension of the
unpacking generalizations introduced in PEP 448, applied to comprehensions.
The goal is to provide a more concise and readable way to flatten sequences
or combine mappings in a comprehension, without needing nested loops or
external functions.

Motivation
----------

Python comprehensions (list, set, and dict comprehensions) provide a
concise way to transform or filter iterable data. A common refactoring
pattern is to convert simple loop constructs into comprehensions for
clarity and brevity. For example, given a list of ``Flight`` objects,
one can collect all departure airports with a list comprehension:

.. code:: python

   # Before refactoring
   points = []
   for f in flights:
       points.append(f.departure)

   # Using a simple list comprehension for a single attribute
   points = [f.departure for f in flights]

However, if each iteration needs to produce **multiple** elements,
current comprehension syntax falls short. Consider that each ``Flight``
has a departure and an arrival location, and we want to build one list
containing **both** departures and arrivals from a list of flights. In
regular Python code, one might write:

.. code:: python

   # Imperative approach to collect multiple values per iteration
   points = []
   for f in flights:
       points += (f.departure, f.arrival)

This loop appends two elements per ``Flight`` (``departure`` and
``arrival``). Ideally, we would like to express this flattening in a
comprehension. Today, the most straightforward comprehension equivalent
is not obvious — one must either use an additional nested loop or an
external helper:

-  **Using a nested comprehension with two for clauses** (one for
   flights, one for points in each flight tuple):

   .. code:: python

      points = [point for f in flights for point in (f.departure, f.arrival)]

   This produces the desired flattened list, but it is somewhat
   unintuitive. The syntax of multiple ``for`` clauses is less
   immediately clear to readers (it essentially double-loops and
   flattens), and also requires creating a tuple ``(f.departure, f.arrival)``
   for each flight and an extra loop to unpack it.

-  **Using itertools.chain.from_iterable**:

   .. code:: python

      from itertools import chain
      points = list(chain.from_iterable((f.departure, f.arrival) for f in flights))

   This avoids a manual nested loop by using a generator expression and
   the ``chain.from_iterable`` flattening utility. While it works, it is
   more verbose and arguably less readable; one must mentally parse the
   generator and the role of ``chain`` in flattening it.

Both approaches achieve the result, but neither is as clear or concise
as a list comprehension typically aspires to be. The
comprehension-with-nested-loop syntax, in particular, is often
considered tricky for newcomers (one needs to remember the order of the nested loop).
Many Python users might even **expect** the unpacking operator
to work inside a comprehension, given that it works in other
iterable-building contexts. For instance, since Python 3.5 we can write:

.. code:: python

   a = [1, 2, 3]
   b = [4, 5, 6]
   combined_list = [*a, *b]      # yields [1, 2, 3, 4, 5, 6]
   combined_set = {*a, *b}      # yields {1, 2, 3, 4, 5, 6}

The ``*`` operator here *flattens* or unpacks two iterables into a new
list or set literal. It is natural to expect that a similar construct
might be possible in a comprehension to flatten elements during
iteration. Indeed, one might try to write a comprehension as follows:

.. code:: python

   # Proposed syntax (currently a SyntaxError)
   points = [*(f.departure, f.arrival) for f in flights]

Intuitively, this syntax suggests: "for each flight, unpack the tuple
``(f.departure, f.arrival)`` into the resulting list." Currently, this
is invalid syntax in Python (it raises a ``SyntaxError`` complaining
that "iterable unpacking cannot be used in comprehension"). This PEP's
motivation is to lift that restriction and allow such syntax, thereby
making comprehensions more general and powerful. The ability to yield
multiple items per iteration in a comprehension would directly address
the patterns above, enabling more readable code for flattening
use-cases.

Beyond this specific example, there are broader use-cases for unpacking
in comprehensions:

-  **Flattening a list of lists**: e.g. converting
   ``[[1,2,3], [4,5,6]]`` into ``[1,2,3,4,5,6]`` in one comprehension,
   rather than using a double loop or ``chain``. Many Python users have
   to look up how to flatten a nested list; an unpacking comprehension
   could make it obvious:

   .. code:: python

      matrix = [[1,2,3], [4,5,6]]
      flat = [*row for row in matrix]        # proposed, flattens each sub-list
      # Equivalent to: flat = [x for row in matrix for x in row]

-  **Flattening a set of sets or other iterables** in a set
   comprehension, similarly.

-  **Merging dictionaries** in a dict comprehension: e.g. combining a
   list of dicts into one dict. Currently one might do
   ``{k:v for d in dicts for k,v in d.items()}``, but with this
   proposal:

   .. code:: python

      dicts = [{"a": 1}, {"b": 2, "c": 3}]
      merged = {**d for d in dicts}         # proposed, merges all dicts into one
      # Equivalent to: merged = {k: v for d in dicts for k, v in d.items()}

In all these scenarios, allowing unpacking in comprehensions would
simplify the code and improve readability by directly reflecting the
idea of flattening or merging. The motivation is to make these patterns
more accessible and idiomatic, leveraging a syntax (``*``/``**``) that
Python programmers already use for similar purposes in other contexts.

Rationale
---------

**Why use the * and ** syntax?** This proposal builds on an
existing, well-understood concept: the unpacking operator. In Python,
``*iterable`` is widely recognized as the way to "flatten" an iterable
into another iterable context (such as in function calls or literal
displays), and ``**mapping`` is used to merge mappings. Applying the
same operators in comprehension output expressions is a consistent
extension of this concept (`Unpacking in
tuple/list/set/dict comprehensions - Python-ideas -
python.org <https://mail.python.org/archives/list/python-ideas@python.org/message/7G732VMDWCRMWM4PKRG6ZMUKH7SUC7SH/#:~:text=Extended%20unpacking%20notation%20%28,set%20with>`__)
(`Unpacking in tuple/list/set/dict
comprehensions - Python-ideas -
python.org <https://mail.python.org/archives/list/python-ideas@python.org/message/7G732VMDWCRMWM4PKRG6ZMUKH7SUC7SH/#:~:text=propose%20,attempt%20to%20argue%20that%20the>`__).
A comprehension with ``*expr`` for a sequence effectively means “extend
the result with the items from ``expr`` on each iteration," which
parallels how ``[*a, *b]`` extends a list with items from ``a`` and
``b``. Likewise, ``{**m for m in mappings}`` would mean “update the
result dict with all items from ``m`` on each iteration," analogous to
``{**m1, **m2}`` merging two dicts.

**Readability and familiarity:** When this idea was first floated years
ago (during the discussions for PEP 448 in 2014), some core developers
expressed concerns about readability (`PEP 448 – Additional Unpacking
Generalizations \|
peps.python.org <https://peps.python.org/pep-0448/#variations#:~:text=>`__).
The comprehension syntax was intentionally limited to one item per
iteration to keep the mental model simple. However, since then, Python
users have become much more familiar with starred unpacking in various
contexts. Features introduced by PEP 448 (extended unpacking in literals
and calls) are now commonplace, and their semantics are well understood.
The proposed comprehension unpacking reads naturally once you know what
``*`` means: for example, ``[ *row for row in matrix ]`` is easily
understood as flattening each ``row``. In fact, evidence of its
intuitive nature can be found in user discussions – people periodically
ask why this syntax isn't allowed or attempt to use it, indicating that
it *feels* like a logical part of the language. Even newcomers, once
they learn about ``*`` for unpacking, often find the double-loop
comprehension idiom harder to grasp than the concept of a "flattening
``*``". Thus, the readability concern has likely diminished over time.

**Consistency with mental model:** A comprehension today can be viewed
as syntactic sugar for a loop that appends to a list (or adds to a set,
or assigns to a dict) one element per iteration. With this PEP, a
comprehension with a starred expression can be understood as a loop that
extends a list (or updates a dict) per iteration. For example:

.. code:: python

   # Proposed semantics illustrated in imperative form:
   result_list = []
   for f in flights:
       result_list.extend((f.departure, f.arrival))

The above loop is precisely what
``[* (f.departure, f.arrival) for f in flights]`` would do. Similarly, a
dict comprehension with ``**`` would ``update`` the result dict in each
iteration. This change is minimal and keeps a clear conceptual model:
*use ``append`` for single items, use ``extend/update`` for starred
items*. This is analogous to how one might teach the difference between
``list.append(x)`` vs ``list.extend([...])`` – the comprehension is just
doing it implicitly. Some have argued that this slightly complicates the
comprehension model since it's no longer a one-to-one correspondence
with a simple append (`Mailman 3 [Python-ideas] Unpacking in
tuple/list/set/dict comprehensions - Python-ideas -
python.org <https://mail.python.org/archives/list/python-ideas@python.org/message/7G732VMDWCRMWM4PKRG6ZMUKH7SUC7SH/#:~:text=Arguments%20against%3A%20,x...%20for%20x%20in>`__).
However, this is an *opt-in* complexity: it only applies when the
comprehension explicitly uses ``*`` or ``**``. In practice, developers
using this syntax are likely those who already understand the concept of
extending vs appending.

**Why now?** The idea of comprehension unpacking was explicitly
considered and set aside when PEP 448 was implemented, largely to avoid
delaying the rest of that PEP's features (`PEP 448 – Additional
Unpacking Generalizations \|
peps.python.org <https://peps.python.org/pep-0448/#variations#:~:text=>`__).
The deferred feature was noted as something that “has not been ruled out
for future proposals" (`PEP 448 – Additional Unpacking Generalizations
\|
peps.python.org <https://peps.python.org/pep-0448/#variations#:~:text=>`__).
Now, a decade later, the landscape is favorable for reconsidering it:
the Python community has ample experience with extended unpacking, and
we have real-world examples where this feature would simplify code. By
revisiting the idea with fresh eyes, and providing strong motivating
use-cases (as in this PEP), we can address the previous concerns. The
core arguments against the idea (that it might be counterintuitive or
hard to teach) can be weighed against the benefit of more expressive
code. Proponents argue that for those who understand ``*`` unpacking,
the comprehension form is actually **more** intuitive than the status
quo alternatives (`Mailman 3 [Python-ideas] Unpacking in
tuple/list/set/dict comprehensions - Python-ideas -
python.org <https://mail.python.org/archives/list/python-ideas@python.org/message/7G732VMDWCRMWM4PKRG6ZMUKH7SUC7SH/#:~:text=ideas%40python,x...%20for%20x%20in>`__).
In terms of implementation and consistency, it's also worth noting that
the change is small and was even prototyped during PEP 448's development
(the reference implementation of PEP 448 had this enabled until it was
deliberately turned off (`PEP 448 – Additional Unpacking Generalizations
\|
peps.python.org <https://peps.python.org/pep-0448/#variations#:~:text=Implementation>`__)).
This indicates that enabling it now would be straightforward, and tools
(like linters, code formatters) would likely have an easy time adapting
since the construct is syntactically clear.

In summary, the rationale for this proposal is that it introduces a
powerful yet simple extension to an existing syntax, aligns with
Python's design philosophy of readability, and solves a recurring need
in a consistent way. It leverages an established operator (``*``/``**``)
for a new but related purpose, thereby minimizing the learning curve and
surprise for Python users.

Specification
-------------

**Syntax changes:** This PEP extends the grammar for comprehensions as
follows:

-  **List and set comprehensions:** Permit an unpacking operator ``*``
   directly before the item expression in a comprehension. In formal
   terms, the syntax:

   .. code:: text

      comprehension ::= "[" starred_expression "for" target_list "in" iterable (comp_iter) "]"
                       | "{" starred_expression "for" target_list "in" iterable (comp_iter) "}"

   is allowed, where ``starred_expression`` is an expression prefixed by
   ``*``. (The same extension applies to generator expressions in
   parentheses, although such an expression would produce a generator of
   flattened items. *Note:* A generator expression cannot be directly
   starred in a function call without parentheses, as per existing
   syntax rules (`PEP 448 – Additional Unpacking Generalizations \|
   peps.python.org <https://peps.python.org/pep-0448/#variations#:~:text=Unbracketed%20comprehensions%20in%20function%20calls%2C,These%20could%20be%20extended%20to>`__),
   so this proposal does not change function call semantics – it only
   concerns the comprehension construct itself.)

   Semantically, a comprehension of the form ``[ *expr for ... ]`` will
   iterate just as ``[expr for ...]`` does over the specified loop(s)
   and conditions, but instead of yielding ``expr`` as a single element
   each iteration, it will iterate over ``expr`` (which must be an
   iterable) and yield all of its elements. In effect, each iteration of
   the comprehension produces zero or more elements in the final result
   (zero if the iterable is empty). A set comprehension
   ``{ *expr for ... }`` behaves analogously, adding all elements of the
   iterable ``expr`` to the resulting set each iteration. Order in set
   comprehensions is of course not guaranteed, as usual.

-  **Dictionary comprehensions:** Permit the ``**`` unpacking operator
   in a similar fashion. The new syntax allows:

   .. code:: text

      dict_comprehension ::= "{" "**" expression "for" target_list "in" iterable (comp_iter) "}"

   In a dict comprehension, using ``**expr`` means that on each
   iteration, ``expr`` must be a mapping (for example, a ``dict``), and
   all its key-value pairs are added to the result dictionary (much like
   ``result_dict.update(expr)``). If duplicate keys occur across
   iterations, the last one wins, just as in ``{**d1, **d2}`` literal
   merges or successive ``dict.update()`` calls – later iterations will
   override earlier ones for duplicate keys. It is an error (likely a
   ``TypeError``) if an ``**expr`` in this context produces something
   that is not a mapping, similarly to how ``**`` behaves in function
   calls and literals.

-  **Mixed usage:** Within a single comprehension, the syntax does not
   allow combining a starred expression with other expressions at the
   same level. For example, ``[x, *y for ...]`` is not a valid
   comprehension syntax (and would be ambiguous). The comprehension's
   output expression must be either a single (non-starred) expression
   yielding one item per iteration, or a single starred expression (or
   double-starred for dict) yielding multiple items per iteration. If
   multiple ``for`` clauses or ``if`` filters are present, they apply to
   the starred form in the same way as they would to a normal element.
   For instance, ``[ *expr for x in xs if cond ]`` will only unpack
   ``expr`` for those ``x`` that satisfy the condition.

The rest of the comprehension syntax (loop nesting, conditional filters)
remains unchanged. This proposal does not introduce any new keywords or
operators — it merely lifts a restriction on the existing ``*`` and
``**`` token usage within comprehensions.

**Evaluation order and scope:** The evaluation order for comprehensions
with unpacking remains the same as for normal comprehensions. The
expression following ``*`` (or ``**``) is evaluated in the innermost
loop scope for each iteration that passes all filters. If that
expression produces an iterable (or mapping, for ``**``), its elements
are processed immediately into the result. If the expression raises an
exception or is not iterable, the comprehension will propagate that
error at runtime (just as a failing expression in a normal comprehension
would). Comprehensions with unpacking still create a new frame for the
loop variable(s), just like existing comprehensions.

**Examples of the new semantics:**

-  List comprehension example: ``[ *range(n) for n in [1, 4, 0, 3] ]``
   would result in a list equivalent to
   ``[*range(1), *range(4), *range(0), *range(3)]``, i.e. it flattens
   each range: ``[0, 0,1,2,3, (nothing), 0,1,2]`` resulting in
   ``[0, 0, 1, 2, 3, 0, 1, 2]``. An empty iterable (like ``range(0)``)
   contributes nothing, just as one would expect.

-  Dict comprehension example: Suppose
   ``dicts = [{"x": 1}, {"y": 2, "z": 3}, {"x": 42}]``. Then
   ``{ **d for d in dicts }`` would produce
   ``{"x": 42, "y": 2, "z": 3}``. The final ``"x"`` comes from the last
   dict in the iteration (overriding the ``"x": 1"`` from the first
   dict). This is exactly how ``{**d1, **d2, **d3}`` or a loop of
   updates would behave.

-  Set comprehension example: ``sets = [{1, 2}, {2, 3}]`` then
   ``{ *s for s in sets }`` yields ``{1, 2, 3}``. If the input sets have
   overlapping elements, the result set naturally deduplicates them.
   Order of iteration does not affect the final set contents.

These rules ensure that the new comprehension behavior aligns with
existing Python semantics for unpacking and for comprehensions, without
surprises. The change required in the compiler is essentially to allow
the starred expression in the grammar and to treat the comprehension
output accordingly (calling an internal extend/merge operation rather
than append for each iteration, conceptually).

Backwards Compatibility
-----------------------

This change is designed to be fully backward compatible. Currently, any
use of ``*`` or ``**`` in a comprehension's output expression is a
syntax error, so no valid Python code will be affected by lifting the
restriction. Code written with the new syntax will of course not run on
older Python versions (it will produce a syntax error on Python 3.13 and
below), but that is expected for any new language feature.

One subtle point is that comprehensions with ``*``/``**`` will change
the internal implementation of how results are accumulated (using an
extend/merge operation instead of append). This has no user-visible
impact except the resulting output, which is exactly what the user
intends. All other behavior (such as the scope of variables,
short-circuiting on exceptions, etc.) remains unchanged.

No existing APIs or semantics are altered by this proposal aside from
the syntax. The ``SyntaxError`` message “iterable unpacking cannot be
used in comprehension" will no longer be emitted in the situations that
become valid. There is an extremely low risk of any code depending on
that specific error. In summary, if code doesn't use the new syntax, it
behaves exactly as before.

Alternatives
------------

Throughout the discussion of this feature, several alternative
approaches and ideas have been considered:

-  **Status quo (nested comprehensions or ``itertools.chain``)**: The
   primary alternative is to continue using the techniques that
   developers use today – either double-loop comprehensions or
   ``chain.from_iterable`` (or writing manual loops). These approaches
   have the disadvantage of being less clear in intent. The nested
   comprehension syntax, while functional, can confuse readers who are
   not used to it, and it doesn't scale well to more complex situations
   (adding conditionals or additional loops can make it quite hard to
   read). The ``chain.from_iterable`` approach introduces a dependency
   on an external utility and a layer of indirection around a generator
   expression. Given that the new syntax is relatively small and easy to
   learn, the status quo feels like an inferior solution in terms of
   code clarity.

-  **A new built-in or method for flattening**: Another idea floated in
   the community is introducing a new helper to flatten iterables (for
   example, a ``list.flatten()`` method or a builtin
   ``flatten(iterable_of_iterables)``). While such a helper could be
   useful, it addresses a narrower need (flattening entire iterables)
   and doesn't generalize to partial comprehension patterns or to dict
   merging. It also doesn't integrate as seamlessly into comprehension
   syntax where additional filters or transformations might be present.
   Using ``*`` in comprehensions, by contrast, works naturally with
   filters (``if`` clauses) and with additional loops if needed (one
   could combine multiple levels of comprehension and still use a
   starred expression at the end). Moreover, the ``*`` operator approach
   is more in line with Python's philosophy of composing small concepts;
   it reuses existing syntax rather than adding a new function.

-  **Allowing multiple items yield via a different syntax**: One could
   imagine a different syntax to indicate multiple yields per iteration,
   such as allowing the comprehension to take a tuple of outputs. For
   instance, something like ``[ (a, b) for ... ]`` flattening
   automatically, or a special keyword. This was generally not favored
   because it would conflict with the current meaning (that would
   normally produce a list of tuple pairs, not flatten them). The
   unpacking operator is the established way to denote “flattening" in
   Python. Another idea mentioned in some threads was changing
   ``list.append`` to accept multiple arguments (so that a comprehension
   could conceptually append multiple items). This was quickly dismissed
   as it would be a significant semantic change to list.append (and one
   can always call ``list.extend`` explicitly). Using ``extend``
   internally via the ``*`` syntax is essentially a cleaner, more
   controlled way to get the same effect without altering core list/dict
   methods semantics.

-  **Do nothing (deferring indefinitely)**: The option remains to simply
   not introduce this feature, as was the case historically. The
   arguments for doing nothing revolve around keeping the language
   simpler and avoiding a feature that might be seen as “nice to have"
   but not necessary. However, given the frequency with which this idea
   has surfaced in discussions over the years and the number of real
   examples where it can be applied, doing nothing means continuing to
   live with less-than-ideal code patterns for flattening. The consensus
   among those in favor is that the benefit in expressiveness and
   symmetry with existing unpacking features outweighs the cost of
   slightly complicating the comprehension syntax.

In the end, the ``*``/``**`` unpacking within comprehensions is favored
because it is **minimal, expressive, and borrows an existing concept**
to solve the problem. It was even noted by previous discussions that
implementing it in CPython would likely involve *removing* a restrictive
check rather than adding new complex logic (`Mailman 3 [Python-ideas]
Unpacking in tuple/list/set/dict comprehensions - Python-ideas -
python.org <https://mail.python.org/archives/list/python-ideas@python.org/message/7G732VMDWCRMWM4PKRG6ZMUKH7SUC7SH/#:~:text=ideas%40python,%60%5B...x...%20for>`__),
which means the complexity cost is low. Given these considerations, the
proposal to enable comprehension unpacking is considered the most direct
and Pythonic solution to the problem at hand.

References
----------

-  PEP 448 – *Additional Unpacking Generalizations* (Python 3.5) (`PEP
   448 – Additional Unpacking Generalizations \|
   peps.python.org <https://peps.python.org/pep-0448/#variations#:~:text=>`__)
   (`PEP 448 – Additional Unpacking Generalizations \|
   peps.python.org <https://peps.python.org/pep-0448/#variations#:~:text=>`__)
   – (Joshua Landau, 2015). This PEP introduced the extended unpacking
   in function calls and literals, and mentioned unpacking in
   comprehensions as a potential future extension that was postponed due
   to readability concerns.

-  Erik Demaine, *“Unpacking in tuple/list/set/dict comprehensions"* –
   Python-ideas mailing list discussion (Oct 2021) (`Mailman 3
   [Python-ideas] Unpacking in tuple/list/set/dict comprehensions -
   Python-ideas -
   python.org <https://mail.python.org/archives/list/python-ideas@python.org/message/7G732VMDWCRMWM4PKRG6ZMUKH7SUC7SH/#:~:text=Extended%20unpacking%20notation%20%28,set%20with>`__)
   (`Mailman 3 [Python-ideas] Unpacking in tuple/list/set/dict
   comprehensions - Python-ideas -
   python.org <https://mail.python.org/archives/list/python-ideas@python.org/message/7G732VMDWCRMWM4PKRG6ZMUKH7SUC7SH/#:~:text=ideas%40python,x...%20for%20x%20in>`__).
   A proposal and debate on allowing ``*``/``**`` in comprehensions,
   summarizing arguments for and against the idea from a prior 2016
   thread.

-  Python Discuss thread *“Using unpacking to generalize comprehensions
   with multiple elements"* (Alex Prengère, Oct 2023) – Initial
   discussion and examples that inspired this PEP, including the
   ``flights`` example and recognition of the historical context.
