# Language-server evidence for the code reader

Researched 2026-10-12 for **Establish language-server coverage and operating constraints**. This is a decision input, not a server selection or implementation specification. Sources are upstream documentation and source inspected on this date; links to moving branches describe that inspected state. No server was installed or exercised. Feature declarations establish an integration opportunity, not navigation correctness, completeness, or speed.

## Recommended starting point

Investigate rust-analyzer, Ruby LSP, Eclipse JDT LS, JetBrains Kotlin LSP, Pyright, typescript-language-server, and the VS Code HTML/CSS servers. Together these cover the requested Rust, Ruby, Java, Kotlin, Python, TypeScript, TSX, JavaScript, HTML, and CSS file families. Keep Solargraph as a Ruby comparison candidate. Do not promise equal “find usages” semantics across languages: the documented gaps below require a product decision.

Make plain reading, file discovery, text search, and lexical highlighting independent of servers. Start semantic services by project/language demand. A process supervisor should expose starting, importing/indexing, ready, failed, and unavailable states, but server readiness alone cannot prove complete indexing. This is a design recommendation derived from the different project-loading and capability models below, not measured performance evidence.

## Evidence for the five navigation features

**D** = definition, **R** = references, **W** = workspace symbols, **S** = document symbols, **H** = hover. “Declared” means upstream initialization/source or documentation exposes the feature; every entry remains empirically unverified in our client.

| Language / candidate | D / R / W / S / H evidence | Material qualification |
| --- | --- | --- |
| Rust / rust-analyzer | All declared in [server capabilities](https://github.com/rust-lang/rust-analyzer/blob/master/crates/rust-analyzer/src/lsp/capabilities.rs). | Build scripts, generated code, macros, toolchain and dependencies affect the model. |
| Ruby / Ruby LSP | All documented in [feature guide](https://shopify.github.io/ruby-lsp/). | References are documented for constants only; method and variable usages must not be represented as complete. Definition/hover cover constants and methods. |
| Ruby alternative / Solargraph | All documented in [integration guide](https://solargraph.org/guides/language-server). | Different inference and documentation model; a feature list does not establish better Ruby usage accuracy. |
| Java / Eclipse JDT LS | All declared by [initialization handler](https://github.com/eclipse-jdtls/eclipse.jdt.ls/blob/main/org.eclipse.jdt.ls.core/src/org/eclipse/jdt/ls/core/internal/handlers/InitHandler.java). | Some are dynamically registered according to client capabilities. Java evidence alone establishes no Kotlin coverage. |
| Kotlin / JetBrains Kotlin LSP | D/R/W evidenced by [release notes](https://github.com/Kotlin/kotlin-lsp/blob/main/RELEASES.md), H by [README](https://github.com/Kotlin/kotlin-lsp/blob/main/README.md), S by [Kotlin provider](https://github.com/Kotlin/kotlin-lsp/blob/main/features-impl/kotlin/src/com/jetbrains/ls/api/features/impl/kotlin/symbols/LSKotlinDocumentSymbolProvider.kt). | Alpha product, partially closed-source; shipped-version and generic-client validation required. |
| Python / Pyright | All declared in [language server base](https://github.com/microsoft/pyright/blob/main/packages/pyright-internal/src/languageServerBase.ts). | Interpreter/import paths and stubs determine what can resolve. |
| TS / TSX / JS / typescript-language-server | All declared in [LSP server](https://github.com/typescript-language-server/typescript-language-server/blob/master/src/lsp-server.ts). | TSX belongs to this service, not the HTML service. Project configuration and TypeScript version matter. |
| HTML / VS Code HTML server | D/R/S/H declared; no W provider in [server initialization](https://github.com/microsoft/vscode/blob/main/extensions/html-language-features/server/src/htmlServer.ts). | Definition/reference handlers dispatch to the language mode at the cursor; capability presence does not imply HTML-wide symbol indexing. |
| CSS / VS Code CSS server | D/R/S/H declared; no W provider in [server initialization](https://github.com/microsoft/vscode/blob/main/extensions/css-language-features/server/src/cssServer.ts). | Definition/reference queries use the current document and stylesheet. This is not evidence of repository-wide CSS-class usages in HTML/TSX. |

## Per-server operating constraints

### Rust

rust-analyzer discovers Cargo projects and needs the appropriate toolchain and standard-library sources; its [installation guide](https://rust-analyzer.github.io/book/installation.html) describes `rust-src`. The [configuration reference](https://rust-analyzer.github.io/book/configuration) documents Cargo metadata autoreload, enabled-by-default build scripts, check-on-save, proc-macro controls, cache priming and configurable worker counts. A separate target directory is possible but duplicates artifacts. Turning off checks does not by itself disable initial build-script/proc-macro work.

Consequently, a reader can still cause Cargo downloads/build output, lockfile interaction, and project code execution. Upstream explicitly describes executable Cargo configuration and build/proc-macro concerns in its [security discussion](https://rust-analyzer.github.io/book/security.html). Decide what “read-only” promises before defaulting these features. Disabling them trades semantic coverage for less activity; it is not a free optimization.

### Ruby

Ruby LSP must use the project's Ruby and gem environment; generic clients need dynamic registration and file watching. The [editor integration guide](https://shopify.github.io/ruby-lsp/editors.html) describes version-manager activation. Its [gemspec](https://github.com/Shopify/ruby-lsp/blob/main/ruby-lsp.gemspec) currently requires Ruby >=3.0. Multiple Gemfiles/Ruby versions can require separate processes; upstream [VS Code integration](https://github.com/Shopify/ruby-lsp/blob/main/vscode/README.md) explains this workspace boundary.

The default [composed bundle](https://shopify.github.io/ruby-lsp/composed-bundle.html) creates `.ruby-lsp` under the project, includes project dependencies, runs Bundler installation and attempts server updates. That conflicts with a promise of zero checkout writes unless launch/setup is changed and verified. The [Rails add-on](https://shopify.github.io/ruby-lsp/rails-add-on) can communicate with the Rails application, so it is a separate integration/execution choice.

Solargraph is worth a targeted comparison because its documentation includes references generally. Its [README](https://github.com/castwide/solargraph) describes YARD/RBS-based gem maps and documentation caching; its [gemspec](https://github.com/castwide/solargraph/blob/master/solargraph.gemspec) currently requires Ruby >=3.1. Neither candidate's documentation proves reliable method usages in metaprogrammed applications. Do not silently substitute text matches for semantic usages.

### Java and Kotlin

Eclipse JDT LS supports Maven, Gradle and standalone Java files, requires Java 21 or newer to run, and has Linux/macOS/Windows launch configurations. Its workspace-specific `-data` directory must be unique. The README's `-Xmx1G` example is a heap option, not observed RSS or a promise of acceptable memory. See [JDT LS README](https://github.com/eclipse-jdtls/eclipse.jdt.ls).

JDT [preferences](https://github.com/eclipse-jdtls/eclipse.jdt.ls/blob/main/org.eclipse.jdt.ls.core/src/org/eclipse/jdt/ls/core/internal/preferences/Preferences.java) include build/import controls, Gradle user-home/JVM selection, offline settings, annotation processing and source downloads. Import may involve dependency/build tooling; moving `-data` alone does not establish zero checkout writes. Test generated-source and dependency-source navigation separately from ordinary Java source.

JetBrains' [Kotlin README](https://github.com/Kotlin/kotlin-lsp/blob/main/README.md) explicitly says alpha, partly closed-source, JVM Gradle/Maven support, experimental Android import, and future KMP support. It is based on IntelliJ technology but that is not a promise of full IntelliJ behavior. [Release notes](https://github.com/Kotlin/kotlin-lsp/blob/main/RELEASES.md) document cross-language reference fixes for mixed Java/Kotlin, Java-file synchronization, workspace symbols in libraries, bundled runtimes, platform-specific distributions and index/import progress. They also document source downloads increasing first-sync time and cache-growth fixes.

**Inference:** mixed Java/Kotlin should be a first-class acceptance fixture. Test Kotlin-to-Java and Java-to-Kotlin definitions/references, external Java changes, generated sources, and multiple Gradle source sets. Do not assume JDT and Kotlin processes automatically combine their indexes or that running both gives complete bidirectional results. Determine which server owns each request; distinguish the standalone Kotlin distribution from separate IntelliJ-branded extensions. The precise standalone launch/configuration and binary redistribution terms remain unresolved.

### Python

Pyright has a standalone Node-based `pyright-langserver` executable, recorded in its [package manifest](https://github.com/microsoft/pyright/blob/main/packages/pyright/package.json). Configure the actual interpreter/environment and project import roots. [Import resolution](https://github.com/microsoft/pyright/blob/main/docs/import-resolution.md) searches local code, extra paths, stubs and installed packages; it may invoke Python to obtain search paths. Navigation may land in type stubs. Editable installs using import hooks have documented limitations.

[Project configuration](https://github.com/microsoft/pyright/blob/main/docs/configuration.md) supports `pyrightconfig.json`, `pyproject.toml`, exclusions and distinct execution environments. Recommend consuming existing settings and storing reader overrides outside the checkout. Neither opening a Python file nor selecting a server installs its missing application dependencies. No full filesystem-write audit was performed.

### TypeScript, TSX, JavaScript, HTML and CSS

typescript-language-server is a community-maintained LSP bridge to TypeScript's `tsserver`; the [README](https://github.com/typescript-language-server/typescript-language-server) documents npm installation and stdio. Do not speak tsserver's custom protocol as if it were LSP. Its [configuration](https://github.com/typescript-language-server/typescript-language-server/blob/master/docs/configuration.md) controls TypeScript location, memory, logging and automatic type acquisition. Read the selected release's [Node requirement](https://github.com/typescript-language-server/typescript-language-server/blob/master/package.json), rather than hard-coding an old minimum. Existing tsconfig/jsconfig, package resolution and installed dependency types affect results; automatic acquisition and logs are potential writes/network activity.

[vscode-langservers-extracted](https://github.com/hrsh7th/vscode-langservers-extracted) packages separate HTML and CSS server executables derived from VS Code. Pin and verify the extracted version against upstream source; upstream `main` is not proof that a published package has the same features. Embedded CSS/JavaScript support exists inside the HTML server, but arbitrary framework templates and repository-wide HTML/CSS/TSX relationships are additional integration work. The [embedded CSS mode](https://github.com/microsoft/vscode/blob/main/extensions/html-language-features/server/src/modes/cssMode.ts) works on extracted document content. TSX must retain the correct TypeScript React language identity and project context. CSS Modules, template languages, and component frameworks need separate capability decisions and fixtures.

## Distribution and platform evidence

Repository license identifiers: rust-analyzer [MIT/Apache-2.0](https://github.com/rust-lang/rust-analyzer); Ruby LSP and Solargraph MIT (gemspecs above); JDT LS EPL-2.0 (README above); Pyright MIT (manifest above); typescript-language-server Apache-2.0 (manifest above); extracted HTML/CSS package MIT ([manifest](https://github.com/hrsh7th/vscode-langservers-extracted/blob/master/package.json)). These are inventory facts, not an audit of transitive binary notices or redistribution obligations. Kotlin's repository has [Apache-2.0 text](https://github.com/Kotlin/kotlin-lsp/blob/main/LICENSE.txt), while its README identifies proprietary components: inspect the exact downloaded distribution before deciding to bundle it.

JDT and Kotlin explicitly document Linux/macOS/Windows artifacts/configuration. Ruby candidates require working Ruby/gem environments; Node candidates require compatible Node and package installations. This research does not certify every OS/architecture combination. Native macOS and Windows remain untested. Prefer a small, versioned supported-server inventory with explicit executable detection; whether installation is manual, reader-managed, or bundled is a product decision still open.

## Client contract and acceptance evidence still needed

LSP initialization negotiates optional capabilities, including position encoding; implement the selected encoding rather than equating Rust byte offsets with protocol character positions. See [initialize specification](https://github.com/microsoft/language-server-protocol/blob/gh-pages/_specifications/lsp/3.17/general/initialize.md). Do not expose an unavailable semantic command as an empty successful search.

External edits still need synchronization in a read-only reader: use versioned `didOpen`/`didChange`/`didClose` for owned open documents and negotiated file watching for other changes. A disk watcher notification does not replace synchronization of an open document. Send updated content before navigation requests and reject stale responses against document/project identity. See [document changes](https://github.com/microsoft/language-server-protocol/blob/gh-pages/_specifications/lsp/3.17/textDocument/didChange.md) and [watched files](https://github.com/microsoft/language-server-protocol/blob/gh-pages/_specifications/lsp/3.17/workspace/didChangeWatchedFiles.md).

Recommended acceptance work, not performed here:

- Pin versions and record negotiated capabilities for every supported server. Test definitions, references, symbols and hover against known locations; count false/missing results, not just successful responses.
- Include Unicode positions, non-file/dependency source targets, file deletion/rename, external edits during navigation, cancellation, crashes/restart and incomplete dependencies.
- Measure first file paint separately from server startup, initial import/indexing and warm navigation. Record reader and complete server/build-process tree memory separately; retain peak, settled and sustained samples.
- Snapshot checkout bytes before/after cold startup, import, refresh and shutdown. Separately inventory external cache/dependency writes and executed build tools. Decide whether “read-only” means no user-edit commands, no source modification, or a genuinely non-writable checkout.

No comparable timings, memory measurements, runtime navigation guarantees, server installs or build-system probes were produced. The largest design risks are Ruby usage completeness, Kotlin/mixed-JVM maturity and distribution, embedded web-language expectations, and server side effects under the read-only promise.
