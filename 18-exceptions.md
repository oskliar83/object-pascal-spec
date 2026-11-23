# 18 — Exceptions

Structured exception handling: `try…except`, `try…finally`, `raise`, exception
filters, re-raising, and nested/inner exceptions.

Shared productions (`StatementList`, `Expression`, `Block`) → [Appendix
B](B-lexical-grammar.md). These are the `TryStmt`/`RaiseStmt` deferred from
[ch.05](05-statements.md).

## Chapter grammar umbrella

```ebnf
TryStmt     = "try" StatementList ( ExceptPart | FinallyPart ) "end" ;
FinallyPart = "finally" StatementList ;
ExceptPart  = "except" ( StatementList
                       | HandlerList [ "else" StatementList ]
                       | "else" StatementList ) ;
HandlerList = ExceptionHandler { ";" ExceptionHandler } [ ";" ] ;
ExceptionHandler = "on" [ Ident ":" ] TypeRef "do" Statement ;
RaiseStmt   = "raise" [ Expression [ "at" Expression ] ] ;
```

The `;` after the last handler is optional, before `end` and before `else`
alike; between two handlers it is required (`on A do X on B do Y` is
`E2066 Missing operator or semicolon`), and it is a single `;`, not a
statement separator around empty statements (`on E: Exception do ; ; end` is
`E2029 'END' expected but ';' found`). dcc64 37.0, 2026-10-30 (spec audit
g6, local/probe-park/spec-audit/g6 X01.dpr, X08.dpr, X10.dpr, X11.dpr).
After that E2066 dcc's recovery reads on in its own way, and the errors it
adds are no rule: `on E: Exception do G := 1 on E: EAbort do G := 2; end;`
gives `E2010 Incompatible types: 'Exception' and 'class of EAbort'` (the
second handler's `E: EAbort` read as a value going to E) and `E2029 '.'
expected but ';' found` (the `try`'s `end` taken for the block's); without
the second `E:` only the E2029 follows. A checker reports the E2066 and need
not imitate the rest (dcc64 37.0, 2026-11-22, PasTree
local/probe-park/sa5-exprs x2).

---

## 18.1 `try…except`

### 18.1.1 The `try…except` block

| | |
|---|---|
| **Introduced** | Delphi 1 (1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Executes guarded statements; if an exception is raised, control transfers to the
`except` part.

**Example**

```pascal
try
  DoRisky;
except
  HandleAnything;     // catch-all form
end;
```

**Semantics & parsing notes**

- ⚠️ *Two `except` shapes:* either a bare `StatementList` (catch-all) **or** a list
  of `on … do` handlers (optionally with `else`). The parser must decide by
  looking for the `on` keyword as the first token of the `except` body.
- ⚠️ *The handler list may be empty before its `else`:* `except else S end`
  compiles and runs S for any exception - the catch-all again, written as an
  `else` with no handler before it (`except else end` too). A catch-all
  statement list cannot take an `else`: `except S; else T end` is `E2153 ';'
  not allowed before 'ELSE'`, also when S is an `if` without `else`, and
  `except S else T end` is `E2029 'END' expected but 'ELSE' found` (dcc64
  37.0, probed 2026-10-10).
- ⚠️ `try` blocks **cannot** have both `except` and `finally` directly — nest them
  (`try try … finally … end except … end`). Reject a single `try` with both parts.
  dcc reads the statement list after `except` up to `end` and fails at the
  `finally`: `try S except T finally U end` is `E2029 'END' expected but
  'FINALLY' found` (dcc64 37.0, 2026-10-30, spec audit g6, X09.dpr).
  ⚠️ The same rule holds for any statement list (dcc64 37.0, 2026-11-20,
  PasTree local/probe-park/sa-parser T1-T8): a `finally`, `except` or
  `until` that ends no construct of the list is `E2029 '<word>' expected
  but '<found>' found` at that word, `<word>` being what the list's own
  construct ends with - `END` after an `except` or `finally` part (an `on`
  handler list too) and inside a `begin`, `UNTIL` inside a `repeat`
  (5.5.4). `try S finally T except U end`, `try S except T except U end`,
  `try S except on E: X do T; finally U end` and `begin S; except T; end`
  are 'END' expected at the second word; `begin S; until C; end` is 'END'
  expected but 'UNTIL'. What dcc reports after it is a cascade; an
  error-tolerant parser reads on past the word, and the statements after it
  keep their nodes.
- ⚠️ *Only the FIRST token decides the shape:* `except ; on E: Exception do
  ... end` is a catch-all statement list whose second statement starts with
  the identifier `on` - `E2003 Undeclared identifier: 'on'` (dcc64 37.0,
  2026-10-30, spec audit g6, local/probe-park/spec-audit/g6 X08.dpr).
  The `E2029 'END' expected but 'DO' found` dcc adds after it is a cascade
  of its recovery, not a rule of the handler syntax: past the error at `on`
  (E2003, or E2066 at `E` when `on` is declared) dcc restarts a statement
  at `E` and reads `E:` as a LABEL - `G := 1 L: G := 2` with `L` used as a
  label later is E2066 then `E2073 Label already defined: 'L'` - with no
  E2003 for an undeclared `E` (alone, `E: G := 2` is `E2003 Undeclared
  identifier: 'E'`). What it reports at the `do` then depends on the
  statement it read after the label: `on E: Exception do` and `G E:
  Exception do` give 'END' expected at the `do`, `G := G E: G := 2 do`
  gives none there, `on E do` and `on E: Exception;` nothing past the
  first error. An error-tolerant parser reports the first error and reads
  on from the next `;`. dcc64 37.0, 2026-11-22 (PasTree
  local/probe-park/sa5-cascade H1-H20).
- *AST:* `TryExceptStmt { body, handlers[], elseBody?, catchAll? }`.

### 18.1.2 `on E: EType do` exception filters

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Typed handlers selected by the exception's class; an optional identifier binds
the exception instance.

**Example**

```pascal
try
  Parse;
except
  on E: EConvertError do ShowMessage(E.Message);
  on E: EAccessViolation do Log(E);
else
  raise;              // unmatched: re-raise
end;
```

**Semantics & parsing notes**

- ⚠️ *Handler order matters:* handlers are tried top-to-bottom; a base-class handler
  placed before a derived one shadows it. The resolver/analyzer may warn; semantics
  follow source order. dcc itself gives no diagnostic: `on E: Exception do A;
  on E: EAbort do B;` compiles clean and runs A for an EAbort (dcc64 37.0,
  2026-10-30, spec audit g6, X04.dpr).
- ⚠️ *The handler type must be a class type:* `on E: Integer`, an interface
  (`on E: IFoo`), a record and a class reference (`on E: TClass`) are each
  `E2021 Class type required`; any class is accepted, `TObject` included, and
  the `TypeRef` may be unit-qualified (`on E: System.SysUtils.Exception`) or a
  generic instance (`on E: EGen<Integer>`). dcc64 37.0, 2026-10-30 (spec
  audit g6, local/probe-park/spec-audit/g6 X01.dpr, X08.dpr). ⚠️ The
  generic instance is checked, not compiled: X08 stops at a syntax error
  before code generation, and a routine with `try except on X: EGen<Integer>
  do ; end` alone stops dcc64 37.0 with `F2084 Internal Error: G16451` (with
  a `raise EGen<Integer>.Create` after it, `AV00D4F404`) - code that
  compiles never has one (2026-11-21, PasTree's local/probe-park/sa4-flow
  R2, R3).
- The bound identifier `E` is scoped to that handler's `Statement`; the instance is
  **freed automatically** when the handler exits (do **not** `Free` it).
  `E` is an ordinary assignable variable: `E := nil` compiles (hint `H2077
  Value assigned to 'E' never used`) and changes nothing the RTL does - a
  bare `raise` after it still re-raises the original exception (dcc64 37.0,
  2026-10-30, spec audit g6, X04.dpr).
- `else` (no type) catches anything unmatched.
- ⚠️ *except-`else` vs `if`-`else`:* an `else` right after a handler whose
  statement is an `if` without `else` is that `if`'s (5.3.1) -
  `on E: EAbort do if A then B else C` runs C for an EAbort when A is False
  and lets any other exception propagate; the except part gets the `else` only
  after a `;` or after the `if`'s own `else` branch (dcc64 37.0, probed
  2026-09-26). After any other statement no `;` is needed: `on E: Exception
  do G := 1 else G := 2; end` is a handler and the except part's `else`
  (dcc64 37.0, 2026-10-30, spec audit g6, X08.dpr).
- *AST:* `ExceptOn { varName?, excType, body }`.

---

## 18.2 `try…finally`

### 18.2.1 The `try…finally` block

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

The `finally` part **always** runs — on normal completion, on exception, or on
early `Exit`/`Break` — for guaranteed cleanup.

**Example**

```pascal
Obj := TThing.Create;
try
  Obj.Use;
finally
  Obj.Free;           // runs no matter what
end;
```

**Semantics & parsing notes**

- ⚠️ `finally` does **not** swallow the exception — it runs, then the exception
  continues propagating unless the block itself raises/exits. Distinguish from
  `except`.
- ⚠️ *No `Exit`, `Break` or `Continue` may leave a `finally` part:* each is
  `E2126 Cannot BREAK, CONTINUE or EXIT out of a FINALLY clause`, also when
  conditional (`finally if C then Exit; end`) and also from a `try` nested
  inside the `finally` part (`finally try Exit; finally end; end`, `finally
  try Exit; except end; end`). Allowed: a `Break` of a loop that lies wholly
  inside the `finally` part, an `Exit` in an anonymous method written there
  or in a nested routine called from there, and all three in an `except`
  part. ⚠️ A `try` whose
  guarded part is empty (`try finally Exit; end`, `try ; finally ...`, `try
  begin end; finally ...`) is NOT checked - dcc compiles it clean, so the rule
  needs a non-empty guarded statement list. dcc stops at the first E2126 in a
  compilation (one is reported however many there are). dcc64 37.0,
  2026-10-30 (spec audit g6, local/probe-park/spec-audit/g6 F1-FI.dpr,
  X06.dpr).
  "Leaving" is lexical: the search for the loop (Break, Continue) or the
  routine (Exit) passes the `finally` part; it ends at a routine or an
  anonymous method. Without a loop at all a `Break` there is E2097 instead
  (5.6). dcc64 37.0, 2026-11-21 (PasTree's local/probe-park/sa4-flow P2).
- `goto` into or out of a `try` statement is E2127, reported at the label
  (see 5.6.4; dcc64 37.0, 2026-10-30, spec audit g6, X05.dpr): `E2127 'GOTO
  L1' leads into or out of TRY statement, FOR-IN statement whose enumerator
  has destructor, or a block containing local variable` - the goto named,
  at the label's line. "Into or out of" is per PART: a goto and its label in
  the same guarded part (`try goto L3; L3: ; finally end`) compile; a goto
  in the guarded part to a label after the statement, or one in front of
  the try to a label in its guarded part, do not.
- *AST:* `TryFinallyStmt { body, finallyBody }`.

---

## 18.3 `raise`

### 18.3.1 Raising and re-raising

| | |
|---|---|
| **Introduced** | Delphi 1; `RaiseOuterException`/inner 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`raise E` throws an exception object; a **bare** `raise` inside a handler
re-raises the current exception.

**Grammar**

```ebnf
RaiseStmt = "raise" [ Expression [ "at" Expression ] ] ;
```

**Example**

```pascal
raise EMyError.Create('boom');     // raise new
// inside an except handler:
raise;                              // re-raise current (preserves stack/origin)
raise E at ReturnAddress;           // raise at a specific address
```

**Semantics & parsing notes**

- ⚠️ *Bare `raise`* is only valid **inside an exception handler** — it re-throws the
  in-flight exception, preserving its original stack trace. Outside a handler it is
  an error: `E2145 Re-raising an exception only allowed in exception handler`.
- ⚠️ *What "inside a handler" means* is **lexical**, and the deciding scope is the
  **nearest enclosing part of a `try` statement**, not any enclosing one. A `try`
  block or a `finally` part therefore *resets* the context an outer handler
  established (dcc32 37.0):

  ```pascal
  try except try finally raise; end; end;   // error — nearest part is `finally`
  try finally try except raise; end; end;   // legal — nearest part is `except`
  try except try raise; except end; end;    // error — nearest part is a try block
  try try except raise; end; finally end;   // legal
  ```

  Both the `on … do` bodies and the `else` branch of an `except` part count as
  handler context. A **plain anonymous method body does not reset it**: a
  bare `raise` written inside a `procedure begin ... end` in a handler is
  accepted, and so is one in a literal with `var`, `const` or `type`
  sections. A **named routine always resets it**: a routine's `raise` is an
  error even when the routine is called from a handler, and so is one in a
  routine declared in an anonymous method's declarations, although that
  routine stands lexically inside the handler.
- ⚠️ *An anonymous method that declares a routine resets the context for its
  whole body.* Written in a handler, such a literal's own `raise` is
  `E2145`, and so is one in a literal written inside it; a plain literal
  around it keeps its own context. A `try ... except` inside it gives the
  context back, as anywhere:

  ```pascal
  try except G := procedure begin raise; end; end;                 // legal
  try except G := procedure var V: Integer; begin raise; end; end; // legal
  try except G := procedure
    procedure N; begin raise; end;                                 // E2145
  begin N; raise; end; end;                                        // E2145
  try except G := procedure
    procedure N; begin end;
  begin G := procedure begin raise; end; end; end;                 // E2145
  try except G := procedure
    procedure N; begin end;
  begin try N; except raise; end; end; end;                        // legal
  ```

  *Evidence:* dcc64 37.0, 2026-10-29, PasTree's local/probe-m-b/b418
  pa.dpr: E2145 on A (the nested routine), B (the literal's own body), I (a
  literal inside it) and K (a nested function); none on C (`var`), D
  (plain), E (`const` and `type`), F and H (a `try ... except` inside) and J
  (a plain literal around one that declares a routine).
- `raise` **transfers ownership** of the exception object to the RTL — do not free
  the raised instance.
- ⚠️ *The raised expression must be a class INSTANCE:* `raise 5`, an
  interface, a record, a class reference (the common slip `raise Exception;`)
  and `raise nil` are each `E2021 Class type required`; any object is
  accepted, a `TObject` variable or `raise TObject.Create` included (18.4.1).
- `at Address` overrides the reported raise location (diagnostics). The
  address must be of a pointer type (`at 5` is `E2017 Pointer type
  required`; `at P`, `at @P`, `at ReturnAddress` compile). `at` is a clause
  word only after an exception expression: in `raise at P;` it is read as the
  operand, an identifier (`E2003 Undeclared identifier: 'at'`). dcc64 37.0,
  2026-10-30 (spec audit g6, local/probe-park/spec-audit/g6 X02.dpr, X12.dpr,
  X07.dpr). `P` is then a stray operand - `E2066 Missing operator or
  semicolon` (chapter 05's missing-separator rule) where `at` names a
  variable, hidden behind the E2003 where it names nothing; with `at`
  declared, `raise at at P` compiles (dcc64 37.0, 2026-11-20,
  local/probe-park/sa2-stmt R1-R3).
- *AST:* `RaiseStmt { exception?, atAddr? }` (no `exception` ⇒ re-raise).

---

## 18.4 Exception hierarchy

### 18.4.1 The `Exception` base class

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

All catchable exceptions derive from `Exception` (RTL); `on E: Exception` catches
the broadest meaningful set.

**Semantics & parsing notes**

- A language-contract class (like `TObject`): the `on` filter type is normally an
  `Exception` descendant. Non-`Exception` objects can technically be raised but are
  outside normal handling: `raise TObject.Create` compiles, is not caught by
  `on E: Exception`, and is caught by `on E: TObject` and by the except
  part's `else` (dcc64 37.0, 2026-10-30, spec audit g6,
  local/probe-park/spec-audit/g6 X02.dpr, X04.dpr).
- *Not parsing* — informs the type-checker's expectations for `on … do` types.

---

## 18.5 Nested & inner exceptions

### 18.5.1 Inner-exception chaining

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Raising a new exception while handling another preserves the original as the
**inner exception** (`Exception.InnerException`), via `Exception.RaiseOuterException`
/ `ThrowOuterException`.

**Example**

```pascal
try
  LowLevel;
except
  Exception.RaiseOuterException(EHighLevel.Create('wrapped'));
end;
```

**Semantics & parsing notes**

- This is an RTL mechanism (method calls), not new syntax — included because it
  changes the *semantics* of nested raises (chaining vs. replacing). No parser
  impact beyond ordinary calls.
- The compiler tracks the "current exception" per thread so nested `try` blocks and
  bare `raise` behave correctly.
- ⚠️ *`BaseException` is the FIRST (innermost) exception of the chain, distinct from
  `InnerException` once the chain is more than one level deep* — `InnerException`
  is only the immediately-preceding exception. With a single nesting they happen to
  be the same object; with two or more they diverge (dcc-verified, dcc32 37.0):

  ```pascal
  try
    raise Exception.Create('Hello');
  except
    try
      Exception.RaiseOuterException(Exception.Create('Another'));
    except
      Exception.RaiseOuterException(Exception.Create('A third'));
    end;
  end;
  // caught E: E.Message           = 'A third'
  //          E.InnerException.Message  = 'Another'   (one level up)
  //          E.BaseException.Message   = 'Hello'      (first/innermost)
  ```
- ⚠️ *`Exception.ToString` concatenates the whole chain's messages*, one per line
  (`sLineBreak`-joined, outermost first), rather than just the current exception's
  `Message` — for the chain above, `E.ToString` is `'A third'` + LF + `'Another'` +
  LF + `'Hello'` (dcc-verified). With no chaining, `ToString` equals `Message`.
- *AST:* not new syntax — `BaseException`/`ToString` are RTL members reached
  through ordinary member access on an `Exception`-typed expression.

### 18.5.2 Intercepting a raise: `Exception.RaisingException`

| | |
|---|---|
| **Introduced** | Delphi 2009 (alongside inner-exception chaining) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`Exception.RaisingException(P: PExceptionRecord)` is a `virtual` hook invoked on the
exception object right after it is constructed but **before** the `raise` actually
transfers control — overriding it gives a single post-creation function that runs
regardless of which constructor was used.

**Example**

```pascal
type
  ECustomException = class(Exception)
  protected
    procedure RaisingException(P: PExceptionRecord); override;
  end;

procedure ECustomException.RaisingException(P: PExceptionRecord);
begin
  Log(Message);          // fires before any `except` handler runs
  inherited;              // required: base impl wires up SetInnerException
end;
```

**Semantics & parsing notes**

- ⚠️ *Fires before the handler, not at construction* — the marker code in an
  overridden `RaisingException` runs strictly between `raise`'s object-construction
  step and the stack search for a handler; ordinary `Create` does not trigger it
  (dcc-verified: overriding `RaisingException` and calling `raise
  ECustomException.Create(...)` prints the override's output before the enclosing
  `except` block's own code runs).
- The base-class implementation calls the internal `SetInnerException` — an
  override that skips `inherited` breaks inner-exception chaining (18.5.1) for that
  exception type.
- Not new syntax — an ordinary virtual method override; included here because it
  changes exception-raising *semantics* (a guaranteed post-construction/pre-raise
  hook point), not because it affects parsing.

---

## 18.6 Exceptions during construction & destruction

### 18.6.1 A failing `Create` still runs `Destroy` (and the pseudo-hooks)

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

If a constructor raises before returning, the runtime immediately invokes the
destructor on the **partially-initialized** object before the exception continues
propagating — the object is never silently discarded without cleanup. The same
non-resumption rule extends to `AfterConstruction`/`BeforeDestruction`: if
`AfterConstruction` raises, `BeforeDestruction` and the regular destructor still run.

**Example**

```pascal
type
  TObjectWithList = class
  private
    FStringList: TStringList;
  public
    constructor Create(Value: Integer);
    destructor Destroy; override;
  end;

constructor TObjectWithList.Create(Value: Integer);
begin
  if Value < 0 then
    raise Exception.Create('Negative value not allowed');  // FStringList never set
  FStringList := TStringList.Create;
end;

destructor TObjectWithList.Destroy;
begin
  if Assigned(FStringList) then      // ⚠️ mandatory guard — see notes
    FreeAndNil(FStringList);
  inherited;
end;
```

**Semantics & parsing notes**

- ⚠️ *A destructor must never assume the constructor ran to completion.* Every
  sub-object/resource field a destructor touches needs an `Assigned` (or
  equivalent) guard, because `Destroy` runs even when `Create` raised partway
  through — the destructor here can be reached with `FStringList` still `nil`
  (dcc-verified, dcc32 37.0: a constructor that sets one field, raises, then
  would have set a second field — the destructor is still entered and observes
  the first field set and the second unset).
- The caller never calls `Free` on an object whose `Create` raised: `raise`
  already triggered `Destroy` for it, and the assignment (`Obj := TFoo.Create`)
  never completes, so there is no reachable reference to free again. This is why
  the standard idiom creates the object *outside* a `try…finally` — construction
  is self-protecting; only post-construction use needs the `finally Obj.Free`.
- ⚠️ *Extends to the pseudo-hooks:* if an override of `AfterConstruction` raises,
  `BeforeDestruction` and `Destroy` both still run on the object (dcc-verified:
  overriding `AfterConstruction` to raise, with `BeforeDestruction` overridden to
  emit a marker — the marker fires, and the exception then propagates out of
  `Create` normally). `AfterConstruction`/`BeforeDestruction` are TObject's
  C++-compatibility pseudo-constructor/pseudo-destructor pair; this is not new
  syntax, but the same lifecycle guarantee applies to them as to `Create`/`Destroy`.
- This is fundamentally an exception-propagation rule rather than an ordinary
  control-flow one: the destructor call is *inserted* by the runtime as part of
  unwinding a failed construction, not written at the call site. See
  [ch.11](11-classes.md) for the general `Create`/`Destroy`/`AfterConstruction`/
  `BeforeDestruction` lifecycle model.
- *AST:* not new syntax — no parser impact; this is a runtime/codegen guarantee
  attached to every constructor call.
