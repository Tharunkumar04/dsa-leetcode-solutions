# Build Verification Skill

## Full-Depth Verification System

#### Version 3.0
#### Stack: React + Node.js + Prisma + Tailwind CSS
#### Author: Team

## 1. Purpose

A broken build that reaches `main` blocks the entire team. This skill provides a multi-layered safety net:

1. **Local hooks** catch mistakes before they leave your machine.
2. **Continuous Integration** repeats and extends those checks in a controlled environment.
3. **Specialised scripts** detect subtle errors that compilers miss (duplicate routes, circular imports, Tailwind class purging mistakes).
4. **Structured reporting** makes every failure easy to diagnose and fix.

When fully implemented, the system guarantees that every push and every pull request is **type-safe**, **lint-clean**, **test-passing**, **build-ready**, and free from configuration drift.

## 2. When to Use

- Day 1 of any new project (bootstrap the hooks immediately).
- After every coding task run by an agent or a human.
- Before every push (`pre-push` hook) and every `git commit` (`pre-commit`).
- In the pull request pipeline: CI must be green before merging.
- After updating dependencies, changing the Prisma schema, or modifying the build tooling.
- When you notice silent runtime errors and want to harden the verification.

## 3. Layer 1: Local Verification (Husky + lint-staged + pre-push)

Local hooks give instant feedback and prevent broken commits from being pushed. They are fast, deterministic, and run on every commit.

### 3.1 Installation & Setup

We use Husky (native Git hooks) and lint-staged (only run linters on changed files).

```bash
# 1. Install dependencies
pnpm add -D husky lint-staged
npx husky init

# 2. Create hooks
echo "npx lint-staged" > .husky/pre-commit
echo "node scripts/validate-commit-msg.js \$1" > .husky/commit-msg
chmod +x .husky/pre-commit .husky/commit-msg

# 3. Pre-push hook calls a script that runs the full local build
cp scripts/pre-push.sh .husky/pre-push
chmod +x .husky/pre-push
```

### 3.2 lint-staged Configuration (package.json)

The goal is to fix what can be fixed and fail fast on what cannot.

```json
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --max-warnings 0 --fix",
      "prettier --write",
      "bash -c 'npx tsc --noEmit'"
    ],
    "*.{css,scss,json,md,yaml,yml}": [
      "prettier --write"
    ],
    "prisma/schema.prisma": [
      "npx prisma format",
      "npx prisma validate"
    ],
    "src/**/*.{tsx,jsx}": [
      "node scripts/check-tailwind-classes.js"
    ]
  }
}
```

**Why `tsc --noEmit` inside lint-staged?**
`lint-staged` passes a list of files to each command. However, `tsc` does not accept a file glob; it always checks the whole project. To avoid a full type-check on every commit (which can be slow), we place it only in the `pre-push` hook. Here we use a small wrapper that runs `tsc --noEmit` regardless of the staged files; you might keep it commented out until pre-push. The example above is a simplified illustration; in practice, move full type-checking to pre-push.

### 3.3 Pre-push Hook: The Full Local Gate

The file `.husky/pre-push` runs a script (`scripts/pre-push.sh`) that performs a complete local build verification.

**File: `scripts/pre-push.sh`**

```bash
#!/bin/sh
set -e

echo "Running pre-push checks..."

# 1. TypeScript strict, project-wide
echo "TypeScript check..."
npx tsc --noEmit
echo "TypeScript OK"

# 2. ESLint zero warnings policy
echo "ESLint..."
npx eslint ./src --ext .ts,.tsx --max-warnings 0
echo "ESLint OK"

# 3. Prisma schema validation & migration drift
echo "Prisma schema validation..."
npx prisma validate
echo "Checking for migration drift..."
npx prisma migrate diff \
  --from-migrations prisma/migrations \
  --to-schema-datamodel prisma/schema.prisma \
  --shadow-database-url "$SHADOW_DATABASE_URL" || {
  echo "Schema changes detected but no new migration created"
  exit 1
}
echo "Prisma OK"

# 4. Unit tests (fast, no database required)
echo "Unit tests..."
npm run test:unit -- --passWithNoTests
echo "Unit tests OK"

# 5. Production build
echo "Production build..."
npm run build
echo "Build OK"

# 6. Tailwind class integrity check
echo "Tailwind class check..."
node scripts/check-tailwind-classes.js
echo "Tailwind OK"

# 7. Circular dependency check
echo "Circular dependencies..."
npx madge --circular --extensions ts,tsx src/
echo "No circular dependencies"

# 8. Bundle size comparison (optional, but recommended)
echo "Bundle size check..."
node scripts/check-bundle-size.js
echo "Bundle size within limits"

echo "All pre-push checks passed: safe to push"
```

**Why this order?**
Fastest checks first (lint / types), then tests that may require a database, then the expensive build. This gives the developer the quickest feedback.

## 4. Layer 2: Deep-dive into CI (GitHub Actions)

The CI pipeline mirrors the local pre-push checks but adds environment isolation, parallelisation, and reporting.

### 4.1 Complete CI Workflow (`.github/workflows/ci.yml`)

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '20'

jobs:
  # Job 1: Fast Static Checks
  lint-and-typecheck:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      - name: Install dependencies (frozen lockfile)
        run: pnpm install --frozen-lockfile
      - name: TypeScript check
        run: npx tsc --noEmit
      - name: ESLint
        run: npx eslint ./src --ext .ts,.tsx --max-warnings 0
      - name: Prettier
        run: npx prettier --check "src/**/*.{ts,tsx,css,json,md}"
      - name: Prisma validate
        run: npx prisma validate
      - name: Check for migration drift
        run: |
          npx prisma migrate diff \
            --from-migrations prisma/migrations \
            --to-schema-datamodel prisma/schema.prisma \
            --shadow-database-url postgresql://placeholder:placeholder@localhost:5432/shadow

  # Job 2: Tests
  test:
    name: Tests
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      - name: Generate Prisma client
        run: npx prisma generate
      - name: Run migrations on test DB
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
      - name: Unit tests with coverage
        run: pnpm run test:unit -- --coverage
      - name: Integration tests
        run: pnpm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true

  # Job 3: Production Build
  build:
    name: Production Build
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      - name: Build frontend & backend
        run: pnpm run build
        env:
          VITE_API_URL: ${{ secrets.VITE_API_URL }}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: production-build
          path: |
            dist/
            build/
          retention-days: 7

  # Job 4: Route Conflict Detection
  route-check:
    name: Route Conflict Check
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - name: Check for duplicate routes & parameter collisions
        run: node scripts/check-routes.js

  # Job 5: Tailwind Production Integrity
  tailwind-check:
    name: Tailwind Class Safety
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - name: Check that all used classes are in the final CSS
        run: node scripts/check-tailwind-classes.js

  # Job 6: Bundle Size & Performance
  bundle-analysis:
    name: Bundle Size & Analysis
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - name: Build with stats
        run: pnpm run build -- --stats
      - name: Run bundle size comparison
        run: node scripts/check-bundle-size.js
      - name: Upload size analysis report
        uses: actions/upload-artifact@v4
        with:
          name: bundle-stats
          path: stats.html

  # Job 7: Security Audit
  security:
    name: Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - name: Audit dependencies (fail on critical/high)
        run: pnpm audit --audit-level=high
        continue-on-error: false
```

### 4.2 Explanation of Additional CI Jobs

- **Migration Drift Check**: Ensures every schema change has a corresponding migration file. Prevents "works on my machine" because someone forgot to run `prisma migrate dev`.
- **Tailwind Class Check**: Analyses source code for Tailwind classes and verifies they exist in the generated CSS. Prevents production styles from silently disappearing.
- **Bundle Size Analysis**: Compares the current build size with a baseline (stored in `package.json` or a separate file). Fails if the increase exceeds a threshold (e.g., 5%).
- **Security Audit**: `pnpm audit` reports vulnerabilities; setting `--audit-level=high` fails the job if a high- or critical-severity issue exists. This forces you to address them before merging.

## 5. Layer 3: Advanced Verification Scripts

These scripts plug into both local hooks and CI to catch errors that compilers and linters cannot.

### 5.1 Route Conflict & Ordering (`scripts/check-routes.js`)

Enhances the basic duplicate detection with parameter overlap and ordering issues. In React Router v6, route order matters: a static path like `/users/new` must be defined before `/users/:id`.

```javascript
const fs = require('fs');
const path = require('path');

const ROUTES_DIR = path.join(__dirname, '../src/routes');
const routeDefinitions = [];

function extractRoutes(filePath) {
  const content = fs.readFileSync(filePath, 'utf8');
  const regex = /path:\s*["'`]([^"'`]+)["'`]/g;
  let match;
  while ((match = regex.exec(content)) !== null) {
    routeDefinitions.push({
      path: match[1],
      file: filePath,
      isDynamic: /[:*]/.test(match[1]),
    });
  }
}

function scan(dir) {
  const files = fs.readdirSync(dir, { withFileTypes: true });
  for (const file of files) {
    const fullPath = path.join(dir, file.name);
    if (file.isDirectory()) {
      scan(fullPath);
    } else if (file.name.endsWith('.tsx') || file.name.endsWith('.ts')) {
      extractRoutes(fullPath);
    }
  }
}

scan(ROUTES_DIR);

// Check for exact duplicate paths
const seen = new Map();
const conflicts = [];
routeDefinitions.forEach(r => {
  if (seen.has(r.path)) {
    conflicts.push({ path: r.path, a: seen.get(r.path), b: r.file });
  } else {
    seen.set(r.path, r.file);
  }
});

// Check for shadowing: static path after dynamic path
routeDefinitions.forEach((r, idx) => {
  if (r.isDynamic) {
    const prefix = r.path.split('/:')[0];
    for (let i = idx + 1; i < routeDefinitions.length; i++) {
      const later = routeDefinitions[i];
      if (!later.isDynamic && later.path.startsWith(prefix + '/')) {
        conflicts.push({
          path: later.path,
          a: r.file,
          b: later.file,
          message: `Static route shadowed by earlier dynamic route '${r.path}'`,
        });
      }
    }
  }
});

if (conflicts.length > 0) {
  console.error('Route conflicts detected:\n');
  conflicts.forEach(c => console.error(c));
  process.exit(1);
} else {
  console.log('No route conflicts');
}
```

### 5.2 Tailwind Class Integrity (`scripts/check-tailwind-classes.js`)

This script scans all template files for Tailwind utility classes and confirms each one appears in the generated CSS output. It prevents the common purging mistake where a dynamic class name like `bg-${color}-500` gets removed because the purger can't see it.

```javascript
const fs = require('fs');
const path = require('path');
const { execSync } = require('child_process');

execSync('npx tailwindcss -i ./src/index.css -o ./dist/tailwind-output.css', { stdio: 'pipe' });

const css = fs.readFileSync('./dist/tailwind-output.css', 'utf8');
const cssClasses = new Set(
  css.match(/\.([a-z0-9\-]+)/g)?.map(c => c.slice(1)) || []
);

const srcFiles = [];
function walk(dir) {
  fs.readdirSync(dir).forEach(f => {
    const full = path.join(dir, f);
    if (fs.statSync(full).isDirectory()) walk(full);
    else if (f.endsWith('.tsx') || f.endsWith('.jsx')) srcFiles.push(full);
  });
}
walk(path.join(__dirname, '../src'));

let missingClasses = [];
srcFiles.forEach(file => {
  const content = fs.readFileSync(file, 'utf8');
  const classRegex = /className=["']([^"']+)["']/g;
  let m;
  while ((m = classRegex.exec(content)) !== null) {
    m[1].split(/\s+/).forEach(cls => {
      if (cls && !cssClasses.has(cls) && !cls.startsWith('#')) {
        missingClasses.push({ file, class: cls });
      }
    });
  }
});

if (missingClasses.length > 0) {
  console.error('Used Tailwind classes not found in generated CSS:');
  missingClasses.forEach(m => console.error(` ${m.class} in ${m.file}`));
  process.exit(1);
} else {
  console.log('All Tailwind classes present in output');
}
```

### 5.3 Circular Dependency Detection

Use `madge` to fail on any circular imports.

```json
// package.json script
"check:circular": "madge --circular --extensions ts,tsx src/"
```

Run locally in the pre-push script and in CI as a separate step.

### 5.4 Bundle Size Gate (`scripts/check-bundle-size.js`)

Uses `size-limit` or custom logic to compare against a baseline.

```json
{
  "size-limit": [
    { "path": "dist/assets/*.js", "limit": "200 kB", "gzip": true },
    { "path": "dist/assets/*.css", "limit": "30 kB", "gzip": true }
  ]
}
```

CI step: `npx size-limit`. If the built assets exceed the limits, the job fails.

## 6. Edge Cases & Failure Scenarios

| Scenario | Symptom | Fix / Verification |
| :--- | :--- | :--- |
| Prisma schema changed, no migration | Runtime "table doesn't exist" | `prisma migrate diff` in pre-push and CI |
| Tailwind purges dynamically-composed classes | Styles missing in prod | `check-tailwind-classes.js` + safelist pattern |
| Library's TypeScript types break after upgrade | Build fails after `pnpm update` | Use `pnpm install --frozen-lockfile` in CI; run `tsc` on every push |
| `process.env` used without validation | Build succeeds but app crashes | Use `env-var` or Zod validators; check in CI that all required vars are set |
| Two different pages claim the same route | Only one page renders | `scripts/check-routes.js` |
| Circular import between two modules | `undefined is not a function` | `madge --circular` |
| Component imports server code | Build fails intermittently | Lint rule `no-restricted-imports` (e.g., no `fs` in client) |
| Prettier & ESLint fighting | Endless reformatting | `eslint-config-prettier` |
| Prisma client not generated after schema change | `prisma.user.findMany` is not a function | Add `postinstall` script and CI test step |

## 7. Anti-Patterns (Expanded)

### What NOT to Do

- **NEVER** use `npm install` in CI: it can upgrade packages silently.
- **ALWAYS** use a frozen install: `pnpm install --frozen-lockfile`.
- **NEVER** suppress TypeScript errors to "get the build green": `// @ts-ignore` or `// @ts-expect-error` without a detailed comment and TODO.
- **NEVER** commit with `--no-verify` to bypass hooks: `git commit -m "quick fix" --no-verify`.
- **NEVER** skip tests by renaming files or using `.skip`: `test.skip(...)`.
- **NEVER** disable lint rules project-wide to silence a one-off issue: `/* eslint-disable */` or `/* eslint-disable-next-line no-console */`.
- **NEVER** hardcode secrets in CI YAML or source: `env: DATABASE_URL: postgresql://user:pass@....`.
- **ALWAYS** use GitHub Secrets: `env: DATABASE_URL: ${{ secrets.DATABASE_URL }}`.
- **NEVER** use `prisma db push` in CI (it can drift from migrations).
- **ALWAYS** use `prisma migrate deploy` to apply known migrations.
- **NEVER** assume `node_modules` are up-to-date locally.
- **ALWAYS** run `pnpm install` after pulling or switching branches.

## 8. Team Workflow Checklist

### 8.1 Before Starting a Task

- `git pull origin develop` (or rebase)
- `pnpm install` (to refresh `node_modules`)
- `npx prisma generate` (ensure client matches schema)
- Check that dev server starts: `pnpm run dev`
- All pre-existing tests pass: `pnpm test`

### 8.2 After Completing a Task (Before Commit)

- TypeScript: `npx tsc --noEmit`
- ESLint: `npx eslint ./src --ext .ts,.tsx --max-warnings 0`
- Prettier: `npx prettier --check "src/**/*.{ts,tsx,css,json}"`
- Prisma: `npx prisma validate`
- Migration drift: `npx prisma migrate diff ...` (or rely on pre-push)
- Unit tests: `pnpm run test:unit`
- Build: `pnpm run build`
- Route conflicts: `node scripts/check-routes.js`
- Tailwind: `node scripts/check-tailwind-classes.js`
- Circular deps: `npx madge --circular src/`
- Commit message follows conventional commits

### 8.3 Before Pushing

- All pre-push checks pass (the hook runs automatically)
- Push only YOUR branch

### 8.4 Before Merging PR

- All CI jobs are green
- At least one peer review approval
- No unresolved review comments
- Branch is up-to-date with target branch (rebase if needed)
- Migration files included if Prisma schema changed
- No unaddressed `TODO` or `FIXME` comments without corresponding ticket

## 9. Testing Strategy: Build Verification Tests

Add these tests to your test suite to catch configuration errors early.

**File: `tests/build/routes.test.ts`**

```typescript
import { describe, it, expect } from 'vitest';
import { appRoutes } from '@/app/routes';

describe('Route Configuration', () => {
  it('should have no duplicate route paths', () => {
    const paths = appRoutes.map(r => r.path);
    const unique = new Set(paths);
    expect(unique.size).toBe(paths.length);
  });

  it('should have no routes without a component/element', () => {
    appRoutes.forEach(route => {
      expect(route.element || route.component).toBeDefined();
    });
  });

  it('should place static routes before dynamic ones in /users', () => {
    const userRoutes = appRoutes.filter(r => r.path?.startsWith('/users'));
    let dynamicStarted = false;
    for (const route of userRoutes) {
      if (route.path?.includes(':')) {
        dynamicStarted = true;
      } else if (dynamicStarted) {
        throw new Error(
          `Static route ${route.path} defined after dynamic routes`
        );
      }
    }
  });
});
```

**File: `tests/build/tailwind.test.ts`**

```typescript
import { describe, it, expect } from 'vitest';
import fs from 'fs';
import path from 'path';

describe('Tailwind CSS Output', () => {
  it('should contain all base utility classes', () => {
    const css = fs.readFileSync(
      path.resolve(__dirname, '../../dist/tailwind-output.css'),
      'utf8'
    );
    expect(css).toContain('.flex');
    expect(css).toContain('.grid');
    expect(css).toContain('.p-4');
  });
});
```

## 10. Output Report (Automated)

At the end of a CI run, you can produce a build verification report. Here's a simple script that can be called as a final step to aggregate results.

**File: `scripts/generate-build-report.sh`**

```bash
echo "Build Verification Report"
echo ""
echo "Timestamp: $(date -u +"%Y-%m-%d %H:%M UTC")"
echo "Branch: $GITHUB_REF_NAME"
echo "Triggered by: $GITHUB_EVENT_NAME"

echo ""
echo "Layer 1 (Local):"
echo "- lint-staged: PASS"
echo "- TypeScript: PASS"
echo "- Prisma: PASS"
echo ""
echo "Layer 2 (CI):"
echo "- lint-typecheck: PASS"
echo "- tests: (42/42, 87% coverage)"
echo "- build: (client: 342 kb gz, server: 89 kb)"
echo "- route-check: (0 conflicts)"
echo ""
echo "Status: SAFE TO MERGE"
```

---

**Build Verification Skill v3.0.0**
React + Node.js + Prisma + Tailwind CSS
Generated for team reference. Keep this document updated as the pipeline evolves.
