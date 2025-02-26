.. index:: ! validation, ! type system, function type, table type, memory type, global type, value type, result type, index space, instantiation. module
.. _type-system:

Conventions
-----------

Validation checks that a WebAssembly module is well-formed.
Only valid modules can be :ref:`instantiated <exec-instantiation>`.

Validity is defined by a *type system* over the :ref:`abstract syntax <syntax>` of a :ref:`module <syntax-module>` and its contents.
For each piece of abstract syntax, there is a typing rule that specifies the constraints that apply to it.
All rules are given in two *equivalent* forms:

1. In *prose*, describing the meaning in intuitive form.
2. In *formal notation*, describing the rule in mathematical form. [#cite-pldi2017]_

.. note::
   The prose and formal rules are equivalent,
   so that understanding of the formal notation is *not* required to read this specification.
   The formalism offers a more concise description in notation that is used widely in programming languages semantics and is readily amenable to mathematical proof.

In both cases, the rules are formulated in a *declarative* manner.
That is, they only formulate the constraints, they do not define an algorithm.
The skeleton of a sound and complete algorithm for type-checking instruction sequences according to this specification is provided in the :ref:`appendix <algo-valid>`.


.. index:: ! context, function type, table type, memory type, tag type, global type, value type, result type, index space, module, function, tag, label type
.. _context:
.. _syntax-labeltype:

Contexts
~~~~~~~~

Validity of an individual definition is specified relative to a *context*,
which collects relevant information about the surrounding :ref:`module <syntax-module>` and the definitions in scope:

* *Types*: the list of types defined in the current module.
* *Functions*: the list of functions declared in the current module, represented by their function type.
* *Tables*: the list of tables declared in the current module, represented by their table type.
* *Memories*: the list of memories declared in the current module, represented by their memory type.
* *Tags*: the list of tags declared in the current module, represented by their tag type.
* *Globals*: the list of globals declared in the current module, represented by their global type.
* *Element Segments*: the list of element segments declared in the current module, represented by their element type.
* *Data Segments*: the list of data segments declared in the current module, each represented by an |ok| entry.
* *Locals*: the list of locals declared in the current function (including parameters), represented by their value type.
* *Labels*: the stack of labels accessible from the current position, represented by their |labeltype|, which is a result type, possibly prepended by a |LCATCH| entry, if the label is surrounding the instructions inside a |CATCH| or |CATCHALL|.
* *Return*: the return type of the current function, represented as an optional result type that is absent when no return is allowed, as in free-standing expressions.
* *References*: the list of :ref:`function indices <syntax-funcidx>` that occur in the module outside functions and can hence be used to form references inside them.

In other words, a context contains a sequence of suitable :ref:`types <syntax-type>` for each :ref:`index space <syntax-index>`,
describing each defined entry in that space.
Locals, labels and return type are only used for validating :ref:`instructions <syntax-instr>` in :ref:`function bodies <syntax-func>`, and are left empty elsewhere.
The label stack is the only part of the context that changes as validation of an instruction sequence proceeds.

More concretely, contexts are defined as :ref:`records <notation-record>` :math:`C` with abstract syntax:

.. math::
   \begin{array}{llll}
   \production{(labeltype)} & \labeltype & ::= & \LCATCH^?~\resulttype\\
   \production{(context)} & C &::=&
     \begin{array}[t]{l@{~}ll}
     \{ & \CTYPES & \functype^\ast, \\
        & \CFUNCS & \functype^\ast, \\
        & \CTABLES & \tabletype^\ast, \\
        & \CMEMS & \memtype^\ast, \\
	& \CTAGS & \tagtype^\ast, \\
        & \CGLOBALS & \globaltype^\ast, \\
        & \CELEMS & \reftype^\ast, \\
        & \CDATAS & {\ok}^\ast, \\
        & \CLOCALS & \valtype^\ast, \\
        & \CLABELS & \labeltype^\ast, \\
        & \CRETURN & \resulttype^?, \\
        & \CREFS & \funcidx^\ast ~\} \\
     \end{array}
   \end{array}

.. _notation-extend:

In addition to field access written :math:`C.\K{field}` the following notation is adopted for manipulating contexts:

* When spelling out a context, empty fields are omitted.

* :math:`C,\K{field}\,A^\ast` denotes the same context as :math:`C` but with the elements :math:`A^\ast` prepended to its :math:`\K{field}` component sequence.

.. note::
   We use :ref:`indexing notation <notation-index>` like :math:`C.\CLABELS[i]` to look up indices in their respective :ref:`index space <syntax-index>` in the context.
   Context extension notation :math:`C,\K{field}\,A` is primarily used to locally extend *relative* index spaces, such as :ref:`label indices <syntax-labelidx>`.
   Accordingly, the notation is defined to append at the *front* of the respective sequence, introducing a new relative index :math:`0` and shifting the existing ones.


.. _valid-notation-textual:

Prose Notation
~~~~~~~~~~~~~~

Validation is specified by stylised rules for each relevant part of the :ref:`abstract syntax <syntax>`.
The rules not only state constraints defining when a phrase is valid,
they also classify it with a type.
The following conventions are adopted in stating these rules.

* A phrase :math:`A` is said to be "valid with type :math:`T`"
  if and only if all constraints expressed by the respective rules are met.
  The form of :math:`T` depends on what :math:`A` is.

  .. note::
     For example, if :math:`A` is a :ref:`function <syntax-func>`,
     then  :math:`T` is a :ref:`function type <syntax-functype>`;
     for an :math:`A` that is a :ref:`global <syntax-global>`,
     :math:`T` is a :ref:`global type <syntax-globaltype>`;
     and so on.

* The rules implicitly assume a given :ref:`context <context>` :math:`C`.

* In some places, this context is locally extended to a context :math:`C'` with additional entries.
  The formulation "Under context :math:`C'`, ... *statement* ..." is adopted to express that the following statement must apply under the assumptions embodied in the extended context.


.. index:: ! typing rules
.. _valid-notation:

Formal Notation
~~~~~~~~~~~~~~~

.. note::
   This section gives a brief explanation of the notation for specifying typing rules formally.
   For the interested reader, a more thorough introduction can be found in respective text books. [#cite-tapl]_

The proposition that a phrase :math:`A` has a respective type :math:`T` is written :math:`A : T`.
In general, however, typing is dependent on a context :math:`C`.
To express this explicitly, the complete form is a *judgement* :math:`C \vdash A : T`,
which says that :math:`A : T` holds under the assumptions encoded in :math:`C`.

The formal typing rules use a standard approach for specifying type systems, rendering them into *deduction rules*.
Every rule has the following general form:

.. math::
   \frac{
     \X{premise}_1 \qquad \X{premise}_2 \qquad \dots \qquad \X{premise}_n
   }{
     \X{conclusion}
   }

Such a rule is read as a big implication: if all premises hold, then the conclusion holds.
Some rules have no premises; they are *axioms* whose conclusion holds unconditionally.
The conclusion always is a judgment :math:`C \vdash A : T`,
and there is one respective rule for each relevant construct :math:`A` of the abstract syntax.

.. note::
   For example, the typing rule for the :math:`\I32.\ADD` instruction can be given as an axiom:

   .. math::
      \frac{
      }{
        C \vdash \I32.\ADD : [\I32~\I32] \to [\I32]
      }

   The instruction is always valid with type :math:`[\I32~\I32] \to [\I32`]
   (saying that it consumes two |I32| values and produces one),
   independent of any side conditions.

   An instruction like |LOCALGET| can be typed as follows:

   .. math::
      \frac{
        C.\CLOCALS[x] = t
      }{
        C \vdash \LOCALGET~x : [] \to [t]
      }

   Here, the premise enforces that the immediate :ref:`local index <syntax-localidx>` :math:`x` exists in the context.
   The instruction produces a value of its respective type :math:`t`
   (and does not consume any values).
   If :math:`C.\CLOCALS[x]` does not exist then the premise does not hold,
   and the instruction is ill-typed.

   Finally, a :ref:`structured <syntax-instr-control>` instruction requires
   a recursive rule, where the premise is itself a typing judgement:

   .. math::
      \frac{
        C \vdash \blocktype : [t_1^\ast] \to [t_2^\ast]
        \qquad
        C,\LABEL\,[t_2^\ast] \vdash \instr^\ast : [t_1^\ast] \to [t_2^\ast]
      }{
        C \vdash \BLOCK~\blocktype~\instr^\ast~\END : [t_1^\ast] \to [t_2^\ast]
      }

   A |BLOCK| instruction is only valid when the instruction sequence in its body is.
   Moreover, the result type must match the block's annotation :math:`\blocktype`.
   If so, then the |BLOCK| instruction has the same type as the body.
   Inside the body an additional label of the corresponding result type is available,
   which is expressed by extending the context :math:`C` with the additional label information for the premise.


.. [#cite-pldi2017]
   The semantics is derived from the following article:
   Andreas Haas, Andreas Rossberg, Derek Schuff, Ben Titzer, Dan Gohman, Luke Wagner, Alon Zakai, JF Bastien, Michael Holman. |PLDI2017|_. Proceedings of the 38th ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI 2017). ACM 2017.

.. [#cite-tapl]
   For example: Benjamin Pierce. |TAPL|_. The MIT Press 2002
