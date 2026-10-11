# Decorator-Aware Class Members Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Delegate only if the user requests it. Commit steps require explicit user authorization.

**Goal:** Document properties once as attributes and distinguish static/class methods while hiding the implicit class-method receiver.

**Architecture:** Classify class-body functions in the AST parser, store properties separately from methods, and use the same extraction in both parsing passes. Extend the existing dictionary contract and class renderer, retaining renderer-specific escaping and source links. Carry property metadata through the existing direct-base inheritance path.

**Tech Stack:** Python >=3.10, standard-library `ast`, existing Griffe docstring formatting, pytest, uv; no new dependencies.

**Spec:** [GitHub issue #51](https://github.com/zzstoatzz/mdxify/issues/51), with the output contract below. Approved for implementation by the user's request to move forward with this plan.

## Global Constraints

- Preserve Python `>=3.10` and add no dependencies.
- Parse source statically; do not import or execute a module to identify decorators.
- Apply existing `include_internal` filtering to properties as well as methods.
- Keep the existing `extract_methods_from_class` entry point and legacy dictionaries usable.
- Use `uv run` for scripts and tests; confirm `pwd` before commands.
- Run the full test suite: `uv run pytest -xvs`.
- For output changes, validate against a real project, with modules importable from the command's working directory.
- Issue #50's overload handling, arbitrary descriptors, cached properties, import-alias resolution, and a general inheritance/MRO rewrite are separate work.

## Output and Data Contract

The section names and labels below were proposed during planning and retained for implementation.

1. Emit `**Properties:**` after the class docstring and before `**Methods:**`; omit empty sections. Keep class-docstring `Attributes:` content intact rather than trying to merge prose with parsed members.
2. Use a single fourth-level heading for `name` and Python code fence containing `name: str`, not `name(self) -> str`. With no getter annotation, render just `name`; do not infer a type from the setter or invent `Any`.
3. Follow the fence with `*Read-only.*` or `*Read/write.*`, based on setter presence. If there is a deleter, use `*Read-only; deletable.*` or `*Read/write; deletable.*`.
4. Use the getter's docstring and source line for the member. Retain nonempty accessor docstrings under `**Setter:**` and `**Deleter:**` within that same entry; format and escape each through the existing pipeline. An explicit `.getter` replacement supplies the final getter's metadata.
5. Methods carry `kind`, one of `instance`, `static`, `class`. Render `*Static method.*` or `*Class method.*` between heading and fence; instance methods keep their current presentation. Static-method signatures retain every argument. Class-method signatures drop exactly the first positional receiver, regardless of its name, without mutating the AST.
6. Recognize bare built-in decorator names and exact `builtins.property`, `builtins.staticmethod`, `builtins.classmethod`. Recognize `@name.getter`, `@name.setter`, and `@name.deleter` for an established same-name property in that class. Other decorators remain opaque; an unrelated `.setter` is not sufficient to create a property. Stacking with `@abstractmethod` must not hide a recognized property or method kind.
7. Aggregate local accessors by property name in source order; ordinary repeated definitions retain the parser's existing behavior. Support sync and async function nodes consistently with existing parsing.
8. Add a `properties` list to parsed class dictionaries. Each property has `name: str`, `annotation: str | None`, `docstring: str`, `line: int`, `has_setter: bool`, `has_deleter: bool`, `setter_docstring: str`, `deleter_docstring: str`, and `is_inherited: bool`. Inherited copies also carry `inherited_from: str` and `source_file: str` when available. Methods retain their existing keys and add `kind`.
9. Preserve current direct-base discovery and first-base-wins ordering. Include inherited properties, retain method kinds, and suppress inherited entries shadowed by locally defined functions/properties (including private or undocumented definitions). A local method can shadow a base property and vice versa. Resolve no new transitive inheritance or import aliases in this feature.
10. Legacy class dictionaries without `properties` render as before; method dictionaries without `kind` are treated as instance methods.

For the issue's `Thing` example, the relevant output becomes:

````markdown
**Properties:**

#### `name`

```python
name: str
```

*Read/write.*

The name.

**Setter:**

Set the name.

**Methods:**

#### `sm`

*Static method.*

```python
sm(x: int) -> int
```

Static.

#### `cm`

*Class method.*

```python
cm() -> 'Thing'
```

Class method.
````

## Review Focus

- Missing getter annotation and setter/deleter combinations must not produce fake types or extra method entries (Task 1 and Task 3).
- Qualified/stacked decorators and unrelated `.setter` decorators must not silently change member kind (Task 1).
- Class-method receivers with unusual names, defaults, positional-only arguments, or keyword-only arguments must yield accurate caller signatures without AST mutation (Task 1).
- Inherited properties and cross-kind local overrides must not produce duplicate headings or wrong source links (Task 2 and Task 3).
- Legacy dictionaries, Markdown output, and MDX-sensitive accessor prose must retain working rendering (Task 3).

---

## File Map

| File | Responsibility |
| --- | --- |
| `src/mdxify/parser.py` | Classify/extract local members, format class-method signatures, propagate properties through inheritance. |
| `src/mdxify/generator.py` | Render property entries and method-kind labels using existing renderers. |
| `tests/test_parser.py` | AST metadata, signatures, filtering, inheritance and override regressions. |
| `tests/test_generator.py` | Exact output, formatting, renderer compatibility and source links. |
| `tests/test_end_to_end.py` | Issue reproduction through CLI, including inheritance and Markdown modes. |
| `README.md` | Describe the new property presentation and method kinds beside existing usage/features. |

No changes to the `Renderer` interface are needed: existing headings, fences, `escape`, and `header_with_source` are sufficient.

### Task 1: Extract Decorator-Aware Local Members

**Files:** Modify `src/mdxify/parser.py` (`extract_function_signature`, `extract_methods_from_class`, `parse_module_fast`); test `tests/test_parser.py`.

**Interfaces:**
- Add `extract_class_members(node: ast.ClassDef, include_internal: bool = False) -> tuple[list[dict[str, Any]], list[dict[str, Any]]]`, returning `(methods, properties)` with the contract above.
- Keep `extract_methods_from_class(node: ast.ClassDef, include_internal: bool = False) -> list[dict[str, Any]]` as a wrapper returning only the methods.
- Extend `extract_function_signature(node: ast.FunctionDef | ast.AsyncFunctionDef, *, skip_first_arg: bool = False) -> str`. The default retains existing tested signatures. Add correct positional-only and keyword-only formatting to support the required class-method cases.
- `parse_module_fast` stores both lists on each local class, without changing its public signature.

- [x] **Step 1: Add failing extraction tests.** Use the issue's complete `Thing` source as a fixture written with `tmp_path` and `dedent`, then parse it. Pin the central contract with these assertions:

  ```python
  cls = parse_module_fast("mre", module_file)["classes"][0]
  assert [m["name"] for m in cls["methods"]] == ["sm", "cm"]
  assert [m["kind"] for m in cls["methods"]] == ["static", "class"]
  assert cls["methods"][0]["signature"] == "sm(x: int) -> int"
  assert cls["methods"][1]["signature"] == "cm() -> 'Thing'"
  assert len(cls["properties"]) == 1
  prop = cls["properties"][0]
  assert prop["name"] == "name"
  assert prop["annotation"] == "str"
  assert prop["docstring"] == "The name."
  assert prop["setter_docstring"] == "Set the name."
  assert prop["has_setter"] is True
  assert prop["has_deleter"] is False
  ```

  Add `test_property_accessor_variants` for read-only, deleter-only, setter+deleter, no annotation, getter replacement, and a private property with both `include_internal` values. Assert one property per name, exact flags, getter line and docstring, and zero accessor entries in `methods`.

  Add `test_decorator_classification` covering exact qualified built-ins, `@abstractmethod` stacking, async class/static methods, plain instance methods, and an unrelated `@other.setter`. Assert exact kinds; unknown decorators preserve plain-method treatment.

  Add `test_classmethod_signature_receiver` with `def build(receiver, /, x: int = 1, *, flag: bool = False) -> str` and assert `build(x: int = 1, *, flag: bool = False) -> str`. Include a second positional-only argument, receiver defaults, variadics, and a static function whose first parameter is literally `cls` (it must stay). Assert `ast.dump(node)` is identical before and after signature extraction. Keep existing top-level signature assertions passing.

- [x] **Step 2: Verify the tests fail for the missing member contract.** Run `uv run pytest tests/test_parser.py -xvs`; expect a failure on property exclusion, missing `kind`/`properties`, or the missing signature option, not fixture syntax.
- [x] **Step 3: Implement the interfaces.** Classify by exact AST decorator shape; aggregate only known local property accessors. Use `ast.unparse` for getter annotations. Build signatures from argument groups without modifying nodes, aligning positional defaults before omitting the receiver. Preserve `/`, keyword-only `*`, defaults, `*args`, and `**kwargs` in their Python order. Use the unified helper in normal class parsing.
- [x] **Step 4: Verify parser regressions pass.** Run `uv run pytest tests/test_parser.py -xvs`; expect all parser tests to pass.
- [ ] **Step 5: Check the diff and, if authorized, commit** the two files with `feat: classify properties and decorated methods`.

### Task 2: Preserve Members Through Inheritance

**Files:** Modify `src/mdxify/parser.py` (`ClassRegistry`, registry first pass, inheritance merge); test `tests/test_parser.py`.

**Interfaces:**
- Consume Task 1's `extract_class_members` and dictionary contracts in the registry-building pass.
- Add `ClassRegistry.get_inherited_properties(self, class_name: str, include_internal: bool = False) -> list[dict[str, Any]]`, parallel to direct-base method retrieval, returning copies with inheritance/source metadata.
- Keep `get_inherited_methods`'s signature and preserve `kind` when copying methods.
- Normal inheritance merge suppresses both inherited lists using all local function/member definition names, before applying output filtering. Sort the merged properties by name, matching existing inherited-method ordering.

- [x] **Step 1: Add failing inheritance tests.** Extend the existing mocked discovery pattern with a private-module base containing a public property, a static method, and a class method. Assert the child has one inherited property, no property accessor in methods, correct `kind`/signatures, and base `source_file`, getter `line`, and `inherited_from` on the property. Assert first-base-wins for a property shared by two direct bases.

  Add `test_local_member_shadows_inherited_member`, covering property-over-property, method-over-property, property-over-method, and undocumented local overrides. Assert a name appears exactly once across the two lists and the local member's metadata wins. Include private-member filtering and compatibility with a manually registered legacy class lacking `properties`.

- [x] **Step 2: Verify the new tests fail.** Run `uv run pytest tests/test_parser.py -xvs`; expect missing inherited properties or duplicate overridden entries.
- [x] **Step 3: Implement the inheritance interfaces.** Use the unified local extraction in both passes, copy inherited properties just as methods are copied, and filter locally shadowed names during the merge. Keep current base resolution and direct-base scope. Do not mutate registry records when annotating inherited copies.
- [x] **Step 4: Verify all parser tests pass.** Run `uv run pytest tests/test_parser.py -xvs`.
- [ ] **Step 5: Check the diff and, if authorized, commit** with `feat: preserve property metadata through inheritance`.

### Task 3: Render Properties and Method Kinds

**Files:** Modify `src/mdxify/generator.py` (class rendering); test `tests/test_generator.py`.

**Interfaces:**
- Consume Task 1's property and method dictionaries, including Task 2's inherited source metadata.
- Preserve `generate_mdx`'s existing signature and return behavior.
- Read optional keys with defaults: `cls.get("properties", [])` and `method.get("kind", "instance")`.
- Use existing renderer methods and `format_docstring_with_griffe` for all getter/setter/deleter prose.

- [x] **Step 1: Add failing output tests.** Add `test_generate_property_and_method_kinds` parameterized over `MDXRenderer()` and `MarkdownRenderer()`, with `repo_url` omitted for exact header assertions. Assert:

  ```python
  assert content.count("#### `name`") == 1
  assert content.index("**Properties:**") < content.index("**Methods:**")
  assert "```python\nname: str\n```" in content
  assert "*Read/write.*" in content
  assert "**Setter:**" in content and "Set the name." in content
  assert "name(self" not in content
  assert "*Static method.*" in content
  assert "*Class method.*" in content
  assert "cm() -> 'Thing'" in content
  ```

  Add `test_property_output_variants` to assert the four exact access labels from the contract, unannotated `name` fences, getter/deleter prose, and no empty Methods section for property-only classes.

  Add `test_property_source_links_and_escaping` for both renderers: a local property links to its getter; an inherited property links to its base file and line. Use an established source-link fixture layout from `tests/test_source_links.py`. Include accessor docstrings containing angle brackets and braces; assert escaped MDX and intact Markdown via the respective renderer's conventions. Ensure accessor docstrings with structured sections use Griffe formatting.

  Preserve `test_generate_mdx_with_content` unchanged to prove legacy dictionaries still work, and assert that absent/empty `properties` never emits a Properties section.

- [x] **Step 2: Verify the tests fail on absent rendering.** Run `uv run pytest tests/test_generator.py -xvs`.
- [x] **Step 3: Implement class rendering.** Insert the property section before the method section, preserving heading levels. Reuse the method source-file fallback logic for getter links. Render access labels and nested accessor prose with the exact contract copy. Add method-kind labels without changing instance-method headings or signatures.
- [x] **Step 4: Verify rendering and source-link regressions.** Run `uv run pytest tests/test_generator.py tests/test_source_links.py -xvs`; expect all tests to pass.
- [ ] **Step 5: Check the diff and, if authorized, commit** with `feat: render properties and method kinds`.

### Task 4: Validate the CLI and Document the Behavior

**Files:** Modify `tests/test_end_to_end.py` and `README.md`.

**Interfaces:** Consume the existing CLI and Tasks 1–3; introduce no flags or configuration options.

- [x] **Step 1: Add `test_cli_decorator_aware_members`.** Create an importable temporary package containing the issue's example and a subclass. Follow the existing subprocess pattern, parameterizing default parsing vs `--include-inheritance`, and `--format mdx` vs `--format md`. Run from the fixture's project directory with `--no-update-nav`. Assert exit code zero, the correct output extension, one property heading per class, attribute-style type display, accessor prose, exact kind labels, and no explicit class-method receiver. In inheritance mode, assert the subclass receives the property without accessor methods.
- [x] **Step 2: Run the CLI regression.** Run `uv run pytest tests/test_end_to_end.py -k decorator_aware -xvs`; expect all four mode combinations to pass. If it fails, identify the parser/renderer integration gap before changing code.
- [x] **Step 3: Update README usage/features.** Describe the Properties section, getter-derived type, access labels, and static/class-method labels with a short example. State that class-method receivers are omitted. Use the existing documentation style.
- [x] **Step 4: Run the full suite.** Run `uv run pytest -xvs`; require zero failures and record the test summary. Run `uv run pre-commit run --all-files`; investigate failures and keep any fixes scoped to this feature. Do not claim baseline failures were caused or fixed without comparing evidence.
- [x] **Step 5: Validate output against a real project.** Prefer the Infrahub Python SDK that surfaced the issue, or FastMCP if it is readily available. Choose and record a source-confirmed example of each member kind. From that project's directory run `uv run --with-editable /absolute/path/to/mdxify mdxify <actual-module> --output-dir /tmp/opencode/decorator-members-output --no-update-nav` (confirm paths first). Repeat with `--include-inheritance` for a source-confirmed subclass. Inspect output for one entry per property, correct type/access flags, correct labels and class-method arguments, and getter/base source links when repository URL options are supplied. Record the project version, exact commands and inspected members. If the project is unavailable, report the blocker rather than substituting the synthetic fixture for this check.
- [ ] **Step 6: Check the final diff and, if authorized, commit** with `test: cover decorator-aware API documentation`.

## Acceptance Checklist

- [x] The issue's property produces one attribute entry with getter type and read/write signal.
- [x] Setter/deleter declarations do not appear as callable methods; their prose remains available in the property entry.
- [x] Static/class methods remain under Methods with explicit kind labels; class-method receiver is omitted.
- [x] Both parsing paths and both output formats satisfy the same contract.
- [x] Directly inherited properties retain source metadata; local cross-kind overrides avoid duplicates.
- [x] Public/private filtering and legacy module-info dictionaries remain supported.
- [x] Full suite, pre-commit, and real-project validation have recorded evidence.

## Scope Notes

Inherited accessor rebinding such as `@Base.name.setter`, aliases such as `@p`, and rebinding an accessor under a different Python name need a separate resolution design; this plan handles conventional same-name local accessors and direct inheritance of completed properties.

## Implementation Evidence

- Full suite: `uv run pytest -xvs` → **137 passed, 1 skipped** (the macOS-only test on Linux).
- `uv run pre-commit run --all-files` → **Ruff and Typecheck passed**.
- CLI regression: MDX/Markdown × default/inheritance → **4 passed**.
- Real project: **Infrahub SDK 1.23.2** generated successfully in default and inheritance modes. Verified `InfrahubNodeBase.hfid`, inherited `InfrahubNode.hfid`, `InfrahubNode.from_graphql`, inherited `_strip_alias`, and read/write client `retry_on_failure`, including getter source links.
- Final author self-review found and fixed local property rebinding and direct-base shadowing edge cases with seven additional RED→GREEN regression cases. Type-only annotations retain inherited properties; actual assignments shadow them. Parsed classes carry `member_names: set[str]` so earlier base bindings suppress later base properties across member kinds.
- The user authorized committing the implementation and pushing the feature branch after verification.

## Plan Self-Review

- Issue requirements map to Tasks 1 and 3; the existing inheritance path is covered by Task 2, CLI/output validation by Task 4.
- All new interface names and dictionary keys are consistent across extraction, inheritance, rendering, and tests.
- Each Review Focus condition has explicit assertions in its owning task.
- The output section and accessor presentation are marked as proposals requiring review, rather than attributed to the issue author.
- No product code or dependency changes are part of preparing this plan.
