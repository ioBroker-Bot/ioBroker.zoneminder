# Template for ioBroker Adapter Copilot Instructions

This is the template file that should be copied to your ioBroker adapter repository as `.github/copilot-instructions.md`.

## How to Use This Template

**Prerequisites:** Ensure you have GitHub Copilot already set up and working in your repository before using this template. If you need help with basic setup, see the [Prerequisites & Setup Guide](README.md#🛠️-prerequisites--basic-github-copilot-setup) in the main repository.

1. Copy this entire content
2. Save it as `.github/copilot-instructions.md` in your adapter repository
3. Customize the sections marked with `[CUSTOMIZE]` if needed
4. Commit the file to enable GitHub Copilot integration

**Note:** If downloading via curl, use the sed command to remove the template comment block:
```bash
curl -o .github/copilot-instructions.md https://raw.githubusercontent.com/DrozmotiX/ioBroker-Copilot-Instructions/main/template.md
sed -i '/^<!--$/,/^-->$/d' .github/copilot-instructions.md
```

---

# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.5.7
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

---

## 📑 Table of Contents

1. [Project Context](#project-context)
2. [Code Quality & Standards](#code-quality--standards)
   - [Code Style Guidelines](#code-style-guidelines)
   - [ESLint Configuration](#eslint-configuration)
3. [Testing](#testing)
   - [Unit Testing](#unit-testing)
   - [Integration Testing](#integration-testing)
   - [API Testing with Credentials](#api-testing-with-credentials)
4. [Development Best Practices](#development-best-practices)
   - [Dependency Management](#dependency-management)
   - [HTTP Client Libraries](#http-client-libraries)
   - [Error Handling](#error-handling)
5. [Admin UI Configuration](#admin-ui-configuration)
   - [JSON-Config Setup](#json-config-setup)
   - [Translation Management](#translation-management)
6. [Documentation](#documentation)
   - [README Updates](#readme-updates)
   - [Changelog Management](#changelog-management)
7. [CI/CD & GitHub Actions](#cicd--github-actions)
   - [Workflow Configuration](#workflow-configuration)
   - [Testing Integration](#testing-integration)

---

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

### ZoneMinder Security Camera Adapter

This adapter connects ioBroker to ZoneMinder, an open-source video surveillance software system. ZoneMinder is designed for monitoring multiple cameras and provides features like motion detection, video recording, and event management.

**Key Adapter Functions:**
- Connect to ZoneMinder server via HTTP/HTTPS API
- Monitor camera states and functions (None, Monitor, Modect, Record, Mocord, Nodect)
- Retrieve monitor information and statistics
- Handle ZoneMinder events and notifications
- Support for WebSocket connections for real-time event monitoring
- Control monitor activation and function modes

**External Dependencies:**
- ZoneMinder server (minimum version not specified)
- WebSocket support for event notifications
- HTTP API authentication (user/password or API key)

**Configuration Requirements:**
- ZoneMinder host URL (default: http://zoneminder/zm)
- Authentication credentials (username/password)
- Polling intervals for monitors and states
- Event monitoring toggle

## Code Quality & Standards

### Code Style Guidelines

- Follow JavaScript/TypeScript best practices
- Use async/await for asynchronous operations
- Implement proper resource cleanup in `unload()` method
- Use semantic versioning for adapter releases
- Include proper JSDoc comments for public methods

**Timer and Resource Cleanup Example:**
```javascript
private connectionTimer?: NodeJS.Timeout;

async onReady() {
  this.connectionTimer = setInterval(() => this.checkConnection(), 30000);
}

onUnload(callback) {
  try {
    if (this.connectionTimer) {
      clearInterval(this.connectionTimer);
      this.connectionTimer = undefined;
    }
    callback();
  } catch (e) {
    callback();
  }
}
```

### ESLint Configuration

**CRITICAL:** ESLint validation must run FIRST in your CI/CD pipeline, before any other tests. This "lint-first" approach catches code quality issues early.

#### Setup
```bash
npm install --save-dev eslint @iobroker/eslint-config
```

#### Configuration (.eslintrc.json)
```json
{
  "extends": "@iobroker/eslint-config",
  "rules": {
    // Add project-specific rule overrides here if needed
  }
}
```

#### Package.json Scripts
```json
{
  "scripts": {
    "lint": "eslint --max-warnings 0 .",
    "lint:fix": "eslint . --fix"
  }
}
```

#### Best Practices
1. ✅ Run ESLint before committing — fix ALL warnings, not just errors
2. ✅ Use `lint:fix` for auto-fixable issues
3. ✅ Don't disable rules without documentation
4. ✅ Lint all relevant files (main code, tests, build scripts)
5. ✅ Keep `@iobroker/eslint-config` up to date
6. ✅ **ESLint warnings are treated as errors in CI** (`--max-warnings 0`). The `lint` script above already includes this flag — run `npm run lint` to match CI behavior locally

#### Common Issues
- **Unused variables**: Remove or prefix with underscore (`_variable`)
- **Missing semicolons**: Run `npm run lint:fix`
- **Indentation**: Use 4 spaces (ioBroker standard)
- **console.log**: Replace with `adapter.log.debug()` or remove

---

## Testing

### Unit Testing

- Use Jest as the primary testing framework
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files

**Example Structure:**
```javascript
describe('AdapterName', () => {
  let adapter;
  
  beforeEach(() => {
    // Setup test adapter instance
  });
  
  test('should initialize correctly', () => {
    // Test adapter initialization
  });
});
```

### Integration Testing

**CRITICAL:** Use the official `@iobroker/testing` framework. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation:** https://github.com/ioBroker/testing

#### Framework Structure

**✅ Correct Pattern:**
```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        // Get adapter object
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) return reject(new Error('Adapter object not found'));

                        // Configure adapter
                        Object.assign(obj.native, {
                            position: '52.520008,13.404954',
                            createHourly: true,
                        });

                        harness.objects.setObject(obj._id, obj);
                        
                        // Start and wait
                        await harness.startAdapterAndWait();
                        await new Promise(resolve => setTimeout(resolve, 15000));

                        // Verify states
                        const stateIds = await harness.dbConnection.getStateIDs('your-adapter.0.*');
                        
                        if (stateIds.length > 0) {
                            console.log('✅ Adapter successfully created states');
                            await harness.stopAdapter();
                            resolve(true);
                        } else {
                            reject(new Error('Adapter did not create any states'));
                        }
                    } catch (error) {
                        reject(error);
                    }
                });
            }).timeout(40000);
        });
    }
});
```

#### Testing Success AND Failure Scenarios

**IMPORTANT:** For every "it works" test, implement corresponding "it fails gracefully" tests.

**Failure Scenario Example:**
```javascript
it('should NOT create daily states when daily is disabled', function () {
    return new Promise(async (resolve, reject) => {
        try {
            harness = getHarness();
            const obj = await new Promise((res, rej) => {
                harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                    if (err) return rej(err);
                    res(o);
                });
            });
            
            if (!obj) return reject(new Error('Adapter object not found'));

            Object.assign(obj.native, {
                createDaily: false, // Daily disabled
            });

            await new Promise((res, rej) => {
                harness.objects.setObject(obj._id, obj, (err) => {
                    if (err) return rej(err);
                    res(undefined);
                });
            });

            await harness.startAdapterAndWait();
            await new Promise((res) => setTimeout(res, 20000));

            const stateIds = await harness.dbConnection.getStateIDs('your-adapter.0.*');
            const dailyStates = stateIds.filter((key) => key.includes('daily'));
            
            if (dailyStates.length === 0) {
                console.log('✅ No daily states found as expected');
                resolve(true);
            } else {
                reject(new Error('Expected no daily states but found some'));
            }

            await harness.stopAdapter();
        } catch (error) {
            reject(error);
        }
    });
}).timeout(40000);
```

#### Key Rules

1. ✅ Use `@iobroker/testing` framework
2. ✅ Configure via `harness.objects.setObject()`
3. ✅ Start via `harness.startAdapterAndWait()`
4. ✅ Verify states via `harness.states.getState()`
5. ✅ Allow proper timeouts for async operations
6. ❌ NEVER test API URLs directly
7. ❌ NEVER bypass the harness system

#### Workflow Dependencies

Integration tests should run ONLY after lint and adapter tests pass:

```yaml
integration-tests:
  needs: [check-and-lint, adapter-tests]
  runs-on: ubuntu-22.04
```

### API Testing with Credentials

For adapters connecting to external APIs requiring authentication:

#### Password Encryption for Integration Tests

```javascript
async function encryptPassword(harness, password) {
    const systemConfig = await harness.objects.getObjectAsync("system.config");
    if (!systemConfig?.native?.secret) {
        throw new Error("Could not retrieve system secret for password encryption");
    }
    
    const secret = systemConfig.native.secret;
    let result = '';
    for (let i = 0; i < password.length; ++i) {
        result += String.fromCharCode(secret[i % secret.length].charCodeAt(0) ^ password.charCodeAt(i));
    }
    return result;
}
```

#### Demo Credentials Testing Pattern

- Use provider demo credentials when available (e.g., `demo@api-provider.com` / `demo`)
- Create separate test file: `test/integration-demo.js`
- Add npm script: `"test:integration-demo": "mocha test/integration-demo --exit"`
- Implement clear success/failure criteria

**Example Implementation:**
```javascript
it("Should connect to API with demo credentials", async () => {
    const encryptedPassword = await encryptPassword(harness, "demo_password");
    
    await harness.changeAdapterConfig("your-adapter", {
        native: {
            username: "demo@provider.com",
            password: encryptedPassword,
        }
    });

    await harness.startAdapter();
    await new Promise(resolve => setTimeout(resolve, 60000));
    
    const connectionState = await harness.states.getStateAsync("your-adapter.0.info.connection");
    
    if (connectionState?.val === true) {
        console.log("✅ SUCCESS: API connection established");
        return true;
    } else {
        throw new Error("API Test Failed: Expected API connection. Check logs for API errors.");
    }
}).timeout(120000);
```

#### ZoneMinder-Specific Testing Considerations

- Mock ZoneMinder API responses to avoid requiring a live ZoneMinder installation
- Test authentication mechanisms and error handling for connection failures
- Validate monitor state changes and function switching
- Test WebSocket connection handling and reconnection logic
- Verify event processing and state updates

Example mock data structure for ZoneMinder API:
```javascript
const mockMonitorData = {
    "monitors": [
        {
            "Monitor": {
                "Id": "1",
                "Name": "Camera1",
                "Function": "Modect",
                "Enabled": "1",
                "Width": "1920",
                "Height": "1080"
            }
        }
    ]
};
```

---

## Development Best Practices

### Dependency Management

- Always use `npm` for dependency management
- Use `npm ci` for installing existing dependencies (respects package-lock.json)
- Use `npm install` only when adding or updating dependencies
- Keep dependencies minimal and focused
- Only update dependencies in separate Pull Requests

**When modifying package.json:**
1. Run `npm install` to sync package-lock.json
2. Commit both package.json and package-lock.json together

**Best Practices:**
- Prefer built-in Node.js modules when possible
- Use `@iobroker/adapter-core` for adapter base functionality
- Avoid deprecated packages
- Document specific version requirements

### HTTP Client Libraries

- **Preferred:** Use native `fetch` API (Node.js 20+ required)
- **Avoid:** `axios` unless specific features are required

**Example with fetch:**
```javascript
try {
  const response = await fetch('https://api.example.com/data');
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }
  const data = await response.json();
} catch (error) {
  this.log.error(`API request failed: ${error.message}`);
}
```

**Other Recommendations:**
- **Logging:** Use adapter built-in logging (`this.log.*`)
- **Scheduling:** Use adapter built-in timers and intervals
- **File operations:** Use Node.js `fs/promises`
- **Configuration:** Use adapter config system

### Error Handling

- Always catch and log errors appropriately
- Use adapter log levels (error, warn, info, debug)
- Provide meaningful, user-friendly error messages
- Handle network failures gracefully
- Implement retry mechanisms where appropriate
- Always clean up timers, intervals, and resources in `unload()` method

**Example:**
```javascript
try {
  await this.connectToDevice();
} catch (error) {
  this.log.error(`Failed to connect to device: ${error.message}`);
  this.setState('info.connection', false, true);
  // Implement retry logic if needed
}
```

---

## Admin UI Configuration

### JSON-Config Setup

Use JSON-Config format for modern ioBroker admin interfaces.

**Example Structure:**
```json
{
  "type": "panel",
  "items": {
    "host": {
      "type": "text",
      "label": "Host address",
      "help": "IP address or hostname of the device"
    }
  }
}
```

**Guidelines:**
- ✅ Use consistent naming conventions
- ✅ Provide sensible default values
- ✅ Include validation for required fields
- ✅ Add tooltips for complex options
- ✅ Ensure translations for all supported languages (minimum English and German)
- ✅ Write end-user friendly labels, avoid technical jargon

### Translation Management

**CRITICAL:** Translation files must stay synchronized with `admin/jsonConfig.json`. Orphaned keys or missing translations cause UI issues and PR review delays.

#### Overview
- **Location:** `admin/i18n/{lang}/translations.json` for 11 languages (de, en, es, fr, it, nl, pl, pt, ru, uk, zh-cn)
- **Source of truth:** `admin/jsonConfig.json` - all `label` and `help` properties must have translations
- **Command:** `npm run translate` - auto-generates translations but does NOT remove orphaned keys
- **Formatting:** English uses tabs, other languages use 4 spaces

#### Critical Rules
1. ✅ Keys must match exactly with jsonConfig.json
2. ✅ No orphaned keys in translation files
3. ✅ All translations must be in native language (no English fallbacks)
4. ✅ Keys must be sorted alphabetically

#### Workflow for Translation Updates

**When modifying admin/jsonConfig.json:**

1. Make your changes to labels/help texts
2. Run automatic translation: `npm run translate`
3. Create validation script (`scripts/validate-translations.js`):

```javascript
const fs = require('fs');
const path = require('path');
const jsonConfig = JSON.parse(fs.readFileSync('admin/jsonConfig.json', 'utf8'));

function extractTexts(obj, texts = new Set()) {
    if (typeof obj === 'object' && obj !== null) {
        if (obj.label) texts.add(obj.label);
        if (obj.help) texts.add(obj.help);
        for (const key in obj) {
            extractTexts(obj[key], texts);
        }
    }
    return texts;
}

const requiredTexts = extractTexts(jsonConfig);
const languages = ['de', 'en', 'es', 'fr', 'it', 'nl', 'pl', 'pt', 'ru', 'uk', 'zh-cn'];
let hasErrors = false;

languages.forEach(lang => {
    const translationPath = path.join('admin', 'i18n', lang, 'translations.json');
    const translations = JSON.parse(fs.readFileSync(translationPath, 'utf8'));
    const translationKeys = new Set(Object.keys(translations));
    
    const missing = Array.from(requiredTexts).filter(text => !translationKeys.has(text));
    const orphaned = Array.from(translationKeys).filter(key => !requiredTexts.has(key));
    
    console.log(`\n=== ${lang} ===`);
    if (missing.length > 0) {
        console.error('❌ Missing keys:', missing);
        hasErrors = true;
    }
    if (orphaned.length > 0) {
        console.error('❌ Orphaned keys (REMOVE THESE):', orphaned);
        hasErrors = true;
    }
    if (missing.length === 0 && orphaned.length === 0) {
        console.log('✅ All keys match!');
    }
});

process.exit(hasErrors ? 1 : 0);
```

4. Run validation: `node scripts/validate-translations.js`
5. Remove orphaned keys manually from all translation files
6. Add missing translations in native languages
7. Run: `npm run lint && npm run test`

#### Add Validation to package.json

```json
{
  "scripts": {
    "translate": "translate-adapter",
    "validate:translations": "node scripts/validate-translations.js",
    "pretest": "npm run lint && npm run validate:translations"
  }
}
```

#### Translation Checklist

Before committing changes to admin UI or translations:
1. ✅ Validation script shows "All keys match!" for all 11 languages
2. ✅ No orphaned keys in any translation file
3. ✅ All translations in native language
4. ✅ Keys alphabetically sorted
5. ✅ `npm run lint` passes
6. ✅ `npm run test` passes
7. ✅ Admin UI displays correctly

---

## Documentation

### README Updates

#### Required Sections
1. **Installation** - Clear npm/ioBroker admin installation steps
2. **Configuration** - Detailed configuration options with examples
3. **Usage** - Practical examples and use cases
4. **Changelog** - Version history (use "## **WORK IN PROGRESS**" for ongoing changes)
5. **License** - License information (typically MIT for ioBroker adapters)
6. **Support** - Links to issues, discussions, community support

#### Documentation Standards
- Use clear, concise language
- Include code examples for configuration
- Add screenshots for admin interface when applicable
- Maintain multilingual support (minimum English and German)
- Always reference issues in commits and PRs (e.g., "fixes #xx")

#### Mandatory README Updates for PRs

For **every PR or new feature**, always add a user-friendly entry to README.md:

- Add entries under `## **WORK IN PROGRESS**` section
- Use format: `* (author) **TYPE**: Description of user-visible change`
- Types: **NEW** (features), **FIXED** (bugs), **ENHANCED** (improvements), **TESTING** (test additions), **CI/CD** (automation)
- Focus on user impact, not technical details

**Example:**
```markdown
## **WORK IN PROGRESS**

* (DutchmanNL) **FIXED**: Adapter now properly validates login credentials (fixes #25)
* (DutchmanNL) **NEW**: Added device discovery to simplify initial setup
```

### Changelog Management

Follow the [AlCalzone release-script](https://github.com/AlCalzone/release-script) standard.

#### Format Requirements

```markdown
# Changelog

<!--
  Placeholder for the next version (at the beginning of the line):
  ## **WORK IN PROGRESS**
-->

## **WORK IN PROGRESS**

- (author) **NEW**: Added new feature X
- (author) **FIXED**: Fixed bug Y (fixes #25)

## v0.1.0 (2023-01-01)
Initial release
```

#### Workflow Process
- **During Development:** All changes go under `## **WORK IN PROGRESS**`
- **For Every PR:** Add user-facing changes to WORK IN PROGRESS section
- **Before Merge:** Version number and date added when merging to main
- **Release Process:** Release-script automatically converts placeholder to actual version

#### Change Entry Format
- Format: `- (author) **TYPE**: User-friendly description`
- Types: **NEW**, **FIXED**, **ENHANCED**
- Focus on user impact, not technical implementation
- Reference issues: "fixes #XX" or "solves #XX"

---

## CI/CD & GitHub Actions

### Workflow Configuration

#### GitHub Actions Best Practices

**Must use ioBroker official testing actions:**
- `ioBroker/testing-action-check@v1` for lint and package validation
- `ioBroker/testing-action-adapter@v1` for adapter tests
- `ioBroker/testing-action-deploy@v1` for automated releases with Trusted Publishing (OIDC)

**Configuration:**
- **Node.js versions:** Test on 20.x, 22.x, 24.x
- **Platform:** Use ubuntu-22.04
- **Automated releases:** Deploy to npm on version tags (requires NPM Trusted Publishing)
- **Monitoring:** Include Sentry release tracking for error monitoring

#### Critical: Lint-First Validation Workflow

**ALWAYS run ESLint checks BEFORE other tests.** Benefits:
- Catches code quality issues immediately
- Prevents wasting CI resources on tests that would fail due to linting errors
- Provides faster feedback to developers
- Enforces consistent code quality

**Workflow Dependency Configuration:**
```yaml
jobs:
  check-and-lint:
    # Runs ESLint and package validation
    # Uses: ioBroker/testing-action-check@v1
    
  adapter-tests:
    needs: [check-and-lint]  # Wait for linting to pass
    # Run adapter unit tests
    
  integration-tests:
    needs: [check-and-lint, adapter-tests]  # Wait for both
    # Run integration tests
```

**Key Points:**
- The `check-and-lint` job has NO dependencies - runs first
- ALL other test jobs MUST list `check-and-lint` in their `needs` array
- If linting fails, no other tests run, saving time
- Fix all ESLint errors before proceeding

### Testing Integration

#### API Testing in CI/CD

For adapters with external API dependencies:

```yaml
demo-api-tests:
  if: contains(github.event.head_commit.message, '[skip ci]') == false
  runs-on: ubuntu-22.04
  
  steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run demo API tests
      run: npm run test:integration-demo
```

#### Testing Best Practices
- Run credential tests separately from main test suite
- Don't make credential tests required for deployment
- Provide clear failure messages for API issues
- Use appropriate timeouts for external calls (120+ seconds)

#### Package.json Integration
```json
{
  "scripts": {
    "test:integration-demo": "mocha test/integration-demo --exit"
  }
}
```

---

[CUSTOMIZE: ZoneMinder-specific adapter architecture, code patterns, and conventions]

## Architecture & File Structure

### Standard ioBroker Adapter Structure
Your adapter should follow the standard ioBroker adapter file structure:

```
adapter-name/
├── admin/              # Admin interface files
│   ├── index.html     # Admin web interface
│   ├── words.js       # Translations
│   └── zoneminder.png # Icon
├── lib/               # Library files
│   └── api.js         # ZoneMinder API client
├── test/              # Test files
│   ├── integration.js
│   └── package.js
├── main.js           # Main adapter file
├── io-package.json   # ioBroker configuration
└── package.json      # NPM configuration
```

### Key Files in ZoneMinder Adapter

**`main.js`** - Main adapter logic:
- Adapter initialization and configuration
- Connection to ZoneMinder API
- Monitor state management
- Event handling and state updates
- WebSocket connection management
- Proper cleanup in unload() method

**`lib/api.js`** - ZoneMinder API client:
- HTTP request handling to ZoneMinder server
- Authentication management
- Monitor control functions
- Event processing
- WebSocket client for real-time notifications

**`io-package.json`** - Adapter configuration:
- Adapter metadata and requirements
- Native configuration schema
- Instance objects definition
- Dependencies on js-controller and admin

## Code Patterns & Best Practices

### ioBroker Adapter Lifecycle
Always implement these key lifecycle methods properly:

```javascript
// In main.js
function startAdapter(options) {
    return adapter = utils.adapter(Object.assign({}, options, {
        name: 'zoneminder',
        ready: main, // Called when adapter is ready
        unload: (callback) => {
            // CRITICAL: Clean up resources
            try {
                clearTimeout(requestInterval);
                if (zm) {
                    zm.websocketClose();
                }
                adapter.log.info('cleaned everything up...');
                callback();
            } catch (e) {
                callback();
            }
        },
        stateChange: (id, state) => {
            // Handle state changes
            if (state && !state.ack) {
                // Handle commands
            }
        }
    }));
}
```

### ZoneMinder API Integration Patterns

#### Monitor Function States
```javascript
const FUNCSTATES = {
    0: 'None',
    1: 'Monitor',
    2: 'Modect',
    3: 'Record',
    4: 'Mocord',
    5: 'Nodect'
};

// Function to change monitor function
functionChange(id, state, callback) {
    this._get2(`/api/monitors/${id}.json?&` + this.authUrl, 
               `Monitor[Function]=${FUNCSTATES[state]}`, callback);
}
```

#### WebSocket Event Handling
```javascript
// WebSocket connection for real-time events
websocketConnect() {
    this.client = new WebSocketClient();
    this.client.on('connectFailed', (error) => {
        this.adapter.log.error('Connect Error: ' + error.toString());
    });
    this.client.on('connect', (connection) => {
        connection.on('message', (message) => {
            if (message.type === 'utf8') {
                const data = JSON.parse(message.utf8Data);
                this.processEvent(data);
            }
        });
    });
}
```

### State Management

#### Creating States
```javascript
// Create monitor states
_createState(sid, name, type, val, callback) {
    adapter.getObject(sid, (err, obj) => {
        if (!obj) {
            adapter.setObject(sid, {
                type: 'state',
                common: {
                    name: name,
                    type: type,
                    role: 'indicator',
                    read: true,
                    write: true,
                    def: val
                },
                native: {}
            }, callback);
        } else if (callback) {
            callback();
        }
    });
}
```

#### State Change Handling
```javascript
// Handle state changes for monitor control
if (id.endsWith('.active')) {
    const monitorId = id.split('.')[2];
    const active = state.val ? 1 : 0;
    zm.activeCange(monitorId, active, (err, result) => {
        if (!err) {
            adapter.setStateAsync(id, { val: state.val, ack: true });
        }
    });
} else if (id.endsWith('.function')) {
    const monitorId = id.split('.')[2];
    const funcState = state.val;
    zm.functionCange(monitorId, funcState, (err, result) => {
        if (!err) {
            adapter.setStateAsync(id, { val: state.val, ack: true });
        }
    });
}
```

### Error Handling Patterns

#### API Error Handling
```javascript
// Proper error handling for ZoneMinder API calls
_get(url, callback) {
    const options = {
        url: this.host + url,
        headers: {
            'User-Agent': 'ioBroker ZoneMinder Adapter'
        },
        timeout: 10000
    };

    rp(options)
        .then((response) => {
            try {
                const json = JSON.parse(response);
                if (typeof callback === 'function') callback(null, json);
            } catch (e) {
                this.adapter.log.error('JSON Parse Error: ' + e.message);
                if (typeof callback === 'function') callback(e);
            }
        })
        .catch((error) => {
            this.adapter.log.error('Request failed: ' + error.message);
            if (typeof callback === 'function') callback(error);
        });
}
```

#### Connection Monitoring
```javascript
// Monitor connection status
checkConnection() {
    this.getVersion((err, data) => {
        if (err) {
            this.adapter.setStateAsync('info.connection', false, true);
            this.adapter.log.warn('ZoneMinder connection lost');
        } else {
            this.adapter.setStateAsync('info.connection', true, true);
        }
    });
}
```

## Dependencies & Libraries

### Core ioBroker Dependencies
- `@iobroker/adapter-core`: Core adapter functionality
- Minimum js-controller version: 5.0.19
- Admin interface compatibility: >=6.13.16

### ZoneMinder-Specific Dependencies
- `request-promise`: HTTP requests to ZoneMinder API (deprecated, consider migration)
- `websocket`: WebSocket client for real-time event monitoring
- `mqtt`: MQTT client for alternative communication (if used)

### Development Dependencies
- `@iobroker/testing`: Official testing framework
- `@iobroker/adapter-dev`: Development tools
- `eslint`: Code linting
- `mocha`: Test runner
- `sinon`: Test mocking

## Configuration

### io-package.json Native Configuration
```json
{
  "native": {
    "host": "http://zoneminder/zm",
    "user": "admin",
    "password": "admin",
    "pollingMon": 5,
    "pollingMonState": 1,
    "zmEvent": false
  }
}
```

### Admin Interface (admin/index.html)
Provide user-friendly configuration interface for:
- ZoneMinder server URL
- Authentication credentials
- Polling intervals
- Event monitoring options
- Connection testing

## Logging & Debugging

### Logging Levels
Use appropriate logging levels throughout the adapter:

```javascript
adapter.log.error('Critical errors that prevent operation');
adapter.log.warn('Important warnings that don\'t stop operation');
adapter.log.info('General information about adapter state');
adapter.log.debug('Detailed information for troubleshooting');
```

### ZoneMinder-Specific Logging
```javascript
// Log API responses for debugging
this.adapter.log.debug('ZoneMinder API response: ' + JSON.stringify(data));

// Log monitor state changes
this.adapter.log.info(`Monitor ${id} function changed to ${FUNCSTATES[state]}`);

// Log WebSocket events
this.adapter.log.debug('ZoneMinder event received: ' + JSON.stringify(event));
```

## Common Patterns to Avoid

### Anti-patterns in ioBroker Development
1. **Don't modify package.json for version tracking** - Use copilot-instructions.md version field
2. **Don't ignore unload() cleanup** - Always properly clean up resources
3. **Don't block the event loop** - Use async/await properly
4. **Don't hardcode delays** - Use configurable timeouts
5. **Don't ignore error states** - Always handle API failures gracefully

### ZoneMinder-Specific Anti-patterns
1. **Don't poll too aggressively** - Respect server resources with reasonable intervals
2. **Don't store credentials in plain text** - Use ioBroker's native secure storage
3. **Don't ignore WebSocket connection failures** - Implement reconnection logic
4. **Don't assume API format** - Always validate response structure
5. **Don't mix sync and async patterns** - Be consistent with promise handling

## Security Considerations

### Authentication & Credentials
- Store sensitive data in adapter's native configuration
- Support multiple authentication methods (user/pass, API keys)
- Validate SSL certificates for HTTPS connections
- Implement proper session management for API access

### API Security
```javascript
// Example secure API request handling
makeSecureRequest(endpoint, data) {
    const options = {
        url: this.config.host + endpoint,
        method: 'POST',
        json: true,
        body: data,
        headers: {
            'Authorization': `Bearer ${this.apiKey}`,
            'User-Agent': 'ioBroker ZoneMinder Adapter'
        },
        timeout: 10000,
        strictSSL: true // Validate SSL certificates
    };
    
    return rp(options);
}
```

## Performance Optimization

### Efficient Polling Strategies
- Use configurable polling intervals
- Implement exponential backoff for failed requests
- Cache frequently accessed data
- Use WebSocket for real-time events instead of polling when possible

### Memory Management
- Clean up timers and intervals in unload()
- Release WebSocket connections properly
- Avoid memory leaks in event handlers
- Monitor adapter memory usage

### Network Optimization
- Batch API requests when possible
- Use compression for large data transfers
- Implement request queuing for high-frequency operations
- Handle rate limiting from ZoneMinder server

## Code Style & Conventions

Follow these conventions when working with this adapter:

### JavaScript/ES6+ Patterns
- Use `const` and `let` instead of `var`
- Prefer async/await over callbacks where possible
- Use template literals for string formatting
- Implement proper error handling with try/catch

### ioBroker Specific Conventions
- Use adapter.log instead of console.log
- Follow the adapter naming convention (lowercase, dots for separation)
- Use proper state roles and types
- Implement proper cleanup in unload method

### ZoneMinder API Patterns
- Use consistent error handling for all API calls
- Implement retry logic for transient failures
- Cache API responses when appropriate
- Validate API response structure before processing

## Updates & Maintenance

This file is managed by the centralized ioBroker Copilot Instructions system. 
- Version updates are handled automatically
- Custom sections marked with [CUSTOMIZE] are preserved during updates
- Template source: https://github.com/DrozmotiX/ioBroker-Copilot-Instructions
- Current template version: 0.5.7

For issues or suggestions regarding these instructions, please visit the template repository.