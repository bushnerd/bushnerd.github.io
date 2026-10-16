# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

The project requires Node.js >= 22 and npm >= 10.9.2. The repository pins Node 22.16.0 in `.node-version`; CI currently tests with Node 24.

```powershell
# Install dependencies
npm install

# Install plugins referenced by the lockfile/config (needed after a fresh clone)
npx quartz plugin install

# Type-check and verify Prettier formatting
npm run check

# Format the repository
npm run format

# Run all tests (Node's test runner through tsx)
npm test

# Run one test file
npx tsx --test quartz/util/path.test.ts

# Build the default content directory into public/
npx quartz build

# Run the local preview server with watch/hot reload
npx quartz build --serve

# Build this repository's documentation site (the package script uses docs/ as input)
npm run docs

# Build the docs site with verbose output and bundle metadata, as CI does
npx quartz build --bundleInfo -d docs -v
```

Tests are colocated with the implementation as `*.test.ts`/`*.test.js`; there is no separate test configuration. `npm run check` runs both `tsc --noEmit` and a repository-wide Prettier check. CI runs `npm install`, `npx quartz plugin install`, `npm run check`, `npm test`, and a docs build.

Useful build flags are `-d/--directory` for the input directory, `-o/--output` for the output directory, `--serve`, `--watch`, `--port` (default 8080), `--wsPort` (default 3001), `-c/--concurrency`, and `--bundleInfo`. `--serve` implies `--watch`.

## Repository architecture

Quartz is a TypeScript/Preact static-site generator. In this checkout, the engine is under `quartz/`; `docs/` is the content directory used by the documentation/deployment workflows, while `content/` is the default starter content directory. `public/`, `.quartz-cache/`, `.quartz/`, and other build/plugin artifacts are generated and ignored.

### CLI and build orchestration

- `package.json` exposes `quartz/bootstrap-cli.mjs` as both the `quartz` script and binary. `bootstrap-cli.mjs` validates the Node version, defines the yargs commands, and dispatches build, sync, create, and plugin operations.
- `quartz/cli/handlers.js` owns the build/dev-server lifecycle and uses esbuild to bundle the TypeScript engine into `.quartz-cache/transpiled-build.mjs`. It also bundles SCSS and `.inline.ts`/`.inline.js` browser resources. `--serve` starts an HTTP server plus a WebSocket hot-reload server.
- `quartz/build.ts` is the main build coordinator. It cleans the output, globs the selected input directory while applying `configuration.ignorePatterns`, parses Markdown, filters processed content, and invokes emitters. Watch mode maintains a content map and uses incremental emitter support when available.

### Content processing pipeline

The build is intentionally split into stages:

1. `quartz/processors/parse.ts` creates unified processors. It applies transformer text hooks, parses Markdown to mdast, applies remark plugins, converts mdast to hast, and applies rehype plugins. For larger inputs it transpiles `quartz/worker.ts` and uses worker threads in chunks.
2. `quartz/processors/filter.ts` runs filter plugins in configured order; each plugin decides whether a processed file is published.
3. `quartz/processors/emit.ts` gathers static resources and runs emitters. `ComponentResources` runs first so hashed CSS/JS names are available, `PageTypeDispatcher` runs next to generate virtual pages, and the remaining emitters run over both real and virtual pages.
4. `quartz/components/renderPage.tsx` converts processed HAST plus Preact components into static HTML. `quartz/components/frames/` resolves the page frame (default, full-width, minimal, or a plugin-provided frame), while components contribute layout slots and browser CSS/scripts.

The central build state is `BuildCtx` in `quartz/util/ctx.ts`. It carries CLI arguments, configuration, all slugs/files, incremental-build state, virtual pages, and emitted resource maps. `quartz/plugins/vfile.ts` defines the processed-content pair passed through the pipeline.

### Configuration and plugin loading

- `quartz.ts` is the TypeScript entrypoint: it loads the site configuration and layout through `quartz/plugins/loader/config-loader.ts` and exports both to the build.
- `quartz.config.yaml` is the active configuration for this checkout; `quartz.config.default.yaml` is the upstream default/template. General site settings, plugin entries, execution `order`, and layout declarations live in YAML. Use `quartz.ts` only when a configuration requires TypeScript callbacks or overrides.
- `config-loader.ts` reads YAML (with legacy JSON fallback), installs/resolves enabled community plugins, validates declared dependencies and ordering, loads component/frame manifests, sorts plugins by order, instantiates the four processing categories, and appends built-in emitters/page types.
- Plugin capabilities are represented in `quartz/plugins/types.ts`: transformers map content, filters publish or discard it, emitters write files/resources, and page types match/render file or virtual-page categories. A plugin can provide more than one capability and can also register components or frames.
- Community plugins are installed under `.quartz/plugins/` and are loaded through their manifests/entry points. The directory and generated index/cache files are ignored; use the plugin CLI rather than hand-editing generated plugin files. Relevant commands include `npx quartz plugin add`, `install`, `remove`, `enable`, `disable`, `config`, `list`, and `prune`.
- Components are registered in the singleton `componentRegistry` (`quartz/components/registry.ts`) and assembled into layout positions by `loadQuartzLayout`. Page-type layout overrides are resolved by `quartz/plugins/pageTypes/dispatcher.ts`; `layout.byPageType.<name>.template` can override a page type's frame.

### Site-specific configuration

The active config builds the `docs/` site in deployment (`.github/workflows/deploy-v5.yaml`) and enables features such as Obsidian-flavored Markdown, search, graph, backlinks, content indexes/RSS, page types, encrypted pages, and Excalidraw support. When changing site behavior, update `quartz.config.yaml`; when changing the generator defaults/templates, also consider `quartz.config.default.yaml` and the corresponding CLI template files in `quartz/cli/templates/`.

### Tests and type boundaries

TypeScript is strict with `noUnusedLocals` and `noUnusedParameters` (`tsconfig.json`), uses Preact's automatic JSX runtime, and resolves JSON modules. Core unit tests cover utilities, plugin/config loading, component/registry behavior, page frames, page rendering, and inline client scripts. Community plugins are external dependencies, so changes to core plugin contracts should account for the public types in `quartz/plugins/types.ts` and the `@quartz-community/types` package.
