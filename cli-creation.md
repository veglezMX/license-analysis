# Progressive Guide: App Scaffolding CLI for React UI Platform

## Overview

**What we're building:** A Node.js CLI tool that scaffolds React applications and generates components with built-in design token integration, testing, and governance.

**Why it matters:** Eliminates bootstrap friction, enforces consistency across teams, and creates a "golden path" from idea to production-ready code.

**End state:** Running `ui-cli create my-app` produces a fully configured React 19 application with tokens, linting, tests, and CI—in under 5 minutes.

**Target audience:** Frontend engineers, tech leads, and platform teams adopting the React UI library.

**Prerequisites:**
- Familiarity with Node.js CLI development
- Understanding of React project structures
- Access to existing design tokens pipeline and UI library

---

## Core Concepts

Before building, internalize these foundational pieces:

### 1. Template Engine Patterns
File-based templates with variable interpolation. Options include Handlebars, EJS, or simple string replacement. The engine reads template files, substitutes placeholders with user-provided values, and writes the output.

### 2. AST Manipulation
For intelligent code insertion—updating imports, registering routes, adding exports to barrel files. Tools like `ts-morph` or `jscodeshift` enable surgical code modifications without string manipulation.

### 3. Interactive Prompts
Gathering user preferences via CLI. Libraries like Clack, Inquirer, or Prompts provide multi-select, confirmations, and text inputs with validation.

### 4. Project Blueprints
Versioned, composable presets that define structure and defaults. A "web-app" blueprint differs from "admin-dashboard" in routing, layout, and feature set—but shares core conventions.

### 5. Plugin Architecture
Extension points for future capabilities (Storybook, i18n, analytics, auth) without bloating the core. Plugins register generators, modify templates, and hook into lifecycle events.

---

## Phases

### Phase 1: Foundation
**Duration:** Week 1-2  
**Goal:** Runnable CLI skeleton with basic `create` command

#### Tasks

| Task | Validation |
|------|------------|
| Initialize monorepo structure (`packages/cli`, `packages/templates`, `packages/core`) | `pnpm install` and `pnpm build` succeed |
| Set up TypeScript with tsup for CLI bundling | `node dist/index.js --version` outputs version |
| Implement command parser using Commander.js | `ui-cli --help` shows available commands |
| Create minimal `create <name>` that copies a static template folder | `ui-cli create test-app` produces folder with files |
| Add basic error handling and colored output (chalk/picocolors) | Errors display in red, success in green |

#### Key Decisions
- **Package manager:** pnpm (workspace support, speed)
- **Command parser:** Commander.js (mature, TypeScript support, subcommands)
- **Build tool:** tsup (fast, zero-config for CLI bundling)

#### Deliverables
```
packages/
├── cli/
│   ├── src/
│   │   ├── index.ts          # Entry point
│   │   ├── commands/
│   │   │   └── create.ts     # Create command
│   │   └── utils/
│   │       └── logger.ts     # Colored output
│   ├── package.json
│   └── tsconfig.json
├── templates/
│   └── base/                 # Static starter template
└── core/                     # Shared utilities (future)
```

#### Checkpoint
You can run the CLI locally and scaffold an empty project shell. The command `ui-cli create my-app` creates a directory with a basic React structure.

---

### Phase 2: Template Engine
**Duration:** Week 3-4  
**Goal:** Dynamic file generation with variable substitution

#### Tasks

| Task | Validation |
|------|------------|
| Integrate EJS as template engine | `.ejs` files render with injected variables |
| Define template variable schema with TypeScript types | Type errors catch missing variables at build time |
| Build recursive file walker that processes templates | Nested directories preserve structure in output |
| Implement conditional file inclusion | `--no-tests` flag excludes `__tests__/` folders |
| Add file renaming support (`__name__.tsx.ejs` → `MyApp.tsx`) | Dynamic filenames resolve correctly |

#### Template Variable Schema
```typescript
interface TemplateContext {
  appName: string;
  appNamePascal: string;
  appNameKebab: string;
  styling: 'css-modules' | 'vanilla-extract' | 'tailwind' | 'styled-components';
  includeTests: boolean;
  testingLibrary: 'vitest' | 'jest';
  includeStorybook: boolean;
  includeRouter: boolean;
  tokenPackage: string;
  year: number;
}
```

#### Template Example
```ejs
// package.json.ejs
{
  "name": "<%= appNameKebab %>",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
<% if (includeTests) { -%>
    "test": "<%= testingLibrary %>"
<% } -%>
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "<%= tokenPackage %>": "latest"
  }
}
```

#### Checkpoint
`ui-cli create my-app --style=vanilla-extract --no-tests` produces a customized project with the correct styling setup and no test configuration.

---

### Phase 3: Interactive Prompts
**Duration:** Week 5  
**Goal:** Guided setup experience for users who don't pass flags

#### Tasks

| Task | Validation |
|------|------------|
| Integrate Clack for modern prompt UI | Prompts display with spinners and styling |
| Define prompt flow sequence | All prompts execute in logical order |
| Map prompt answers to template context | Answers populate `TemplateContext` correctly |
| Implement `--yes` flag for default values | Non-interactive mode skips all prompts |
| Add preset shortcuts (`--preset=admin`) | Preset auto-selects bundled options |

#### Prompt Flow
```
1. What is your project name? [text input]
2. Which styling solution? [select: css-modules, vanilla-extract, tailwind, styled-components]
3. Include testing setup? [confirm]
   └─ If yes: Which test runner? [select: vitest, jest]
4. Include Storybook? [confirm]
5. Include React Router? [confirm]
6. [Summary] Create project with these settings? [confirm]
```

#### Presets Definition
```typescript
const presets: Record<string, Partial<TemplateContext>> = {
  'web-app': {
    styling: 'vanilla-extract',
    includeTests: true,
    testingLibrary: 'vitest',
    includeStorybook: false,
    includeRouter: true,
  },
  'admin-dashboard': {
    styling: 'vanilla-extract',
    includeTests: true,
    testingLibrary: 'vitest',
    includeStorybook: true,
    includeRouter: true,
  },
  'marketing-site': {
    styling: 'tailwind',
    includeTests: false,
    includeStorybook: false,
    includeRouter: true,
  },
};
```

#### Checkpoint
Running `ui-cli create` without arguments launches an interactive wizard. Non-technical users can scaffold apps by answering questions.

---

### Phase 4: Design Token Integration
**Duration:** Week 6-7  
**Goal:** Generated apps consume tokens correctly by default

#### Tasks

| Task | Validation |
|------|------------|
| Include token package as dependency in templates | `package.json` lists `@org/design-tokens` |
| Wire ThemeProvider in app shell | Root component wraps children with provider |
| Generate token-aware style files per styling solution | Styles import and use token variables |
| Create example component using tokens | Button component demonstrates token usage |
| Add token documentation link in generated README | Developers know where to find token docs |

#### Styling Integration Matrix

| Styling Solution | Token Consumption Pattern |
|------------------|--------------------------|
| CSS Modules | CSS custom properties from tokens build |
| Vanilla Extract | Import tokens as TypeScript constants |
| Tailwind | Extend theme config with token values |
| Styled Components | ThemeProvider with token object |

#### Generated Structure Example (Vanilla Extract)
```
src/
├── styles/
│   ├── theme.css.ts      # Token-based theme definition
│   ├── sprinkles.css.ts  # Utility classes from tokens
│   └── global.css.ts     # Reset + base styles
├── components/
│   └── Button/
│       ├── Button.tsx
│       └── Button.css.ts # Uses theme tokens
└── App.tsx               # Wrapped with style provider
```

#### Checkpoint
Generated apps have working token integration. Changing a token value in the source propagates to the scaffolded app after rebuild.

---

### Phase 5: Component Generator
**Duration:** Week 8-9  
**Goal:** `add component` command generates consistent components

#### Tasks

| Task | Validation |
|------|------------|
| Implement `add component <Name>` command | Command recognized and executes |
| Detect project styling solution from config | Generator uses correct style template |
| Generate component file with proper structure | Component follows UI library patterns |
| Add `--with-tests` flag for test file | Test file created alongside component |
| Add `--with-story` flag for Storybook story | Story file created with basic stories |
| Update barrel exports automatically | `index.ts` includes new component export |

#### Command Interface
```bash
ui-cli add component Button
ui-cli add component UserCard --with-tests --with-story
ui-cli add component Modal --path=src/features/auth/components
```

#### Generated Component Structure
```
src/components/Button/
├── Button.tsx           # Component implementation
├── Button.css.ts        # Styles (varies by styling solution)
├── Button.test.tsx      # Tests (if --with-tests)
├── Button.stories.tsx   # Storybook (if --with-story)
├── Button.types.ts      # TypeScript interfaces
└── index.ts             # Barrel export
```

#### Component Template (Base)
```tsx
// <%= componentName %>.tsx.ejs
import { forwardRef } from 'react';
import type { <%= componentName %>Props } from './<%= componentName %>.types';
import * as styles from './<%= componentName %>.css';

export const <%= componentName %> = forwardRef<<%= elementType %>, <%= componentName %>Props>(
  function <%= componentName %>(props, ref) {
    const { children, className, ...rest } = props;

    return (
      <<%= elementTag %>
        ref={ref}
        className={styles.root}
        {...rest}
      >
        {children}
      </<%= elementTag %>>
    );
  }
);
```

#### Checkpoint
`ui-cli add component Card --with-tests --with-story` creates a complete, properly-wired component matching UI library conventions.

---

### Phase 6: Page & Feature Generators
**Duration:** Week 10-11  
**Goal:** Higher-level generators for pages and feature modules

#### Tasks

| Task | Validation |
|------|------------|
| Implement `add page <Name>` command | Creates page component in pages directory |
| Auto-register route in router config | New page accessible at generated path |
| Implement `add feature <Name>` command | Creates feature folder with full structure |
| Support feature-based architecture patterns | Components, hooks, utils grouped by feature |
| Add `--crud` flag for data-driven features | Generates list, detail, create, edit pages |

#### Page Generator Output
```bash
ui-cli add page Dashboard
```
```
src/pages/Dashboard/
├── Dashboard.tsx
├── Dashboard.css.ts
├── Dashboard.loader.ts    # Route loader (if using data routers)
└── index.ts
```

#### Feature Generator Output
```bash
ui-cli add feature Users --crud
```
```
src/features/users/
├── components/
│   ├── UserList/
│   ├── UserDetail/
│   ├── UserForm/
│   └── index.ts
├── hooks/
│   ├── useUsers.ts
│   ├── useUser.ts
│   └── index.ts
├── api/
│   └── users.api.ts
├── types/
│   └── user.types.ts
├── utils/
│   └── user.utils.ts
└── index.ts
```

#### Checkpoint
Complex features can be scaffolded with one command, maintaining consistent architecture across the codebase.

---

### Phase 7: Governance & Validation
**Duration:** Week 12-13  
**Goal:** Enforce standards and catch drift

#### Tasks

| Task | Validation |
|------|------------|
| Create `lint` command to check project structure | Reports deviations from conventions |
| Validate naming conventions (files, components, exports) | Incorrect names flagged with suggestions |
| Check for proper token usage in styles | Missing token imports reported |
| Implement `--fix` flag for auto-corrections | Fixable issues resolved automatically |
| Add CI integration mode (`--ci`) with exit codes | CI fails on validation errors |

#### Validation Rules
```typescript
const rules: ValidationRule[] = [
  {
    id: 'component-structure',
    description: 'Components must have index.ts barrel export',
    validate: (componentDir) => hasFile(componentDir, 'index.ts'),
    fix: (componentDir) => createBarrelExport(componentDir),
  },
  {
    id: 'token-usage',
    description: 'Style files must import from design tokens',
    validate: (styleFile) => importsFrom(styleFile, '@org/design-tokens'),
    severity: 'warning',
  },
  {
    id: 'naming-convention',
    description: 'Component files must be PascalCase',
    validate: (file) => isPascalCase(getFileName(file)),
    fix: (file) => renameFile(file, toPascalCase(getFileName(file))),
  },
];
```

#### Command Interface
```bash
ui-cli lint                    # Check all rules
ui-cli lint --fix              # Auto-fix where possible
ui-cli lint --ci               # Exit 1 on any error
ui-cli lint --rule=token-usage # Check specific rule
```

#### Checkpoint
Teams can run `ui-cli lint --ci` in their pipelines to enforce consistency without manual review.

---

### Phase 8: Configuration & Customization
**Duration:** Week 14-15  
**Goal:** Allow team-specific customization

#### Tasks

| Task | Validation |
|------|------------|
| Define `.ui-cli.json` configuration schema | CLI reads and validates config file |
| Support custom template directories | Team templates override defaults |
| Implement team presets via config | `--preset=team-x` loads custom settings |
| Add generator aliases and shortcuts | Teams can rename commands |
| Support extending/overriding validation rules | Custom rules load from config |

#### Configuration Schema
```typescript
interface CliConfig {
  // Template customization
  templates?: {
    directory?: string;      // Custom templates path
    extends?: string;        // Extend another preset
  };
  
  // Default values
  defaults?: Partial<TemplateContext>;
  
  // Generator settings
  generators?: {
    component?: {
      defaultPath?: string;
      includeTests?: boolean;
      includeStory?: boolean;
    };
  };
  
  // Validation
  rules?: {
    enabled?: string[];
    disabled?: string[];
    custom?: string;         // Path to custom rules
  };
  
  // Team presets
  presets?: Record<string, Partial<TemplateContext>>;
}
```

#### Example Configuration
```json
{
  "$schema": "https://ui-cli.dev/schema.json",
  "defaults": {
    "styling": "vanilla-extract",
    "testingLibrary": "vitest"
  },
  "generators": {
    "component": {
      "defaultPath": "src/components",
      "includeTests": true
    }
  },
  "rules": {
    "disabled": ["naming-convention"],
    "custom": "./cli-rules"
  }
}
```

#### Checkpoint
Teams can customize the CLI behavior without forking, maintaining compatibility with core updates.

---

### Phase 9: Upgrade & Migration Tools
**Duration:** Week 16-17  
**Goal:** Help projects stay current

#### Tasks

| Task | Validation |
|------|------------|
| Implement `upgrade` command | Detects and applies available updates |
| Create version tracking in generated projects | `.ui-cli-version` stores template version |
| Build codemod system for breaking changes | Codemods transform code automatically |
| Add `diff` command to show template changes | Users see what changed between versions |
| Implement rollback capability | Failed upgrades can be reverted |

#### Upgrade Flow
```bash
ui-cli upgrade                 # Interactive upgrade wizard
ui-cli upgrade --check         # Show available updates only
ui-cli upgrade --to=2.0.0      # Upgrade to specific version
ui-cli upgrade --dry-run       # Show changes without applying
```

#### Codemod Example
```typescript
// codemods/v2-token-imports.ts
export const codemod: Codemod = {
  name: 'v2-token-imports',
  description: 'Update token imports for v2 package structure',
  fromVersion: '1.x',
  toVersion: '2.x',
  transform(file, api) {
    const j = api.jscodeshift;
    return j(file.source)
      .find(j.ImportDeclaration, { source: { value: '@org/tokens' } })
      .forEach(path => {
        path.node.source.value = '@org/design-tokens/css';
      })
      .toSource();
  },
};
```

#### Checkpoint
Projects created months ago can run `ui-cli upgrade` and receive the latest templates, conventions, and dependency updates.

---

### Phase 10: Plugin Ecosystem
**Duration:** Week 18-20  
**Goal:** Extensible architecture for future growth

#### Tasks

| Task | Validation |
|------|------------|
| Define plugin API interface | TypeScript types document extension points |
| Implement plugin discovery and loading | Plugins in `node_modules` auto-discovered |
| Create lifecycle hooks system | Plugins can hook into create, add, lint |
| Build first-party plugins (storybook, i18n, auth) | Plugins install and work correctly |
| Document plugin development guide | External teams can build plugins |

#### Plugin Interface
```typescript
interface CliPlugin {
  name: string;
  version: string;
  
  // Extend commands
  commands?: CommandDefinition[];
  
  // Add generators
  generators?: GeneratorDefinition[];
  
  // Modify templates
  templateTransforms?: TemplateTransform[];
  
  // Add validation rules
  rules?: ValidationRule[];
  
  // Lifecycle hooks
  hooks?: {
    'pre:create'?: (context: CreateContext) => Promise<void>;
    'post:create'?: (context: CreateContext) => Promise<void>;
    'pre:add'?: (context: AddContext) => Promise<void>;
    'post:add'?: (context: AddContext) => Promise<void>;
  };
}
```

#### First-Party Plugins
```
@org/cli-plugin-storybook    # Storybook integration and generators
@org/cli-plugin-i18n         # Internationalization setup
@org/cli-plugin-auth         # Authentication boilerplate
@org/cli-plugin-analytics    # Analytics integration
@org/cli-plugin-api          # API layer generators (React Query, etc.)
```

#### Checkpoint
The CLI is extensible. Teams can build and share plugins for their specific needs without modifying core.

---

## Project: Reference Implementation

Build this end-to-end to validate the guide:

### Milestone 1: Basic Scaffolding (Phase 1-3)
Create a working CLI that scaffolds a React app with interactive prompts.

**Acceptance Criteria:**
- [ ] `ui-cli create my-app` produces runnable React app
- [ ] Interactive prompts collect user preferences
- [ ] `--yes` flag uses sensible defaults
- [ ] Generated app starts with `npm run dev`

### Milestone 2: Token Integration (Phase 4)
Wire design tokens into generated projects.

**Acceptance Criteria:**
- [ ] Generated apps import token package
- [ ] Styles use token values (not hardcoded)
- [ ] ThemeProvider configured at app root
- [ ] Example component demonstrates token usage

### Milestone 3: Component Generation (Phase 5-6)
Add component and feature generators.

**Acceptance Criteria:**
- [ ] `ui-cli add component X` creates complete component
- [ ] Tests and stories generated with flags
- [ ] Barrel exports updated automatically
- [ ] Feature generator creates full module structure

### Milestone 4: Governance (Phase 7-8)
Implement validation and configuration.

**Acceptance Criteria:**
- [ ] `ui-cli lint` catches convention violations
- [ ] `--fix` auto-corrects fixable issues
- [ ] `.ui-cli.json` customizes behavior
- [ ] CI mode exits with proper codes

### Milestone 5: Production Ready (Phase 9-10)
Add upgrade tools and plugin system.

**Acceptance Criteria:**
- [ ] `ui-cli upgrade` updates existing projects
- [ ] Codemods handle breaking changes
- [ ] Plugin API documented and working
- [ ] First plugin published and functional

---

## Best Practices

### Template Authoring
- Keep templates minimal; avoid over-abstraction
- Use clear variable names (`appName` not `n`)
- Include helpful comments that survive generation
- Test templates with edge cases (special characters, long names)

### Generator Design
- Generate code that looks hand-written
- Follow existing project conventions when adding to projects
- Provide escape hatches for advanced users
- Document what gets generated and why

### Governance Balance
- Start permissive, tighten over time
- Make rules fixable where possible
- Explain why rules exist in error messages
- Allow teams to disable rules with justification

### Plugin Development
- Keep plugins focused (one capability per plugin)
- Version plugins alongside CLI compatibility
- Provide migration guides for plugin updates
- Test plugins against CLI version matrix

### User Experience
- Fast is better than feature-rich
- Clear error messages with suggested fixes
- Respect existing project structure
- Support both interactive and scripted usage

---

## Troubleshooting

### Common Issues

**"Command not found: ui-cli"**
- Ensure global install: `npm install -g @org/ui-cli`
- Or use npx: `npx @org/ui-cli create my-app`
- Check PATH includes npm global bin directory

**"Template rendering failed"**
- Check variable names match schema exactly
- Ensure no syntax errors in `.ejs` templates
- Run with `--verbose` for detailed error stack

**"Could not detect project type"**
- Ensure you're in a project created by the CLI
- Check for `.ui-cli.json` or `ui-cli` field in `package.json`
- Run `ui-cli init` to add CLI support to existing project

**"Validation rule X is failing incorrectly"**
- Check rule configuration in `.ui-cli.json`
- Disable rule temporarily: `ui-cli lint --disable=X`
- Report false positives to CLI maintainers

**"Plugin not loading"**
- Verify plugin is installed in project: `npm ls @org/cli-plugin-x`
- Check plugin compatibility with CLI version
- Run `ui-cli doctor` to diagnose plugin issues

### Debug Mode

Enable verbose logging for troubleshooting:

```bash
DEBUG=ui-cli:* ui-cli create my-app
```

Log levels:
- `ui-cli:cli` - Command parsing and execution
- `ui-cli:template` - Template rendering
- `ui-cli:generator` - Generator operations
- `ui-cli:plugin` - Plugin loading and hooks

---

## Success Metrics

Track these KPIs to measure adoption and impact:

| Metric | Target | Measurement |
|--------|--------|-------------|
| Time to first running app | < 5 minutes | Timed user tests |
| Time to add compliant component | < 1 minute | Timed user tests |
| Bootstrap ticket reduction | -80% | Jira ticket count |
| CLI adoption rate | 90% of new projects | Project survey |
| Lint pass rate | 95% on first CI run | CI metrics |
| Developer satisfaction | > 4.5/5 | Quarterly survey |

---

## Resources

### Dependencies
- [Commander.js](https://github.com/tj/commander.js) - Command parsing
- [Clack](https://github.com/natemoo-re/clack) - Interactive prompts
- [EJS](https://ejs.co/) - Template engine
- [ts-morph](https://ts-morph.com/) - TypeScript AST manipulation
- [picocolors](https://github.com/alexeyraspopov/picocolors) - Terminal colors
- [tsup](https://tsup.egoist.dev/) - TypeScript bundler

### Inspiration
- [Angular CLI](https://angular.io/cli) - Gold standard for framework CLIs
- [Create React App](https://create-react-app.dev/) - Zero-config bootstrapping
- [Nx](https://nx.dev/) - Monorepo tools and generators
- [Plop](https://plopjs.com/) - Micro-generator framework
- [Hygen](https://www.hygen.io/) - Code generator

### Internal Links
- Design Tokens Documentation
- React UI Library Storybook
- Platform Team Confluence Space
- CLI GitHub Repository
