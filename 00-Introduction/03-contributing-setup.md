# Setting Up for React Contribution

## Cloning and Building

```bash
# Clone the repo
git clone https://github.com/facebook/react.git
cd react

# Install dependencies
yarn install

# Build all packages
yarn build

# Build a specific package
yarn build react react-dom

# Run the test suite
yarn test

# Run tests for a specific file
yarn test ReactFiber

# Run tests in watch mode
yarn test --watch
```

## Key Build Artifacts

After `yarn build`, compiled files appear in `build/`:

```
build/
├── node_modules/
│   ├── react/
│   │   ├── cjs/
│   │   │   ├── react.development.js    # Development build (warnings)
│   │   │   └── react.production.min.js # Production build (minified)
│   │   └── umd/
│   ├── react-dom/
│   └── scheduler/
└── facebook-www/           # Facebook internal builds
```

## Feature Flags

React uses feature flags to gate experimental features. They live in:

```
packages/shared/ReactFeatureFlags.js          # Default flags
packages/shared/forks/ReactFeatureFlags.www.js # Facebook flags
packages/shared/forks/ReactFeatureFlags.test-renderer.js
```

Example flags:

```javascript
// packages/shared/ReactFeatureFlags.js

export const enableSuspenseCallback = false;
export const enableTransitionTracing = false;
export const enableLegacyHidden = false;
export const enableCPUSuspense = false;
```

To test with a feature flag enabled, modify the flag in the test renderer fork
or create a test that explicitly enables it.

## Running Specific Test Suites

```bash
# Unit tests (Jest)
yarn test packages/react-reconciler

# Integration tests
yarn test packages/react-dom/src/__tests__

# Test a specific concept
yarn test ReactFiberWorkLoop
yarn test ReactHooks
yarn test ReactSuspense
yarn test ReactFiberLane

# Type checking
yarn flow

# Lint
yarn lint
```

## Debugging React Internals

### Method 1: Build and Link

```bash
# Build React
cd react
yarn build

# In your test app
# Point to local build
npm link ../react/build/node_modules/react
npm link ../react/build/node_modules/react-dom
```

### Method 2: Using __DEV__ Breakpoints

In development builds, React includes extensive `__DEV__` only code:

```javascript
// You'll see this pattern everywhere in the source
if (__DEV__) {
  // Validation, warnings, component stack traces
  // Strip-dead-code removed in production builds
  console.error(
    'Invalid hook call. Hooks can only be called inside...'
  );
}
```

### Method 3: React DevTools Profiler

1. Install React DevTools browser extension
2. Open DevTools → Profiler tab
3. Record an interaction
4. Inspect commit timings, fiber tree, and re-render reasons

## Code Conventions in the React Codebase

1. **Flow types** — React uses Flow (not TypeScript) for type checking
2. **No classes in new code** — Modern React internals use functions + closures
3. **Feature flags** — All new features behind flags
4. **`__DEV__` guards** — Development-only warnings and validation
5. **Monomorphic code** — Avoids polymorphism for V8 optimization
6. **Bitwise operations** — Lanes, effect tags, fiber flags all use bitmasks
7. **No external dependencies** — Zero runtime dependencies in core packages
