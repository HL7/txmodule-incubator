# Terminology Module Incubator

This repository contains the source for the **Terminology Module Incubator** HL7 FHIR Implementation Guide (IG).

- **Canonical URL:** http://hl7.org/fhir/uv/txmodule-incubator
- **Package ID:** `hl7.fhir.uv.txmodule-incubator`
- **FHIR Version:** R5 (5.0.0)
- **Sponsoring Work Group:** [HL7 Terminology Infrastructure (Vocab)](http://www.hl7.org/Special/committees/Vocab)
- **License:** [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/)

## About

The Terminology Module Incubator is a workspace for developing and testing terminology-related
resources that are planned to be incorporated into a future version of the FHIR core specification,
but where iteration can happen faster than the core (R6) release cycle.

The current scope includes:

- An extension to the **Terminology Service Module** describing how to maintain a closure table.
- `OperationDefinition` resources for terminology operations such as `$closure` (ConceptMap) and
  `$find-matches` (CodeSystem).

Published content is available at: <http://hl7.org/fhir/uv/txmodule-incubator> (once published).

## Builds

| Build | Location |
| --- | --- |
| **Continuous integration (CI) build** — latest commit on the default branch | <https://build.fhir.org/ig/HL7/txmodule-incubator/> |
| **CI build QA report** | <https://build.fhir.org/ig/HL7/txmodule-incubator/qa.html> |
| **CI build history / branches** | <https://build.fhir.org/ig/HL7/txmodule-incubator/branches/> |
| **Published versions (once released)** | <http://hl7.org/fhir/uv/txmodule-incubator/history.html> |

The CI build is produced automatically by the HL7 [auto-builder](https://github.com/FHIR/auto-ig-builder)
whenever a commit is pushed to this repository. Allow a few minutes after pushing for the build to
appear.

## Repository Layout

| Path | Purpose |
| --- | --- |
| [sushi-config.yaml](sushi-config.yaml) | IG metadata (id, version, pages, menu, parameters). |
| [ig.ini](ig.ini) | IG Publisher entry point. |
| [input/pagecontent/](input/pagecontent) | Narrative pages (Markdown / XHTML) rendered into the IG. |
| [input/resources/codesystem/](input/resources/codesystem) | `OperationDefinition` and example resources for CodeSystem. |
| [input/resources/conceptmap/](input/resources/conceptmap) | `OperationDefinition` and example resources for ConceptMap. |
| [input/fsh/](input/fsh) | FSH is used only to auto-generate the `ImplementationGuide` resource — no profile content is defined here. |
| [input/includes/](input/includes) | Page fragments included in the rendered output. |
| [fsh-generated/](fsh-generated) | Output of SUSHI (auto-generated, do not edit by hand). |
| [input-cache/](input-cache) | Cached publisher jar, terminology cache, and schemas. |
| [output/](output) | Rendered IG output (auto-generated). |
| [temp/](temp), [template/](template) | Build scratch space and template assets. |

## Prerequisites

To build the IG locally you need:

- **Java 11+** (required by the HL7 IG Publisher)
- **[SUSHI](https://fshschool.org/docs/sushi/installation/)** (`npm install -g fsh-sushi`) — used to
  generate the `ImplementationGuide` resource from FSH
- **Internet access** on first build to download the IG Publisher jar and to access the public
  terminology server (`tx.fhir.org`)
- **Ruby + Jekyll** are no longer required at runtime; this repo ships a self-contained
  [`jekyll.cs`](jekyll.cs) shim invoked by the publisher

## Building the IG

### 1. Update the IG Publisher (one-time, then occasionally)

```bat
_updatePublisher.bat
```

Downloads the latest `publisher.jar` from
[HL7/fhir-ig-publisher](https://github.com/HL7/fhir-ig-publisher/releases) into `input-cache/`.

The shell equivalents are also provided: [_updatePublisher.sh](_updatePublisher.sh).

### 2. Run a single build

```bat
_genonce.bat
```

This runs SUSHI (implicitly via the publisher) and the IG Publisher once, producing the rendered
site under [output/](output). Open [output/index.html](output/index.html) in a browser to view it.

The shell equivalent is [_genonce.sh](_genonce.sh).


### Offline builds

The build scripts detect lack of connectivity to `tx.fhir.org` and automatically pass `-tx n/a` to
the publisher so the build can proceed without a terminology server.

## Authoring Notes

- **Narrative pages** live in [input/pagecontent/](input/pagecontent). New pages must also be
  registered in the `pages:` and (typically) `menu:` sections of [sushi-config.yaml](sushi-config.yaml).
- **OperationDefinitions** and other conformance resources are authored as XML/JSON under
  [input/resources/](input/resources). The `path-resource` parameter in
  [sushi-config.yaml](sushi-config.yaml) lists the directories the publisher scans.
- **FSH is not used for profiles** in this IG — see [input/fsh/readme.md](input/fsh/readme.md). The
  `fsh-generated/` folder is auto-produced and should not be edited manually.
- **QA results** after a build are written to [output/qa.html](output/qa.html) and
  [temp/qa/](temp/qa). Warnings to be ignored are listed in
  [input/ignoreWarnings.txt](input/ignoreWarnings.txt).

## Contributing

Issues and pull requests are welcome. For substantive changes to terminology operations or service
behavior, please raise the topic with the
[HL7 Terminology Infrastructure (Vocab) Work Group](http://www.hl7.org/Special/committees/Vocab)
or open an issue on the [HL7 FHIR JIRA](https://jira.hl7.org/projects/FHIR) against the
*FHIR Specification Feedback* project, component *txmodule-incubator*.

## License

Content in this repository is published under the
[Creative Commons Zero v1.0 Universal (CC0-1.0)](https://creativecommons.org/publicdomain/zero/1.0/)
license, as declared in [sushi-config.yaml](sushi-config.yaml).
