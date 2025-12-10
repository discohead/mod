# GitHub Copilot Instructions for MOD

## Repository Overview

**MOD** is a TypeScript/React library for building modular audio applications using the Web Audio API. It provides composable React components that connect like hardware modular synthesizer modules using refs as "patch cables".

- **Type**: Monorepo (npm workspaces)
- **Languages**: TypeScript, React, CSS
- **Size**: ~47 audio/UI components across 3 packages
- **Runtime**: Node.js 20.x, npm 10.x
- **Build Tools**: tsup (library), Vite (demo), VitePress (docs)
- **Testing**: Jest with React Testing Library

## Package Structure

```
mod/
├── packages/core/          # Main library (@mode-7/mod) - published to npm
│   ├── src/
│   │   ├── components/     # Audio modules (sources, processors, cv, mixers, output, visualizations)
│   │   ├── context/        # AudioProvider React context
│   │   ├── hooks/          # useModStream, useControlledState, useModStreamToMediaStream
│   │   ├── modui/          # 11 UI components (Slider, Knob, XYPad, etc.)
│   │   ├── types/          # ModStream, ModStreamRef types
│   │   └── __tests__/      # Jest tests (914 tests total)
│   └── dist/               # Build output (generated, gitignored)
├── packages/demo/          # Interactive playground app
│   └── dist/ → ../../docs/public/playground/  # Builds into docs
├── docs/                   # VitePress documentation site
│   ├── guide/              # User guides
│   ├── api/                # API reference
│   └── .vitepress/dist/    # Build output (generated, gitignored)
└── .github/workflows/deploy.yml  # CI/CD pipeline
```

## Build & Development Commands

### Installation

**ALWAYS run `npm install` from the repository root**, not from individual packages. This installs dependencies for all workspaces:

```bash
npm install    # or `npm ci` in CI environments
```

### Building

**Build order matters!** The demo depends on the core library being built first.

```bash
# Build core library only (required before demo)
npm run build --workspace=packages/core

# Build demo (requires core built first)
npm run build --workspace=packages/demo

# Build all packages in correct order
npm run build:all

# Build documentation
npm run docs:build
```

**Note**: CLAUDE.md files are excluded from the VitePress build to prevent dead link warnings. These files contain guidance for AI assistants and are not part of the user-facing documentation.

### Development

```bash
# Start demo playground with hot reload (localhost:5173)
npm run dev

# Start documentation dev server (localhost:5173)
npm run docs:dev

# Watch mode for core library (auto-rebuild on changes)
npm run dev --workspace=packages/core
```

### Testing

**Tests must be run from `packages/core` directory**, not from root (root has no test script):

```bash
cd packages/core
npm run test              # Silent mode (CI-friendly)
npm run test:verbose      # With output
npm run test:watch        # Watch mode for development
npm run test:coverage     # Coverage report

# Run specific test file
npm test -- Filter.test
```

**Test Duration**: Full test suite takes ~15 seconds (914 tests across 34 suites).

## CI/CD Workflow

GitHub Actions workflow (`.github/workflows/deploy.yml`) on push to `main`:

1. Checkout code
2. Setup Node.js 20 with npm cache
3. `npm ci`
4. `npm run build --workspace=packages/core`
5. `npm run build --workspace=packages/demo`
6. `npm run docs:build`
7. Prepare deployment directory (copy index.html, docs dist)
8. Deploy to GitHub Pages

**To replicate CI locally**:
```bash
npm ci
npm run build --workspace=packages/core
npm run build --workspace=packages/demo
npm run docs:build
```

## Key Architectural Patterns

### Component Implementation Pattern

All audio components follow a standard structure (see `packages/core/src/components/processors/Filter.tsx` as reference):

1. **Three Usage Patterns**:
   - Render props: `<Filter>{({ frequency, setFrequency }) => ...}</Filter>`
   - Controlled props: `<Filter frequency={x} onFrequencyChange={setX} />`
   - Imperative refs: `filterRef.current?.getState()`

2. **Props Structure**:
   - `input`/`output`: ModStreamRef connections (signal routing)
   - `cv`: Optional ModStreamRef for modulation
   - `cvAmount`: Scale factor for CV modulation
   - `enabled`: Boolean for bypass routing
   - Controlled props with `onChange` callbacks
   - `children`: Render prop function

3. **State Management**: Uses `useControlledState` hook for controlled/uncontrolled mode

4. **Web Audio Integration**: Audio nodes created in `useEffect`, cleaned up on unmount

### Signal Flow Architecture

```
Source → [Processors] → Monitor → Speakers
            ↑
         CV (LFO, ADSR, etc.)
```

- **ModStreamRef**: Created via `useModStream()` hook, connects modules
- **Monitor required**: Audio only plays when signal reaches Monitor component
- **CV Modulation**: `finalValue = baseValue + (cvSignal × cvAmount)`

## Common Development Tasks

### Adding a New Audio Component

1. Create `packages/core/src/components/{category}/{Name}.tsx` (copy Filter.tsx pattern)
2. Export from `packages/core/src/index.ts`
3. Add test in `packages/core/src/__tests__/{Name}.test.tsx`
4. Add to playground: `packages/demo/src/moduleDefinitions.ts` and `ModuleRenderer.tsx`
5. Create documentation: `docs/api/{category}/{name}.md`
6. Run tests: `cd packages/core && npm test`
7. Build and verify: `npm run build --workspace=packages/core`

### Making Changes to Existing Components

1. Edit source in `packages/core/src/components/`
2. Run related tests: `cd packages/core && npm test -- --testPathPattern="ComponentName"`
3. Rebuild: `npm run build --workspace=packages/core`
4. Test in playground: `npm run dev`
5. Update docs if API changed

### Modifying UI Components (ModUI)

ModUI components are in `packages/core/src/modui/`:
- 11 pre-built audio controls (Slider, Knob, XYPad, Button, etc.)
- Include both logic and styles
- No external dependencies (headless architecture)
- Built alongside core library

## Important Files & Locations

### Configuration Files
- `package.json` (root): Workspace config, main scripts
- `packages/core/package.json`: Library metadata, build/test scripts, tsup config
- `packages/core/tsconfig.json`: TypeScript config for library
- `packages/demo/vite.config.ts`: Demo build config (outputs to docs/public/playground)
- `jest.config.js` (root & packages/core): Jest configuration
- `docs/.vitepress/config.mjs`: Documentation site config, sidebar structure

### Key Source Files
- `packages/core/src/index.ts`: All public exports (145 lines)
- `packages/core/src/types/ModStream.ts`: Core type definitions
- `packages/core/src/hooks/useModStream.ts`: Main hook for creating connections
- `packages/core/src/context/AudioContext.tsx`: AudioProvider wrapper
- `packages/core/src/__tests__/setup.ts`: Jest Web Audio API mocks

### Documentation
- `CLAUDE.md` (root): Top-level guidance for AI assistants
- `packages/core/CLAUDE.md`: Component patterns, testing details
- `packages/demo/CLAUDE.md`: Playground architecture
- `docs/CLAUDE.md`: VitePress documentation structure
- `CONTRIBUTING.md`: Contribution guidelines
- `GETTING_STARTED.md`: Quick start guide

## Common Pitfalls & Workarounds

### Build Issues

**Problem**: "Cannot find module '@mode-7/mod'" in demo
- **Cause**: Core library not built
- **Fix**: `npm run build --workspace=packages/core`

**Problem**: Vite warns "outDir is not inside project root"
- **Expected**: Demo builds to `../../docs/public/playground/` for deployment
- **Fix**: Add `--emptyOutDir` flag if needed: `npm run playground:build`



### Testing Issues

**Problem**: "Missing script: test" when running `npm test` from root
- **Cause**: Tests only exist in packages/core
- **Fix**: `cd packages/core && npm test`

**Problem**: Tests timeout or fail sporadically
- **Cause**: Web Audio API async operations
- **Fix**: Tests have 10s timeout configured, should be sufficient

**Problem**: Coverage report doesn't include new file
- **Cause**: May need to clear Jest cache
- **Fix**: `cd packages/core && npm test -- --clearCache`

### Development Issues

**Problem**: Changes to core library not reflected in demo
- **Cause**: Stale build artifacts
- **Fix**: `npm run build --workspace=packages/core` then restart dev server

**Problem**: Audio not playing in browser
- **Causes**: 
  1. Missing Monitor component in signal chain
  2. Browser autoplay policy (requires user interaction)
  3. AudioContext suspended
- **Debug**: Check browser console for Web Audio errors

## Environment & Dependencies

- **Node.js**: 20.x (specified in GitHub Actions)
- **npm**: 10.x
- **TypeScript**: 5.3.3
- **React**: 18.2.0 (peer dependency)
- **Build Time**: ~3 seconds for core, ~2 seconds for demo, ~7 seconds for docs

### Security Notes
- Repository has 4 moderate severity vulnerabilities (npm audit warns)
- These are in devDependencies and don't affect runtime
- Run `npm audit fix` carefully as it may cause breaking changes

## Trust These Instructions

These instructions have been validated by running all build, test, and deployment commands. When working in this repository:

1. **Always install from root**: `npm install` (workspace-aware)
2. **Build core first**: Before demo or making changes
3. **Test from packages/core**: `cd packages/core && npm test`
4. **Follow component patterns**: Use Filter.tsx as reference
5. **Check CI workflow**: `.github/workflows/deploy.yml` for deployment steps

Only search for additional information if these instructions are incomplete or found to be incorrect. Most common tasks and issues are documented above.
