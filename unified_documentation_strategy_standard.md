---
title: "Unified Documentation Intermediate Representation (UDIR)"
subtitle: "Research specification and adapter profiles for the August 2026 TIOBE top 20 programming languages"
document_version: "1.0.0-research-draft"
schema_version: "1.0"
research_date: "2026-09-01"
ranking_basis: "TIOBE Index for August 2026"
normative_language: "RFC 2119-style MUST/SHOULD/MAY"
---

# Unified Documentation Intermediate Representation (UDIR)

## Status and purpose

This document is a research-backed design for ingesting the native documentation conventions of twenty major programming languages into one loss-aware intermediate representation. It is intended to serve as:

1. a data-model specification;
2. an adapter implementation guide;
3. a conformance and edge-case test plan; and
4. a source registry for continued maintenance.

It does **not** propose that programmers abandon Javadoc, rustdoc, Go doc comments, DocC, POD, Rd, XML documentation, or any other native authoring system. A universal source-comment syntax would break native IDE support, compiler validation, ecosystem tooling, and established conventions. UDIR instead standardizes the **extracted semantics**, while preserving the complete native input and its provenance.

The core transformation is:

```text
native source + build context + compiler/indexer output + catalog metadata
                                 |
                                 v
                    language/dialect adapter
                                 |
                                 v
                 native parse tree + symbol graph
                                 |
                                 v
             UDIR semantic records + diagnostics + raw payload
                                 |
                                 v
            one renderer/search/index/AI ingestion platform
```

## Ranking scope

There is no universally authoritative definition of the “top 20” programming languages. This report uses the **TIOBE Index for August 2026**, the most recent complete monthly table available on the research date. TIOBE measures language popularity signals; it explicitly does not rank language quality or lines of code.

| Rank | Language | UDIR language ID |
|---:|---|---|
| 1 | Python | `python` |
| 2 | C | `c` |
| 3 | C++ | `cpp` |
| 4 | Java | `java` |
| 5 | C# | `csharp` |
| 6 | JavaScript | `javascript` |
| 7 | Visual Basic | `visual-basic-dotnet` |
| 8 | SQL | `sql` |
| 9 | R | `r` |
| 10 | Rust | `rust` |
| 11 | Delphi/Object Pascal | `delphi` |
| 12 | Scratch | `scratch` |
| 13 | PHP | `php` |
| 14 | Go | `go` |
| 15 | Fortran | `fortran` |
| 16 | Ruby | `ruby` |
| 17 | Swift | `swift` |
| 18 | Perl | `perl` |
| 19 | COBOL | `cobol` |
| 20 | Assembly language | `assembly` |

Ranking source: [TIOBE Index](https://www.tiobe.com/tiobe-index/), captured 2026-09-01.

---

# Part I — Normative platform design

## 1. Design principles

### 1.1 Normalize semantics, not punctuation

`@param`, `<param>`, `- Parameter name:`, `Args:`, an Rd `\arguments` item, and a Go prose paragraph can all communicate parameter documentation. They are not syntactically interchangeable, and some are more structured than others. An adapter MUST map them to common semantic fields while retaining:

- the original source text;
- the native parsed form;
- the exact dialect and processor;
- the binding method;
- confidence and loss information; and
- source spans.

### 1.2 The code model is authoritative for code facts

When a compiler, semantic model, or dialect-aware parser can determine a symbol’s name, kind, signature, visibility, parameter order, type, generic constraints, inheritance, or source location, that result MUST take precedence over prose tags. A comment may record a second `documentedType`, but it MUST NOT silently replace the actual type.

Example: a PHPDoc block says `@return string`, but the PHP declaration says `: int`. UDIR stores `int` as the actual type, `string` as the documented type, and emits `UDIR-TYPE-CONFLICT`.

### 1.3 Preserve before interpreting

Every record MUST retain the native comment or documentation payload. Unknown tags, directives, XML elements, Markdown extensions, generator plug-in nodes, and vendor fields MUST survive under `native.parsed`, `native.unknownConstructs`, or `extensions`.

A renderer may ignore unsupported extensions. An ingestion pipeline may not delete them merely because the current UDIR version lacks a semantic field.

### 1.4 Build context is part of identity

The same source can expose different APIs under conditional compilation, feature flags, target triples, SQL dialects, package versions, compiler versions, source formats, or preprocessors. Therefore:

- C/C++ compile commands;
- Rust Cargo features and target;
- Go build tags, GOOS, GOARCH, and cgo state;
- Swift target and compiler version;
- Fortran preprocessing and source form;
- COBOL dialect and source format;
- SQL vendor/version/schema;
- assembly architecture/assembler/syntax mode; and
- Java release/module paths

MUST be captured when they affect extraction. Distinct configurations SHOULD produce distinct `variantKey` values rather than being flattened into one falsely universal symbol.

### 1.5 Documentation inheritance is provenance, not copy-and-forget

Inherited, included, generated, amended, or merged documentation MUST record its origin and transformation path. UDIR MAY materialize the resolved text for rendering, but it MUST also retain the inheritance/include relation and native rule that caused it.

### 1.6 No execution by default

Documentation is not inert in every ecosystem. Examples, includes, macros, embedded expressions, generators, catalog queries, and compiler plug-ins can execute code or read arbitrary files. Extractors MUST default to static parsing with:

- no network;
- no subprocesses;
- no module import;
- no macro/procedural-macro execution unless sandboxed and explicitly enabled;
- no doctest/example execution;
- no R `\Sexpr`;
- no DocC or Javadoc external include traversal outside an allowlisted root;
- no SQL connection unless explicitly configured read-only; and
- no untrusted HTML/script execution.

## 2. Authority taxonomy

A language profile MUST declare the authority of each documentation mechanism. “Official” is not a single category.

| Authority | Meaning | Examples |
|---|---|---|
| `language-standard` | Defined by the language specification or normative language documentation | Rust doc comments; Python docstring semantics |
| `compiler-vendor-official` | Recognized or emitted by the principal compiler/vendor, but not necessarily portable across implementations | C# XML documentation; Delphi XML documentation |
| `ecosystem-official` | Maintained by the language project or shipped as its standard documentation tool | Go documentation packages; Ruby RDoc; Swift DocC; Perl POD |
| `dialect-vendor-specific` | Official only for a vendor/dialect | PostgreSQL `COMMENT ON`; SQL Server extended properties; MASM `COMMENT` |
| `de-facto` | Widely used but not language-standard | Doxygen for C/C++; JSDoc for JavaScript; phpDocumentor PHPDoc; FORD for Fortran |
| `none` | No semantic documentation mechanism exists at that layer | ISO C comments; Scratch API documentation; generic assembly |

A profile can contain several mechanisms with different authorities. For example, JavaScript has ECMAScript lexical comments (`language-standard`) but JSDoc semantics (`de-facto`).

## 3. UDIR record categories

| `recordKind` | Use |
|---|---|
| `symbol-documentation` | Documentation bound to a language symbol or database object |
| `module-documentation` | Module, crate, unit, package-file, or namespace-level documentation |
| `package-documentation` | Distribution/package-level documentation |
| `conceptual-document` | Guides, tutorials, articles, overviews, and hand-written pages |
| `catalog-documentation` | Deployed database/catalog metadata |
| `workspace-annotation` | Comments that annotate a visual workspace rather than an API, notably Scratch |
| `example` | Standalone example or executable-documentation unit |
| `relationship` | Explicit graph relation when it cannot be embedded in one symbol record |
| `diagnostic` | Extraction, validation, conflict, or loss report |

A renderer SHOULD use one visual design across record categories but MUST NOT falsely display a workspace annotation as a function contract or a SQL catalog comment as source-authoritative documentation.

## 4. Canonical identity

### 4.1 Required identity layers

Each symbol SHOULD carry three identity layers:

1. **Native ID** — compiler, documentation generator, symbol graph, catalog key, or assembler name, when available.
2. **Canonical identity tuple** — normalized language, package/module/assembly/database context, symbol kind, qualified name, overload signature, and variant.
3. **UDIR record ID** — deterministic hash of the canonical tuple.

Recommended form:

```text
recordId = "udir:v1:" + language.id + ":" +
           base64url(sha256(canonical-json(identityTuple)))
```

The canonical JSON MUST use sorted keys, UTF-8, Unicode normalization form NFC, and no insignificant whitespace. The human-readable name is stored separately; consumers MUST NOT reverse-engineer it from the hash.

### 4.2 Native IDs to preserve

Adapters SHOULD preserve the strongest identity their ecosystem provides:

| Ecosystem | Preferred native identity |
|---|---|
| C/C++ | Clang USR plus compile configuration; linkage name where relevant |
| Java | module/package/binary name plus erased or JVM descriptor-level member signature |
| .NET | compiler XML documentation ID plus assembly identity |
| Rust | rustdoc/rustc item identity or DefPath-based identity plus crate/version/features |
| Swift | compiler USR / symbol-graph precise identifier |
| Go | module version + import path + receiver + exported symbol |
| Python | distribution/module/qualified name plus source span; identity is less stable under rebinding |
| JavaScript | module URL/package export path + AST symbol + source span; virtual JSDoc symbols need explicit IDs |
| SQL | vendor/server/database/schema/object kind/name/signature |
| R | package + Rd topic/alias + object/method identity |
| Ruby | gem/version + namespace + singleton/instance marker + method name |
| Perl | distribution/module plus declared package/subroutine where statically recoverable |
| Scratch | project/target/object ID; custom-block procedure code plus mutation IDs |
| COBOL | program/library + program/function/class/entry ID + dialect/build variant |
| Assembly | object format + architecture + object/module + linkage symbol/version |
| Fortran | package/module/submodule/generic/specific procedure identity + compiler variant |
| Delphi | package/unit/namespace/type/member + compiler-emitted identity where available |

### 4.3 One record per overload and conditional variant

Overloads, template specializations, SQL overloaded routines, generic/specific Fortran procedures, Swift overloads, .NET indexers/operators, and C++ conversion functions MUST be separate records. A language-level overload group MAY be represented as an additional grouping record.

Conditional declarations SHOULD not be merged unless all semantic properties and documentation are identical. Otherwise create variants and connect them with `overloads`, `specializes`, or a project-specific relation.

## 5. Binding model

`provenance.binding` records how text became associated with an entity:

| Binding | Meaning |
|---|---|
| `compiler` | Compiler directly recognized/emitted the documentation |
| `semantic-model` | Language service/indexer attached it to a resolved symbol |
| `ast-adjacent` | Parser attached an immediately preceding native comment/string |
| `explicit-tag` | A native tag names the documented virtual or real entity |
| `catalog-key` | Database/object catalog key |
| `graph-edge` | Symbol graph or visual block graph relation |
| `heuristic` | Heading/name/proximity convention without native binding |
| `unbound` | File/workspace prose with no reliable symbol target |

Consumers SHOULD display lower-confidence or heuristic bindings distinctly when correctness matters.

## 6. Content model

### 6.1 Semantic fields

UDIR normalizes the following concepts:

- summary and long description;
- named/custom sections;
- parameters and type parameters;
- return values and yielded values;
- exceptions, errors, error codes, and failure conditions;
- examples and expected output;
- preconditions, postconditions, invariants, safety requirements, panic behavior, permissions, complexity, threading, side effects, and other contracts;
- links and cross-references;
- lifecycle: since, deprecation, replacement, removal, stability, and availability;
- authorship, license, keywords, categories, and grouping;
- symbol relationships; and
- raw native extensions.

Not every language can supply every field. Missing data MUST remain missing; it must not be inferred merely to make the rendered layout look complete.

### 6.2 Documentation AST

All prose is represented as a safe, renderer-independent document tree. Core nodes include paragraphs, headings, lists, tables, code blocks, inline code, links, symbol links, emphasis, admonitions, block quotes, math, images, directives, raw HTML, raw native content, and unknown nodes.

The native markup dialect MUST be recorded. A CommonMark parser must not be applied blindly to RDoc, POD, Rd, Go doc syntax, XML documentation, Doxygen commands, or DocC directives.

### 6.3 Type AST

Types are normalized into a recursive tree, while retaining native spelling. Core forms include named, generic, union, intersection, nullable, tuple, function, array/slice/map/set, pointer/reference, lifetime, wildcard, literal, dynamic, void, never, unknown, and raw.

UDIR intentionally permits both:

- `signature.parameters[].type` — the actual language/compiler type; and
- `documentation.parameters[].documentedType` — the type asserted in prose markup.

This separation is mandatory for dynamic languages and stale comments.

## 7. Canonical JSON Schema

The following schema is the machine-readable contract shipped beside this document as `udir.schema.json`.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "urn:udir:schema:1.0",
  "title": "Unified Documentation Intermediate Representation (UDIR)",
  "description": "Language-neutral, loss-aware representation for source and catalog documentation.",
  "type": "object",
  "required": [
    "schemaVersion",
    "recordId",
    "recordKind",
    "language",
    "documentation",
    "native",
    "provenance"
  ],
  "additionalProperties": false,
  "properties": {
    "schemaVersion": {
      "type": "string",
      "pattern": "^1\\.0(?:\\.[0-9]+)?(?:-[A-Za-z0-9.-]+)?$"
    },
    "recordId": {
      "type": "string",
      "minLength": 1
    },
    "recordKind": {
      "enum": [
        "symbol-documentation",
        "module-documentation",
        "package-documentation",
        "conceptual-document",
        "catalog-documentation",
        "workspace-annotation",
        "example",
        "relationship",
        "diagnostic"
      ]
    },
    "language": {
      "$ref": "#/$defs/languageContext"
    },
    "buildContext": {
      "$ref": "#/$defs/buildContext"
    },
    "symbol": {
      "$ref": "#/$defs/symbol"
    },
    "documentation": {
      "$ref": "#/$defs/documentation"
    },
    "native": {
      "$ref": "#/$defs/nativePayload"
    },
    "provenance": {
      "$ref": "#/$defs/provenance"
    },
    "relations": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/relation"
      },
      "default": []
    },
    "diagnostics": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/diagnostic"
      },
      "default": []
    },
    "lossFlags": {
      "type": "array",
      "items": {
        "enum": [
          "ambiguous-binding",
          "conditional-variant",
          "dynamic-unobservable",
          "execution-suppressed",
          "generated-symbol",
          "identity-unstable",
          "inherited-doc-expanded",
          "markup-sanitized",
          "native-unknown-tag",
          "catalog-source-drift",
          "type-conflict",
          "unresolved-symbol-link",
          "unsupported-native-node",
          "dialect-assumed",
          "source-map-missing",
          "partial-merge",
          "heuristic-section-mapping",
          "native-output-version-unknown"
        ]
      },
      "uniqueItems": true,
      "default": []
    },
    "extensions": {
      "type": "object",
      "additionalProperties": true,
      "default": {}
    }
  },
  "$defs": {
    "languageContext": {
      "type": "object",
      "required": [
        "id",
        "displayName",
        "authority"
      ],
      "additionalProperties": false,
      "properties": {
        "id": {
          "type": "string",
          "pattern": "^[a-z0-9][a-z0-9+._-]*$"
        },
        "displayName": {
          "type": "string"
        },
        "dialect": {
          "type": [
            "string",
            "null"
          ]
        },
        "version": {
          "type": [
            "string",
            "null"
          ]
        },
        "toolchain": {
          "type": [
            "string",
            "null"
          ]
        },
        "toolchainVersion": {
          "type": [
            "string",
            "null"
          ]
        },
        "sourceFormat": {
          "type": [
            "string",
            "null"
          ]
        },
        "authority": {
          "enum": [
            "language-standard",
            "compiler-vendor-official",
            "ecosystem-official",
            "dialect-vendor-specific",
            "de-facto",
            "none"
          ]
        },
        "authoritySources": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": []
        }
      }
    },
    "buildContext": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "workspace": {
          "type": [
            "string",
            "null"
          ]
        },
        "package": {
          "type": [
            "string",
            "null"
          ]
        },
        "module": {
          "type": [
            "string",
            "null"
          ]
        },
        "target": {
          "type": [
            "string",
            "null"
          ]
        },
        "platform": {
          "type": [
            "string",
            "null"
          ]
        },
        "configuration": {
          "type": [
            "string",
            "null"
          ]
        },
        "featureFlags": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": []
        },
        "defines": {
          "type": "object",
          "additionalProperties": {
            "type": [
              "string",
              "number",
              "boolean",
              "null"
            ]
          },
          "default": {}
        },
        "dependencyVersions": {
          "type": "object",
          "additionalProperties": {
            "type": "string"
          },
          "default": {}
        },
        "sourceFiles": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": []
        },
        "buildDigest": {
          "type": [
            "string",
            "null"
          ]
        }
      }
    },
    "symbol": {
      "type": "object",
      "required": [
        "canonicalId",
        "kind",
        "name"
      ],
      "additionalProperties": false,
      "properties": {
        "canonicalId": {
          "type": "string",
          "minLength": 1
        },
        "nativeId": {
          "type": [
            "string",
            "null"
          ]
        },
        "alternateIds": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": []
        },
        "kind": {
          "enum": [
            "namespace",
            "package",
            "module",
            "crate",
            "unit",
            "program",
            "file",
            "class",
            "interface",
            "trait",
            "protocol",
            "struct",
            "record",
            "union",
            "enum",
            "enum-case",
            "type-alias",
            "delegate",
            "annotation-type",
            "function",
            "method",
            "constructor",
            "destructor",
            "operator",
            "conversion",
            "subscript",
            "property",
            "field",
            "constant",
            "variable",
            "event",
            "macro",
            "template",
            "concept",
            "procedure",
            "paragraph",
            "section",
            "entry",
            "database",
            "schema",
            "table",
            "view",
            "column",
            "index",
            "constraint",
            "trigger",
            "sequence",
            "routine",
            "custom-block",
            "block",
            "label",
            "data-symbol",
            "unknown"
          ]
        },
        "name": {
          "type": "string"
        },
        "displayName": {
          "type": [
            "string",
            "null"
          ]
        },
        "qualifiedName": {
          "type": [
            "string",
            "null"
          ]
        },
        "visibility": {
          "enum": [
            "public",
            "protected",
            "internal",
            "package",
            "private",
            "file-private",
            "local",
            "exported",
            "unexported",
            "unknown"
          ]
        },
        "modifiers": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": []
        },
        "signatures": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/signature"
          },
          "default": []
        },
        "source": {
          "$ref": "#/$defs/sourceSpan"
        },
        "variantKey": {
          "type": [
            "string",
            "null"
          ]
        },
        "generated": {
          "type": "boolean",
          "default": false
        }
      }
    },
    "signature": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "display": {
          "type": "string"
        },
        "canonical": {
          "type": [
            "string",
            "null"
          ]
        },
        "language": {
          "type": [
            "string",
            "null"
          ]
        },
        "parameters": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/parameter"
          },
          "default": []
        },
        "typeParameters": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/typeParameter"
          },
          "default": []
        },
        "returns": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/returnValue"
          },
          "default": []
        },
        "throwsDeclared": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/typeExpression"
          },
          "default": []
        },
        "constraints": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": []
        },
        "receiver": {
          "$ref": "#/$defs/parameter"
        },
        "sourceOfTruth": {
          "enum": [
            "compiler",
            "semantic-model",
            "ast",
            "catalog",
            "documentation",
            "heuristic"
          ]
        }
      }
    },
    "parameter": {
      "type": "object",
      "required": [
        "name"
      ],
      "additionalProperties": false,
      "properties": {
        "name": {
          "type": "string"
        },
        "nativeName": {
          "type": [
            "string",
            "null"
          ]
        },
        "position": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0
        },
        "direction": {
          "enum": [
            "in",
            "out",
            "inout",
            "receiver",
            "variadic",
            "positional-only",
            "keyword-only",
            "unknown"
          ]
        },
        "type": {
          "$ref": "#/$defs/typeExpression"
        },
        "documentedType": {
          "$ref": "#/$defs/typeExpression"
        },
        "defaultValue": {
          "type": [
            "string",
            "null"
          ]
        },
        "optional": {
          "type": "boolean",
          "default": false
        },
        "variadic": {
          "type": "boolean",
          "default": false
        },
        "documentation": {
          "$ref": "#/$defs/docNode"
        }
      }
    },
    "typeParameter": {
      "type": "object",
      "required": [
        "name"
      ],
      "additionalProperties": false,
      "properties": {
        "name": {
          "type": "string"
        },
        "position": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0
        },
        "variance": {
          "enum": [
            "in",
            "out",
            "invariant",
            "unknown"
          ]
        },
        "bounds": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/typeExpression"
          },
          "default": []
        },
        "documentation": {
          "$ref": "#/$defs/docNode"
        }
      }
    },
    "returnValue": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "name": {
          "type": [
            "string",
            "null"
          ]
        },
        "position": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0
        },
        "type": {
          "$ref": "#/$defs/typeExpression"
        },
        "documentedType": {
          "$ref": "#/$defs/typeExpression"
        },
        "condition": {
          "type": [
            "string",
            "null"
          ]
        },
        "documentation": {
          "$ref": "#/$defs/docNode"
        }
      }
    },
    "typeExpression": {
      "type": [
        "object",
        "null"
      ],
      "additionalProperties": false,
      "properties": {
        "kind": {
          "enum": [
            "named",
            "generic",
            "union",
            "intersection",
            "nullable",
            "tuple",
            "function",
            "array",
            "slice",
            "map",
            "set",
            "pointer",
            "reference",
            "lifetime",
            "wildcard",
            "literal",
            "dynamic",
            "void",
            "never",
            "unknown",
            "raw"
          ]
        },
        "name": {
          "type": [
            "string",
            "null"
          ]
        },
        "native": {
          "type": [
            "string",
            "null"
          ]
        },
        "arguments": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/typeExpression"
          },
          "default": []
        },
        "members": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/typeExpression"
          },
          "default": []
        },
        "nullable": {
          "type": [
            "boolean",
            "null"
          ]
        },
        "modifiers": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": []
        },
        "targetId": {
          "type": [
            "string",
            "null"
          ]
        }
      }
    },
    "documentation": {
      "type": "object",
      "required": [
        "summary",
        "description"
      ],
      "additionalProperties": false,
      "properties": {
        "summary": {
          "$ref": "#/$defs/docNode"
        },
        "description": {
          "$ref": "#/$defs/docNode"
        },
        "sections": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/namedSection"
          },
          "default": []
        },
        "parameters": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/documentedParameter"
          },
          "default": []
        },
        "typeParameters": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/documentedParameter"
          },
          "default": []
        },
        "returns": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/documentedReturn"
          },
          "default": []
        },
        "yields": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/documentedReturn"
          },
          "default": []
        },
        "errors": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/documentedError"
          },
          "default": []
        },
        "contracts": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/contract"
          },
          "default": []
        },
        "examples": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/example"
          },
          "default": []
        },
        "seeAlso": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/linkTarget"
          },
          "default": []
        },
        "lifecycle": {
          "$ref": "#/$defs/lifecycle"
        },
        "authors": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": []
        },
        "license": {
          "type": [
            "string",
            "null"
          ]
        },
        "keywords": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": []
        }
      }
    },
    "namedSection": {
      "type": "object",
      "required": [
        "kind",
        "content"
      ],
      "additionalProperties": false,
      "properties": {
        "kind": {
          "type": "string"
        },
        "title": {
          "type": [
            "string",
            "null"
          ]
        },
        "content": {
          "$ref": "#/$defs/docNode"
        },
        "nativeName": {
          "type": [
            "string",
            "null"
          ]
        }
      }
    },
    "documentedParameter": {
      "type": "object",
      "required": [
        "name",
        "documentation"
      ],
      "additionalProperties": false,
      "properties": {
        "name": {
          "type": "string"
        },
        "position": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0
        },
        "documentation": {
          "$ref": "#/$defs/docNode"
        },
        "documentedType": {
          "$ref": "#/$defs/typeExpression"
        },
        "sourceNativeTag": {
          "type": [
            "string",
            "null"
          ]
        }
      }
    },
    "documentedReturn": {
      "type": "object",
      "required": [
        "documentation"
      ],
      "additionalProperties": false,
      "properties": {
        "name": {
          "type": [
            "string",
            "null"
          ]
        },
        "position": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0
        },
        "documentation": {
          "$ref": "#/$defs/docNode"
        },
        "documentedType": {
          "$ref": "#/$defs/typeExpression"
        },
        "sourceNativeTag": {
          "type": [
            "string",
            "null"
          ]
        }
      }
    },
    "documentedError": {
      "type": "object",
      "required": [
        "documentation"
      ],
      "additionalProperties": false,
      "properties": {
        "type": {
          "$ref": "#/$defs/typeExpression"
        },
        "code": {
          "type": [
            "string",
            "null"
          ]
        },
        "condition": {
          "type": [
            "string",
            "null"
          ]
        },
        "documentation": {
          "$ref": "#/$defs/docNode"
        },
        "declaredInSignature": {
          "type": [
            "boolean",
            "null"
          ]
        },
        "sourceNativeTag": {
          "type": [
            "string",
            "null"
          ]
        }
      }
    },
    "contract": {
      "type": "object",
      "required": [
        "kind",
        "documentation"
      ],
      "additionalProperties": false,
      "properties": {
        "kind": {
          "enum": [
            "precondition",
            "postcondition",
            "invariant",
            "safety",
            "panic",
            "threading",
            "complexity",
            "side-effect",
            "permission",
            "requirement",
            "other"
          ]
        },
        "documentation": {
          "$ref": "#/$defs/docNode"
        },
        "machineReadable": {
          "type": [
            "string",
            "object",
            "array",
            "number",
            "boolean",
            "null"
          ]
        },
        "sourceNativeTag": {
          "type": [
            "string",
            "null"
          ]
        }
      }
    },
    "example": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "title": {
          "type": [
            "string",
            "null"
          ]
        },
        "documentation": {
          "$ref": "#/$defs/docNode"
        },
        "code": {
          "type": [
            "string",
            "null"
          ]
        },
        "language": {
          "type": [
            "string",
            "null"
          ]
        },
        "execution": {
          "enum": [
            "never",
            "parse-only",
            "compile",
            "run",
            "compile-fail",
            "run-no-output-check",
            "external"
          ]
        },
        "expectedOutput": {
          "type": [
            "string",
            "null"
          ]
        },
        "hiddenLines": {
          "type": "array",
          "items": {
            "type": "integer",
            "minimum": 1
          },
          "default": []
        },
        "source": {
          "$ref": "#/$defs/sourceSpan"
        }
      }
    },
    "lifecycle": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "since": {
          "type": [
            "string",
            "null"
          ]
        },
        "deprecated": {
          "type": [
            "object",
            "null"
          ],
          "additionalProperties": false,
          "properties": {
            "message": {
              "$ref": "#/$defs/docNode"
            },
            "replacement": {
              "$ref": "#/$defs/linkTarget"
            },
            "since": {
              "type": [
                "string",
                "null"
              ]
            },
            "nativeMechanisms": {
              "type": "array",
              "items": {
                "type": "string"
              },
              "default": []
            }
          }
        },
        "removed": {
          "type": [
            "string",
            "null"
          ]
        },
        "stability": {
          "type": [
            "string",
            "null"
          ]
        },
        "availability": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": []
        }
      }
    },
    "docNode": {
      "type": [
        "object",
        "null"
      ],
      "required": [
        "kind"
      ],
      "additionalProperties": false,
      "properties": {
        "kind": {
          "enum": [
            "document",
            "paragraph",
            "text",
            "heading",
            "list",
            "list-item",
            "table",
            "table-row",
            "table-cell",
            "code-block",
            "inline-code",
            "link",
            "symbol-link",
            "emphasis",
            "strong",
            "admonition",
            "blockquote",
            "thematic-break",
            "math",
            "image",
            "directive",
            "raw-html",
            "raw-native",
            "unknown"
          ]
        },
        "text": {
          "type": [
            "string",
            "null"
          ]
        },
        "children": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/docNode"
          },
          "default": []
        },
        "level": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 1,
          "maximum": 6
        },
        "language": {
          "type": [
            "string",
            "null"
          ]
        },
        "title": {
          "type": [
            "string",
            "null"
          ]
        },
        "target": {
          "$ref": "#/$defs/linkTarget"
        },
        "attributes": {
          "type": "object",
          "additionalProperties": true,
          "default": {}
        },
        "raw": {
          "type": [
            "string",
            "null"
          ]
        }
      }
    },
    "linkTarget": {
      "type": [
        "object",
        "null"
      ],
      "additionalProperties": false,
      "properties": {
        "kind": {
          "enum": [
            "symbol",
            "url",
            "file",
            "fragment",
            "tutorial",
            "specification",
            "unknown"
          ]
        },
        "target": {
          "type": "string"
        },
        "label": {
          "type": [
            "string",
            "null"
          ]
        },
        "resolvedId": {
          "type": [
            "string",
            "null"
          ]
        },
        "status": {
          "enum": [
            "resolved",
            "unresolved",
            "ambiguous",
            "external",
            "suppressed"
          ]
        }
      }
    },
    "nativePayload": {
      "type": "object",
      "required": [
        "sourceSyntax",
        "markup",
        "raw"
      ],
      "additionalProperties": false,
      "properties": {
        "sourceSyntax": {
          "type": "string"
        },
        "markup": {
          "type": "string"
        },
        "raw": {
          "type": "string"
        },
        "parsed": {},
        "generator": {
          "type": [
            "string",
            "null"
          ]
        },
        "generatorVersion": {
          "type": [
            "string",
            "null"
          ]
        },
        "outputFormat": {
          "type": [
            "string",
            "null"
          ]
        },
        "unknownConstructs": {
          "type": "array",
          "items": {},
          "default": []
        }
      }
    },
    "provenance": {
      "type": "object",
      "required": [
        "binding",
        "confidence",
        "sources"
      ],
      "additionalProperties": false,
      "properties": {
        "binding": {
          "enum": [
            "compiler",
            "semantic-model",
            "ast-adjacent",
            "explicit-tag",
            "catalog-key",
            "graph-edge",
            "heuristic",
            "unbound"
          ]
        },
        "confidence": {
          "type": "number",
          "minimum": 0,
          "maximum": 1
        },
        "coverage": {
          "enum": [
            "direct",
            "inherited",
            "generated",
            "inferred",
            "missing",
            "suppressed"
          ]
        },
        "inheritancePath": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": []
        },
        "sources": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/provenanceSource"
          }
        },
        "extractor": {
          "type": "string"
        },
        "extractorVersion": {
          "type": [
            "string",
            "null"
          ]
        },
        "extractedAt": {
          "type": "string",
          "format": "date-time"
        }
      }
    },
    "provenanceSource": {
      "type": "object",
      "required": [
        "kind",
        "uri"
      ],
      "additionalProperties": false,
      "properties": {
        "kind": {
          "enum": [
            "source",
            "compiler-output",
            "catalog",
            "generated-file",
            "external-file",
            "runtime",
            "heuristic"
          ]
        },
        "uri": {
          "type": "string"
        },
        "span": {
          "$ref": "#/$defs/sourceSpan"
        },
        "authority": {
          "type": [
            "string",
            "null"
          ]
        },
        "digest": {
          "type": [
            "string",
            "null"
          ]
        },
        "version": {
          "type": [
            "string",
            "null"
          ]
        }
      }
    },
    "sourceSpan": {
      "type": [
        "object",
        "null"
      ],
      "additionalProperties": false,
      "properties": {
        "uri": {
          "type": "string"
        },
        "startLine": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 1
        },
        "startColumn": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 1
        },
        "endLine": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 1
        },
        "endColumn": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 1
        },
        "byteStart": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0
        },
        "byteEnd": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0
        }
      }
    },
    "relation": {
      "type": "object",
      "required": [
        "kind",
        "target"
      ],
      "additionalProperties": false,
      "properties": {
        "kind": {
          "enum": [
            "contains",
            "member-of",
            "inherits",
            "implements",
            "overrides",
            "overloads",
            "specializes",
            "aliases",
            "reexports",
            "extends",
            "mixes-in",
            "uses",
            "provides",
            "documents",
            "example-of",
            "generated-from",
            "corresponds-to",
            "attached-to",
            "calls",
            "references"
          ]
        },
        "target": {
          "$ref": "#/$defs/linkTarget"
        },
        "nativeKind": {
          "type": [
            "string",
            "null"
          ]
        },
        "conditional": {
          "type": [
            "string",
            "null"
          ]
        },
        "provenance": {
          "type": [
            "string",
            "null"
          ]
        }
      }
    },
    "diagnostic": {
      "type": "object",
      "required": [
        "code",
        "severity",
        "message"
      ],
      "additionalProperties": false,
      "properties": {
        "code": {
          "type": "string"
        },
        "severity": {
          "enum": [
            "info",
            "warning",
            "error"
          ]
        },
        "message": {
          "type": "string"
        },
        "source": {
          "$ref": "#/$defs/sourceSpan"
        },
        "relatedIds": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": []
        }
      }
    }
  }
}
```

## 8. Adapter manifest

Every adapter SHOULD publish a machine-readable manifest resembling:

```yaml
adapter_id: udir.java.jdk
adapter_version: 1.0.0
language_ids: [java]
supported_versions: ["8-26"]
authority_mechanisms:
  - id: javadoc-standard-doclet
    authority: compiler-vendor-official
    syntax: [traditional, markdown]
inputs:
  - source-tree
  - module-path
  - class-path
  - release
outputs:
  - udir-1.0
security:
  executes_user_code: false
  reads_external_includes: allowlisted
identity:
  native_ids: [javadoc-reference, jvm-descriptor]
loss_policy:
  preserve_unknown_tags: true
  preserve_raw_comment: true
conformance_fixture_set: udir-java-v1
```

The manifest MUST disclose whether extraction can execute code, load plug-ins, expand macros, contact a database, follow includes, or depend on a compiler build.

## 9. Extraction pipeline

A conforming implementation SHOULD use these stages:

1. **Inventory** — identify language, files, package/module structure, dialect, versions, and build configuration.
2. **Static parse** — create the language AST without importing or running the project.
3. **Semantic/index pass** — resolve symbols and relationships using the native compiler or language service where practical.
4. **Native documentation parse** — parse the exact native syntax into a native AST.
5. **Binding** — associate native docs with symbols using native rules, not a universal adjacency heuristic.
6. **Identity** — assign native and canonical IDs.
7. **Semantic normalization** — map supported native constructs into UDIR.
8. **Preservation** — retain raw text, unsupported constructs, processor configuration, and source maps.
9. **Resolution** — resolve symbol links, includes, inheritance, and aliases under a controlled policy.
10. **Validation** — compare parameter names, returns, errors, type arguments, links, visibility, and duplicate ownership.
11. **Security filtering** — sanitize HTML, prevent traversal, suppress execution, and cap resource use.
12. **Emission** — produce deterministic JSON records and diagnostics.

### 9.1 Source-of-truth precedence

For code facts:

```text
compiler semantic model
  > compiler/indexer symbol graph
  > dialect-aware AST
  > deployed catalog (for deployed SQL objects)
  > documentation declaration/type tag
  > heuristic
```

For prose ownership:

```text
directly bound native documentation
  > native amendment/extension mechanism
  > native include/inherit mechanism
  > project-configured convention
  > heuristic heading/proximity match
```

These orders answer different questions. A deployed SQL catalog can be authoritative for the live database while a migration file is authoritative for repository intent; both MUST remain separately attributable.

## 10. Conflict rules

A conforming platform MUST emit diagnostics instead of silently selecting convenient text.

| Code | Condition | Required behavior |
|---|---|---|
| `UDIR-PARAM-MISSING` | Actual parameter has no matching documentation entry | Keep actual parameter; report missing docs |
| `UDIR-PARAM-ORPHAN` | Documentation names no actual parameter | Preserve entry; mark unresolved |
| `UDIR-TYPE-CONFLICT` | Documented and actual types disagree | Preserve both; actual type remains authoritative |
| `UDIR-RETURN-VOID` | Return documentation exists for no-value declaration | Preserve and warn; language-specific exceptions allowed |
| `UDIR-ERROR-UNRESOLVED` | Exception/error type cannot resolve | Preserve native spelling and unresolved link |
| `UDIR-LINK-AMBIGUOUS` | Cross-reference matches multiple symbols | Preserve all candidates; do not guess |
| `UDIR-DOC-DUPLICATE` | Multiple direct docs compete for one symbol | Apply native precedence; retain rejected fragments |
| `UDIR-VARIANT-MERGE` | Conditional variants differ | Split variants |
| `UDIR-NATIVE-UNKNOWN` | Unknown tag/node/directive | Preserve in extensions |
| `UDIR-INCLUDE-BLOCKED` | Include escapes allowed root or is unavailable | Do not read it; record blocked target |
| `UDIR-EXECUTION-SUPPRESSED` | Native construct could execute | Preserve source; do not execute |
| `UDIR-CATALOG-DRIFT` | Source and deployed catalog comments differ | Keep separate records and relationship |
| `UDIR-BINDING-HEURISTIC` | Association lacks native proof | Set lower confidence and loss flag |

## 11. Feature matrix

Legend: **S** = natively structured; **C** = convention/processor feature; **H** = heuristic only; **—** = no standard mechanism. “Official layer” is the strongest semantic documentation layer, not merely lexical comment syntax.

| Language | Official layer | Native markup | Params | Returns | Errors | Links | Inherit/merge | Executable examples | Stable native symbol ID |
|---|---|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Python | language docstrings; PEP conventions | unspecified; dialects vary | C/H | C/H | C/H | C | runtime/processor | `doctest` C | limited |
| C | none; Doxygen de facto | Doxygen/Markdown-like | C | C | C | C | C | C | compiler USR |
| C++ | none; Doxygen de facto | Doxygen/Markdown-like | C | C | C | C | C | C | compiler USR |
| Java | Javadoc standard doclet | HTML or Markdown + tags | S | S | S | S | S | snippet/tool-specific | strong |
| C# | compiler XML docs | XML | S | S | S | S | C | C | compiler XML ID |
| JavaScript | JSDoc de facto | JSDoc + configured Markdown | S | S | S | S | C | C | weak/processor |
| Visual Basic | compiler XML docs | XML | S | S | S | S | C | C | compiler XML ID |
| SQL | dialect comments/catalog metadata | plain text/vendor | H | H | H | vendor | vendor | — | catalog key |
| R | Rd official package artifact | Rd | S | S | C | S | roxygen C | Rd examples C | topic/alias |
| Rust | language doc comments + rustdoc | Markdown + rustdoc | C | C | C | S | reexport/trait C | S | strong/toolchain |
| Delphi | vendor XML docs | XML subset | S | S | S | S | vendor C | C | vendor XML |
| Scratch | workspace comments only | plain text | — | — | — | — | graph attachment | — | object IDs |
| PHP | PHPDoc de facto | DocBlock + tags | S | S | S | S | C | C | parser-derived |
| Go | ecosystem-official doc comments | Go doc syntax | C | C | C | S | package concatenation | S | strong |
| Fortran | none; FORD de facto | Markdown + FORD | C | C | C | C | C | C | parser/compiler |
| Ruby | RDoc ecosystem-official | RDoc/Markdown/RD/TomDoc | C | C | C | S | reopen/RBS C | C | namespace/method |
| Swift | DocC ecosystem-official | Markdown + DocC | S | S | S | S | extension files C | C | compiler USR |
| Perl | POD language ecosystem | POD | H | H | H | S | — | C | weak |
| COBOL | none | plain comments | H | H | H | — | COPY/preprocess | — | parser-derived |
| Assembly | none; assembler-specific comments | plain text | H | H | H | — | include/macro | — | object symbol |

## 12. Cross-language normalization map

| UDIR semantic | Common native inputs |
|---|---|
| `summary` | Javadoc first sentence/`{@summary}`; XML `<summary>`; DocC abstract; Rd `\title` plus description policy; Python/RDoc/POD first paragraph by configured convention |
| `description` | Javadoc main description; XML `<remarks>`; Python body; rustdoc/Go/RDoc/POD prose; Rd `\description`/`\details` |
| `parameters` | Javadoc/JSDoc/PHPDoc `@param`; XML `<param>`; DocC parameter fields; Rd `\arguments`; Python style sections; Doxygen commands |
| `typeParameters` | Javadoc `@param <T>`; XML `<typeparam>`; JSDoc `@template`; Doxygen `@tparam`; static-analysis PHPDoc extensions |
| `returns` | `@return`/`@returns`; XML `<returns>`/`<value>`; DocC `Returns`; Rd `\value`; prose conventions |
| `yields` | JSDoc `@yields`; Python/Ruby conventions; Rust iterator prose; native language semantics |
| `errors` | Javadoc/Doxygen/PHPDoc/JSDoc `@throws`; XML `<exception>`; DocC `Throws`; Rust `# Errors`/`# Panics`; R/Go/Python conventions |
| `contracts` | Doxygen `@pre/@post/@invariant`; Rust Safety/Panics; DocC callouts; XML custom elements; prose headings |
| `examples` | Javadoc snippets; rustdoc code fences/doctests; Go `Example` tests; Python doctest; Rd `\examples`; XML `<example>/<code>`; RDoc/POD sections |
| `lifecycle.deprecated` | Javadoc `@deprecated` plus annotation; XML `<deprecated>` custom or language attributes; Go `Deprecated:`; PHPDoc/JSDoc tags; Rust `#[deprecated]`; DocC metadata/callout; Rd lifecycle |
| `seeAlso` | Javadoc `@see`; XML `<see>/<seealso>`; JSDoc/PHPDoc `@see`; Rd `\seealso`; POD `L<>`; rustdoc/Go/DocC symbol links |
| `relations` | compiler inheritance/implements; Javadoc uses/provides; JSDoc augments/mixes; symbol graphs; R S3/S4; SQL dependencies; Scratch block graph |
| `extensions` | custom Javadoc taglets; arbitrary XML; JSDoc/PHPDoc plug-ins; roxygen custom tags; DocC directives; formatter-specific POD; vendor metadata |

---

# Part II — Language adapter reports

Each report contains a machine-readable profile followed by implementation guidance. “Loss risk” estimates how likely a comment-only parser is to misrepresent documentation without the language/build model.


<a id="lang-python"></a>
## 13. Python

```yaml
profile_version: 1
language_id: python
rank: 1
primary_mechanism: runtime docstrings
authority: language-standard
convention_authority: informational-pep
native_syntax:
  - first string literal in module/function/class/method body
  - attribute docstring
  - additional docstring
native_markup: unspecified
common_markup_dialects: [reStructuredText/Sphinx, Google, NumPy]
preferred_symbol_source: Python AST plus static import/package resolution
preferred_doc_source: source AST
loss_risk_without_semantics: high
execution_required: false
```

### 13.1 Native strategy

Python gives docstrings a language-defined runtime role: the first string literal in a module, function, class, or method body becomes the object’s `__doc__`. PEP 257 standardizes high-level conventions—one-line versus multi-line form, indentation, summary style, and what public objects should document—but deliberately does **not** choose a field markup language.

PEP 257 also defines two statically extractable forms that do not become runtime `__doc__` values:

- **attribute docstrings** immediately following a simple assignment at module, class, or `__init__` scope; and
- **additional docstrings** immediately following another docstring.

A UDIR adapter SHOULD collect all three forms and distinguish them by native kind. It MUST NOT import modules merely to read `__doc__`; import-time code can mutate state, perform I/O, fail because optional dependencies are absent, or replace documentation dynamically.

```python
"""Package-level summary.

Longer package description.
"""

DEFAULT_TIMEOUT = 30
"""Default request timeout in seconds."""

def fetch(user_id: int) -> "User":
    """Return the requested user.

    Args:
        user_id: Numeric user identifier.

    Returns:
        The matching user.

    Raises:
        UserNotFoundError: If the user does not exist.
    """
```

Only the first literal in `fetch` is language-recognized. `Args`, `Returns`, and `Raises` above are Google-style conventions, not Python grammar and not PEP 257 semantics.

### 13.2 Markup and field dialects

The adapter SHOULD support multiple explicit docstring dialects:

| Dialect | Typical parameter form | Notes |
|---|---|---|
| Plain PEP 257 | prose only | No structured field grammar |
| reStructuredText/Sphinx | `:param name:`, `:type name:`, `:returns:`, `:rtype:`, `:raises Type:` | Roles and directives can be extended by Sphinx domains and plug-ins |
| Google style | `Args:`, `Returns:`, `Raises:` | Section grammar is convention/tool dependent |
| NumPy style | underlined `Parameters`, `Returns`, `Raises`, `Examples` sections | Richer tabular section conventions |
| Epydoc-style legacy | `@param`, `@type`, `@return` | Retain as dialect-specific input |
| Unknown/custom | arbitrary text | Parse as document AST and preserve headings |

Dialect detection MUST be conservative. A heading named “Returns” inside narrative prose is not necessarily a structured return field. Preferred order:

1. project configuration;
2. generator configuration (`conf.py`, `pyproject.toml`, etc.);
3. explicit file/module directive if one exists;
4. high-confidence syntax detection;
5. plain prose fallback.

### 13.3 Symbol and signature extraction

Use `ast` or a concrete-syntax/semantic parser to collect:

- modules, classes, functions, async functions, methods, nested definitions;
- decorators;
- positional-only, regular, keyword-only, variadic, and keyword-variadic parameters;
- defaults;
- annotations and type comments;
- generators/async generators;
- properties and descriptor-like decorators;
- overload declarations in stubs or guarded typing blocks;
- source spans.

Type annotations and `.pyi` stubs are code facts. Types repeated in docstrings become `documentedType`. For distribution-level identity, include package/distribution name and version when available because the same import path may exist in unrelated environments.

### 13.4 Binding and inheritance

Direct docstrings bind through AST structure. Runtime `inspect.getdoc` may clean indentation and inherit documentation for some descriptors/classes/methods; that resolved runtime behavior is not the same as direct source ownership. Store:

- direct source docstring;
- cleaned form used for rendering;
- inherited source ID and resolution algorithm, when expanded;
- any runtime-observed `__doc__` separately from static source.

Documentation assigned later—such as `function.__doc__ = "..."`—is dynamic. A static adapter SHOULD record the assignment as a possible mutation but MUST NOT execute it. Dynamically generated functions/classes may have no statically recoverable symbol or docstring.

### 13.5 Normalization rules

| Native construct | UDIR mapping |
|---|---|
| first sentence/line | `documentation.summary`, using configured dialect rules |
| remaining body | `documentation.description` |
| `Args` / `:param:` / NumPy Parameters | `documentation.parameters[]` |
| `Keyword Args` | parameters with `direction: keyword-only` when matched to AST |
| `Returns` | `documentation.returns[]` |
| `Yields` | `documentation.yields[]` |
| `Raises` / `:raises:` | `documentation.errors[]` |
| `Warns`, `Warnings`, `Notes` | named sections/admonitions |
| `Examples` / doctest prompts | `documentation.examples[]` |
| `Deprecated` directive/section | lifecycle or native extension depending dialect |
| Sphinx roles such as `:class:` | `symbol-link` with domain metadata |
| unknown directives/fields | native node plus `extensions.python.*` |

### 13.6 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| `python -OO` or optimized AST | Docstrings may be removed from bytecode/optimized AST | Parse original source; record missing-source condition |
| Runtime mutation of `__doc__` | Static and runtime docs differ | Keep runtime observation as separate provenance; never overwrite source |
| Decorators replace functions/classes | Displayed runtime object differs from syntactic declaration | Retain decorated declaration and optional resolved wrapper relation |
| `functools.wraps` | Wrapper copies metadata/docstring | Record generated/copied provenance if runtime analysis is enabled |
| `.py` and `.pyi` both exist | Signatures and docs may be split or conflict | Merge by semantic identity; retain source priority and diagnostics |
| `typing.overload` | Multiple signatures share one implementation | One overload record per signature plus overload group; define documentation inheritance policy |
| Properties/descriptors | Getter, setter, property object, and class attribute may each carry docs | Model accessor relations; do not collapse text without provenance |
| Dataclasses/attrs/Pydantic | Fields and constructors can be generated | Mark generated symbols and generator/version; source comments may bind to fields differently |
| C-extension/built-in object | Source AST unavailable; signatures may be text or Argument Clinic data | Use runtime/index metadata only in trusted environment; lower confidence |
| Attribute/additional docstrings | Not runtime-visible | Preserve as source documentation records |
| F-strings/concatenated literals | Not every expression is a valid docstring; compile-time literal concatenation can be | Follow Python AST/grammar, not token proximity |
| Name rebinding | Qualified name later points to another object | Identity remains declaration/source based; model aliases/rebinding separately |
| Nested/local definitions | Not public import API but still documentable | Use local scope path and visibility `local` |
| Sphinx autodoc import | May execute arbitrary module code | Do not use by default; sandbox if explicitly enabled |
| Sphinx `include`, `literalinclude`, custom roles | File traversal or plug-in execution | Allowlist roots; preserve unexpanded directive when blocked |
| Unicode indentation/tabs | Cleaning may differ between tools | Preserve raw bytes/text and separately store normalized clean form |

### 13.7 Adapter acceptance tests

A Python adapter MUST include fixtures for positional-only and keyword-only parameters, async generators, nested functions, properties, dataclasses, `.pyi` overloads, attribute docstrings, dynamic `__doc__` assignments, all supported field dialects, unresolved Sphinx roles, and source compiled with optimization.

### 13.8 Primary sources

- [PEP 257 — Docstring Conventions](https://peps.python.org/pep-0257/)
- [Python `pydoc`](https://docs.python.org/3/library/pydoc.html)
- [Python `doctest`](https://docs.python.org/3/library/doctest.html)
- [Python `inspect.getdoc`](https://docs.python.org/3/library/inspect.html#inspect.getdoc)
- [Python `ast`](https://docs.python.org/3/library/ast.html)
- [PEP 287 — reStructuredText Docstring Format](https://peps.python.org/pep-0287/)

---

<a id="lang-c"></a>
## 14. C

```yaml
profile_version: 1
language_id: c
rank: 2
primary_mechanism: none-at-language-level
authority: none
lexical_comments_authority: language-standard
de_facto_mechanism: Doxygen
vendor_semantic_mechanism: Clang documentation comments
native_markup: processor-dependent
preferred_symbol_source: compiler AST using exact compilation database
preferred_doc_source: compiler-attached raw comments plus Doxygen parser
loss_risk_without_semantics: critical
execution_required: false
```

### 14.1 Native strategy

The C language defines line and block comments lexically but does not assign documentation semantics to a special comment form. `/** ... */`, `/*! ... */`, `///`, and `//!` become documentation only because a tool adopts that convention.

Doxygen is the dominant cross-platform convention. Clang also parses and attaches many documentation-comment forms to declarations and exposes a structured comment AST. These are not interchangeable with an ISO C documentation standard, because no such standard exists.

```c
/**
 * Opens a connection.
 *
 * @param[in]  url       Endpoint URL.
 * @param[out] connection Receives the allocated connection.
 * @return 0 on success; a negative error code otherwise.
 * @pre url != NULL
 * @post On success, *connection owns a live handle.
 */
int connection_open(const char *url, struct connection **connection);
```

### 14.2 Doxygen model

Common Doxygen semantics relevant to UDIR include:

- brief and detailed descriptions;
- `@param` with optional `[in]`, `[out]`, or `[in,out]`;
- `@tparam` where a C-like extension or mixed C++ input applies;
- `@return` and `@retval`;
- `@throws`/`@exception`;
- contracts such as `@pre`, `@post`, and `@invariant`;
- notes, warnings, attention, remarks, todo, bugs;
- deprecation, version, since, authorship, copyright;
- links and references;
- code/example blocks;
- grouping/modules/pages;
- explicit entity commands including files, functions, variables, typedefs, structs, unions, enums, enum values, macros, and aliases;
- conditional sections and aliases configured in the Doxygen project.

Doxygen input is configurable. The same bytes can parse differently depending on `JAVADOC_AUTOBRIEF`, `QT_AUTOBRIEF`, `MARKDOWN_SUPPORT`, `ALIASES`, preprocessing settings, macro expansion, and source filters. The adapter MUST retain the effective Doxygen configuration or a digest of it.

### 14.3 Symbol extraction

A robust C adapter requires the exact compiler configuration:

- language standard/version;
- target triple and ABI;
- include paths;
- predefined and project macros;
- forced includes;
- source encoding;
- compiler extensions;
- preprocessing mode.

Use a compilation database where available. Parse original source and the selected preprocessed variant. The original file provides human source spans and comments; the semantic/preprocessed view provides declarations that actually exist.

Function signatures, pointer qualifiers, array parameter adjustment, attributes, calling conventions, linkage, visibility, typedef expansion, and nullability annotations MUST come from the compiler AST. A Doxygen type string is supplemental.

### 14.4 Binding rules

The adapter SHOULD use compiler-attached comments first. If an explicit Doxygen entity command documents a symbol away from its declaration, bind by that command and mark `explicit-tag`. A pure proximity parser is unsafe around:

- preprocessor branches;
- multiple declarators in one declaration;
- typedef declarations;
- macro-generated declarations;
- forward declarations and definitions;
- comments between attributes/specifiers and declarations.

When documentation exists on both a prototype and definition, retain both fragments. Apply a project-configurable merge policy; do not assume one universally wins.

### 14.5 Normalization rules

| Doxygen/native construct | UDIR mapping |
|---|---|
| brief description | `summary` |
| detailed description | `description` |
| `@param[in/out/in,out]` | parameter docs plus direction |
| `@return` | general return description |
| `@retval value` | return entry with condition/value metadata |
| `@pre`, `@post`, `@invariant` | contracts |
| `@exception`, `@throws` | errors, even though C has no language exceptions |
| `@warning`, `@note`, `@attention`, `@remark` | admonition/section |
| `@deprecated`, `@since`, `@version` | lifecycle |
| `@defgroup`, `@ingroup`, `@addtogroup`, `@{`, `@}` | grouping relations |
| `@file`, `@fn`, `@var`, `@typedef`, `@def`, etc. | explicit virtual/real symbol binding |
| custom alias/command | native extension |
| macro definition documentation | symbol kind `macro` with build variant |

### 14.6 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Multiple declarators (`int a, b;`) | One comment ambiguously documents two symbols | Use compiler comment attachment; split only with explicit native evidence |
| Prototype and definition | Duplicate or divergent docs | Preserve both with source role; configurable resolution |
| `typedef struct {...} Name` | Tag name and typedef name are distinct identities | Create separate symbols and alias/correspondence relations |
| Anonymous struct/union/enum | No stable source name | Derive scoped synthetic identity from owner/span; mark unstable |
| Function-pointer typedef | Parameter docs may describe callback arguments, not typedef declaration parameters | Model callable type signature separately |
| Macros | Token substitution can create symbols or change signatures | Capture macro definition and expansion provenance; one variant per macro environment |
| Conditional compilation | Mutually exclusive APIs collapse incorrectly | Extract build variants; never merge differing declarations |
| Include-generated API | Original ownership spans multiple headers | Preserve include stack and generated status |
| Comments inside disabled branches | Text exists but symbol does not in selected build | Store optional unbound/disabled annotation or extract another variant |
| C attributes/annotations | Tool may ignore contracts/availability | Extract from AST and merge with docs |
| `_Generic`, compiler builtins | Apparent function-like API is not an ordinary function | Preserve native kind/extension |
| `#define` line continuations | Comment may terminate replacement list or be stripped | Follow preprocessing/token rules |
| Doxygen aliases | Custom commands change grammar | Load configuration safely; unknown aliases preserved |
| Doxygen preprocessing | Documentation may depend on macro expansion | Record preprocessing mode and raw pre-expansion text |
| C23/C compiler differences | Same source accepted differently | Pin compiler/version and standard |
| Header copied into several libraries | Same qualified C name has different ABI/build meaning | Include package/library/target identity |

### 14.7 Recommended adapter stack

1. Clang tooling or an equivalent compiler frontend for symbols and attached comments.
2. Compilation database (`compile_commands.json`) or an explicit build matrix.
3. Doxygen-compatible comment parser configured per project.
4. Optional Doxygen XML ingestion as a secondary representation.
5. Object/debug metadata correlation when ABI-level identity matters.

Do not invoke arbitrary Doxygen input filters or preprocessing commands in an untrusted repository.

### 14.8 Primary sources

- [Doxygen — Documenting the code](https://www.doxygen.nl/manual/docblocks.html)
- [Doxygen — Special commands](https://www.doxygen.nl/manual/commands.html)
- [Clang documentation comments](https://clang.llvm.org/docs/InternalsManual.html#comment-handling)
- [ISO C language standards information](https://www.open-std.org/jtc1/sc22/wg14/)

---

<a id="lang-cpp"></a>
## 15. C++

```yaml
profile_version: 1
language_id: cpp
rank: 3
primary_mechanism: none-at-language-level
authority: none
lexical_comments_authority: language-standard
de_facto_mechanism: Doxygen
vendor_semantic_mechanism: Clang documentation comments
native_markup: processor-dependent
preferred_symbol_source: compiler AST using exact build configuration
preferred_doc_source: compiler-attached raw comments plus configured parser
loss_risk_without_semantics: critical
execution_required: false
```

### 15.1 Native strategy

Like C, C++ standardizes comments as lexical elements but does not define a Javadoc-equivalent semantic documentation language. Doxygen and compiler/IDE documentation-comment support form the practical ecosystem standard. The C profile applies, but C++ multiplies identity, overload, template, inheritance, and lookup edge cases.

```cpp
/**
 * Parses a value from text.
 *
 * @tparam T destination type
 * @param text UTF-8 input
 * @return parsed value
 * @throws parse_error if the input is invalid
 */
template <typename T>
requires Parsable<T>
T parse(std::string_view text);
```

### 15.2 Semantic extraction

The compiler semantic model MUST be authoritative for:

- namespace and module membership;
- class/struct/union/enum identity;
- overload sets and exact signatures;
- cv/ref qualifiers, `noexcept`, explicit object parameters, calling conventions;
- templates, parameter packs, concepts, constraints, and specializations;
- aliases and using-declarations;
- virtual overrides, final overriders, and inheritance;
- friends;
- operators, conversion functions, constructors, destructors, deduction guides;
- defaulted/deleted functions;
- attributes and availability;
- exported module declarations.

A string-only signature parser is inadequate. It will misidentify overloads, dependent types, anonymous namespaces, local types, aliases, and template specializations.

### 15.3 Documentation inheritance and duplication

Doxygen can inherit documentation under configured rules, and compilers can expose overridden relationships. UDIR SHOULD store direct docs on each declaration, then materialize inherited docs only as a resolved view with an inheritance path.

Declarations may occur repeatedly:

- primary declaration;
- redeclarations;
- out-of-line definition;
- explicit specialization;
- generated instantiation;
- friend declaration;
- module interface and implementation partitions.

Collect all comment-bearing redeclarations. Choose a display fragment through an explicit policy, preferably the canonical declaration or public header/module interface, while retaining alternatives and diagnostics.

### 15.4 C++-specific normalization

| Native construct | UDIR mapping |
|---|---|
| `@tparam` | type-parameter documentation matched by semantic parameter identity |
| concept/requires prose | constraints plus description; machine semantics from AST |
| class template specialization | separate symbol related by `specializes` |
| overload group docs | optional grouping record; never substitute for overload-specific docs |
| `@copydoc`, `@copybrief`, `@copydetails`, inherited docs | relation plus resolved fragment |
| operators/conversions | exact symbol kind and compiler-native ID |
| `@relates`, `@relatesalso` | relation/grouping extension |
| modules/groups/pages | module or conceptual records |
| exception specification + `@throws` | actual declared spec and documented errors kept separately |

### 15.5 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Overloaded functions | Name-only references bind incorrectly | Resolve with compiler USR/signature; ambiguous links remain ambiguous |
| Function templates and specializations | Docs on primary are copied as if exact | Model primary/specialization relation and provenance |
| Constrained overloads | Same surface signature differs by requires-clause | Include normalized constraints in canonical identity |
| Operators and conversion functions | Ad hoc parser mangles names | Use compiler-native symbol kind/USR |
| Constructors/destructors | Return fields are nonsensical; implicit members may exist | Validate fields by symbol kind; mark implicit/generated |
| Defaulted/deleted functions | API exists but behavior/availability differs | Preserve modifiers and docs; renderer labels status |
| Friend declarations | Lexical owner differs from semantic namespace/class association | Preserve lexical and semantic containers |
| Inline/out-of-line definitions | Comment may be attached at either location | Aggregate redeclarations with source roles |
| Anonymous namespaces/types | IDs unstable across moves | Use compiler identity plus file/module digest; mark internal |
| Type aliases and alias templates | Docs may describe alias or target | Keep separate symbols with `aliases` |
| Using-declaration/re-export | Docs may surface imported member | Model reexport relation; avoid copying ownership |
| Multiple/virtual inheritance | Naive inherited-doc order is ambiguous | Follow processor’s exact algorithm; retain candidate path |
| Covariant return/overrides | Base docs may not describe derived result | Keep actual signature and inherited prose provenance |
| Modules and header units | File path no longer equals API module | Use module ownership from compiler build |
| Macros generating declarations | Source spans and docs originate at invocation/definition | Preserve expansion stack and generated status |
| Explicit instantiations | May not be independently documentable | Relate to template; follow selected generator visibility |
| Concepts | Some tools treat them as pages/functions | Use native symbol kind `concept`; preserve generator fallback |
| Lambdas/local classes | No portable user-facing name | Synthetic scoped identity; usually non-public |
| `decltype(auto)`, dependent types | Type unavailable until instantiation | Store dependent/raw type and compiler resolution state |
| ABI vs source signature | Linkage identity may differ by ABI/toolchain | Preserve mangled/linkage name when relevant |
| Doxygen + Clang parse differences | Tags/Markdown/attachment differ | Record processor and maintain separate native ASTs when needed |

### 15.6 Adapter acceptance tests

Fixtures MUST cover overloads, all operator categories, conversion functions, templates and partial/explicit specializations, concepts, parameter packs, aliases, multiple inheritance, redeclarations across headers and modules, macro-generated declarations, conditional compilation, anonymous namespaces, and Doxygen custom aliases.

### 15.7 Primary sources

- [C++ working draft — comments](https://eel.is/c++draft/lex.comment)
- [Doxygen — Documenting the code](https://www.doxygen.nl/manual/docblocks.html)
- [Doxygen — Special commands](https://www.doxygen.nl/manual/commands.html)
- [Clang documentation comments](https://clang.llvm.org/docs/InternalsManual.html#comment-handling)
- [Clang LibTooling](https://clang.llvm.org/docs/LibTooling.html)

---

<a id="lang-java"></a>
## 16. Java

```yaml
profile_version: 1
language_id: java
rank: 4
primary_mechanism: Javadoc standard doclet documentation comments
authority: compiler-vendor-official
ecosystem_status: de-facto-standard-syntax
native_syntax:
  - traditional /** ... */
  - markdown /// consecutive lines
native_markup:
  traditional: HTML plus inline/block tags
  markdown: CommonMark/GFM-table-oriented Markdown plus HTML and tags
preferred_symbol_source: javac language model / DocTrees
preferred_doc_source: DocTrees or standard-doclet-compatible parser
loss_risk_without_semantics: high
execution_required: false
```

### 16.1 Native strategy

JDK 26 recognizes two Javadoc documentation-comment forms:

1. traditional comments beginning `/**`; and
2. Markdown documentation comments made from consecutive `///` lines.

Both can occur in the same source file. Only one comment is recognized for a declaration, and when several precede it, the closest is used. Comments must appear immediately before documentable declarations, before annotations and modifiers.

Traditional comments combine HTML with Javadoc inline and block tags. Markdown comments combine Markdown, allowed HTML, and the same tag system. The main description precedes all block tags; after the first block tag, the main description cannot resume.

```java
/// Retrieves a user.
///
/// The result is cached for the lifetime of the request.
///
/// @param id the user identifier
/// @return the matching user
/// @throws UserNotFoundException if no user exists
public User getUser(long id) { ... }
```

### 16.2 Standard tags in JDK 26

The standard doclet specifies the following tags. Adapters MUST retain tag spelling, placement, and version because some tags are valid only on certain declaration kinds.

| Category | Tags |
|---|---|
| Identity/metadata | `@author`, `@version`, `@since`, `@deprecated`, `@hidden` |
| Parameters/results/errors | `@param`, `@return`, `@throws`, `@exception` |
| Links/text | `{@link}`, `{@linkplain}`, `@see`, `{@code}`, `{@literal}`, `{@summary}`, `{@index}` |
| Paths/values/properties | `{@docRoot}`, `{@value}`, `{@systemProperty}` |
| Inheritance | `{@inheritDoc}` |
| Snippets | `{@snippet}` and snippet markup directives |
| Modules/services | `@uses`, `@provides` |
| Specifications | `@spec` |
| Serialization | `@serial`, `@serialData`, `@serialField` |

Custom tags/taglets are permitted by the tool ecosystem. Common OpenJDK/source conventions such as `@apiNote`, `@implSpec`, and `@implNote` must be recorded as configured/custom tags unless the selected doclet formally defines them. Do not mislabel them as universally standard Javadoc tags.

### 16.3 Documentable locations and auxiliary files

Javadoc comments can document modules, packages, classes/interfaces/enums/records/annotation interfaces, constructors, methods, fields, enum constants, and annotation elements. Package and conceptual documentation may additionally come from:

- `package-info.java`;
- `module-info.java`;
- legacy `package.html`;
- `doc-files/*.html`;
- `doc-files/*.md`;
- overview files and custom doclet inputs.

UDIR SHOULD emit package/module/conceptual records for these inputs and preserve whether content was treated as traditional HTML or Markdown.

### 16.4 Summary and method completeness

The standard doclet normally derives the summary from the first sentence of the main description; `{@summary ...}` can define it explicitly. Store both the selected summary and the native selection mechanism.

For methods, the current standard doclet specification defines expected documentation items: main description, each type parameter, each formal parameter, non-void return, and declared exceptions. Overriding methods can inherit missing items by omission under the doclet’s rules. Therefore a comment containing only one local parameter description can still resolve to a complete rendered method page.

UDIR MUST distinguish:

- direct item text;
- implicit inheritance by omission;
- explicit `{@inheritDoc}`;
- unresolved/missing items;
- the overridden source selected by the doclet.

Do not apply method inheritance rules to fields, constructors, types, or arbitrary similarly named members.

### 16.5 References and symbol identity

Javadoc references can identify modules, packages, types, fields, constructors, and methods, with parameter lists to disambiguate overloads. The adapter SHOULD resolve references through `javac`/`DocTrees` using the same module path, class path, source path, and release settings as the documentation build.

Canonical identity SHOULD include:

- module when using the Java Platform Module System;
- package;
- binary type name, including nested type encoding;
- member kind;
- erased/JVM descriptor or semantically equivalent overload signature;
- release/build variant.

Generic display signatures and binary identity are separate. A method reference cannot directly address an individual parameter or record component using the normal member-reference form; preserve fragment or tag-specific semantics.

### 16.6 Snippets

`{@snippet}` can contain inline code or refer to external files/regions. Snippet markup can highlight, replace, link, and define regions. It is a structured mini-language, not merely a code fence.

A secure adapter MUST:

- retain inline or external source origin;
- resolve external paths only inside allowlisted roots;
- preserve regions and directives as native nodes;
- forbid overlapping link transformations where the native processor reports an error;
- avoid compiling or running snippets by default;
- record JDK introduction/version constraints.

### 16.7 Normalization rules

| Javadoc construct | UDIR mapping |
|---|---|
| main-description first sentence | summary with selection metadata |
| rest of main description | description |
| `@param name` | parameter docs |
| `@param <T>` | type-parameter docs |
| `@return` | return docs |
| `@throws`/`@exception` | error docs; retain preferred/deprecated spelling |
| `@deprecated` | lifecycle message; correlate with `@Deprecated` annotation |
| `@since` | lifecycle `since` |
| `@see`, `{@link}`, `{@linkplain}` | see-also/symbol-link |
| `{@inheritDoc}` / omission | inheritance relation and resolved fragment |
| `{@snippet}` | example/code AST plus directives |
| `@spec` | link target kind `specification` |
| `@uses`, `@provides` | service relations |
| serialization tags | named semantic extensions under `extensions.java.serialization` |
| unknown/custom taglet | lossless native extension |

### 16.8 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Traditional and Markdown comments | Parser assumes one grammar | Detect per comment and record syntax |
| Multiple comments before declaration | Tool merges all | Use closest recognized comment; preserve ignored comments as ordinary annotations if requested |
| Annotation between comment and declaration | Naive adjacency fails | Follow Javadoc declaration rule via parser |
| Markdown comment interrupted by blank/non-`///` line | Comment terminates unexpectedly | Follow lexical grouping exactly |
| `*/` in traditional content | Terminates comment | Preserve source parse error; Markdown form can represent such text |
| Block tag after prose | Main description cannot resume | Parse tag boundaries before markup rendering |
| Custom taglets | Semantics unavailable without plug-in | Preserve tag; optionally run allowlisted taglet in sandbox |
| `@deprecated` without annotation, or annotation without tag | Source/API lifecycle views disagree | Preserve both mechanisms and report policy diagnostic |
| Records | Components, accessors, canonical constructor, and fields can be related/generated | Model generated relations; do not duplicate one comment indiscriminately |
| Annotation elements | Default values and return semantics differ from ordinary methods | Use exact symbol kind |
| Enum constants | Field-like but separately documentable | Use `enum-case` |
| Overloaded link | Reference omits parameters and is ambiguous | Retain candidates and emit link ambiguity |
| Generic/varargs reference spelling | Native reference grammar differs from source display | Resolve with javac, preserve original spelling |
| Module/package doc files | Ownership differs from class comments | Emit module/package/conceptual records |
| Inheritance by omission | Text appears direct after rendering | Mark each inherited field with path/provenance |
| Unchecked exceptions | May be documented without `throws` clause | Keep documented errors; do not declare them missing solely from signature |
| External snippet files | Traversal and source drift | Allowlist, hash, and source-map included region |
| Raw HTML | XSS or invalid HTML | Sanitize for output; retain original node and loss flag |
| DocLint differences | Tool/release produces different diagnostics | Pin JDK/doclet version and store diagnostics |
| Generated sources/annotation processors | API may exist only after generation | Extract generated-source variant with generator provenance |
| Multi-release JAR/source | Same API name differs by release | Variant by release |
| `package.html` legacy | Different reference-resolution requirements | Record legacy source kind and fully qualified reference behavior |

### 16.9 Recommended adapter stack

Use the JDK compiler language model, `DocTrees`, and the standard-doclet parsing behavior for the selected JDK. Prefer source extraction over scraping generated HTML. Generated HTML is a presentation artifact and may lose direct/inherited distinction, raw tags, source spans, and custom doclet semantics.

### 16.10 Primary sources

- [JDK 26 Javadoc Documentation Comment Specification](https://docs.oracle.com/en/java/javase/26/docs/specs/javadoc/doc-comment-spec.html)
- [Javadoc tool specification](https://docs.oracle.com/en/java/javase/26/docs/specs/man/javadoc.html)
- [Markdown documentation comments](https://docs.oracle.com/en/java/javase/26/javadoc/using-markdown-documentation-comments.html)
- [DocLint](https://docs.oracle.com/en/java/javase/26/docs/specs/man/javadoc.html#doclint)

---

<a id="lang-csharp"></a>
## 17. C#

```yaml
profile_version: 1
language_id: csharp
rank: 5
primary_mechanism: compiler XML documentation comments
authority: compiler-vendor-official
native_syntax:
  - "/// XML fragments"
  - "/** XML fragments */"
native_markup: XML with compiler-recognized and custom elements
preferred_symbol_source: Roslyn semantic model
preferred_doc_source: Roslyn syntax/semantic APIs plus compiler XML output
loss_risk_without_semantics: high
execution_required: false
```

### 17.1 Native strategy

C# documentation comments are structured XML associated with the following declaration. The compiler can emit an XML documentation file that combines comment content with compiler-generated API identity strings. `///` is the conventional form; `/** ... */` is also supported with defined prefix-trimming behavior.

```csharp
/// <summary>Retrieves a user.</summary>
/// <param name="id">The user identifier.</param>
/// <returns>The matching user.</returns>
/// <exception cref="UserNotFoundException">
/// Thrown when no user exists.
/// </exception>
public User GetUser(long id) => ...;
```

The XML file is not embedded as runtime assembly metadata. It normally ships beside the assembly. Reflection alone therefore cannot recover it.

### 17.2 Recommended XML vocabulary

The compiler and IDE understand a common vocabulary, while well-formed custom XML is allowed. A UDIR adapter SHOULD recognize:

| Purpose | Elements |
|---|---|
| Summary/details | `<summary>`, `<remarks>`, `<para>` |
| Parameters/generics | `<param name="">`, `<paramref name="">`, `<typeparam name="">`, `<typeparamref name="">` |
| Results/properties | `<returns>`, `<value>` |
| Errors | `<exception cref="">` |
| Examples/code | `<example>`, `<code>`, `<c>` |
| References | `<see cref="">`, `<see href="">`, `<seealso cref="">`, `<seealso href="">` |
| Lists | `<list>`, `<listheader>`, `<item>`, `<term>`, `<description>` |
| External composition | `<include file="" path="">` |
| Inheritance | `<inheritdoc/>` or `<inheritdoc cref="" path="">`, as supported by consuming tools |
| Permissions/legacy consumers | `<permission cref="">` |
| Arbitrary extensions | any well-formed XML not reserved by a processor |

Not all consumers render every element identically. The compiler validates and transforms selected attributes such as `name` and `cref`; downstream tools such as DocFX, Sandcastle, IDEs, or custom processors decide presentation and may implement additional behavior.

### 17.3 Compiler XML member IDs

The emitted XML uses a flat `<members>` list. Each `<member name="...">` has a compiler-defined ID beginning with:

- `N:` namespace;
- `T:` type;
- `F:` field;
- `P:` property/indexer;
- `M:` method, constructor, or operator;
- `E:` event; or
- `!:` an error/unresolved form.

Member IDs encode fully qualified names and, where needed, parameter types, arrays, pointers, by-reference forms, generic arity, conversion return types, and other signature details. UDIR MUST retain the exact compiler ID as `symbol.nativeId`; it is the primary bridge from XML to .NET metadata.

The XML ID alone is not globally unique. Add assembly name, version/public key identity where relevant, target framework, and build configuration to the canonical identity.

### 17.4 Roslyn-first extraction

Prefer Roslyn over hand parsing. Roslyn provides:

- exact declaration/symbol binding;
- overload identities;
- partial declaration relationships;
- explicit interface implementations;
- nullable annotations;
- records, primary constructors, synthesized symbols;
- source spans and syntax trivia;
- XML documentation diagnostics;
- resolved `cref` targets.

The compiler XML output remains valuable because it captures the compiler’s canonical IDs and normalization. Keep both the source-native XML fragment and emitted member XML.

### 17.5 Merge and inheritance

Partial declarations and documentation inheritance require native rules:

- Documentation on partial types may be combined from parts by compiler/tool behavior.
- For partial methods/members, defining and implementing declarations can have different precedence.
- `<inheritdoc>` is a directive whose expansion is typically performed by documentation consumers rather than represented as ordinary compiler metadata.
- Interface/base documentation may be inherited by IDEs or generators even when the source element is empty.

UDIR MUST store direct fragments separately and represent expansion through `relations` and `provenance.inheritancePath`. It SHOULD store the processor name/version that selected or expanded text.

### 17.6 Normalization rules

| XML/native construct | UDIR mapping |
|---|---|
| `<summary>` | summary |
| `<remarks>` and top-level additional prose | description/named sections |
| `<param>` | parameter docs matched against Roslyn parameter symbol |
| `<typeparam>` | type-parameter docs |
| `<returns>` | returns |
| `<value>` | property value/return-like documentation |
| `<exception cref>` | documented error plus resolved symbol link |
| `<example>`/`<code>` | examples/code AST |
| `<see>` | inline link |
| `<seealso>` | see-also |
| `<paramref>`/`<typeparamref>` | typed local reference nodes |
| `<list>` | list/table-like doc AST preserving native list type |
| `<include>` | include directive plus optionally resolved fragment |
| `<inheritdoc>` | inheritance directive/relation |
| custom element | native/extension node |
| language attributes such as `[Obsolete]` | lifecycle facts merged from semantic model, not XML alone |

### 17.7 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Invalid XML | Compiler/generator drops or mangles content | Preserve raw fragment and compiler diagnostics; emit best-effort native tree |
| Bare `///` text without elements | XML output exists but common consumers ignore it | Preserve as raw/top-level prose with consumer-compatibility warning |
| `cref` generic syntax | Angle brackets require escaping or braces | Resolve through Roslyn; preserve original attribute |
| Overloads/operators/conversions | Name-only identity collides | Use compiler ID and symbol |
| Indexers | Property ID carries parameters | Keep `property`/`subscript` kind and exact ID |
| Explicit interface implementation | Names contain dots encoded specially in XML IDs | Preserve native compiler ID; model implementation relation |
| Nullable reference types | Legacy XML IDs do not fully encode nullable intent | Store nullable type from Roslyn signature |
| `ref`, `in`, `out`, `scoped`, ref returns | Direction/metadata can be lost in prose | Extract from semantic model and preserve ID representation |
| Tuples | Element names may not be fully represented in IDs | Store tuple type AST and metadata separately |
| Records/primary constructors | Parameters create properties and synthesized members | Model generated relations and direct doc targets |
| Partial type/member | Fragments may concatenate or one part may win | Follow compiler/tool rules and retain each source fragment |
| Extension methods/members | Display owner differs from declaring static type | Store declaration owner plus extension-target relation |
| File-local types | Names and visibility have special source scope | Include file identity and visibility |
| Source generators | Documented APIs may exist only in generated source | Mark generated, include generator/version and generated span |
| Multi-target project | APIs/docs vary by TFM/preprocessor symbols | Emit variants keyed by target framework/configuration |
| `<include>` XPath | Reads external XML and selects arbitrary nodes | Root-allowlist, size limits, safe XML parser, no external entities |
| XML external entities | XXE/resource attacks | Disable DTD/external entities |
| `<inheritdoc>` processor differences | Different generators resolve different source/path | Record processor and preserve unresolved directive |
| Custom tags | One renderer understands, another loses semantics | Preserve full XML subtree in extensions |
| Documentation file absent at runtime | Reflection ingestion assumes docs are metadata | Treat XML as separate build artifact; report absence |

### 17.8 Primary sources

- [Microsoft — Generate XML API documentation comments](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/xmldoc/)
- [Microsoft — Recommended XML documentation tags](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/xmldoc/recommended-tags)
- [C# language specification — Documentation comments](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/documentation-comments)
- [Roslyn .NET Compiler Platform SDK](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/)

---

<a id="lang-javascript"></a>
## 18. JavaScript

```yaml
profile_version: 1
language_id: javascript
rank: 6
primary_mechanism: none-at-ECMAScript-level
authority: none
lexical_comments_authority: language-standard
de_facto_mechanism: JSDoc
native_syntax: "/** ... */ immediately associated by JSDoc parser"
native_markup: JSDoc tags plus processor-configured text/Markdown
preferred_symbol_source: ECMAScript/TypeScript-capable AST and module resolver
preferred_doc_source: JSDoc parser/doclets plus raw AST comments
loss_risk_without_semantics: critical
execution_required: false
```

### 18.1 Native strategy

ECMAScript defines comments but no semantic API-documentation format. JSDoc is the long-established de facto system. JSDoc recognizes blocks beginning exactly `/**`; ordinary `/*` blocks and blocks beginning `/***` are not JSDoc documentation blocks.

```javascript
/**
 * Retrieves a user.
 *
 * @param {number} id - The user identifier.
 * @returns {Promise<User>} The matching user.
 * @throws {UserNotFoundError} If no user exists.
 */
export async function getUser(id) {}
```

Unlike Java, JSDoc commonly carries type information because JavaScript declarations may not. In TypeScript projects, those types may duplicate or conflict with compiler types.

### 18.2 JSDoc semantic surface

A production adapter SHOULD recognize the current JSDoc tag vocabulary and maintain a versioned registry. Major categories include:

| Category | Representative tags |
|---|---|
| Values and signatures | `@param`, `@returns`/`@return`, `@yields`/`@yield`, `@throws`/`@exception`, `@type`, `@this`, `@template` |
| Virtual types | `@typedef`, `@callback`, `@enum`, `@property` |
| Entity declaration | `@class`, `@constructor`, `@function`, `@method`, `@interface`, `@namespace`, `@module`, `@file`, `@event`, `@external` |
| Name/ownership | `@name`, `@memberof`, `@inner`, `@instance`, `@static`, `@global`, `@lends`, `@constructs` |
| Relations | `@augments`/`@extends`, `@implements`, `@mixes`, `@mixin`, `@borrows`, `@fires`/`@emits`, `@listens` |
| Visibility/behavior | `@public`, `@protected`, `@private`, `@package`, `@readonly`, `@constant`, `@abstract`, `@virtual`, `@override`, `@async`, `@generator` |
| Lifecycle/metadata | `@deprecated`, `@since`, `@version`, `@author`, `@license`, `@copyright`, `@todo` |
| Navigation/examples | `@see`, `@link` inline forms, `@example`, `@tutorial` |
| Inheritance | `@inheritdoc` |
| Plug-in/custom | processor-defined tags and dictionary changes |

The exact tag set and parsing behavior depend on JSDoc version, tag dictionary, templates, and plug-ins. Store that environment.

### 18.3 Type grammar

JSDoc supports a type-expression grammar with names, unions, nullable/non-nullable markers, optional/default parameters, variadics, functions, arrays/objects, generics, record-like structures, and other forms. In practice there are overlapping dialects:

- JSDoc’s own interpretation;
- Closure Compiler annotations;
- TypeScript’s JSDoc support;
- IDE-specific extensions;
- project-specific tags.

The adapter MUST preserve the raw type expression and identify the parser/dialect. It SHOULD normalize into UDIR’s type AST only when the grammar is known. Unknown forms remain `raw`.

When TypeScript or another semantic compiler is present:

- compiler type is `signature.*.type`;
- JSDoc type is `documentedType`;
- conflicts produce diagnostics;
- JSDoc-only virtual types remain valid documentation symbols.

### 18.4 Symbols and modules

JavaScript’s runtime and module dynamism make symbol identity difficult. Support at least:

- ECMAScript modules: named/default exports, aliases, star reexports, namespace objects;
- CommonJS: `module.exports`, `exports.name`, reassignment patterns;
- class methods, fields, private fields, accessors, static blocks;
- prototype assignments;
- object-literal members;
- function declarations/expressions/arrows;
- destructuring;
- virtual `@name`, `@typedef`, and `@callback` symbols;
- package `exports` maps and conditional exports.

The adapter SHOULD distinguish lexical declaration, exported API path, and runtime property path. One declaration can have several exports; one export can reexport another declaration.

### 18.5 JSDoc doclets

`jsdoc -X`/`--explain` can emit JSON doclets. These are useful as a native processor output, but not sufficient alone:

- plug-ins can mutate them;
- source spans and raw syntax may be incomplete;
- dynamic code patterns may be inferred differently by version;
- a template may apply further behavior;
- type strings still need dialect handling.

Store doclets under `native.parsed`, then correlate them with an AST and module graph.

### 18.6 Normalization rules

| JSDoc construct | UDIR mapping |
|---|---|
| description/summary | summary and description per configured sentence/paragraph policy |
| `@param` including property paths | parameter docs; nested property docs retained as structured path |
| `@returns` | returns |
| `@yields` | yields |
| `@throws` | errors |
| `@template` | type parameters |
| `@typedef`, `@callback` | virtual symbol record |
| `@property` | child documented property, not necessarily runtime field |
| `@module`, `@namespace`, `@file` | module/namespace/file record |
| `@name`, `@memberof`, `@lends` | explicit binding/ownership directives |
| `@augments`, `@implements`, `@mixes` | relations |
| access tags | visibility claim; compare with actual/private syntax |
| `@deprecated`, `@since`, `@version` | lifecycle |
| `@example` | example |
| `@see`, inline links, tutorials | links/conceptual relationships |
| custom/unknown tag | extension |

### 18.7 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| ESM versus CommonJS | Same file interpreted under wrong module system | Use package type/extensions/build config |
| Conditional package exports | One package exposes different APIs by condition | Create export variants |
| Default export anonymous declaration | No stable source name | Use export-path identity plus source span |
| Reexports/star exports | Docs copied to wrong owner or cycles | Build module graph; retain declaration and reexport relation |
| `@name` virtual symbols | No AST declaration exists | Create virtual symbol with explicit-tag binding |
| Prototype assignment | Method owner missed by declaration-only parser | Analyze assignment patterns with confidence |
| Computed property names | Static name may be unavailable | Resolve literals/constants when safe; otherwise dynamic/raw |
| Private `#field` vs `@private` | Language privacy and documentation visibility differ | Preserve both actual and claimed visibility |
| Destructured parameters | JSDoc names paths while AST parameter is a pattern | Build binding tree; map nested paths without inventing formal parameters |
| Default/rest/optional syntax | Type grammar and runtime signature encode differently | Preserve actual parameter flags and native JSDoc notation |
| Multiple `@augments` | JavaScript permits mixin-like models; generator behavior varies | Store every relation and processor semantics |
| `@lends`/object literals | Comment changes ownership interpretation | Retain explicit directive and source object relation |
| Overload tags/tool extensions | Core JS has no overload declarations | Store processor-specific overload records and link to implementation |
| TypeScript JSDoc | Compiler recognizes only a subset/its own rules | Set dialect `typescript-jsdoc` and use TypeScript compiler |
| Decorator/transpiler output | Source API differs from emitted runtime | Keep source and emitted variants with generated-from relation |
| Minified/generated bundles | Comments may be stripped or license-only | Prefer original source maps/source package |
| Plug-ins/tag dictionaries | Same block yields different doclets | Record configuration and disable untrusted plug-ins |
| Markdown/HTML in descriptions | XSS and parser variance | Parse declared markup, sanitize output, preserve raw |
| Global script collisions | Identical names in different script realms | Include project/script container identity |
| Dynamic `Object.defineProperty`/Proxy | API unobservable statically | Mark dynamic-unobservable; optional trusted runtime probe separate |
| Function reassignment/aliasing | Documentation ownership changes at runtime | Keep declaration docs and model obvious aliases separately |

### 18.8 Primary sources

- [ECMAScript lexical grammar — Comments](https://tc39.es/ecma262/multipage/ecmascript-language-lexical-grammar.html#sec-comments)
- [JSDoc — Getting started](https://jsdoc.app/about-getting-started)
- [JSDoc — Block and inline tags](https://jsdoc.app/#block-tags)
- [JSDoc — Command-line arguments](https://jsdoc.app/about-commandline)
- [JSDoc — Configuring JSDoc](https://jsdoc.app/about-configuring-jsdoc)

---

<a id="lang-visual-basic"></a>
## 19. Visual Basic (.NET)

```yaml
profile_version: 1
language_id: visual-basic-dotnet
rank: 7
primary_mechanism: compiler XML documentation comments
authority: compiler-vendor-official
native_syntax: "three-apostrophe XML fragments"
native_markup: XML with compiler-recognized and custom elements
preferred_symbol_source: Roslyn Visual Basic semantic model
preferred_doc_source: Roslyn plus compiler XML output
loss_risk_without_semantics: high
execution_required: false
```

### 19.1 Native strategy

Visual Basic uses three apostrophes followed by XML content:

```vb
''' <summary>Retrieves a user.</summary>
''' <param name="id">The user identifier.</param>
''' <returns>The matching user.</returns>
''' <exception cref="UserNotFoundException">
''' No user exists.
''' </exception>
Public Function GetUser(id As Long) As User
    ' ...
End Function
```

Adjacent XML documentation lines form one comment associated with the following declaration. The compiler emits the same broad .NET XML documentation artifact model used by C#, including compiler-generated member IDs, but parsing and name resolution follow Visual Basic language rules.

Ordinary apostrophe comments and `REM` comments are not XML documentation comments.

### 19.2 XML vocabulary and normalization

Use the same semantic mapping as C# for `<summary>`, `<remarks>`, `<param>`, `<typeparam>`, `<returns>`, `<value>`, `<exception>`, `<example>`, `<code>`, `<see>`, `<seealso>`, `<paramref>`, `<typeparamref>`, `<list>`, `<include>`, and consumer-supported `<inheritdoc>`.

The adapter MUST preserve the language as Visual Basic, even when the emitted XML member ID points to the same CLR metadata shape as equivalent C#. Source syntax and semantic concepts are not identical.

### 19.3 Visual Basic symbol concerns

Use the Visual Basic Roslyn semantic model for:

- case-insensitive name binding;
- bracketed identifiers;
- modules;
- default/indexed properties;
- events and custom events;
- delegates;
- `ByRef`, `Optional`, and `ParamArray`;
- generic constraints;
- extension methods;
- operators and widening/narrowing conversions;
- partial classes and methods;
- interface implementation clauses;
- source line mappings;
- generated designer code.

Canonical identity should still correlate with CLR metadata and compiler XML ID, while display names preserve VB spelling.

### 19.4 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Case-insensitive identifiers | Case-sensitive matcher reports orphan params/links | Match using VB symbol identity; retain original spelling |
| Bracketed identifiers | Raw name includes syntax delimiters | Store semantic name and source display separately |
| Default properties/indexers | Rendered as ordinary property, losing invocation semantics | Use property/indexer metadata and parameter list |
| `ByRef` | Direction confused with documentation prose | Extract actual modifier; map direction carefully |
| `Optional`/default parameters | Default expression omitted | Capture semantic default and source text |
| `ParamArray` | Variadic parameter not recognized | Set variadic flag |
| Modules | CLR static class representation leaks into docs | Preserve source kind `module` plus CLR relation |
| Custom events | Add/remove/raise accessors are special | Model event and accessors without duplicating docs |
| `Handles`/`WithEvents` | Relationship affects behavior but not signature | Preserve as language-specific relations/extensions |
| Partial types/methods | Different fragments win/merge | Follow VB compiler/tool rules; preserve each fragment |
| Designer/generated files | Large generated public API overwhelms docs | Mark generated; configurable visibility/filter |
| XML `cref` resolution | Imports, aliases, and VB syntax affect binding | Resolve with VB Roslyn compilation |
| C# and VB docs in one assembly | Same output schema, different source conventions | Keep language per symbol and one assembly graph |
| Namespace declarations | Cannot always own emitted doc member records | Preserve conceptual/file docs separately if unbound |
| XML include/DTD | Traversal/XXE | Same secure XML rules as C# |
| Multi-target/preprocessor constants | Different declarations/doc fragments | Variant per configuration |
| Compiler-generated backing members | Metadata symbol exists but no source doc target | Mark generated and relate to source property/event |
| Legacy VB language variants | VB6/VBA syntax is not VB.NET XML docs | Require dialect ID; do not apply this profile to VB6/VBA |

### 19.5 Primary sources

- [Microsoft — XML documentation comments in Visual Basic](https://learn.microsoft.com/en-us/dotnet/visual-basic/programming-guide/program-structure/documenting-your-code-with-xml)
- [Visual Basic XML comment tags](https://learn.microsoft.com/en-us/dotnet/visual-basic/language-reference/xmldoc/)
- [Visual Basic language specification](https://learn.microsoft.com/en-us/dotnet/visual-basic/reference/language-specification/)
- [Roslyn .NET Compiler Platform SDK](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/)

---

<a id="lang-sql"></a>
## 20. SQL

```yaml
profile_version: 1
language_id: sql
rank: 8
primary_mechanism: no-cross-dialect-documentation-standard
authority: none
mechanisms:
  - source comments
  - vendor catalog/object comments
  - vendor extended properties
  - procedural-language comments
required_dialect: true
preferred_symbol_source: dialect parser plus optional read-only live catalog
preferred_doc_source: both repository source and deployed catalog, never silently merged
loss_risk_without_semantics: critical
execution_required: false
live_catalog_access: opt-in-read-only
```

### 20.1 Native strategy

“SQL” is a family of standards, products, procedural extensions, migration tools, and client syntaxes. There is no universal Javadoc-like documentation strategy. A platform must model at least three independent channels:

1. **Source comments** in scripts and routine bodies, commonly `--` to end of line and `/* ... */`, with dialect-specific nesting and exceptions.
2. **Catalog/object documentation**, such as PostgreSQL/Oracle `COMMENT ON`.
3. **Vendor metadata properties**, such as SQL Server extended properties or MySQL table/column `COMMENT` clauses.

```sql
-- Repository intent: user table for the identity service.
CREATE TABLE app_user (
    id bigint PRIMARY KEY,
    email text NOT NULL
);

COMMENT ON TABLE app_user IS
    'Canonical application user records.';

COMMENT ON COLUMN app_user.email IS
    'Normalized login email.';
```

The source comment is an annotation. The `COMMENT ON` statements create deployed catalog metadata. They are not the same record and can drift.

### 20.2 Required dialect profile

Every SQL record MUST identify at least:

- vendor/dialect (`postgresql`, `oracle`, `sql-server`, `mysql`, `sqlite`, `db2`, etc.);
- product/version;
- client or migration parser when relevant;
- default catalog/database/schema;
- identifier case-folding and quoting rules;
- procedural language for routines (`plpgsql`, T-SQL, PL/SQL, SQL/PSM, etc.);
- deployment/environment identity if catalog data is read.

Do not use a generic SQL parser for object identity across vendors.

### 20.3 Vendor documentation channels

#### PostgreSQL

PostgreSQL supports lexical comments and `COMMENT ON` for many object kinds. Comments are stored in system catalogs and can be viewed through object-description functions and client tools. Routine identity may require argument types because functions/procedures can be overloaded.

#### Oracle Database

Oracle supports `COMMENT ON TABLE`, `COMMENT ON COLUMN`, and related documented object-comment forms. Comments appear in data dictionary views. PL/SQL source comments remain separate from schema object comments.

#### SQL Server

SQL Server commonly uses extended properties, especially `MS_Description`, through procedures/functions such as `sp_addextendedproperty` and catalog views. Extended properties are arbitrary name/value metadata attached at hierarchical levels; `MS_Description` is a convention, not a universal language field.

#### MySQL

MySQL supports table and column comments in DDL, plus routine/event/view metadata in vendor-specific forms. Length, storage, display, and dump behavior are version/object dependent. MySQL also recognizes comment forms used for executable/versioned comments and optimizer hints; these must not be treated as inert prose.

### 20.4 UDIR object model

SQL symbols should cover:

- server/instance, database/catalog, schema;
- table, view/materialized view;
- column;
- constraint and key;
- index;
- sequence;
- user-defined type/domain;
- function, procedure, aggregate;
- trigger/event;
- package/module where the dialect has one;
- role/policy;
- synonym/alias;
- extension-specific objects.

Canonical identity requires object kind because many databases allow names to overlap across namespaces. For routines, include input signature and routine kind. Preserve quoted spelling and normalized comparison form separately.

### 20.5 Source versus catalog provenance

Emit separate records:

```yaml
recordKind: symbol-documentation
provenance:
  sources:
    - kind: source
      uri: migrations/V014__users.sql
```

and:

```yaml
recordKind: catalog-documentation
provenance:
  sources:
    - kind: catalog
      uri: postgresql://prod/app/public/pg_description/...
```

Link them with `corresponds-to`. When text differs, emit `UDIR-CATALOG-DRIFT`. Never overwrite repository intent with production metadata or vice versa.

### 20.6 Routine documentation

Procedures/functions often have source comments describing parameters, return sets, side effects, transaction behavior, security definer/invoker semantics, exceptions/SQLSTATE, and locking. Most dialects do not structure those fields.

UDIR MAY support a project convention, for example Doxygen-style comments above routines, but it must be identified as `de-facto`/project-specific. Actual routine parameters, modes (`IN`, `OUT`, `INOUT`), types, defaults, volatility/determinism, result shape, and privileges come from the parser/catalog.

### 20.7 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Dialect not specified | Comments, quoting, DDL, and object kinds misparse | Dialect is mandatory; otherwise `dialect-assumed` and low confidence |
| `--` behavior in clients/dialects | Whitespace/newline rules differ | Use vendor/client lexer |
| Nested block comments | Supported in some systems, not others | Use versioned dialect grammar |
| Executable/version comments | Comment-looking text changes execution | Classify as directive, not documentation |
| Optimizer hints | `/*+ ... */` is semantically active | Preserve as code directive |
| Dynamic SQL strings | Apparent comments are string content | Parse host routine/string boundaries |
| `COMMENT ON` transactions | Deployment may roll back or fail on privilege | Treat source statement and live metadata separately |
| Object rename | Catalog key/ID and textual name diverge | Preserve internal object ID where exposed; history relation if available |
| Drop/recreate | Same name is a new object | Include deployment snapshot and stable catalog ID where possible |
| Overloaded routines | Name-only comment target is ambiguous | Include argument signature and routine kind |
| `OUT` parameters/table returns | Return shape represented as parameters | Normalize both signature and return columns with native relation |
| Quoted identifiers | Case and Unicode normalization break identity | Preserve exact quoted name and dialect comparison form |
| Temporary objects | Session-scoped identity | Include session/snapshot; usually exclude from durable docs |
| Synonyms/search path | Display name resolves differently by context | Store fully qualified target and search-path context |
| Table inheritance/partitioning | Comments may not inherit | Model physical/logical relations; never assume text inheritance |
| Views | Column comments can be absent or derived | Keep explicit catalog docs; derivation is separate |
| Generated columns | Prose may claim behavior that expression determines | Store expression as code fact |
| Extended properties | Arbitrary hierarchy/name/value types | Preserve property name/type and map only configured names |
| Metadata length/encoding | DDL silently truncates or rejects | Preserve source text and catalog result; diagnostic on mismatch |
| Dump/restore tools | Some metadata omitted by options | Record extraction/deployment tool and snapshot |
| Privilege-limited catalog read | Missing metadata looks empty | Emit access diagnostic; do not declare undocumented |
| Cross-database references | Target resolver lacks remote context | Keep unresolved/external relation |
| Migration branches | Same version number has different SQL | Include repository commit/digest |
| ORM-generated schema | Source-of-truth may be model code | Link generated SQL to model artifact; mark generated |
| Embedded SQL in another language | Two lexers and symbol graphs interact | Host-language adapter owns source; SQL sub-adapter owns embedded fragment |
| SQL injection/security examples | Rendering executes copied snippets | Examples are inert by default |

### 20.8 Recommended adapter stack

1. Dialect-aware parser for repository SQL.
2. Migration-tool parser for delimiter/batch semantics.
3. Optional read-only catalog collectors per vendor.
4. Safe normalization of identifiers and object signatures.
5. Source-to-deployed comparison with explicit environment/snapshot.
6. Project-configurable routine comment dialect, disabled unless declared.

Catalog collection MUST use least privilege, bounded queries, and secret redaction. Connection strings and object definitions may contain sensitive data and MUST NOT be emitted into public documentation by default.

### 20.9 Primary sources

- [PostgreSQL — Lexical structure and comments](https://www.postgresql.org/docs/current/sql-syntax-lexical.html#SQL-SYNTAX-COMMENTS)
- [PostgreSQL — `COMMENT`](https://www.postgresql.org/docs/current/sql-comment.html)
- [Microsoft SQL Server — `sp_addextendedproperty`](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-addextendedproperty-transact-sql)
- [Oracle Database SQL Language Reference — `COMMENT`](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/COMMENT.html)
- [MySQL Reference Manual — Comments](https://dev.mysql.com/doc/refman/8.4/en/comments.html)
- [MySQL Reference Manual — `CREATE TABLE`](https://dev.mysql.com/doc/refman/8.4/en/create-table.html)

---

<a id="lang-r"></a>
## 21. R

```yaml
profile_version: 1
language_id: r
rank: 9
primary_mechanism: Rd package documentation files
authority: ecosystem-official
source_authoring_mechanism: roxygen2
source_authoring_authority: de-facto
native_syntax:
  official_artifact: ".Rd files"
  common_source: "#' roxygen comments"
native_markup: Rd macro language
preferred_symbol_source: R parser plus namespace/package metadata
preferred_doc_source: tools::parse_Rd output, with optional roxygen source correlation
loss_risk_without_semantics: high
execution_required: false
```

### 21.1 Native strategy

For R packages, the official documentation artifacts are files in the `man/` directory using the **R documentation (Rd)** format. Rd is a macro language processed by R’s documentation tools into text, HTML, LaTeX/PDF, and other representations.

A common authoring workflow uses roxygen2 comments beginning `#'` near R declarations. roxygen2 generates `.Rd`, `NAMESPACE`, and related artifacts. R package tooling treats the generated Rd as the package documentation input, but source-oriented systems should retain both the roxygen source and generated Rd.

```r
#' Add two numbers
#'
#' @param x First value.
#' @param y Second value.
#' @return The sum of `x` and `y`.
#' @examples
#' add(1, 2)
#' @export
add <- function(x, y) x + y
```

Possible generated Rd:

```rd
\name{add}
\alias{add}
\title{Add two numbers}
\usage{
add(x, y)
}
\arguments{
\item{x}{First value.}
\item{y}{Second value.}
}
\value{
The sum of \code{x} and \code{y}.
}
\description{
Add two numbers
}
\examples{
add(1, 2)
}
```

### 21.2 Rd structure

A UDIR adapter SHOULD understand at least these Rd concepts:

| Category | Representative Rd macros/directives |
|---|---|
| Identity/indexing | `\name`, `\alias`, `\docType`, `\keyword`, `\concept` |
| Summary/body | `\title`, `\description`, `\details`, `\section` |
| API | `\usage`, `\arguments`, `\item`, `\value`, `\format` |
| Provenance | `\author`, `\source`, `\references` |
| Navigation | `\seealso`, `\link`, `\linkS4class` |
| Examples | `\examples`, `\dontrun`, `\donttest`, `\dontshow` |
| Formatting | `\code`, `\preformatted`, `\emph`, `\strong`, lists, tables, equations |
| Conditional/dynamic | `\if`, `\ifelse`, `\Sexpr` |
| Media | `\figure` and related output-specific content |
| Extensibility | user-defined macros and package-specific conventions |

Rd is not Markdown. Backslashes, braces, modes, and macro arguments have formal parsing rules. Parse with R’s `tools::parse_Rd` or equivalent behavior rather than regex.

### 21.3 Topics, aliases, and symbols

An Rd file describes a **help topic**, not necessarily one source symbol. A topic can have:

- several `\alias` entries;
- several functions in `\usage`;
- methods for a generic;
- a class and its constructor;
- a dataset;
- a package overview;
- operators with non-identifier names;
- concepts/keywords independent of symbols.

Therefore UDIR SHOULD emit:

1. a topic/conceptual record for the Rd page;
2. symbol records for statically resolvable aliases/usages; and
3. `documents` relations from the topic to symbols.

Do not use `\name` as a globally unique symbol ID. It is a filename/topic key.

### 21.4 R object systems

The adapter must account for:

- ordinary functions and objects;
- S3 generics/methods such as `print.foo`;
- S4 classes, methods, signatures, slots, and generics;
- Reference Classes;
- R6 and other package-defined object systems;
- datasets and data objects;
- operators and replacement functions;
- active bindings;
- native routines registered through C/C++/Fortran;
- reexports.

Actual formals, defaults, namespace exports, and method registrations should come from source/package metadata. Rd `\usage` is documentation text and can intentionally abbreviate or diverge.

### 21.5 roxygen2 mapping

Common roxygen2 tags map as follows:

| roxygen2 input | UDIR |
|---|---|
| title/first block | summary/title policy |
| `@description`, `@details` | description/sections |
| `@param` | parameter docs |
| `@return` | return docs |
| `@examples`, `@examplesIf` | examples with execution condition |
| `@seealso`, `@references` | links/references |
| `@family`, `@concept`, `@keywords` | grouping/keywords |
| `@section` | named section |
| `@inherit`, `@inheritParams`, `@inheritSection`, `@template` | include/inheritance relation |
| `@rdname`, `@aliases` | topic ownership/aliases |
| `@name` | explicit topic identity |
| `@export`, `@exportS3Method`, `@import`, `@importFrom` | namespace/build metadata, not prose |
| `@rawRd` | native Rd extension |
| unknown custom tag | extension |

Roxygen tags that affect `NAMESPACE` MUST NOT be displayed as ordinary documentation fields, but they can enrich symbol visibility and package relations.

### 21.6 Examples and execution

Rd examples may be classified by wrappers:

- ordinary examples are candidates for checking;
- `\dontrun{}` generally indicates code not run in normal checks;
- `\donttest{}` has check-policy implications;
- `\dontshow{}` hides code from rendered examples but can still affect execution/context.

UDIR must preserve wrapper semantics in `example.execution` and native attributes. It MUST NOT run examples during ingestion. A separate sandboxed validation service may execute them under an explicit R/package environment.

### 21.7 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Multiple aliases per Rd file | One page falsely becomes one symbol | Topic record plus separate symbol relations |
| One alias in multiple files | Help index conflict/tool warning | Preserve both; report duplicate alias |
| `\usage` differs from actual formals | Stale or simplified signature replaces code | Actual source signature wins; Rd form is documented signature |
| S3 method naming | Dot interpreted as namespace/member syntax | Use S3 registration/generic identity |
| S4 method signatures | Name alone is insufficient | Include dispatch signature and package |
| Datasets | Adapter expects callable symbol | Use variable/data symbol and `\format` schema |
| Replacement functions | Name ends `<-` and first/last parameters have conventions | Preserve native name/signature |
| Operators | Non-syntactic names/aliases | Preserve quoted/native spelling |
| `\Sexpr` | Documentation parsing can execute R | Never evaluate by default; mark execution suppressed |
| `\if`/`\ifelse` | Output differs by format | Preserve condition and optionally emit format variants |
| User macros | Parser needs package macro definitions | Load declarative definitions safely; unknown macro preserved |
| `\dontrun`/`\donttest`/`\dontshow` nesting | Example behavior flattened | Preserve wrappers as AST attributes |
| Roxygen inheritance | Generated text loses original owner | Correlate generated Rd and source maps; preserve inheritance path |
| `@rdname` | Distant blocks contribute one topic | Aggregate explicitly; retain each source fragment |
| Collate order | Objects may be defined/redefined across files | Use package `Collate` and namespace metadata when resolving |
| Conditional definitions | OS/dependency/source conditions alter API | Build variants or mark conditional |
| Lazy data | Object structure unavailable without loading package | Use metadata/static declarations; optional trusted inspection |
| Native routines | R docs correspond to C/Fortran symbols | Cross-language relation; do not merge identities |
| Encoding/macros | Output differs or parser errors | Preserve declared encoding and raw bytes |
| Generated Rd not committed | Only roxygen source exists | Generate only in controlled toolchain, or parse source tags directly |
| Rd generated manually and roxygen source both present | Drift | Compare and report; define project source-of-truth |
| Links to unavailable packages | Unresolved cross-package references | Preserve external target and package constraint |

### 21.8 Recommended adapter stack

1. Parse R source without evaluating it.
2. Parse `DESCRIPTION`, `NAMESPACE`, and package metadata.
3. Parse committed `.Rd` using `tools::parse_Rd` behavior.
4. Parse roxygen2 source blocks and correlate generated topics when present.
5. Never evaluate `\Sexpr`, examples, package code, or vignettes by default.
6. Store generator/R/Roxygen versions because output can vary.

### 21.9 Primary sources

- [Writing R Extensions — Writing R documentation files](https://cran.r-project.org/doc/manuals/r-release/R-exts.html#Writing-R-documentation-files)
- [R `tools::parse_Rd`](https://stat.ethz.ch/R-manual/R-devel/library/tools/html/parse_Rd.html)
- [R `tools::checkRd`](https://stat.ethz.ch/R-manual/R-devel/library/tools/html/checkRd.html)
- [roxygen2 — Rd documentation](https://roxygen2.r-lib.org/articles/rd.html)
- [roxygen2 tag index](https://roxygen2.r-lib.org/reference/tags-index.html)

---

<a id="lang-rust"></a>
## 22. Rust

```yaml
profile_version: 1
language_id: rust
rank: 10
primary_mechanism: language doc comments and doc attributes processed by rustdoc
authority: language-standard
tool_authority: ecosystem-official
native_syntax:
  outer: ["///", "/** ... */", "#[doc = ...]"]
  inner: ["//!", "/*! ... */", "#![doc = ...]"]
native_markup: Markdown plus rustdoc extensions
preferred_symbol_source: rustc/rustdoc semantic model and Cargo metadata
preferred_doc_source: rustdoc representation plus original attributes/comments
loss_risk_without_semantics: high
execution_required: false
```

### 22.1 Native strategy

Rust makes documentation comments syntactic sugar for `doc` attributes:

- `///` and `/** ... */` are outer doc comments attached to the following item;
- `//!` and `/*! ... */` are inner doc comments attached to the containing item, commonly a module or crate;
- `#[doc = "..."]` and `#![doc = "..."]` are the attribute forms.

```rust
/// Parses a user identifier.
///
/// # Errors
///
/// Returns [`ParseUserIdError`] when the text is invalid.
///
/// # Examples
///
/// ```
/// let id = parse_user_id("42")?;
/// # Ok::<(), ParseUserIdError>(())
/// ```
pub fn parse_user_id(text: &str) -> Result<UserId, ParseUserIdError> {
    // ...
}
```

Rustdoc interprets Markdown and adds symbol-link resolution, doctest behavior, search aliases, reexport controls, and other attributes.

### 22.2 Outer versus inner ownership

The adapter MUST preserve comment orientation:

- outer docs bind to the item that follows;
- inner docs document the enclosing crate/module/item;
- a sequence of doc attributes/comments contributes ordered fragments;
- generated attributes such as `#[doc = include_str!("...")]` may require controlled expansion.

Rust’s grammar has exact edge behavior: comments beginning with extra slash/asterisk combinations may be ordinary comments rather than doc comments; bare carriage-return characters are not allowed in doc comments; nested block comments are supported, but a block doc comment still terminates according to comment delimiters, not Markdown awareness.

### 22.3 Markdown and conventional sections

Rustdoc does not require one rigid tag grammar for parameters and results. Common sections are conventions:

- `# Examples`;
- `# Panics`;
- `# Errors`;
- `# Safety`;
- `# Abort` where relevant;
- implementation-specific notes and platform sections.

Parameter and return descriptions are often prose. The adapter SHOULD parse well-known headings as semantic sections, but parameter-level mapping is convention-based unless a project adopts a stricter style.

`# Safety` is especially important for `unsafe` functions/traits/implementations. It maps to a `safety` contract, but the actual `unsafe` modifier comes from the compiler.

### 22.4 Intra-doc links

Rustdoc can resolve links to items by name using Markdown link syntax and shortcut forms, with namespace disambiguators and anchors. Resolution depends on scope, imports, reexports, associated items, macros, and the toolchain.

Store:

- original link text/destination;
- namespace/disambiguator;
- resolved precise item ID;
- candidate set when ambiguous;
- toolchain/version;
- whether the link points through a reexport.

Do not reduce every link to a URL. Symbol links should remain graph references so the final renderer can generate its own routes.

### 22.5 Documentation attributes

Important rustdoc-related attributes include:

- `doc(hidden)` to suppress an item from generated documentation;
- `doc(alias = "...")` for search aliases;
- `doc(inline)` and `doc(no_inline)` affecting reexport presentation;
- crate-level HTML/logo/favicon/playground/root URL settings;
- `doc(cfg(...))` and availability-like display where supported;
- arbitrary generated `doc` string fragments.

Treat presentation-only attributes as native metadata while mapping semantic visibility, alias, and conditional information where appropriate.

### 22.6 Doctests

Rustdoc extracts Rust code blocks as documentation tests unless attributes indicate otherwise. Code-fence attributes can include:

- Rust language marker;
- `no_run`;
- `ignore` and conditional ignore forms;
- `should_panic`;
- `compile_fail`;
- edition selection;
- expected diagnostic/error-code annotations;
- other rustdoc-specific flags.

Lines beginning with `#` can be hidden from rendered output while remaining part of the compiled test. Preserve both rendered and executable forms.

The ingestion adapter MUST set execution to `never` or `parse-only`. A separate Rust sandbox can compile/run examples under the correct crate features, edition, target, dependencies, and test harness.

### 22.7 Symbol graph and output

Rustdoc JSON can provide structured item information, but its status/version stability must be recorded; historically it has required nightly/unstable tooling. HTML scraping is a last resort.

A robust adapter uses:

- Cargo metadata for workspace/package/features;
- rustc/rustdoc for semantic items and relationships;
- a pinned rustdoc JSON schema or compiler API;
- original source for raw comment/attribute spans;
- optional source expansion records for generated items.

### 22.8 Normalization rules

| Native construct | UDIR mapping |
|---|---|
| first paragraph | summary policy |
| remaining Markdown | description |
| well-known headings | semantic/named sections |
| `# Errors` | errors section; individual errors may remain prose |
| `# Panics` | panic contracts |
| `# Safety` | safety contract |
| code fence/doctest | example with execution attributes |
| intra-doc link | symbol-link |
| `#[deprecated]` | lifecycle from compiler attribute |
| `#[doc(hidden)]` | native presentation visibility/suppression |
| `#[doc(alias)]` | alternate search IDs/keywords |
| reexport doc attributes | reexport relation/presentation metadata |
| `include_str!` doc value | external fragment with expansion provenance |
| unknown rustdoc attribute | extension |

### 22.9 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Outer vs inner docs | Module docs bind to next item | Respect attribute orientation |
| Multiple doc attributes | Source order lost | Preserve ordered fragments and spans |
| `include_str!` in `doc` | Reads external file during expansion | Allowlist root; hash included file; preserve unexpanded expression if blocked |
| `cfg`/Cargo features | Items and docs differ per build | Emit variants with feature/target context |
| Declarative/procedural macros | Symbols/docs generated during expansion | Prefer compiler output; mark generated and retain expansion provenance |
| Reexports | Item appears under several paths | One declaration identity plus reexport path records; honor inline/no-inline metadata |
| Traits and implementations | Same method exists as requirement/default/impl/inherent method | Separate symbol identities and override/implements relations |
| Blanket implementations | Combinatorial/generated docs | Mark generated; allow renderer filtering |
| Associated types/consts | Parameter-centric model misses items | Use exact symbol kinds |
| Unsafe implementations | Safety contract may belong to trait/impl rather than method | Bind to native item and propagate only by explicit policy |
| Hidden doctest lines | Rendered code differs from compiled code | Store both views |
| `compile_fail` | Successful compilation means test failure | Preserve execution expectation |
| `ignore` conditions | Toolchain/target conditions matter | Store condition, never reduce to boolean |
| Edition attributes | Same code parses differently | Record example edition |
| Intra-doc link namespace ambiguity | Type/macro/value share name | Preserve disambiguator and candidates |
| Private items/document-private-items | Output selection differs | Store actual visibility and extraction option |
| `#[doc(hidden)]` | Public API omitted from docs | Preserve suppression separately from visibility |
| Rustdoc JSON instability | Schema changes | Pin toolchain and schema; set native-output-version flag |
| CR characters/block delimiters | Parse failures | Preserve raw bytes and compiler diagnostics |
| Async functions/`impl Trait` | Source and lowered signatures differ | Store source display and semantic type representation |
| Lifetimes elided | Display versus inferred identity differs | Use compiler identity; retain source spelling |
| Panic/error prose | Not structurally tied to concrete variants | Map heading section with lower confidence unless links resolve |
| FFI symbols | Rust item and foreign linkage name differ | Preserve ABI/linkage and cross-language relation |

### 22.10 Primary sources

- [The Rust Reference — Comments](https://doc.rust-lang.org/reference/comments.html)
- [The rustdoc book — How to write documentation](https://doc.rust-lang.org/rustdoc/how-to-write-documentation.html)
- [The rustdoc book — Documentation tests](https://doc.rust-lang.org/rustdoc/write-documentation/documentation-tests.html)
- [The rustdoc book — Intra-doc links](https://doc.rust-lang.org/rustdoc/write-documentation/linking-to-items-by-name.html)
- [Cargo `doc`](https://doc.rust-lang.org/cargo/commands/cargo-doc.html)
- [rustdoc JSON types](https://doc.rust-lang.org/nightly/nightly-rustc/rustdoc_json_types/)

---

<a id="lang-delphi"></a>
## 23. Delphi/Object Pascal

```yaml
profile_version: 1
language_id: delphi
rank: 11
primary_mechanism: Delphi XML documentation comments
authority: compiler-vendor-official
native_syntax: "/// XML documentation before declaration"
native_markup: vendor-defined XML subset
preferred_symbol_source: Delphi compiler parser/metadata
preferred_doc_source: compiler-emitted Delphi XML plus source comments
loss_risk_without_semantics: high
execution_required: false
dialect_required: true
```

### 23.1 Native strategy

Modern Delphi supports XML documentation comments beginning `///` before a symbol. The compiler can emit XML documentation with source and program-structure information. Despite superficial similarity to .NET XML comments, Delphi’s emitted XML model and symbol organization are not the C#/Visual Basic `<members><member name="M:...">` format.

```pascal
/// <summary>Retrieves a user.</summary>
/// <param name="AId">The user identifier.</param>
/// <returns>The matching user.</returns>
/// <exception cref="EUserNotFound">
/// Raised when the user does not exist.
/// </exception>
function GetUser(AId: Int64): TUser;
```

Supported/documented XML elements include summary and paragraph content, inline/code blocks, remarks, parameters, return values, references, exceptions, and permissions. The exact set and rendering behavior are tied to RAD Studio/compiler versions.

### 23.2 Compiler-emitted documentation

The Delphi compiler’s documentation output represents units, namespaces, types, ancestors, members, fields, properties, methods, parameters, enum elements, source files/lines, and development notes. An adapter SHOULD consume this structure when available because it resolves declaration ownership and overloads more reliably than comment proximity alone.

Store:

- compiler version and target platform;
- package/project configuration;
- emitted XML schema/root/version if present;
- exact source path separately from a redacted/public path;
- raw comment XML;
- compiler-emitted semantic structure.

Absolute source paths may appear in generated output and can leak developer names or filesystem layout. Redact only in the publishing layer; retain secured provenance internally.

### 23.3 Source declaration ownership

For routines declared in an interface section and implemented later, API documentation normally belongs to the declaration. A naive nearest-comment scan of the implementation can create duplicates or document private implementation details.

The adapter must understand:

- units and namespaces;
- interface versus implementation sections;
- classes, records, interfaces, helpers;
- methods and properties;
- overloaded routines;
- generics;
- class/static methods;
- operators;
- events/delegates;
- nested types and scoped enums;
- visibility sections;
- conditional compilation.

### 23.4 XML mapping

| Delphi XML element | UDIR |
|---|---|
| `<summary>` | summary |
| `<para>`/`<remarks>` | description/paragraphs |
| `<c>`/`<code>` | inline/code block |
| `<param name>` | parameter docs |
| `<returns>` | returns |
| `<exception cref>` | error docs and symbol link |
| `<see cref>` | symbol link |
| `<permission cref>` | permission contract |
| unknown XML | native extension |
| compiler ancestor/member nodes | symbol relations and containment |

Do not run a .NET XML documentation parser without a Delphi-specific layer. It may parse the comment fragment but misread the compiler output and native identities.

### 23.5 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Delphi vs Free Pascal/Object Pascal | Syntax/tool semantics differ | Require dialect/compiler ID; separate adapters |
| Interface declaration vs implementation | Duplicate or wrong ownership | Prefer public declaration; preserve implementation fragment separately |
| Overloads | Name-only IDs collide | Use compiler-emitted signature and source declaration |
| Default parameters | Implementation may omit/repeat defaults differently | Source/semantic declaration is authoritative |
| Properties | Read/write methods, index parameters, default property | Model property and accessor relations |
| Record/class helpers | Extension-like members appear under target | Preserve declaring helper and extended-type relation |
| Generic constraints | Comment parser misses semantics | Extract from compiler model |
| Operators | Symbol names and signatures are special | Use native symbol kind/identity |
| Conditional defines | Platform API differs | Variant by project/platform/defines |
| Unit aliases/namespaces | Same type display name resolves differently | Include fully qualified unit/namespace |
| Scoped enums | Enum value identity differs by compiler setting | Record compiler option and qualified identity |
| Anonymous methods | No stable public symbol | Synthetic scoped identity if documented |
| Compiler-generated members | Backing/accessor artifacts pollute API | Mark generated and filter by source mapping |
| XML parse errors | Help Insight/compiler may ignore content | Preserve raw and compiler diagnostics |
| Unsupported/custom XML | Renderer drops data | Preserve subtree as extension |
| Inherited members | Compiler output may list inherited members | Distinguish declaration owner from display container |
| Source-path leakage | Generated XML exposes local paths | Redaction policy at export |
| Package/binary-only dependency | Docs available without source or vice versa | Preserve compiler XML/native binary identity and confidence |
| Character encodings | Older source/compiler versions differ | Capture encoding/code page |
| `{$REGION}` and directives | Comment-like or structural text affects parsing | Use compiler lexer; preserve directives separately |
| Help Insight cache/version | IDE display differs from build output | Treat as consumer view, not source of truth |

### 23.6 Recommended adapter stack

Prefer compiler-emitted XML plus a Delphi-aware source parser. If compiler invocation is unavailable, use a parser configured for the exact Delphi version and project defines, then mark identities and overload resolution with lower confidence.

### 23.7 Primary sources

- [Embarcadero — XML Documentation for Delphi Code](https://docwiki.embarcadero.com/RADStudio/Florence/en/XML_Documentation_for_Delphi_Code)
- [Embarcadero — XML Documentation Comments](https://docwiki.embarcadero.com/RADStudio/Florence/en/XML_Documentation_Comments)
- [Embarcadero — Delphi compiler documentation generation](https://docwiki.embarcadero.com/RADStudio/Florence/en/Delphi_Compiler)

---

<a id="lang-scratch"></a>
## 24. Scratch

```yaml
profile_version: 1
language_id: scratch
rank: 12
primary_mechanism: workspace comments
authority: ecosystem-official
authority_scope: current Scratch VM implementation
api_documentation_mechanism: none
storage: Scratch 3 project JSON inside SB3 archive
native_markup: plain text
preferred_symbol_source: validated project JSON block graph
preferred_doc_source: target-scoped comment records
loss_risk_without_graph: critical
execution_required: false
record_kind_default: workspace-annotation
```

### 24.1 Native strategy

Scratch does not have a textual API-documentation language comparable to Javadoc. Its native comments are visual workspace annotations that can be free-floating or attached to a block.

In the current Scratch VM serializer, each target’s `comments` object maps a comment ID to:

```json
{
  "blockId": "optional-associated-block-id",
  "x": 120,
  "y": 80,
  "width": 200,
  "height": 120,
  "minimized": false,
  "text": "Explain this script."
}
```

The fields are implementation-level project data:

- `blockId`: attached block or null/unset;
- `x`, `y`: workspace position;
- `width`, `height`: expanded visual dimensions;
- `minimized`: UI state;
- `text`: annotation text.

These must normalize to `workspace-annotation`, not fabricated parameter/return documentation.

### 24.2 Project structure

An `.sb3` file is a ZIP-based project archive whose `project.json` describes targets, variables, lists, broadcasts, blocks, comments, monitors, extensions, and metadata. Assets are separate archive entries.

Each target—stage or sprite—has its own block and comment maps. Block records form an ID graph through `next`, `parent`, inputs, fields, mutations, and comment links.

A safe adapter should:

1. open the archive without executing it;
2. enforce archive entry count/size/path limits;
3. validate `project.json`;
4. build target-scoped ID maps;
5. deserialize comments as annotations;
6. resolve `blockId` links;
7. separately infer custom-block symbols from procedure definition/prototype mutations.

### 24.3 Custom blocks as the closest API unit

Scratch custom blocks can be modeled as `custom-block` symbols. Their identity comes from procedure definition/prototype metadata, including procedure code, argument IDs, names/defaults, and warp behavior. Comments do not themselves define the custom block’s contract.

A comment attached to:

- a custom-block definition may be related to the custom-block symbol with `attached-to`;
- a call documents that call site, not necessarily the procedure globally;
- an arbitrary internal block documents that block/script region;
- no block remains a free workspace annotation.

Automatic promotion of definition-attached prose into formal parameter/return docs should be opt-in and heuristic, with low confidence.

### 24.4 IDs and current compatibility behavior

Current VM code serializes comment records and associates attached comments with block IDs. During deserialization, newer VM behavior may synthesize an attached comment ID based on the parent block ID rather than honoring an older arbitrary saved ID. Therefore:

- preserve saved/native comment ID;
- preserve effective/runtime ID separately;
- record the rewrite relation;
- never assume comment IDs are stable across editor versions.

### 24.5 Normalization rules

| Scratch construct | UDIR |
|---|---|
| target/stage/sprite | module/container-like project entity |
| workspace comment | `workspace-annotation` |
| `blockId` | `attached-to` relation |
| comment geometry/UI state | `extensions.scratch.workspace` |
| block opcode | native block kind/operator |
| custom block prototype | `custom-block` symbol |
| procedure arguments | custom-block signature parameters |
| procedure code/display | native signature/display name |
| script stack | graph/containment relation |
| extension opcode | dialect/extension-specific block |
| project metadata | build/source context |

### 24.6 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Free-floating comment | Forced onto nearest block | Keep unbound annotation with target and geometry |
| Missing/deleted `blockId` | Dangling relation crashes importer | Preserve unresolved attachment and diagnostic |
| Attached-comment ID rewrite | Round-trip ID changes | Store saved and effective IDs |
| Duplicate IDs/corrupt JSON | Map entries overwrite | Validate before normalization; reject or namespace recovery IDs |
| Same block ID across targets | Global map collides | IDs are target-scoped in canonical tuple |
| Comment on custom-block call | Mistaken for procedure docs | Bind to call block; optional relation to procedure |
| Comment on custom-block definition | Prose treated as structured contract | Keep annotation; heuristic promotion only by policy |
| Locale-dependent labels | Rendered block text changes identity | Use opcodes and mutation IDs, not localized labels |
| Unknown extension opcode | Block parser drops script | Preserve native block and extension ID |
| Procedure mutation JSON strings | Argument arrays encoded as strings/legacy forms | Use VM-compatible deserializer and preserve raw mutation |
| Hacked/obsolete blocks | Editor may repair or discard | Parse losslessly where possible; diagnostic and version |
| Shadow blocks | Compact serialization and repair behavior | Follow VM graph semantics; preserve source JSON |
| Detached comments after repair | Editor rewrites graph | Keep original and normalized graph versions |
| UI geometry | Mistaken for semantic ordering | Store only as presentation metadata |
| Minimized state | Text overlooked by visual scraping | Read serialized text regardless of UI state |
| Sprite cloning | Runtime clones do not equal authored targets | Document original authored targets; runtime traces separate |
| Cloud variables/monitors | Sensitive or runtime state leaks | Treat project definition separately from live values |
| Malicious ZIP | Path traversal/decompression bomb | Safe archive extraction with strict limits |
| Malformed SVG/assets | Renderer security issue | Do not render unsanitized assets during doc extraction |
| Project extensions | May load remote/custom code in modified editors | Never execute extensions; parse metadata only |
| Editor/project-format evolution | Fields and repair rules change | Pin VM/project semver and preserve unknown fields |
| Visual-only meaning | Spatial grouping conveys human intent | Preserve coordinates and optional script-region inference with low confidence |

### 24.7 Recommended adapter stack

Use the current Scratch VM serialization schema/implementation as the compatibility reference, but do not instantiate or run the VM for untrusted projects. Build a read-only JSON graph parser and maintain fixtures from multiple project/editor versions.

### 24.8 Primary sources

- [Scratch VM repository](https://github.com/scratchfoundation/scratch-vm)
- [Current Scratch editor monorepo](https://github.com/scratchfoundation/scratch-editor)
- [Current Scratch VM SB3 serializer](https://github.com/scratchfoundation/scratch-editor/blob/develop/packages/scratch-vm/src/serialization/sb3.js)
- [Current Scratch VM comment model](https://github.com/scratchfoundation/scratch-editor/blob/develop/packages/scratch-vm/src/engine/comment.js)
- [Scratch developers](https://scratch.mit.edu/developers)

---

<a id="lang-php"></a>
## 25. PHP

```yaml
profile_version: 1
language_id: php
rank: 13
primary_mechanism: native doc-comment tokens with de-facto PHPDoc semantics
authority: de-facto
native_language_support: reflection exposes raw doc comments when retained
de_facto_processor: phpDocumentor
native_syntax: "/** ... */"
native_markup: PHPDoc summary/description/tags plus inline tags
preferred_symbol_source: static PHP AST and package/autoload metadata
preferred_doc_source: dedicated DocBlock parser plus raw source
loss_risk_without_semantics: high
execution_required: false
```

### 25.1 Native strategy

PHP recognizes documentation comments lexically and can expose raw comments through reflection, but the language does not standardize the complete semantics of PHPDoc tags. phpDocumentor supplies the principal documentation grammar and renderer ecosystem. Proposed PHPDoc standardization through PHP-FIG did not become an accepted PSR, so an adapter must not label every PHPDoc dialect as a language standard.

```php
/**
 * Retrieve a user.
 *
 * @param positive-int $id User identifier.
 * @return User
 * @throws UserNotFoundException When no user exists.
 */
function getUser(int $id): User
{
    // ...
}
```

Native parameter and return types are code facts. `positive-int` above is a documentation/static-analysis refinement and belongs in `documentedType` or a constraint extension.

### 25.2 PHPDoc structure

A DocBlock typically contains:

1. summary;
2. optional long description;
3. block tags;
4. inline tags within prose.

phpDocumentor’s tag reference includes major tags such as:

| Category | Tags |
|---|---|
| Signatures/types | `@param`, `@return`, `@var`, `@throws` |
| Virtual members | `@method`, `@property`, `@property-read`, `@property-write` |
| Visibility/API | `@api`, `@internal`, `@ignore`, `@final`, `@static`, `@global` |
| Ownership/grouping | `@package`, `@subpackage` |
| Lifecycle | `@deprecated`, `@since`, `@version` |
| Navigation/reuse | `@see`, `@link`, `@uses`, `@used-by`, `@inheritdoc` |
| Examples/source | `@example`, `@source`, `@filesource` |
| Metadata | `@author`, `@copyright`, `@license`, `@todo` |
| Legacy/deprecated tags | `@category` and other version-specific entries |

The adapter SHOULD version the tag registry and preserve tags unknown to the configured processor.

### 25.3 Competing type dialects

PHPDoc type syntax has expanded beyond basic phpDocumentor forms through static-analysis ecosystems such as PHPStan and Psalm. Common extensions include:

- generics/templates;
- variance;
- array and object shapes;
- integer ranges and refined scalar types;
- class-string and key/value relationships;
- conditional types;
- callable signatures;
- assertion tags;
- type aliases/imports;
- tool-prefixed variants such as `@phpstan-*` and `@psalm-*`.

A universal parser that accepts every extension without identifying its dialect will produce incorrect semantics. Store:

- parser/dialect name and version;
- raw type string;
- normalized type tree where supported;
- vendor extension node;
- relation to actual native type.

### 25.4 Native language metadata

Modern PHP has native:

- scalar/class/union/intersection/DNF types;
- nullable types;
- attributes;
- promoted constructor properties;
- enums;
- readonly classes/properties;
- first-class callables;
- property hooks and evolving language features;
- a native `#[Deprecated]` attribute in current PHP generations.

Documentation lifecycle should combine PHPDoc and native attributes without collapsing them. Actual deprecation is strongest when compiler/runtime/tooling recognizes the native mechanism; PHPDoc remains valuable for message/replacement/since prose.

### 25.5 Symbols and ownership

The adapter must understand:

- namespaces and imports;
- classes, interfaces, traits, enums;
- functions and constants;
- methods/properties/class constants/enum cases;
- anonymous classes and closures;
- trait insertion and conflict resolution;
- conditional declarations;
- magic members declared through PHPDoc;
- Composer package and autoload mappings.

Magic `@method`/`@property` entries create **virtual documented symbols**. They must not be confused with actual declarations. Relate them to their owner and mark `binding: explicit-tag`.

### 25.6 Inheritance

phpDocumentor and static-analysis tools can inherit or combine DocBlocks, including inline `{@inheritdoc}` and tags on parent/interface/trait members. Behavior varies by processor and version.

Store:

- direct child block;
- selected parent/interface/trait source;
- tag-by-tag merge result;
- processor/version;
- conflicts among multiple interfaces or traits;
- virtual versus actual member status.

### 25.7 Normalization rules

| PHPDoc/native construct | UDIR |
|---|---|
| summary/description | summary/description |
| `@param` | parameter docs plus documented type |
| `@return` | return docs/type |
| `@throws` | errors |
| `@var` | variable/property documented type |
| `@method` | virtual method symbol |
| `@property*` | virtual property symbol and access mode |
| `@template` tool dialect | type parameter extension |
| `@deprecated` + `#[Deprecated]` | lifecycle with multiple native mechanisms |
| `@since`, `@version` | lifecycle |
| `@see`, `@link` | links |
| `@inheritdoc` | inheritance directive |
| `@internal`, `@api`, `@ignore` | documentation visibility/stability metadata |
| `@package` | grouping/package hint; compare with Composer namespace/package |
| attributes | semantic metadata from AST, separate from DocBlock |
| unknown prefixed tag | vendor extension |

### 25.8 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Native/PHPDoc type conflict | Stale doc type replaces code | Actual type wins; retain documented type and diagnostic |
| Union/intersection/DNF | Simplistic parser changes precedence | Use native grammar AST and lossless type tree |
| PHPStan/Psalm syntax | phpDocumentor parser rejects/refolds | Select dialect or multi-parser; preserve raw |
| Magic members | Renderer claims runtime declaration | Mark virtual and explicit-tag bound |
| Traits | Docs/owners copied ambiguously | Model trait declaration, insertion, aliases, conflict resolution |
| `{@inheritdoc}` | Processor merge differs | Store processor and inheritance path |
| Constructor property promotion | One syntax creates parameter and property | Separate related symbols and documentation targets |
| Property hooks | Getter/setter hook docs and property docs collide | Use language-version-aware AST and relation |
| Enum cases | Treated as class constants only | Preserve `enum-case` |
| Anonymous class/closure | Weak identity | Source-scoped synthetic ID |
| Conditional declarations | API depends on runtime condition | Mark conditional/dynamic; static branches as variants where resolvable |
| Multiple namespaces per file | File-level block binds incorrectly | Follow AST namespace scope |
| Attributes adjacent to DocBlocks | Naive proximity breaks | Bind through AST declaration |
| Reflection comments missing | Runtime/opcache/build strips comments or source absent | Prefer static source; report runtime absence |
| Composer classmap/generated files | Duplicate source identities | Use package/autoload and generated provenance |
| `eval`/runtime class generation | Static API unavailable | Mark dynamic-unobservable; optional trusted runtime probe |
| Alias imports | Doc type names resolve under namespace/import context | Use semantic name resolver |
| `self`, `static`, `$this`, `parent` | Context-sensitive documented types | Preserve contextual kind and resolve target |
| Variadics/by-reference | Tags omit native modifiers | Extract modifiers from AST |
| Named arguments | Parameter names are API | Detect renames/mismatch as breaking-documentation diagnostic |
| Inline HTML/Markdown | XSS/parser variation | Sanitize output; record declared parser |
| Custom annotations used by frameworks | Documentation parser treats behavior metadata as prose | Preserve under framework extension; do not execute reflection hooks |

### 25.9 Recommended adapter stack

Use a version-aware static parser, Composer metadata, and a dedicated DocBlock parser. Do not rely on runtime Reflection for primary extraction because loading arbitrary PHP executes code and comments may not be available in deployed builds.

### 25.10 Primary sources

- [PHP Reflection — `getDocComment`](https://www.php.net/manual/en/reflectionclass.getdoccomment.php)
- [phpDocumentor PHPDoc reference](https://docs.phpdoc.org/guide/references/phpdoc/index.html)
- [phpDocumentor — What is a DocBlock?](https://docs.phpdoc.org/guide/getting-started/what-is-a-docblock.html)
- [PHP-FIG standards status](https://www.php-fig.org/psr/)
- [PHP attributes overview](https://www.php.net/manual/en/language.attributes.overview.php)

---

<a id="lang-go"></a>
## 26. Go

```yaml
profile_version: 1
language_id: go
rank: 14
primary_mechanism: Go doc comments
authority: ecosystem-official
native_syntax: ordinary line/block comments immediately before declarations
native_markup: Go doc comment syntax
preferred_symbol_source: go/parser, go/types, go/packages, module metadata
preferred_doc_source: go/doc and go/doc/comment
loss_risk_without_build_context: high
execution_required: false
```

### 26.1 Native strategy

Go uses ordinary comments in a specially positioned and conventionally worded role. A doc comment appears immediately before a top-level package, const, func, type, or var declaration with no intervening blank line. Exported names should have doc comments.

```go
// Open opens the named file.
//
// Deprecated: Use OpenContext when cancellation is required.
func Open(name string) (*File, error) {
    // ...
}
```

Package comments conventionally begin `Package name ...`. A declaration comment conventionally begins with the declared name. These conventions support readable source and tooling but are not an `@param` tag system.

### 26.2 Go doc markup

The official Go doc syntax supports:

- paragraphs;
- headings;
- links to Go packages and symbols;
- lists;
- preformatted/code blocks;
- notes of the form `MARKER(uid): body`;
- deprecation paragraphs beginning `Deprecated: `.

It intentionally omits complex formatting such as arbitrary raw HTML. `gofmt` can canonicalize documentation formatting. The syntax resembles a deliberately restricted Markdown subset but must be parsed with `go/doc/comment`, not a generic Markdown parser.

Known limitations matter to normalization: nested lists are not supported as a full hierarchical construct, and indentation determines code blocks.

### 26.3 Structured semantics from prose and code

Go docs normally describe parameters, returns, and errors in prose rather than tags. Actual signature structure comes from `go/types`. UDIR SHOULD:

- extract parameter/result names and types from code;
- retain prose as description;
- optionally apply conservative language-aware sentence heuristics;
- avoid pretending a sentence mentioning `x` is a formal parameter field;
- map `Deprecated:` natively;
- map recognized notes;
- create links through the Go doc resolver.

Named result parameters may be documented by name. Unnamed results should remain positional.

### 26.4 Examples

Go has an official executable-example convention in `_test.go` files:

- `Example`;
- `ExampleFoo`;
- `ExampleFoo_Bar`;
- suffix variants for multiple examples;
- `Output:` or `Unordered output:` comments.

The testing/doc tooling associates examples with packages, functions, types, and methods according to naming rules. Examples with expected output can be executed by `go test` and displayed in documentation.

UDIR must store:

- associated symbol;
- complete example body;
- display code;
- expected output and ordered/unordered mode;
- build tags and package context;
- execution disabled by default.

### 26.5 Package and build context

A package can span many files. The package comment should generally occur once, but tooling may concatenate multiple package comments. Build constraints determine which files exist for a target.

Capture:

- module path/version;
- package import path;
- package name;
- GOOS/GOARCH;
- build tags;
- cgo state;
- toolchain version;
- generated file markers;
- test/external-test package distinction.

pkg.go.dev documentation is built in a particular context and may not reflect every platform-specific variant. UDIR SHOULD be capable of a build matrix.

### 26.6 Symbol identity and methods

Canonical Go identity should include module version, import path, and:

- top-level name; or
- receiver base type and method name;
- instantiated/generic context when documenting an instantiation rather than declaration.

Pointer and value receiver method sets differ. Embedding can promote methods without transferring declaration ownership. Alias declarations and type definitions are different. Preserve these relations.

### 26.7 Directives versus docs

Comments beginning with `//go:` and related compiler/tool directives are semantically active and must not be interpreted as prose documentation. Build constraints also use comment syntax. Generated-code markers, line directives, cgo directives, and linter pragmas need classification.

A doc comment block can contain directive-looking lines; follow Go parser/tool rules and preserve directives separately where applicable.

### 26.8 Normalization rules

| Go construct | UDIR |
|---|---|
| declaration doc comment | summary/description using Go parser |
| package comment | package documentation |
| `[pkg.Symbol]` link | symbol-link |
| heading/list/code | doc AST |
| `Deprecated:` paragraph | lifecycle deprecation |
| `MARKER(uid):` note | named diagnostic/note extension |
| `Example...` function | example and `example-of` relation |
| `Output:` | expected output |
| `Unordered output:` | expected output with unordered comparison |
| embedded member promotion | relation, not copied ownership |
| `//go:` directive | source directive extension, excluded from docs |
| grouped declarations | group/spec-specific comment handling |

### 26.9 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Multiple package comments | One silently wins | Follow `go/doc` behavior; preserve individual fragments and concatenation |
| Grouped `const`/`var` declarations | Group comment applied to each item incorrectly | Preserve group and per-spec comments; follow Go doc rules |
| `iota` blocks | Values derived, comments interleaved | Use AST/type info and per-spec association |
| Embedded fields | Promoted methods/fields appear owned by embedding type | Keep declaration owner and promotion relation |
| Pointer vs value receiver | Methods displayed under wrong set | Store receiver form and method-set relations |
| Type aliases vs definitions | Semantics collapsed | Preserve alias relation and native declaration kind |
| Generics | Type parameters/constraints lost | Extract from `go/types`; prose remains separate |
| Build tags | Platform API merged incorrectly | Variant by GOOS/GOARCH/tags |
| cgo | Generated/imported C symbols vary | Record cgo/toolchain and generated relations |
| Generated files | Docs duplicate generator source | Mark generated and choose project policy |
| Internal packages | Public name is not universally importable | Preserve module visibility/path rules |
| External test package | Example association/package differs | Follow testing naming and package context |
| Example suffix parsing | Helper example misbound | Use official association rules |
| No `Output:` | Example may render but not run as a test | Store execution semantics accurately |
| `Unordered output:` | Ordered comparison assumed | Preserve comparison mode |
| Ambiguous short links | Same import name in files | Resolve using package-wide Go doc rules; report ambiguity |
| Indented prose | Accidentally becomes code block | Use `go/doc/comment` and retain raw formatting |
| Nested lists | Generic Markdown creates unsupported structure | Represent native flattened result plus raw text |
| Deprecated package | Entire package hidden/filter behavior varies | Lifecycle on package, not every symbol unless processor chooses display inheritance |
| `//go:`/build directives | Rendered as docs | Classify semantically active comments before documentation parse |
| `go:linkname` | Source/API/linkage identity diverges | Preserve unsafe linkage relation |
| Vendoring/replacements | Same import path resolves to different code | Include module graph/version/replacement digest |

### 26.10 Recommended adapter stack

Use `go/packages` for build-aware loading in syntax/types mode without executing package code, `go/parser`/`go/ast`, `go/types`, `go/doc`, and `go/doc/comment`. Use `go list`/module metadata in a constrained environment as needed.

### 26.11 Primary sources

- [Go Doc Comments](https://go.dev/doc/comment)
- [`go/doc`](https://pkg.go.dev/go/doc)
- [`go/doc/comment`](https://pkg.go.dev/go/doc/comment)
- [Go examples in tests](https://go.dev/blog/examples)
- [`go doc` command](https://pkg.go.dev/cmd/doc)
- [pkg.go.dev documentation](https://go.dev/about#adding-a-package)

---

<a id="lang-fortran"></a>
## 27. Fortran

```yaml
profile_version: 1
language_id: fortran
rank: 15
primary_mechanism: none-at-language-standard-level
authority: none
de_facto_mechanism: FORD
alternative_mechanism: Doxygen and project conventions
native_syntax:
  common_ford: ["!> preceding documentation", "!! trailing/continuation documentation"]
native_markup: Markdown plus FORD extensions
preferred_symbol_source: versioned Fortran parser/compiler AST
preferred_doc_source: FORD-compatible parser plus raw comments
loss_risk_without_build_context: critical
execution_required: false
dialect_and_source_form_required: true
```

### 27.1 Native strategy

The Fortran language defines comments but no standard semantic documentation-comment system. Modern Fortran projects commonly use **FORD**, which extracts special comment forms and renders Markdown-based API documentation. Doxygen is also used, with different syntax and feature coverage.

A common FORD pattern is:

```fortran
!> Compute a normalized vector.
!!
!! @param x Input vector.
!! @return Normalized vector.
pure function normalize(x) result(y)
    real, intent(in) :: x(:)
    real :: y(size(x))
    ! ...
end function normalize
```

FORD supports documentation preceding entities and trailing/continuation documentation, with behavior affected by free/fixed source form and project configuration.

### 27.2 FORD content model

FORD supports Markdown-oriented prose, symbol links, admonition-like blocks, source display, project pages, and metadata. It also recognizes a limited set of Doxygen-style commands and supports aliases/includes/configuration features.

A UDIR adapter SHOULD identify:

- summary/description paragraphs;
- parameters and return/result descriptions when structured commands are used;
- notes, warnings, tips, and other admonitions;
- symbol links;
- code blocks and examples;
- project pages and module/type/procedure grouping;
- external includes and environment substitutions;
- custom aliases/macros.

Because FORD behavior is configuration-dependent, record its project file and version.

### 27.3 Fortran semantic model

Fortran’s API cannot be recovered reliably from comments. Extract from a compiler/parser:

- modules and submodules;
- programs, block data, subroutines, functions;
- generic and explicit interfaces;
- module procedures and submodule implementations;
- derived types, inheritance/extension, type-bound procedures;
- operators and assignment interfaces;
- procedure pointers;
- abstract interfaces;
- parameters, variables, components;
- `intent(in/out/inout)`, `optional`, `value`, `pointer`, `allocatable`, `target`;
- array rank/shape, coarrays;
- kind and length parameters;
- result variables;
- visibility (`public`/`private`);
- `bind(C)` names and interoperability;
- elemental/pure/impure/recursive qualifiers.

The function’s result variable may have a different name from the function. UDIR should model the result as a return value and preserve the named result symbol relation.

### 27.4 Source forms and preprocessing

The adapter must know whether a file uses free or fixed source form. Column rules, continuation, comment sentinels, and line limits differ. Extensions such as `.f`, `.for`, `.f90`, `.F90`, and build flags are hints, not proof.

Preprocessing can include:

- C preprocessor;
- compiler-specific preprocessors;
- fypp or other generators;
- conditional compilation;
- include files.

Extract original and generated/preprocessed variants when practical. Comments may disappear or be generated. Preserve source maps.

### 27.5 Semantically active comment lines

OpenMP, OpenACC, and compiler directives often begin with comment-like sentinels such as `!$omp`, `!$acc`, or fixed-form variants. They are not documentation prose; they affect compilation/execution.

Classify:

1. ordinary comments;
2. FORD/Doxygen documentation comments;
3. conditional sentinel lines;
4. compiler directives;
5. preprocessor lines.

Never feed all leading `!` lines into a documentation parser.

### 27.6 Normalization rules

| Native/FORD construct | UDIR |
|---|---|
| preceding/trailing doc block | direct symbol docs according to FORD binding |
| `@param` | parameter docs matched to semantic dummy argument |
| `@return` | function result docs |
| Markdown prose | doc AST |
| FORD admonition | admonition/contract/named section |
| symbol link | symbol-link |
| module/project page | module/conceptual record |
| external include | include directive with safe resolution |
| generic interface | grouping/overload-like relation |
| `intent` | actual parameter direction from code |
| `bind(C)` | FFI/linkage relation |
| directive sentinels | source directive extension, not docs |

### 27.7 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Fixed vs free source form | Comment and continuation parse incorrectly | Require source form/build flags |
| `!$omp`/`!$acc` | Active directive rendered as prose | Classify sentinel directives first |
| Compiler directives | Comment-looking line changes optimization | Preserve as code metadata |
| Generic interfaces | One name maps to many specifics | Group record plus specific procedure records |
| Module procedure/submodule body | Declaration and implementation docs split | Model correspondence and choose public declaration policy |
| External procedure without explicit interface | Signature incomplete | Lower confidence; use build/interface metadata |
| Function result variable | Return docs bound to local variable only | Relate result variable to return semantic |
| Optional/intent/value | Documentation direction conflicts | Code semantics win; diagnostic |
| Assumed/deferred/assumed-rank arrays | Simplistic type string loses shape | Rich native type extension plus normalized rank |
| Coarrays | Array parser drops codimensions | Preserve coarray metadata |
| Kind/length parameters | `real(kind=...)` collapsed | Preserve expressions and resolved values where compiler provides |
| Type-bound procedures | Binding name differs from implementation procedure | Separate binding and procedure identities |
| Operators/assignment generics | Non-identifier API names | Preserve native generic operator identity |
| Include files | Docs/source spans originate elsewhere | Include stack/source map |
| CPP/FPP/fypp | Generated variants and comments differ | Record generator/macros and variant |
| Case insensitivity | Case-sensitive doc names mismatch | Resolve semantically, retain spelling |
| Common blocks/namelists | Not ordinary symbols | Dedicated kind/extension representation |
| `bind(C)` | Fortran and linker/C names differ | Preserve binding name and cross-language relation |
| FORD include/environment expansion | Traversal/data leak | Allowlist; disable env expansion unless explicit |
| FORD aliases | Custom syntax changes parsing | Store config and unknown alias nodes |
| Doxygen vs FORD markers | Same comment parsed differently | Declare processor per project/file |
| Legacy Hollerith/fixed-form quirks | Modern parser fails | Dialect/compiler version and raw fallback |
| Generated stdlib docs | Renderer output scraped instead of source | Prefer source+FORD config, retain generated output only as secondary |

### 27.8 Recommended adapter stack

Use a Fortran compiler frontend or mature parser with module and interface awareness, plus a FORD-compatible documentation parser. Record compiler, standard level, extensions, source form, preprocessing, and FORD version/configuration. Do not run arbitrary FORD preprocessors or include external files outside a sandbox.

### 27.9 Primary sources

- [FORD — Writing documentation](https://forddocs.readthedocs.io/en/latest/user_guide/writing_documentation.html)
- [FORD documentation](https://forddocs.readthedocs.io/)
- [Fortran-lang standard library API generated by FORD](https://stdlib.fortran-lang.org/)
- [Fortran standards community (WG5)](https://wg5-fortran.org/)
- [OpenMP Fortran directives](https://www.openmp.org/specifications/)

---

<a id="lang-ruby"></a>
## 28. Ruby

```yaml
profile_version: 1
language_id: ruby
rank: 16
primary_mechanism: RDoc
authority: ecosystem-official
native_syntax: comments preceding Ruby/C-extension declarations
native_markup_dialects: [RDoc, Markdown, RD, TomDoc]
additional_signature_source: RBS
preferred_symbol_source: Ruby parser plus RDoc/RBS model
preferred_doc_source: RDoc parser/store plus raw source
loss_risk_without_semantics: high
execution_required: false
```

### 28.1 Native strategy

RDoc is Ruby’s standard documentation system and ships tools for generating HTML and command-line `ri` documentation. Comments immediately preceding classes, modules, methods, constants, and attributes are associated with those entities.

```ruby
##
# Retrieves a user.
#
# Raises UserNotFoundError if no matching user exists.
def get_user(id)
  # ...
end
```

RDoc can parse Ruby, C extension source, RBS signature files, and standalone documentation files. Current RDoc supports multiple markup formats:

- RDoc markup, currently the default for Ruby/C source;
- Markdown;
- RD;
- TomDoc.

RDoc documentation recommends Markdown for new standalone documents and indicates a future shift toward Markdown, but adapters must not assume that shift has already occurred. Detect configuration per project/file/comment.

### 28.2 Markup and directives

RDoc supports directives that change parsing, visibility, or inferred declarations. Important classes include:

| Purpose | Representative directives |
|---|---|
| Signature display | `:call-seq:`, `:args:`/`:arg:` |
| Metaprogrammed methods | `:method:`, `:singleton-method:` |
| Metaprogrammed attributes | `:attr:`, `:attr_reader:`, `:attr_writer:`, `:attr_accessor:` |
| Inclusion/exclusion | `:nodoc:`, `:doc:`, `:stopdoc:`, `:startdoc:`, `:enddoc:` |
| Organization | `:section:`, `:category:` |
| Markup/project | `:markup:`, `:include:`, `:title:` |
| Yield information | version-supported yield argument/return directives |
| C extensions | `Document-class`, `Document-module`, `Document-method`, `Document-const`, `Document-global`, `Document-variable`, `call-seq` |

Directive support is parser/version-specific. Preserve unknown directives.

### 28.3 RBS integration

RDoc can parse `.rbs` files. When an RBS declaration matches an object documented from Ruby source, its comments and type signatures can extend existing documentation.

UDIR MUST keep:

- Ruby source declaration;
- RBS declaration/signature;
- comments from both sources;
- merge relation and processor/version;
- conflicts between runtime syntax, RBS, and prose.

RBS types are code/interface declarations, not merely documented types. Depending on project policy, they may be the authoritative public signature for dynamic Ruby.

### 28.4 Dynamic and reopened code

Ruby classes/modules are open and may be declared across files. Methods can be added through:

- ordinary reopenings;
- `define_method`;
- `class_eval`/`module_eval`;
- delegation macros;
- ActiveSupport/framework DSLs;
- refinements;
- native C extensions;
- generated code.

RDoc has directives to document some metaprogrammed entities. These produce virtual/documented symbols with explicit provenance. A static adapter should not execute arbitrary DSL code.

### 28.5 C extensions

RDoc’s C parser recognizes patterns and directives around Ruby C API calls such as class/module/method definitions. The C function name, Ruby-visible method name, arity, singleton/instance ownership, and call sequence can differ.

Model:

- C implementation symbol;
- Ruby API symbol;
- `implemented-by`/`corresponds-to` relation;
- declared call sequence;
- actual registered arity where statically recoverable;
- C comment source.

### 28.6 Normalization rules

| RDoc construct | UDIR |
|---|---|
| first paragraph/sentence by markup | summary |
| remaining content | description |
| automatic cross-reference | symbol-link with RDoc resolution metadata |
| `:call-seq:` | documented signatures, possibly multiple |
| `:args:` | documented parameter display |
| metaprogramming directives | virtual symbol declaration |
| `:nodoc:`/doc controls | documentation suppression/visibility |
| `:section:`/`:category:` | grouping |
| `:include:` | external include directive |
| RBS signature | actual/interface signature with source provenance |
| C `Document-*` | explicit Ruby symbol binding from C |
| `ri` store | generated/native output, not source of truth |

### 28.7 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Reopened class/module | One file overwrites or duplicates page | Merge members/fragments by semantic namespace; retain source ownership |
| Method redefinition | Historical definition appears current | Use load/order policy if known; keep shadowed declarations with diagnostics |
| Singleton vs instance method | Same display name collides | Include `.`/`#` semantic distinction in identity |
| Aliases | Docs copied without relation | Model alias and processor copy behavior |
| Visibility changed later | Declaration comment claims wrong visibility | Use semantic/static control flow where recoverable; preserve uncertainty |
| `define_method` | No ordinary method AST | Honor explicit RDoc directives or safe literal analysis |
| `method_missing` | Infinite virtual API claims | Only configured/explicit virtual symbols; never infer arbitrary names |
| Refinements | Method visible only in lexical activation | Include refinement container and variant context |
| Modules mixed in | Methods displayed under including class | Keep owner and mixin relation |
| C extensions | Ruby and C names/signatures differ | Cross-language relation and RDoc C parser |
| RBS merge | Conflicting comments/types | Preserve each source and diagnostic |
| Multiple markup formats | Generic Markdown corrupts RDoc markup | Detect/configure native markup |
| `:include:` | Arbitrary file read | Restrict roots and size; preserve blocked directive |
| `:call-seq:` multiple forms | One method has several call shapes | Multiple documented signatures linked to one implementation |
| Operator methods | Parser/linker mishandles punctuation | Use Ruby semantic names |
| Attribute macros | Getter/setter symbols generated | Create related generated symbols |
| Constants assigned dynamically | Value/type unavailable | Preserve declaration and dynamic status |
| Autoload | Resolving symbol could load code | Do not trigger autoload |
| Framework DSLs | Static extractor misses API | Plug-in architecture with declarative/sandboxed analyzers |
| Encodings/magic comments | Source parse changes | Respect encoding and parser version |
| Endless methods/pattern syntax/new Ruby | Old parser fails | Pin current Ruby/RDoc parser |
| RDoc store/version | Generated cache not portable | Treat as versioned secondary output |
| Preview server/plugins | Tool execution/network risk | Static parser only for untrusted input |

### 28.8 Recommended adapter stack

Use current RDoc parsers/stores, a Ruby syntax parser such as Prism/current official parser APIs, and an RBS parser. Prefer source and RBS over rendered HTML/RI output. Add framework-specific extractors only through declared, sandboxed plug-ins.

### 28.9 Primary sources

- [RDoc — Ruby Documentation System](https://ruby.github.io/rdoc/)
- [RDoc Markup](https://ruby.github.io/rdoc/RDoc/Markup.html)
- [RDoc Ruby parser](https://ruby.github.io/rdoc/RDoc/Parser/Ruby.html)
- [RDoc C parser](https://ruby.github.io/rdoc/RDoc/Parser/C.html)
- [RBS](https://ruby.github.io/rbs/)

---

<a id="lang-swift"></a>
## 29. Swift

```yaml
profile_version: 1
language_id: swift
rank: 17
primary_mechanism: Swift documentation comments plus DocC
authority: ecosystem-official
native_syntax: ["///", "/** ... */"]
native_markup: Markdown plus DocC field syntax, callouts, directives, and symbol links
conceptual_storage: .docc documentation catalogs
preferred_symbol_source: compiler-emitted symbol graphs / Swift semantic model
preferred_doc_source: symbol graph plus DocC parser and source comments
loss_risk_without_semantics: high
execution_required: false
```

### 29.1 Native strategy

Swift source documentation uses `///` or `/** ... */` comments. Apple’s and the Swift project’s DocC tool combines source comments with compiler-emitted symbol graphs and optional documentation catalogs.

```swift
/// Retrieves a user.
///
/// - Parameter id: The user identifier.
/// - Returns: The matching user.
/// - Throws: ``UserNotFoundError`` when no user exists.
func user(id: Int) throws -> User {
    // ...
}
```

For several parameters:

```swift
/// Updates a user.
///
/// - Parameters:
///   - id: The user identifier.
///   - name: The new display name.
/// - Returns: The updated user.
func update(id: Int, name: String) -> User { ... }
```

The first paragraph commonly becomes the abstract/summary. Remaining prose becomes discussion, followed by structured sections and callouts.

### 29.2 DocC field syntax and callouts

A UDIR adapter SHOULD recognize DocC’s structured documentation fields:

- singular `- Parameter name: ...`;
- grouped `- Parameters:` with nested parameter entries;
- `- Returns: ...`;
- `- Throws: ...`.

DocC also recognizes callout-style list items. Common callouts include notes, warnings, important information, tips, preconditions, postconditions, requirements, invariants, attention, authorship, bugs, complexity, copyright, dates, experiments, remarks, see-also, since, todo, and version information. Exact supported callouts/directives evolve, so parsing MUST be tied to a DocC version and unknown forms preserved.

Map callouts semantically only when their meaning is stable:

| Callout | UDIR |
|---|---|
| Note/Remark/Tip/Important/Warning/Attention | admonition |
| Precondition/Requires | precondition or requirement contract |
| Postcondition | postcondition contract |
| Invariant | invariant contract |
| Complexity | complexity contract |
| Throws | errors |
| Returns | returns |
| See Also | see-also |
| Since/Version | lifecycle |
| Author/Authors/Copyright | metadata |
| Todo/Bug/Experiment | named section/diagnostic metadata |

### 29.3 Symbol links

DocC supports links to symbols using a symbol-link syntax such as double backticks. Resolution uses the symbol graph, lexical context, module ownership, overload signatures, and path components.

The adapter MUST retain:

- original symbol-link spelling;
- resolved symbol-graph precise identifier;
- display representation/language;
- overload candidate set if ambiguous;
- external dependency/module relationship;
- fallback URL/fragment where applicable.

Do not turn a precise symbol relationship into a hard-coded generated-site URL.

### 29.4 Documentation catalogs

A `.docc` documentation catalog can contain:

- Markdown articles;
- tutorials and tutorial chapters;
- documentation extension files;
- images and other resources;
- technology/module metadata.

Documentation extensions target symbols and can add or replace portions of source documentation according to DocC’s merge rules. UDIR should represent extension pages as independent source fragments related to the target symbol, then materialize the resolved DocC view with processor/version provenance.

Conceptual pages are first-class `conceptual-document` records. Tutorials should preserve hierarchy, landmarks/steps, resources, and directives rather than being flattened into one description string.

### 29.5 Symbol graphs and language representations

Swift compilers emit symbol graphs containing symbols, declarations, source locations, documentation-comment lines, and relationships. SymbolKit/DocC consumes these graphs. They may include multiple language representations, especially Swift and Objective-C views of the same API.

Store:

- precise identifier/USR;
- module and target;
- access level;
- declaration fragments;
- navigator title;
- path components;
- relationships;
- source location;
- availability;
- SPI/internal metadata where emitted;
- language representation.

A Swift and Objective-C representation can correspond to the same underlying declaration but have different names/signatures. Model them as representations/corresponding symbols, not as accidental duplicates.

### 29.6 Semantic extraction

Use the compiler for:

- modules, types, extensions, protocols, actors;
- functions/methods/initializers/deinitializers/subscripts/operators;
- properties and accessors;
- associated types and generic parameters;
- generic requirements;
- async/throws/rethrows;
- ownership/isolation/sendability modifiers;
- opaque/result types;
- availability and platform annotations;
- synthesized members and conformances;
- reexports and extension ownership.

Doc fields are matched to semantic parameter names. Swift has external argument labels and local parameter names; both must be preserved. Documentation conventions generally name the local parameter, while display signatures include labels.

### 29.7 Normalization rules

| Swift/DocC construct | UDIR |
|---|---|
| first paragraph/abstract | summary |
| discussion | description |
| Parameter/Parameters fields | parameter docs |
| Returns | returns |
| Throws | errors section |
| callouts | contracts/admonitions/lifecycle/sections |
| double-backtick symbol link | symbol-link |
| `@available` and compiler availability | lifecycle availability |
| source `deprecated` availability message | lifecycle deprecation |
| documentation extension | amendment/replacement fragment relation |
| tutorial/article | conceptual document |
| symbol graph relation | UDIR relation |
| Objective-C representation | corresponding representation relation |
| unknown DocC directive | extension/native node |

### 29.8 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| External vs local parameter names | Docs matched to argument label only | Store both; match according to Swift semantic symbol |
| `_` argument labels | Empty/underscore name lost | Preserve label and local name separately |
| Overloads | Symbol link/name ambiguous | Resolve precise identifier; retain candidates |
| Initializers/subscripts/operators | Path syntax/parser treats as ordinary method | Use symbol graph kind and precise ID |
| Extensions in another file/module | Lexical owner confused with nominal type | Preserve declaring extension/module and extended-type relation |
| Protocol requirement/default implementation/conformance | Docs copied across distinct symbols | Separate requirement, default implementation, and witness relations |
| Synthesized conformances/members | API has no source comment | Mark generated and compiler/toolchain provenance |
| Async/throws/rethrows | Display and lowered ABI differ | Preserve source declaration and semantic flags |
| Actors/global actors/isolation | Concurrency contract omitted | Extract semantic isolation metadata and optional prose contracts |
| Opaque `some` and existential `any` | Type parser flattens distinctions | Preserve native type AST |
| Availability by platform/version | One generic since field is insufficient | Store structured native availability extensions plus normalized strings |
| SPI/package access | Public renderer leaks nonpublic API | Preserve actual access/SPI and apply export policy |
| Swift/Objective-C representations | Duplicate pages or wrong docs | Link corresponding representations and retain representation-specific text |
| C/C++ interoperability | Imported symbols lack Swift source docs | Preserve imported origin and cross-language identity |
| Documentation extension precedence | Source text silently overwritten | Retain every fragment and DocC merge operation |
| Duplicate extension files | Build emits conflicting pages | Follow DocC diagnostics; preserve conflict |
| Tuple return element docs | No universally stable structured field mapping | Store return tuple type plus prose/custom section; mark heuristic if split |
| Symbol graph version changes | Parser loses fields | Pin schema/toolchain; preserve unknown JSON |
| Reexports | API path differs from declaration module | Model reexport relation and display path |
| Conditional compilation | Symbols/docs vary by target/config | Variant key from Swift build settings |
| Macros | Generated declarations/docs need expansion | Use compiler output; mark generated and macro provenance |
| DocC directives/resources | File traversal or unsupported directive | Allowlist catalog root and preserve raw directive |
| Code snippets/tutorial actions | Potential execution | Never run during ingestion |
| Raw HTML/links/images | XSS/privacy/resource issues | Sanitize rendering; preserve source and resource digest |

### 29.9 Recommended adapter stack

Use compiler-emitted symbol graphs as the semantic backbone, original source for raw comments/spans, and DocC/Swift-DocC parsers for documentation catalogs and markup. Pin Swift and DocC versions because directives, symbol graph fields, and resolution behavior evolve.

### 29.10 Primary sources

- [Swift DocC](https://www.swift.org/documentation/docc/)
- [Documenting a Swift framework or package](https://www.swift.org/documentation/docc/documenting-a-swift-framework-or-package)
- [Writing symbol documentation in source files](https://www.swift.org/documentation/docc/documenting-a-swift-framework-or-package#Write-documentation-comments-in-your-source-code)
- [Linking to symbols and other content](https://www.swift.org/documentation/docc/linking-to-symbols-and-other-content)
- [Swift SymbolKit](https://github.com/swiftlang/swift-docc-symbolkit)
- [Swift DocC repository](https://github.com/swiftlang/swift-docc)

---

<a id="lang-perl"></a>
## 30. Perl

```yaml
profile_version: 1
language_id: perl
rank: 18
primary_mechanism: Plain Old Documentation (POD)
authority: ecosystem-official
native_syntax: command/ordinary/verbatim POD paragraphs embedded in Perl or standalone .pod
native_markup: POD
preferred_symbol_source: Perl parser/indexer plus package/distribution metadata
preferred_doc_source: Pod::Simple-compatible parse tree
loss_risk_without_symbol_convention: high
execution_required: false
```

### 30.1 Native strategy

POD is Perl’s native documentation format. It can be embedded in modules/scripts or stored in standalone `.pod` files. Perl ignores POD blocks in valid source positions, while formatters convert them to text, man pages, HTML, and other outputs.

```perl
=head1 NAME

My::Users - user lookup functions

=head1 FUNCTIONS

=head2 get_user($id)

Returns the requested user.

Throws C<UserNotFound> when no user exists.

=cut

sub get_user {
    my ($id) = @_;
    # ...
}
```

POD is document-oriented, not symbol-tag-oriented. Headings such as `FUNCTIONS` and `=head2 get_user($id)` are conventions, not a compiler binding. Most symbol associations will therefore be heuristic or project-configured.

### 30.2 Paragraph types and commands

POD is paragraph based:

- **ordinary paragraphs**;
- **verbatim paragraphs**, conventionally indicated by indentation;
- **command paragraphs** beginning with `=` at the start of a line.

Core commands include:

| Purpose | Commands |
|---|---|
| Start/end | `=pod`, `=cut` |
| Headings | `=head1`, `=head2`, `=head3`, `=head4` |
| Lists | `=over`, `=item`, `=back` |
| Encoding | `=encoding` |
| Formatter-specific data | `=for`, `=begin`, `=end` |

Formatting codes include:

- `B<>` bold;
- `I<>` italic;
- `C<>` code;
- `L<>` links;
- `E<>` escapes/entities;
- `F<>` filenames;
- `S<>` nonbreaking text;
- `X<>` index entries;
- `Z<>` zero-width content.

Nested and alternate delimiters have detailed parsing rules. Use `Pod::Simple`/the POD specification, not regular expressions.

### 30.3 Conventional sections

Common POD document sections include:

- NAME;
- SYNOPSIS;
- DESCRIPTION;
- METHODS;
- FUNCTIONS;
- OPTIONS or ARGUMENTS;
- RETURN VALUES;
- ERRORS or DIAGNOSTICS;
- EXAMPLES;
- SEE ALSO;
- AUTHOR;
- COPYRIGHT AND LICENSE.

UDIR MAY map these headings to semantic sections, but the mapping is convention-based. Parameter and return extraction from signatures/headings/prose SHOULD be labeled heuristic unless an additional formal convention is configured.

### 30.4 Embedding rules

POD must occur where Perl expects a new statement. It cannot safely begin in the middle of an expression. A POD block begins with a command paragraph and returns to Perl with `=cut`. At file end, `__END__` or `__DATA__` interactions and blank-line requirements affect recognition by formatters.

A critical parser warning: a standalone POD parser does not necessarily understand Perl tokenization. Text inside a here-document could resemble POD commands. For source files, tokenize/parse Perl first or use a POD extractor that respects source boundaries.

### 30.5 Symbol model

Perl APIs may include:

- packages/modules;
- subroutines;
- methods distinguished largely by calling convention;
- constants and package variables;
- object systems supplied by frameworks;
- import/export lists;
- AUTOLOAD/method generation;
- XS/C implementations;
- signatures/prototypes/attributes.

POD itself does not create stable symbol IDs. Build identity from distribution, module/package, declared subroutine, source span, and export metadata. Explicit project conventions can add structured directives/tags under an extension namespace.

### 30.6 Normalization rules

| POD construct | UDIR |
|---|---|
| NAME summary after dash | package summary by conventional parser |
| SYNOPSIS | named section/examples |
| DESCRIPTION | description |
| METHODS/FUNCTIONS headings | grouping; child symbol binding heuristic |
| RETURN VALUES | returns section, generally unbound prose |
| ERRORS/DIAGNOSTICS | errors section |
| EXAMPLES | examples |
| SEE ALSO / `L<>` | links |
| AUTHOR/LICENSE | metadata |
| heading/list/verbatim | doc AST |
| `X<>` | keywords/index entry |
| `=for`/`=begin` | formatter-specific native directive |
| unrecognized command/code | extension/unknown node |

### 30.7 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| No per-symbol native binding | Heading guessed as wrong subroutine | Mark heuristic; allow project mapping |
| Multi-package source file | POD assigned to first/last package globally | Track package scope and heading targets |
| Here-doc contains `=head1` | POD parser extracts string content | Parse/tokenize Perl before POD extraction |
| POD inside invalid source position | Perl compile failure but formatter accepts | Record source/parser diagnostic |
| Missing blank lines | Older formatters fail or merge paragraphs | Preserve raw layout and selected parser version |
| `__END__`/`__DATA__` | Documentation/data boundary confused | Follow Perl syntax and blank-line guidance |
| Alternate/nested formatting delimiters | Regex parser truncates content | Use POD parser AST |
| `=encoding` late/missing | Text decoded incorrectly | Honor directive and source bytes |
| Formatter-specific `=for`/`=begin` | Content lost in generic rendering | Preserve target formatter and raw body |
| Same heading repeated | Symbol sections merge unexpectedly | Retain hierarchy/source order |
| Prototypes/signatures | Heading signature differs from code | Actual parser signature wins; heading is documented signature |
| AUTOLOAD/metaprogramming | API absent statically | Explicit docs become virtual symbols; no arbitrary inference |
| Exporter lists | Public API not equal all subs | Parse declarative export metadata where safe |
| Object-system frameworks | Classes/attributes generated | Plug-in/dialect-specific extraction, sandboxed |
| XS functions | Perl-visible symbol implemented in C | Cross-language relation |
| POD after generated code | Source map unavailable | Mark generated/source-map missing |
| Link target forms | Manpage/module/section/URL ambiguity | Preserve parsed link structure and resolution status |
| Verbatim line width | Formatters wrap differently | Preserve exact text; renderer controls display |
| `podchecker` passes but renderer differs | Processor-specific output | Store parser/formatter versions and optionally test multiple |
| Embedded HTML via formatter data | XSS | Sanitize selected output; raw remains secured |
| International text | Formatter encoding mismatch | Normalize Unicode after authoritative decoding, preserve bytes digest |

### 30.8 Recommended adapter stack

Use a Perl-aware source parser or tokenizer to locate POD safely, then parse with `Pod::Simple`-compatible semantics. Implement conventional section-to-symbol mapping as an optional, confidence-scored layer. Never load modules to discover methods or exports in untrusted code.

### 30.9 Primary sources

- [Perl POD format (`perlpod`)](https://perldoc.perl.org/perlpod)
- [POD detailed specification (`perlpodspec`)](https://perldoc.perl.org/perlpodspec)
- [`Pod::Simple`](https://perldoc.perl.org/Pod::Simple)
- [`podchecker`](https://perldoc.perl.org/podchecker)
- [Perl module documentation guidance (`perlnewmod`)](https://perldoc.perl.org/perlnewmod)

---

<a id="lang-cobol"></a>
## 31. COBOL

```yaml
profile_version: 1
language_id: cobol
rank: 19
primary_mechanism: no-semantic-documentation-standard
authority: none
native_mechanisms:
  - fixed-format comment lines
  - floating/inline comment syntax by standard/dialect
  - identification-division informational paragraphs
preferred_symbol_source: dialect-aware COBOL parser with copybook/preprocessor context
preferred_doc_source: source comments plus configured project convention
loss_risk_without_dialect: critical
execution_required: false
dialect_required: true
source_format_required: true
```

### 31.1 Native strategy

COBOL has rich source-comment traditions but no cross-vendor semantic API documentation language. Comment syntax depends on source format and dialect.

Common forms include:

- fixed-format indicator-area comment lines, typically `*` in the indicator column;
- `/` comment/page-eject behavior in some fixed-format environments;
- floating/inline comments beginning `*>` in modern dialects/source formats;
- informational paragraphs in the `IDENTIFICATION DIVISION`, such as authoring, installation, dates, security, or remarks fields, depending on standard/vendor generation.

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. GET-USER.
       AUTHOR. PLATFORM-TEAM.

      *> Retrieves a user by identifier.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-USER-ID      PIC 9(18).
       01  LK-STATUS       PIC S9(9) COMP.
       PROCEDURE DIVISION USING LK-USER-ID
                                LK-STATUS.
```

The `*>` line is prose. The callable interface is inferred from `PROGRAM-ID`, `LINKAGE SECTION`, and `PROCEDURE DIVISION USING/RETURNING`, not from a standard doc tag.

### 31.2 Dialect and source format

A UDIR adapter MUST identify:

- vendor/compiler: IBM Enterprise COBOL, GnuCOBOL, Micro Focus, or another dialect;
- language level;
- fixed, free, variable, or vendor-specific source format;
- column rules and sequence/indicator areas;
- copybook search paths;
- conditional compilation variables;
- embedded-language preprocessors;
- character encoding/code page;
- target/runtime environment.

The same physical line can be code, continuation, comment, debug line, or ignored text depending on columns and options.

### 31.3 Symbol/API model

Potential documentable entities include:

- program, function, class, method, factory/object definitions where supported;
- nested programs;
- `ENTRY` points;
- sections and paragraphs;
- data items and group items;
- files/records;
- copybooks;
- interfaces described by `LINKAGE SECTION`;
- database/CICS calls and external program references.

Do not model every paragraph as a public function. Paragraphs and sections are control-flow labels unless project rules explicitly expose them.

A callable program identity should include library/load module, program ID, entry point, calling convention, dialect, and build variant. Data arguments require hierarchical PICTURE/USAGE/OCCURS/REDEFINES metadata; flattening to a single scalar type loses the ABI.

### 31.4 Copybooks and preprocessing

`COPY` and `REPLACE` can inject declarations, procedure code, and comments. Conditional compilation and vendor preprocessors can alter the resulting program. Embedded SQL and CICS commonly pass through preprocessors.

UDIR SHOULD maintain:

- original source location;
- copybook source location;
- expansion stack;
- replacement substitutions;
- preprocessed/generated location;
- selected build condition;
- source-map confidence.

A comment next to an expanded copybook item may apply to every inclusion, while an including program may add local context. Preserve both.

### 31.5 Project conventions

Organizations often use fixed comment banners or fields such as:

- program purpose;
- inputs/outputs;
- called/calling programs;
- files/tables;
- return codes;
- change history;
- abend/error conditions.

These are valuable but not standardized. Support them through declared templates, for example:

```yaml
cobol_comment_profile:
  id: enterprise-banner-v3
  fields:
    - header: PROGRAM PURPOSE
      maps_to: documentation.summary
    - header: INPUTS
      maps_to: documentation.parameters
    - header: RETURN CODES
      maps_to: documentation.errors
```

The parser MUST preserve unmatched lines and emit heuristic confidence. Change-history banners should normally become source-history metadata, not the current API contract.

### 31.6 Normalization rules

| COBOL construct | UDIR |
|---|---|
| PROGRAM-ID/FUNCTION-ID/CLASS-ID | symbol identity/kind |
| ENTRY | additional callable symbol related to program |
| PROCEDURE DIVISION USING | parameters |
| RETURNING | return |
| LINKAGE SECTION item | parameter/data-layout type |
| `*>` or fixed comment | annotation/direct docs only by configured binding |
| IDENTIFICATION fields | authorship/metadata where semantically valid |
| section/paragraph | control-flow symbol, visibility project-defined |
| COPY | include/generated-from relation |
| embedded SQL/CICS | sublanguage fragment/relation |
| return-code banner | errors only under declared convention |
| change log | named metadata/history extension |

### 31.7 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Wrong source format | Columns make comments/code swap roles | Source format mandatory |
| Sequence/identification areas | Numbers/text treated as code/docs | Preserve column regions |
| Continuation lines | Comment or literal concatenation misparsed | Use dialect lexer |
| Debug lines | `D` indicator behavior depends on option | Record debug mode/build variant |
| `*>` support/version | Legacy compiler rejects or treats differently | Dialect/version gate |
| COPY/REPLACE | Docs bind to expanded symbol with no origin | Preserve expansion stack and replacements |
| Conditional compilation | Mutually exclusive records/interfaces merged | Variant by condition |
| Embedded SQL | `--`/`/*` meanings belong to SQL sublanguage | Delegate fragment to SQL adapter after host preprocessing rules |
| DB2/CICS preprocessors | Comments can be stripped/rewritten | Preserve original and generated source |
| Multiple program units per file | File-level banner assigned globally | Bind by program boundaries |
| Nested programs | Qualified identity omitted | Include nesting and visibility |
| `ENTRY` | Alternative signatures missed | Separate entry symbols/signatures |
| Linkage data hierarchy | Flat parameter list loses groups | Recursive data-layout extension |
| `REDEFINES`/`OCCURS DEPENDING ON` | Type model cannot express overlay/dynamic arrays | Preserve native data schema and constraints |
| PICTURE/USAGE/sign/computational formats | Generic numeric type changes ABI | Native type metadata is mandatory |
| 88-level condition names | Treated as independent variables | Model named-value/condition relation |
| Paragraph fall-through/ALTER/GO TO | Control-flow labels misrepresented as callable | Keep paragraph kind and graph separately |
| External name truncation/case | Source ID differs from runtime program name | Preserve compiler/runtime linkage name |
| EBCDIC/code pages | Text/doc identifiers decode incorrectly | Record encoding and preserve byte digest |
| Compiler listing comments | Generated listing mistaken for source | Mark generated output and source-map |
| Informational paragraphs obsolete/vendor-specific | Metadata misread | Version/dialect field registry |
| Change-history banners | Old facts presented as current contract | Separate history metadata |
| Security/privacy in banners | Names/tickets/internal hosts leak | Redaction policy at publication |
| Copybook reused as data contract | One comment has many consumers | Shared symbol plus inclusion relations |
| Mixed fixed/free source directives | File changes format midstream | Honor compiler directives and segment variants |

### 31.8 Recommended adapter stack

Use a compiler-grade, dialect-aware parser with copybook expansion and source mapping. Parse original and preprocessed forms. Add project-specific banner parsers declaratively; never hard-code one enterprise’s fields as universal COBOL semantics.

### 31.9 Primary sources

- [IBM Enterprise COBOL documentation](https://www.ibm.com/docs/en/cobol-zos)
- [IBM Enterprise COBOL Language Reference](https://www.ibm.com/docs/en/cobol-zos/6.5.0)
- [GnuCOBOL Programmer’s Guide](https://gnucobol.sourceforge.io/guides.html)
- [GnuCOBOL project](https://gnucobol.sourceforge.io/)

---

<a id="lang-assembly"></a>
## 32. Assembly language

```yaml
profile_version: 1
language_id: assembly
rank: 20
primary_mechanism: no-universal-language-or-documentation-standard
authority: none
required_context:
  - assembler
  - assembler_version
  - architecture
  - syntax_mode
  - object_format
  - ABI
native_comments: assembler-and-target-specific
preferred_symbol_source: assembler parser plus object/debug/unwind metadata
preferred_doc_source: source comments under project-specific convention
loss_risk_without_context: critical
execution_required: false
```

### 32.1 Native strategy

“Assembly language” is not one language. Comment syntax, directives, macros, local labels, object metadata, and symbol rules depend on the assembler, target architecture, syntax mode, and object format.

Examples:

```asm
; NASM-style comment
global add_two
add_two:
    lea eax, [rdi + rsi]
    ret
```

```asm
# GNU assembler comment on many targets/syntaxes
.globl add_two
.type add_two, @function
add_two:
    lea (%rdi,%rsi), %eax
    ret
.size add_two, .-add_two
```

```asm
; MASM line comment
COMMENT !
This is a MASM block comment using ! as delimiter.
!
```

GNU `as` supports C-style block comments and target-specific line comment characters; `#` can also have line-marker/preprocessor significance. NASM uses `;` for comments. MASM supports `;` and a `COMMENT` directive with a chosen delimiter. Applying one lexer to all assembly is unsafe.

### 32.2 Required adapter context

Every record MUST carry:

- assembler and version (`gas`, `nasm`, `masm`, `armasm`, etc.);
- architecture and ISA revision;
- syntax mode (for example AT&T versus Intel where applicable);
- object format (ELF, COFF/PE, Mach-O, flat binary, etc.);
- ABI/calling convention;
- preprocessor/macro system;
- include paths/defines;
- target platform.

A file extension such as `.s`, `.S`, `.asm`, or `.inc` is only a hint.

### 32.3 Symbol semantics

Public API identity should come from code/object metadata, not nearby prose alone:

- labels and scoped/local labels;
- export/global/public directives;
- external/import directives;
- symbol type/size directives;
- sections and visibility;
- weak aliases and symbol versions;
- procedures such as MASM `PROC`/`ENDP`;
- debug information;
- unwind/CFI metadata;
- object-file symbol tables.

For ELF/GNU assembly, `.globl`, `.type`, `.size`, `.hidden`, `.weak`, `.symver`, and section directives can establish ABI-level facts. NASM and MASM use different directives. Preserve both source-level and object-level names.

### 32.4 Documentation conventions

There is no universal parameter/return tag standard. Projects often use banners describing:

- calling convention;
- input/output registers;
- clobbers;
- stack alignment;
- flags;
- memory effects;
- preconditions;
- constant-time/security properties;
- exceptions/faults;
- ABI ownership.

UDIR should support a declared profile:

```yaml
assembly_comment_profile:
  id: x86-register-contract-v1
  fields:
    input: documentation.parameters
    output: documentation.returns
    clobbers: extensions.assembly.clobbers
    flags: extensions.assembly.flags
    faults: documentation.errors
    requires: documentation.contracts
```

Register inputs are not ordinary source-language parameters unless correlated with an ABI signature. Preserve register-specific contracts in an assembly extension.

### 32.5 Object and source correlation

A build can provide stronger evidence:

- symbol table identifies exported symbols;
- DWARF/PDB/Stabs can identify functions, source lines, and types;
- unwind information identifies frame behavior;
- linker maps identify aliases and addresses.

However, comments are absent from object files. Correlate source documentation to object symbols via assembler/debug source maps. If debug data is absent, use source directives/labels with lower confidence.

Disassembly alone cannot recover comments and often cannot determine reliable function boundaries or high-level types. Do not claim documentation completeness from a binary-only input.

### 32.6 Normalization rules

| Assembly construct | UDIR |
|---|---|
| exported/global/public symbol | public function/data/label candidate |
| symbol type/PROC directive | symbol kind |
| symbol size/end directive | extent metadata |
| extern/import | external relation |
| weak/alias/version directive | alias/version relation |
| section | containment/linkage metadata |
| register contract comment | parameter/return plus assembly extension under configured profile |
| clobber/flags/stack notes | contracts/extensions |
| CFI/unwind | machine-readable frame/unwind metadata |
| macro definition | macro symbol |
| macro expansion | generated-from relation |
| local/numeric label | local symbol/control-flow node |
| line marker/preprocessor directive | source mapping, not documentation |

### 32.7 Edge-case report

| Edge case | Failure mode | Required handling |
|---|---|---|
| Wrong assembler | Comment delimiter/directives misparsed | Assembler/version mandatory |
| Target-specific `gas` comments | `#`, `;`, `@`, or other characters differ | Use target backend rules |
| `.S` preprocessing | C preprocessor changes comments/tokens | Capture preprocessor config and source maps |
| `#` line markers | Documentation parser treats mapping directive as prose | Classify as preprocessor/source directive |
| MASM `COMMENT` delimiter | Block terminates on arbitrary chosen character | Parse directive exactly |
| Macro-local labels | IDs collide across expansions | Include expansion instance/scoped identity |
| Numeric/anonymous labels | `1f`/`1b` style references have no stable global name | Local control-flow identity only |
| Conditional assembly | APIs vary by defines/architecture | Variant per condition |
| Syntax-mode switch | Same text tokenizes differently | Track mode changes in file |
| Intel vs AT&T operands | Signature/semantics inferred backwards | Never infer ABI from operand syntax alone |
| Function lacks type/size directives | Boundaries ambiguous | Use debug/unwind/linker evidence or low confidence |
| Data label mistaken for function | Export alone insufficient | Combine section, type, usage, debug data |
| Weak aliases/versioned symbols | Several public names share implementation | Separate alias/version records |
| Symbol decoration | `_name`, `@N`, mangling differ by ABI | Preserve source, object, and demangled IDs |
| CFI/unwind directives | Treated as comments or ordinary instructions | Parse as metadata/contracts |
| Handwritten vs compiler-generated assembly | Source comments/intent differ | Mark generated and preserve source language relation |
| Inline assembly | Host parser owns fragment boundaries | Nested assembly adapter with host context/constraints |
| Register aliases | Architecture mode changes register meaning | Architecture/ISA metadata |
| Clobber conventions | Comment conflicts with inline-asm constraint/object behavior | Preserve both and diagnose where analyzable |
| Self-modifying/code-data overlap | Symbol kind/boundaries unstable | Raw range records and warning |
| Includes | Same label/comments reused in many units | Include stack and object/module identity |
| Disassembly input | No native comments/source ownership | Binary-derived records only; no fabricated documentation |
| Debug info stripped | Source correlation unavailable | Mark source-map missing |
| Multiple ABIs | Same symbol uses platform-specific registers | Variant by ABI/platform |
| Privileged instructions/faults | Examples unsafe to execute | Never run code during extraction |
| Semicolon inside another assembler syntax | Comment assumption corrupts operand | Versioned lexer only |

### 32.8 Recommended adapter stack

Implement assembler-specific frontends rather than one “assembly parser.” Correlate source with object/debug metadata when possible. Support documentation fields only through declared project/ABI profiles, and preserve unstructured comments otherwise.

### 32.9 Primary sources

- [GNU assembler — Comments](https://sourceware.org/binutils/docs/as/Comments.html)
- [GNU assembler documentation](https://sourceware.org/binutils/docs/as/)
- [NASM language](https://www.nasm.us/doc/nasm03.html)
- [Microsoft MASM `COMMENT`](https://learn.microsoft.com/en-us/cpp/assembler/masm/comment-masm)
- [Microsoft MASM reference](https://learn.microsoft.com/en-us/cpp/assembler/masm/microsoft-macro-assembler-reference)
- [Arm assembler documentation](https://developer.arm.com/documentation/dui0801/latest/)

---

# Part III — Cross-language edge-case and platform reports

## 33. Cross-language edge-case taxonomy

The following cases recur across several ecosystems. They should be represented as first-class test categories, not patched independently in each renderer.

### 33.1 Symbol identity and overloads

| Case | Languages especially affected | Required UDIR behavior |
|---|---|---|
| Same name, different signatures | Java, C++, C#, VB, Rust traits/impls, Swift, SQL, Delphi, Fortran generics | One symbol record per overload/specific declaration; overload-group relation optional |
| Same API under several export paths | JavaScript, Rust, Swift, Ruby, PHP, Go aliases, SQL synonyms | Preserve declaration identity and every alias/reexport path |
| Source name differs from runtime/linkage name | C/C++, Fortran `bind(C)`, COBOL, assembly, .NET explicit interface members, Swift/ObjC | Store source, native compiler, linkage, and display identities separately |
| Anonymous or generated symbols | C/C++, JavaScript, PHP, Ruby, Swift macros, Rust macros, compiler-generated .NET | Source-scoped synthetic ID plus generated/unstable flag |
| Topic/page documents several symbols | R Rd, POD, conceptual DocC, package docs | Page record plus explicit `documents` relations |
| One symbol has several language representations | Swift/Objective-C, FFI, native extensions | Corresponding representation relation; do not collapse display signatures |
| Name rebinding/redefinition | Python, JavaScript, Ruby, PHP | Keep declarations and alias/shadow relations; do not pretend static load order is universally known |

### 33.2 Binding ambiguity

A comment parser MUST NOT assume “nearest comment wins” globally. Native attachment rules differ:

- Java requires one recognized documentation comment immediately before a documentable declaration.
- C# and VB bind XML documentation through compiler syntax.
- Python docstrings are statements inside a body.
- Rust inner comments bind to the enclosing item, while outer comments bind forward.
- Go binds comments immediately before declarations but has group/package rules.
- R topics and Perl POD are document-oriented.
- Scratch comments use explicit block IDs.
- SQL catalog comments use object keys.
- Doxygen/JSDoc/RDoc can explicitly name or create virtual entities.
- COBOL/assembly/project banners usually have no native binding.

When more than one binding is plausible, the adapter MUST emit all candidates, `ambiguous-binding`, and a confidence score. A renderer may ask a user to resolve ambiguity, but ingestion must remain deterministic.

### 33.3 Fragment composition, inheritance, and replacement

Documentation can be composed from multiple sources:

| Mechanism | Examples |
|---|---|
| Override inheritance | Javadoc `inheritDoc` and omission; .NET/DocFX inheritdoc; Doxygen copy/inherit |
| Inclusion | C# `<include>`; RDoc `:include:`; FORD includes; Javadoc snippets; `#[doc = include_str!]` |
| Partial/reopened declarations | C# and VB partials; Ruby reopenings; C/C++ redeclarations |
| Generated source | annotation/source generators; macros; roxygen; RBS; Delphi compiler XML |
| Extension/amendment pages | Swift DocC documentation extensions |
| Reexport presentation | Rust, Swift, JavaScript, Go |
| Topic aggregation | Rd `@rdname`, aliases, multiple usages |
| Copybook expansion | COBOL |
| Preprocessor/macro expansion | C/C++, Fortran, assembly, SQL migration tools |

The platform MUST store immutable **doc fragments** before producing a resolved view. A fragment record SHOULD contain:

- fragment ID;
- native source;
- owner/binding target;
- priority under the native processor;
- condition/variant;
- include or inheritance target;
- source span/digest;
- parser/processor version;
- normalized doc AST;
- rejected/conflicting status.

Resolved documentation is a derived artifact and should be reproducible from fragments plus processor policy.

### 33.4 Types and signatures

The data model must support four simultaneous representations:

1. source display signature;
2. semantic/compiler signature;
3. ABI/linkage signature where relevant;
4. documented signature/type.

Examples of legitimate divergence:

- Python docs express refined types not present in annotations.
- PHPDoc/Psalm/PHPStan types are richer than PHP native declarations.
- JSDoc types may be the only types or may conflict with TypeScript.
- R Rd `\usage` can intentionally abbreviate actual formals.
- RDoc `:call-seq:` can show several call shapes for one implementation.
- C array parameters adjust to pointers semantically.
- .NET XML IDs erase or encode information differently from source display.
- Swift has external labels, local parameter names, and ABI lowering.
- Fortran generic names dispatch to specific procedures.
- COBOL hierarchical data layouts are not scalar type strings.
- Assembly register contracts and object symbols do not imply a high-level function type.

UDIR MUST never destroy any representation during normalization.

### 33.5 Conditional APIs

Conditionality can arise from:

- C/C++ preprocessor definitions;
- Rust `cfg` and Cargo features;
- Go build constraints;
- Swift conditional compilation/targets;
- Fortran preprocessing;
- Delphi conditional defines;
- COBOL conditional compilation/preprocessors;
- assembly conditional assembly;
- Java multi-release builds/modules;
- .NET target frameworks and constants;
- JavaScript conditional exports;
- SQL environment/migration state;
- R package platform conditions.

A single project snapshot can therefore contain multiple valid API graphs. The platform SHOULD model:

```text
repository snapshot
    ├── build variant A
    │     └── symbol/document graph A
    └── build variant B
          └── symbol/document graph B
```

It SHOULD offer a “union view” only as a query/rendering projection that labels conditions.

### 33.6 Dynamic and generated APIs

Static extraction cannot fully observe:

- Python metaclasses/decorators/runtime assignments;
- JavaScript proxies/dynamic property creation;
- Ruby DSLs, `method_missing`, eval;
- PHP runtime declarations/eval/framework containers;
- R metaprogramming and runtime registration;
- compiler/procedural macros;
- source generators;
- database objects created dynamically;
- assembler macros with computed names.

Use three distinct states:

| State | Meaning |
|---|---|
| `generated` | A known tool/compiler produced the symbol; provenance available |
| `virtual` | Documentation explicitly declares a symbol without a native declaration |
| `dynamic-unobservable` | Existence/shape depends on execution and is not safely statically recoverable |

Optional runtime probes MUST run in a separate trust domain and produce separate provenance. They may enrich but not overwrite static records.

### 33.7 Semantically active “comments”

Several comment-looking forms affect compilation or execution:

- SQL optimizer hints/versioned executable comments;
- Go `//go:` directives and build constraints;
- Fortran OpenMP/OpenACC/compiler sentinels;
- assembly preprocessor/line-marker/comment ambiguities;
- COBOL debug lines and compiler directives;
- C/C++ pragmas/macros around comments;
- linter/tool suppression comments.

Adapters MUST classify lexical comments before documentation extraction. Active directives belong in code/build metadata, not the documentation description.

### 33.8 Executable documentation

Potentially executable constructs include:

- Python doctests;
- Rust doctests;
- Go examples;
- R examples and Rd `\Sexpr`;
- documentation generators with plug-ins;
- code snippets compiled by downstream tools;
- Sphinx/DocC/Javadoc include and plug-in mechanisms;
- SQL examples against a database.

The ingestion service MUST be non-executing. An optional validation service should use:

- isolated container/VM;
- no secrets;
- read-only or disposable filesystem;
- no network by default;
- CPU/memory/time/process limits;
- language dependency lockfile;
- exact toolchain;
- explicit user authorization;
- captured stdout/stderr/exit code;
- separate trust and provenance labels.

### 33.9 Markup and rendering loss

The following cannot be safely collapsed to generic Markdown without preservation:

- Javadoc inline/block tags and snippets;
- .NET/Delphi XML elements;
- Rd macros and output-mode conditionals;
- POD formatting/formatter blocks;
- RDoc directives and markup modes;
- DocC directives/tutorial structures;
- Doxygen commands/groups;
- Go’s restricted syntax;
- rustdoc hidden doctest lines;
- JSDoc/PHPDoc custom tags.

Renderers SHOULD convert normalized nodes to a common visual design, but MUST retain native nodes for unsupported semantics and provide a fallback representation.

### 33.10 Internationalization and encoding

Required behavior:

- decode according to language/file directives and compiler settings;
- preserve a hash of original bytes;
- normalize identity strings to NFC only for canonical hashing, not source display;
- retain exact case and quoted spelling;
- record case-folding semantics separately;
- avoid locale-dependent sorting/identity;
- preserve right-to-left and combining text safely;
- sanitize bidirectional-control characters in code display without silently deleting them;
- distinguish translated documentation from source-language documentation;
- allow multiple localized fragments linked to one symbol.

## 34. Security model

### 34.1 Threat matrix

| Threat | Examples | Mandatory controls |
|---|---|---|
| Code execution | Importing Python/PHP/Ruby modules; doctests; R `\Sexpr`; generator plug-ins | Static parsing; execution off; isolated optional validator |
| Process execution | Doxygen filters, Sphinx extensions, compiler plug-ins | Deny by default; allowlisted sandbox only |
| Path traversal | XML includes, DocC resources, FORD/RDoc includes, Javadoc snippets | Canonicalize path; workspace-root allowlist; no symlink escape |
| XML attacks | .NET/Delphi external entities and entity expansion | DTD/external entities disabled; expansion and depth limits |
| Archive attacks | Scratch `.sb3`, source packages | Entry/path/count/size/compression limits |
| Parser denial of service | Deep nesting, huge comments, pathological regexes/types | Streaming/iterative parsers, depth/size/time limits |
| XSS/content injection | Raw HTML, SVG, links, custom templates | Sanitization, CSP, safe URL schemes, isolated previews |
| SSRF/network fetch | External images/includes/spec links, plug-ins | No fetch during ingest; proxy/allowlist if later enabled |
| Secret leakage | SQL credentials/catalogs, source paths, internal URLs/tickets | Secret scanning, field-level access, redaction at publication |
| Supply-chain compromise | Third-party parsers/generators | Pin versions/checksums; sandbox; SBOM; minimal privileges |
| Repository escape | Git submodules/symlinks/includes | Resolve against snapshot manifest; never follow untrusted links by default |
| Data poisoning | Malicious docs targeting search/AI | provenance, trust labels, moderation, prompt-injection isolation |
| Link spoofing | Unicode confusables/unsafe schemes | resolved IDs, display URL policy, scheme allowlist |
| Resource exfiltration | HTML images, tutorial resources | proxy or block remote loads |
| Binary parsing risk | object/debug files | hardened parser process and file limits |

### 34.2 Trust labels

Every fragment SHOULD have a trust label:

- `first-party-source`;
- `generated-first-party`;
- `dependency-source`;
- `compiler-output`;
- `live-catalog`;
- `runtime-observation`;
- `external-link`;
- `heuristic`;
- `untrusted-upload`.

Trust affects display, execution eligibility, link fetching, and AI retrieval. It must not affect whether raw provenance is retained.

### 34.3 AI ingestion controls

Because the user intends to ingest documentation into a documentation tool, possibly with AI features, treat documentation as untrusted content:

- never let doc text alter system/tool instructions;
- separate code/document context from control prompts;
- store provenance with every retrieved chunk;
- cap retrieved text per source;
- filter secrets and access-controlled records before embedding/retrieval;
- preserve symbol IDs so citations point to exact versions;
- do not execute commands found in examples;
- display whether text is direct, inherited, generated, or heuristic.

## 35. Conformance levels

An adapter can claim one of four levels per language/dialect/version.

| Level | Name | Requirements |
|---:|---|---|
| 0 | Lossless intake | Raw bytes/text, syntax classification, source spans, parser diagnostics, no execution |
| 1 | Structural documentation | Native documentation AST, unknown-node preservation, safe rendering |
| 2 | Symbol-aware normalization | Semantic symbol graph, stable IDs, parameters/types/relations, UDIR field mapping |
| 3 | Native-processor parity | Document binding, link resolution, inheritance/includes, output selection closely match pinned official/de-facto processor |
| 4 | Variant and ecosystem completeness | Build matrix, generated/FFI symbols, package/catalog integration, conformance fixtures and drift detection |

A tool MUST state level by adapter, not globally. It may be Level 3 for Java and Level 1 for COBOL banners.

## 36. Adapter conformance suite

### 36.1 Required universal fixture classes

Every adapter fixture repository SHOULD contain:

1. empty and malformed documentation;
2. Unicode and mixed line endings;
3. all native comment forms;
4. every supported symbol kind;
5. every native structured field;
6. unknown/custom tags;
7. overloaded or duplicate names;
8. broken and ambiguous links;
9. direct versus inherited/included/generated docs;
10. conditional variants;
11. source/generated/output drift;
12. large/deep/pathological input;
13. unsafe includes/raw HTML/URLs;
14. documented-versus-actual type conflicts;
15. missing and orphan parameters;
16. aliases/reexports;
17. multi-file/package/module ownership;
18. cross-language/FFI relationships;
19. deterministic repeat extraction;
20. version-upgrade golden-output comparison.

### 36.2 Language-specific fixture minimums

| Adapter | Mandatory special fixtures |
|---|---|
| Python | attribute/additional docstrings, `.pyi` overloads, decorators, dynamic `__doc__`, Google/NumPy/Sphinx dialects |
| C | typedef/tag split, macros, prototypes/definitions, conditional compilation, multiple declarators |
| C++ | operators, templates/specializations, concepts, modules, redeclarations, inheritance |
| Java | both comment syntaxes, every standard tag, inheritance by omission, snippets, modules/records |
| C# | XML IDs, partials, explicit interface implementations, records, nullable types, includes/inheritdoc |
| JavaScript | ESM/CJS/conditional exports, virtual symbols, prototype patterns, destructuring, plug-ins |
| Visual Basic | case-insensitive names, modules, default properties, partials, `ByRef`/`ParamArray` |
| SQL | at least PostgreSQL, SQL Server, Oracle, MySQL; source/catalog drift; overloaded routines; hints |
| R | aliases/topics, S3/S4, datasets, roxygen inheritance, examples wrappers, `\Sexpr` suppression |
| Rust | inner/outer docs, cfg/features, macros, reexports, intra-doc links, doctest flags |
| Delphi | interface/implementation, overloads, helpers, conditional defines, emitted XML |
| Scratch | attached/free comments, missing blocks, ID rewrites, custom blocks, corrupt archives |
| PHP | native/PHPDoc conflicts, magic members, traits, tool-specific types, attributes |
| Go | grouped declarations, build tags, examples, links, embedded methods, directives |
| Fortran | fixed/free form, directives, generics, submodules, result variables, preprocessing |
| Ruby | reopenings, singleton methods, directives, C extension, RBS merge, markup variants |
| Swift | parameter labels, overload links, extensions, protocols/witnesses, DocC catalogs, representations |
| Perl | POD embedding, here-doc false positives, formatter blocks, encoding, heuristic method headings |
| COBOL | fixed/free formats, copybooks, continuation, embedded SQL, nested programs, hierarchical data |
| Assembly | GAS/NASM/MASM, comment characters, macros, local labels, conditional assembly, object correlation |

### 36.3 Golden data

Each fixture SHOULD produce:

- raw native payload;
- normalized UDIR JSON;
- diagnostics;
- native processor comparison output where safe;
- source map;
- deterministic digest.

Golden files MUST be keyed by adapter and toolchain version. A parser upgrade that changes output requires an explicit migration review rather than silently rewriting the index.

### 36.4 Round-trip policy

UDIR is not generally a source-code round-trip format. It is intentionally richer in provenance but can be semantically lossy across native systems. Therefore:

- **native-to-UDIR-to-native byte equality is not required**;
- native raw payload equality MUST be retained;
- UDIR-to-common-renderer semantic equivalence SHOULD be tested;
- a source rewriter MUST use the native AST/raw payload and adapter-specific formatter, not regenerate comments from normalized fields alone.

## 37. Suggested storage architecture

A scalable data platform should separate immutable evidence from derived views.

### 37.1 Logical entities

| Entity | Purpose |
|---|---|
| `repository_snapshot` | Commit/archive/database snapshot identity |
| `build_variant` | Toolchain, dialect, feature flags, target, dependencies |
| `source_artifact` | File, catalog snapshot, generated XML/JSON, object/debug artifact |
| `native_fragment` | Raw comment/doc block/topic/catalog property and native AST |
| `symbol` | Canonical symbol identity and semantic signature |
| `symbol_alias` | Native IDs, export paths, linkage names, aliases |
| `doc_binding` | Fragment-to-symbol association with method/confidence/condition |
| `normalized_record` | UDIR JSON for direct or resolved view |
| `relation` | Symbol/document graph edge |
| `diagnostic` | Validation, conflict, security, loss, or parser report |
| `blob` | Deduplicated raw text/binary/output content by digest |
| `source_registry` | Authority URLs and captured versions |
| `adapter_run` | Adapter/config/version, timing, success, logs, sandbox policy |

### 37.2 Recommended keys

```text
repository_snapshot(snapshot_id, repository_uri, revision, content_digest)
build_variant(variant_id, snapshot_id, language_id, context_digest)
source_artifact(artifact_id, variant_id, uri, kind, digest)
native_fragment(fragment_id, artifact_id, span, syntax, raw_blob_id, native_ast_blob_id)
symbol(symbol_id, variant_id, canonical_id, native_id, kind, qualified_name)
doc_binding(fragment_id, symbol_id, binding, confidence, priority)
normalized_record(record_id, symbol_id, schema_version, view_kind, json_blob_id)
relation(source_id, kind, target_id, condition, provenance)
diagnostic(diagnostic_id, run_id, code, severity, artifact_id, span)
adapter_run(run_id, adapter_id, adapter_version, variant_id, policy_digest)
```

A document database can store full UDIR records, but a relational/graph side index is still useful for symbol links, overloads, inheritance, and impact analysis. Raw blobs should be content-addressed to deduplicate repeated inherited/generated text.

### 37.3 Direct and resolved views

Store at least:

- `direct` — only text directly owned by the source symbol/topic;
- `resolved-native` — after native inheritance/includes/extension rules;
- `render-safe` — after sanitization and blocked-resource placeholders;
- `search` — chunked/indexed projection with source IDs;
- `latest` — pointer, not a destructive overwrite of older snapshot data.

### 37.4 Incremental ingestion

A change detector SHOULD hash:

- source artifact bytes;
- parser/adapter version;
- build context;
- dependency/module graph;
- external included file digests;
- generator config;
- source registry mapping version.

Recompute downstream records only when one of these inputs changes. Link resolution and inherited documentation require dependency-aware invalidation.

## 38. Common rendering contract

The unified visual renderer SHOULD present a consistent order:

1. symbol name, kind, language, package/module, availability;
2. canonical/display signatures;
3. summary;
4. lifecycle/deprecation warning;
5. description;
6. parameters/type parameters;
7. returns/yields;
8. errors/panics/faults;
9. contracts and side effects;
10. examples;
11. named/custom sections;
12. see-also and relationships;
13. provenance and source links;
14. extraction diagnostics when requested.

It MUST label:

- direct, inherited, included, generated, virtual, catalog, or heuristic text;
- actual versus documented types where conflicting;
- conditional build applicability;
- unresolved/ambiguous links;
- suppressed unsafe content;
- processor/dialect when behavior is nonportable.

Native-only material should appear in a structured “Additional native metadata” area rather than disappearing.

## 39. Search and chunking contract

For semantic search or AI retrieval:

- one chunk should not cross symbol boundaries unless it is a page-level overview;
- parameter/return/error fields should retain the parent symbol ID;
- inherited text should cite both displayed symbol and original owner;
- aliases should resolve to the declaration but retain the queried path;
- code examples should be separate child chunks linked to the symbol;
- catalog environment/security labels must be part of access filtering;
- diagnostics and unknown tags should be searchable but lower-ranked by default;
- generated boilerplate can be de-duplicated by digest;
- chunk text must retain section role so “Returns” is not confused with narrative prose.

Suggested chunk metadata:

```json
{
  "recordId": "udir:v1:java:...",
  "symbolId": "udir:v1:java:...",
  "sectionRole": "parameter",
  "parameterName": "id",
  "language": "java",
  "qualifiedName": "com.example.UserService.getUser",
  "coverage": "inherited",
  "sourceOwnerId": "udir:v1:java:base...",
  "variantKey": "jdk26-linux-x64",
  "trust": "first-party-source",
  "accessPolicy": "repository-members"
}
```

## 40. Example normalized records

These are abbreviated illustrations; the JSON Schema remains authoritative.

### 40.1 Java method with inherited field

```json
{
  "schemaVersion": "1.0",
  "recordId": "udir:v1:java:example",
  "recordKind": "symbol-documentation",
  "language": {
    "id": "java",
    "displayName": "Java",
    "dialect": "Javadoc standard doclet",
    "version": "26",
    "toolchain": "OpenJDK",
    "toolchainVersion": "26",
    "sourceFormat": "markdown-doc-comment",
    "authority": "compiler-vendor-official",
    "authoritySources": ["java:javadoc:jdk26"]
  },
  "symbol": {
    "canonicalId": "java:module/com.example/UserService#getUser(long)",
    "nativeId": "com.example.UserService#getUser(long)",
    "alternateIds": [],
    "kind": "method",
    "name": "getUser",
    "displayName": "getUser",
    "qualifiedName": "com.example.UserService.getUser",
    "visibility": "public",
    "modifiers": ["public"],
    "signatures": [],
    "source": null,
    "variantKey": "jdk26",
    "generated": false
  },
  "documentation": {
    "summary": {"kind": "paragraph", "text": "Retrieves a user.", "children": [], "level": null, "language": null, "title": null, "target": null, "attributes": {}, "raw": null},
    "description": {"kind": "document", "text": null, "children": [], "level": null, "language": null, "title": null, "target": null, "attributes": {}, "raw": null},
    "sections": [],
    "parameters": [
      {
        "name": "id",
        "position": 0,
        "documentation": {"kind": "paragraph", "text": "the user identifier", "children": [], "level": null, "language": null, "title": null, "target": null, "attributes": {"coverage": "inherited"}, "raw": null},
        "documentedType": null,
        "sourceNativeTag": "@param"
      }
    ],
    "typeParameters": [],
    "returns": [],
    "yields": [],
    "errors": [],
    "contracts": [],
    "examples": [],
    "seeAlso": [],
    "lifecycle": {"since": null, "deprecated": null, "removed": null, "stability": null, "availability": []},
    "authors": [],
    "license": null,
    "keywords": []
  },
  "native": {
    "sourceSyntax": "///",
    "markup": "Javadoc Markdown",
    "raw": "/// Retrieves a user.",
    "parsed": {},
    "generator": "javadoc standard doclet",
    "generatorVersion": "26",
    "outputFormat": null,
    "unknownConstructs": []
  },
  "provenance": {
    "binding": "compiler",
    "confidence": 1,
    "coverage": "inherited",
    "inheritancePath": ["base-method-id"],
    "sources": [{"kind": "source", "uri": "src/UserService.java", "span": null, "authority": "direct", "digest": null, "version": null}],
    "extractor": "udir-java",
    "extractorVersion": "1.0.0",
    "extractedAt": "2026-09-01T00:00:00Z"
  },
  "relations": [],
  "diagnostics": [],
  "lossFlags": ["inherited-doc-expanded"],
  "extensions": {}
}
```

### 40.2 SQL source and deployed catalog drift

```yaml
source_record:
  recordKind: symbol-documentation
  symbol:
    canonicalId: "postgresql:repo:public.app_user:table"
  documentation:
    summary: "Canonical application user records."
  provenance:
    binding: ast-adjacent
    sources:
      - kind: source
        uri: migrations/V014__users.sql

catalog_record:
  recordKind: catalog-documentation
  symbol:
    canonicalId: "postgresql:prod:app:public.app_user:table"
  documentation:
    summary: "Application login accounts."
  provenance:
    binding: catalog-key
    sources:
      - kind: catalog
        uri: "postgresql://prod/app/public/app_user"

relation:
  kind: corresponds-to
diagnostic:
  code: UDIR-CATALOG-DRIFT
```

### 40.3 Scratch annotation

```yaml
recordKind: workspace-annotation
language:
  id: scratch
symbol:
  kind: block
  canonicalId: "scratch:project-123:sprite-7:block-abc"
documentation:
  summary: null
  description:
    kind: paragraph
    text: "Keep this loop synchronized with the music."
relations:
  - kind: attached-to
    target:
      kind: symbol
      target: "scratch:project-123:sprite-7:block-abc"
extensions:
  scratch:
    commentIdSaved: "comment-old"
    commentIdEffective: "block-abc_comment"
    x: 120
    y: 80
    width: 200
    height: 120
    minimized: false
```

## 41. Implementation roadmap

### 41.1 Foundation

Build these components before language adapters:

1. UDIR schema package and validators.
2. content-addressed raw/native AST storage.
3. doc AST and type AST libraries.
4. canonical identity/hashing library.
5. source-span and fragment model.
6. diagnostics registry.
7. safe markup renderer.
8. link-resolution graph.
9. adapter process sandbox and capability manifest.
10. golden-fixture runner.

### 41.2 Initial adapters

A practical first release should target ecosystems with strong structured compiler/tool APIs:

- Java/Javadoc;
- C# and Visual Basic XML documentation via Roslyn;
- Go;
- Rust;
- Swift/DocC;
- Python with explicit docstring dialect configuration.

Then add:

- JavaScript/JSDoc and PHP/PHPDoc, where dialect control is central;
- Ruby/RDoc/RBS and R/Rd/roxygen;
- C/C++ via Clang+Doxygen;
- vendor-specific SQL adapters;
- Delphi;
- Fortran/FORD;
- Perl/POD;
- COBOL;
- assembler-specific adapters;
- Scratch.

This order is an engineering recommendation, not a ranking of language importance. It starts with stronger native symbol identities and testable semantics.

### 41.3 Adapter boundary

Each adapter process should expose:

```text
probe(input) -> supported language/dialect/version confidence
inventory(input, policy) -> build variants and required capabilities
extract(variant, policy) -> source artifacts, fragments, symbols, relations
normalize(native_graph, udir_version) -> UDIR records
validate(records, native_graph) -> diagnostics
resolve(records, graph, policy) -> resolved-native view
```

The core platform owns storage, security policy, schema validation, rendering, indexing, and access control. Adapters own native syntax and semantics.

### 41.4 Explicit non-goals for version 1

Do not make these version-1 promises:

- byte-perfect regeneration of native comments;
- arbitrary framework/metaprogramming discovery;
- universal execution of examples;
- one generic SQL/assembly/COBOL parser;
- perfect documentation inheritance parity without a pinned processor;
- automatic conversion of all prose into structured parameters/errors;
- lossless conversion of every native markup feature into generic Markdown;
- stable IDs across package renames or major compiler identity changes without mapping data.

## 42. Versioning and governance

### 42.1 Separate versions

Track independently:

- UDIR schema version;
- language profile/mapping version;
- adapter implementation version;
- native parser/compiler/generator version;
- source registry capture date;
- renderer version;
- sanitizer policy version.

A schema-compatible adapter update can still change symbol binding or inherited text; that must trigger review and new golden data.

### 42.2 Change policy

- Adding an optional extension field: mapping minor version.
- Adding a new enum value that old validators reject: schema minor/major according to compatibility policy.
- Changing canonical identity components: schema/identity major version.
- Changing native precedence/inheritance behavior: adapter major version.
- Correcting a source link or prose guidance: document patch version.
- Adding a newly popular language: profile collection minor version; existing ranking snapshot remains immutable.

### 42.3 Source review

Official and de-facto tools evolve. Review at least:

- monthly for ranking metadata if a “current top 20” view is offered;
- per language/tool release for syntax/processor changes;
- immediately for security advisories in parsers/generators;
- annually for inactive/legacy source links.

Keep previous profile snapshots so old documentation builds remain reproducible.

---

# Appendix A — Machine-readable language registry

```yaml
registry_version: 1
ranking:
  provider: TIOBE
  edition: 2026-08
  captured: 2026-09-01
languages:
  - rank: 1
    id: python
    name: Python
    primary: docstrings
    authority: language-standard
    structured_fields: dialect-dependent
  - rank: 2
    id: c
    name: C
    primary: Doxygen/Clang conventions
    authority: de-facto
    structured_fields: processor-dependent
  - rank: 3
    id: cpp
    name: C++
    primary: Doxygen/Clang conventions
    authority: de-facto
    structured_fields: processor-dependent
  - rank: 4
    id: java
    name: Java
    primary: Javadoc
    authority: compiler-vendor-official
    structured_fields: native
  - rank: 5
    id: csharp
    name: C#
    primary: XML documentation comments
    authority: compiler-vendor-official
    structured_fields: native
  - rank: 6
    id: javascript
    name: JavaScript
    primary: JSDoc
    authority: de-facto
    structured_fields: processor-dependent
  - rank: 7
    id: visual-basic-dotnet
    name: Visual Basic
    primary: XML documentation comments
    authority: compiler-vendor-official
    structured_fields: native
  - rank: 8
    id: sql
    name: SQL
    primary: vendor source/catalog metadata
    authority: dialect-vendor-specific
    structured_fields: vendor-dependent
  - rank: 9
    id: r
    name: R
    primary: Rd
    authority: ecosystem-official
    structured_fields: native
  - rank: 10
    id: rust
    name: Rust
    primary: rustdoc
    authority: language-standard
    structured_fields: convention-plus-tool
  - rank: 11
    id: delphi
    name: Delphi/Object Pascal
    primary: Delphi XML documentation
    authority: compiler-vendor-official
    structured_fields: native
  - rank: 12
    id: scratch
    name: Scratch
    primary: workspace comments
    authority: ecosystem-official
    structured_fields: none-for-api
  - rank: 13
    id: php
    name: PHP
    primary: PHPDoc
    authority: de-facto
    structured_fields: processor-dependent
  - rank: 14
    id: go
    name: Go
    primary: Go doc comments
    authority: ecosystem-official
    structured_fields: limited
  - rank: 15
    id: fortran
    name: Fortran
    primary: FORD
    authority: de-facto
    structured_fields: processor-dependent
  - rank: 16
    id: ruby
    name: Ruby
    primary: RDoc
    authority: ecosystem-official
    structured_fields: convention-plus-tool
  - rank: 17
    id: swift
    name: Swift
    primary: DocC
    authority: ecosystem-official
    structured_fields: native
  - rank: 18
    id: perl
    name: Perl
    primary: POD
    authority: ecosystem-official
    structured_fields: document-oriented
  - rank: 19
    id: cobol
    name: COBOL
    primary: comments/project conventions
    authority: none
    structured_fields: project-specific
  - rank: 20
    id: assembly
    name: Assembly language
    primary: assembler/project conventions
    authority: none
    structured_fields: project-specific
```

# Appendix B — Source registry

All sources below are primary language-project, compiler/vendor, or principal documentation-tool sources unless noted as a de-facto ecosystem tool. Capture date: **2026-09-01**.

| Source ID | Scope | Authority | URL |
|---|---|---|---|
| `ranking:tiobe:2026-08` | Top-20 ranking | ranking provider | <https://www.tiobe.com/tiobe-index/> |
| `python:pep257` | Docstring semantics/conventions | Python PEP | <https://peps.python.org/pep-0257/> |
| `python:pydoc` | Standard doc renderer | Python docs | <https://docs.python.org/3/library/pydoc.html> |
| `python:doctest` | Executable examples | Python docs | <https://docs.python.org/3/library/doctest.html> |
| `python:inspect-getdoc` | Runtime cleaning/inheritance | Python docs | <https://docs.python.org/3/library/inspect.html#inspect.getdoc> |
| `python:ast` | Static source model | Python docs | <https://docs.python.org/3/library/ast.html> |
| `python:pep287` | reStructuredText proposal | Python PEP | <https://peps.python.org/pep-0287/> |
| `c:doxygen-docblocks` | De-facto documentation blocks | Doxygen | <https://www.doxygen.nl/manual/docblocks.html> |
| `c:doxygen-commands` | De-facto command registry | Doxygen | <https://www.doxygen.nl/manual/commands.html> |
| `c:clang-comments` | Compiler comment model | LLVM/Clang | <https://clang.llvm.org/docs/InternalsManual.html#comment-handling> |
| `c:wg14` | Language standards body | ISO WG14 | <https://www.open-std.org/jtc1/sc22/wg14/> |
| `cpp:lex-comment` | C++ lexical comments | C++ draft | <https://eel.is/c++draft/lex.comment> |
| `cpp:libtooling` | Semantic extraction | LLVM/Clang | <https://clang.llvm.org/docs/LibTooling.html> |
| `java:javadoc:jdk26` | Javadoc syntax/tags | Oracle/OpenJDK | <https://docs.oracle.com/en/java/javase/26/docs/specs/javadoc/doc-comment-spec.html> |
| `java:javadoc-tool:jdk26` | Javadoc tool | Oracle/OpenJDK | <https://docs.oracle.com/en/java/javase/26/docs/specs/man/javadoc.html> |
| `java:javadoc-markdown:jdk26` | Markdown comments | Oracle/OpenJDK | <https://docs.oracle.com/en/java/javase/26/javadoc/using-markdown-documentation-comments.html> |
| `dotnet:csharp-xmldoc` | C# XML docs | Microsoft | <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/xmldoc/> |
| `dotnet:csharp-tags` | Recommended XML tags | Microsoft | <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/xmldoc/recommended-tags> |
| `dotnet:csharp-spec-docs` | Documentation comment spec | Microsoft | <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/documentation-comments> |
| `dotnet:vb-xmldoc` | VB XML docs | Microsoft | <https://learn.microsoft.com/en-us/dotnet/visual-basic/programming-guide/program-structure/documenting-your-code-with-xml> |
| `dotnet:vb-tags` | VB XML tags | Microsoft | <https://learn.microsoft.com/en-us/dotnet/visual-basic/language-reference/xmldoc/> |
| `dotnet:roslyn` | Semantic model/compiler platform | Microsoft | <https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/> |
| `javascript:ecma-comments` | Lexical comments | Ecma TC39 | <https://tc39.es/ecma262/multipage/ecmascript-language-lexical-grammar.html#sec-comments> |
| `javascript:jsdoc` | De-facto documentation system | JSDoc | <https://jsdoc.app/about-getting-started> |
| `javascript:jsdoc-tags` | Tag registry | JSDoc | <https://jsdoc.app/#block-tags> |
| `sql:postgres-comments` | PostgreSQL lexical comments | PostgreSQL | <https://www.postgresql.org/docs/current/sql-syntax-lexical.html#SQL-SYNTAX-COMMENTS> |
| `sql:postgres-comment-ddl` | PostgreSQL object docs | PostgreSQL | <https://www.postgresql.org/docs/current/sql-comment.html> |
| `sql:sqlserver-extended-property` | SQL Server object metadata | Microsoft | <https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-addextendedproperty-transact-sql> |
| `sql:oracle-comment` | Oracle object docs | Oracle | <https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/COMMENT.html> |
| `sql:mysql-comments` | MySQL comments | Oracle/MySQL | <https://dev.mysql.com/doc/refman/8.4/en/comments.html> |
| `r:r-exts-rd` | Rd specification/guidance | R Core | <https://cran.r-project.org/doc/manuals/r-release/R-exts.html#Writing-R-documentation-files> |
| `r:parse-rd` | Rd parser | R Core | <https://stat.ethz.ch/R-manual/R-devel/library/tools/html/parse_Rd.html> |
| `r:roxygen-rd` | Source authoring | roxygen2 | <https://roxygen2.r-lib.org/articles/rd.html> |
| `r:roxygen-tags` | Tag registry | roxygen2 | <https://roxygen2.r-lib.org/reference/tags-index.html> |
| `rust:reference-comments` | Doc comment syntax | Rust project | <https://doc.rust-lang.org/reference/comments.html> |
| `rust:rustdoc-writing` | rustdoc authoring | Rust project | <https://doc.rust-lang.org/rustdoc/how-to-write-documentation.html> |
| `rust:rustdoc-tests` | Doctests | Rust project | <https://doc.rust-lang.org/rustdoc/write-documentation/documentation-tests.html> |
| `rust:rustdoc-links` | Intra-doc links | Rust project | <https://doc.rust-lang.org/rustdoc/write-documentation/linking-to-items-by-name.html> |
| `delphi:xml-docs` | Delphi XML docs | Embarcadero | <https://docwiki.embarcadero.com/RADStudio/Florence/en/XML_Documentation_for_Delphi_Code> |
| `scratch:vm` | Scratch VM | Scratch Foundation | <https://github.com/scratchfoundation/scratch-vm> |
| `scratch:editor` | Current editor/VM monorepo | Scratch Foundation | <https://github.com/scratchfoundation/scratch-editor> |
| `scratch:sb3-serializer` | Comment/project serialization | Scratch Foundation | <https://github.com/scratchfoundation/scratch-editor/blob/develop/packages/scratch-vm/src/serialization/sb3.js> |
| `scratch:comment-model` | Comment fields | Scratch Foundation | <https://github.com/scratchfoundation/scratch-editor/blob/develop/packages/scratch-vm/src/engine/comment.js> |
| `php:reflection-doccomment` | Raw comment runtime API | PHP | <https://www.php.net/manual/en/reflectionclass.getdoccomment.php> |
| `php:phpdocumentor` | De-facto PHPDoc | phpDocumentor | <https://docs.phpdoc.org/guide/references/phpdoc/index.html> |
| `php:fig-status` | PSR status | PHP-FIG | <https://www.php-fig.org/psr/> |
| `go:doc-comments` | Official comment format | Go project | <https://go.dev/doc/comment> |
| `go:doc-package` | Documentation extraction | Go project | <https://pkg.go.dev/go/doc> |
| `go:doc-comment-package` | Markup parser | Go project | <https://pkg.go.dev/go/doc/comment> |
| `go:examples` | Executable examples | Go project | <https://go.dev/blog/examples> |
| `fortran:ford-writing` | De-facto docs | FORD | <https://forddocs.readthedocs.io/en/latest/user_guide/writing_documentation.html> |
| `fortran:stdlib` | Major community usage | Fortran-lang | <https://stdlib.fortran-lang.org/> |
| `fortran:wg5` | Standards body | ISO WG5 | <https://wg5-fortran.org/> |
| `ruby:rdoc` | Official documentation tool | Ruby/RDoc | <https://ruby.github.io/rdoc/> |
| `ruby:rdoc-ruby-parser` | Ruby source parser | Ruby/RDoc | <https://ruby.github.io/rdoc/RDoc/Parser/Ruby.html> |
| `ruby:rdoc-c-parser` | C extension parser | Ruby/RDoc | <https://ruby.github.io/rdoc/RDoc/Parser/C.html> |
| `ruby:rbs` | Interface signatures | Ruby | <https://ruby.github.io/rbs/> |
| `swift:docc` | DocC | Swift project | <https://www.swift.org/documentation/docc/> |
| `swift:symbolkit` | Symbol graph model | Swift project | <https://github.com/swiftlang/swift-docc-symbolkit> |
| `swift:docc-source` | DocC implementation | Swift project | <https://github.com/swiftlang/swift-docc> |
| `perl:perlpod` | POD format | Perl | <https://perldoc.perl.org/perlpod> |
| `perl:perlpodspec` | Detailed POD parsing | Perl | <https://perldoc.perl.org/perlpodspec> |
| `perl:pod-simple` | Parser | Perl | <https://perldoc.perl.org/Pod::Simple> |
| `perl:podchecker` | Validation | Perl | <https://perldoc.perl.org/podchecker> |
| `cobol:ibm` | IBM Enterprise COBOL docs | IBM | <https://www.ibm.com/docs/en/cobol-zos> |
| `cobol:gnucobol` | GnuCOBOL docs/project | GnuCOBOL | <https://gnucobol.sourceforge.io/guides.html> |
| `asm:gas-comments` | GNU assembler comments | GNU Binutils | <https://sourceware.org/binutils/docs/as/Comments.html> |
| `asm:nasm-language` | NASM source syntax | NASM | <https://www.nasm.us/doc/nasm03.html> |
| `asm:masm-comment` | MASM block comments | Microsoft | <https://learn.microsoft.com/en-us/cpp/assembler/masm/comment-masm> |
| `asm:arm` | Arm assembler docs | Arm | <https://developer.arm.com/documentation/dui0801/latest/> |

# Appendix C — Diagnostic code namespace

Recommended stable prefixes:

| Prefix | Owner |
|---|---|
| `UDIR-CORE-*` | schema/storage/identity |
| `UDIR-BIND-*` | comment-to-symbol binding |
| `UDIR-TYPE-*` | actual/documented type issues |
| `UDIR-LINK-*` | cross-reference resolution |
| `UDIR-INHERIT-*` | inheritance/include/merge |
| `UDIR-VARIANT-*` | conditional/build variants |
| `UDIR-SEC-*` | security policy |
| `UDIR-NATIVE-*` | unknown/invalid native syntax |
| `UDIR-SOURCE-*` | spans/maps/artifacts |
| `UDIR-CATALOG-*` | SQL/catalog drift/access |
| `UDIR-EXEC-*` | suppressed/sandboxed execution |
| `UDIR-ADAPTER-*` | parser/tool failures |
| `UDIR-RENDER-*` | sanitizer/unsupported rendering |

# Appendix D — Research limitations

1. The language list is a dated popularity snapshot, not an objective definition of importance.
2. Several languages intentionally have no official semantic documentation standard; this report labels de-facto tools rather than upgrading them to “official.”
3. Vendor and generator behavior changes by version. The source registry records the research date, but adapter implementations must pin actual toolchains.
4. Documentation plug-ins and organization-specific conventions are effectively unbounded. UDIR preserves extensions instead of claiming exhaustive semantic understanding.
5. Dynamic/metaprogrammed APIs cannot always be recovered without execution. The platform favors safe, explicit incompleteness over unsafe or fabricated certainty.
6. Native renderer parity requires golden testing against the selected processor. The common schema alone cannot reproduce every layout or resolution quirk.
7. URLs and documentation sites can move. Store source IDs, capture dates, and archived metadata in the implementation’s source registry.
8. This report focuses on source/API documentation. Repository READMEs, architecture decision records, OpenAPI/AsyncAPI, database modeling tools, notebooks, and generated protocol schemas should be integrated as additional conceptual/schema adapters rather than forced into a language comment profile.

---

# Conclusion

The viable universal standard is a **loss-aware intermediate representation**, not one source-comment language. UDIR keeps native authoring intact, obtains code facts from semantic tooling, records build and dialect context, preserves every native construct, and emits consistent semantic records for rendering, search, and AI ingestion. The most important implementation property is not how many tags it recognizes; it is whether every transformation remains attributable, reproducible, and honest about ambiguity or loss.
