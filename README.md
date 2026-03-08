# Sigma File Manager Extensions Registry

This repository contains the official registry of extensions for [Sigma File Manager](https://github.com/aleksey-hoffman/sigma-file-manager).

## Table of Contents

- [Browsing Extensions](#browsing-extensions)
- [Documentation](#documentation)
- [Quick Start](#quick-start)
- [Extension Architecture](#extension-architecture)
- [Best Practices](#best-practices)
- [Submitting to Registry](#submitting-to-registry)
- [Examples](#examples)

---

## Browsing Extensions

Extensions can be browsed and installed directly from the Sigma File Manager app via the **Extensions** page in the sidebar.

---

## Documentation

The extension API documentation is maintained in the wiki:

- [Extensions Wiki](https://github.com/sigma-hub/sfm-extensions/wiki)
- [Getting Started](https://github.com/sigma-hub/sfm-extensions/wiki/Getting-Started)
- [API Reference](https://github.com/sigma-hub/sfm-extensions/wiki/API-Reference)
- [UI Reference](https://github.com/sigma-hub/sfm-extensions/wiki/UI-Reference)
- [Manifest Reference](https://github.com/sigma-hub/sfm-extensions/wiki/Manifest-Reference)
- [Best Practices](https://github.com/sigma-hub/sfm-extensions/wiki/Best-Practices)
- [API Types](https://raw.githubusercontent.com/aleksey-hoffman/sigma-file-manager/main/packages/api/index.d.ts)
- [Package Schema](https://raw.githubusercontent.com/aleksey-hoffman/sigma-file-manager/main/packages/api/manifest.schema.json)

---

## Quick Start

### 1. Create Your Extension Repository

```bash
mkdir my-extension
cd my-extension
git init
```

### 2. Install API Types

```bash
npm install -D @sigma-file-manager/api
```

### 3. Create `package.json`

```json
{
  "$schema": "https://raw.githubusercontent.com/aleksey-hoffman/sigma-file-manager/main/packages/api/manifest.schema.json",
  "id": "your-username.my-extension",
  "name": "My Extension",
  "version": "1.0.0",
  "repository": "https://github.com/your-username/my-extension",
  "license": "MIT",
  "extensionType": "api",
  "main": "index.js",
  "permissions": ["commands", "notifications"],
  "engines": {
    "sigmaFileManager": ">=2.0.0"
  }
}
```

### 4. Create `index.js`

```javascript
async function activate(context) {
  console.log('Extension activated!', context.extensionPath);
  
  sigma.commands.registerCommand(
    { id: 'hello', title: 'Say Hello' },
    () => {
      sigma.ui.showNotification({
        title: 'Hello!',
        subtitle: 'My first extension is working!',
        type: 'success'
      });
    }
  );
}

function deactivate() {
  console.log('Extension deactivated');
}

module.exports = { activate, deactivate };
```

### 5. Test Locally

In Sigma File Manager:
1. Go to **Extensions** page
2. Click **Install from folder**
3. Select your extension folder

### 6. Push your extension to Github

```bash
git add .
git commit -m "Initial release"
git tag v1.0.0
git push origin main --tags
```

---

## Extension Architecture

### Registry vs Package

The extension system uses two complementary sources of metadata:

| Source | Location | Controller | Purpose |
|--------|----------|------------|---------|
| **Registry** | This repo (`registry.json`) | SFM maintainers | Curated marketplace metadata |
| **Package** | Extension repo (`package.json`) | Extension developer | Runtime configuration |

### Design Principles

1. **Trust & Curation**: Fields that affect user trust (publisher, description, categories) are controlled by the registry
2. **Developer Control**: Technical/runtime fields (permissions, entry point, version) are controlled by the package
3. **Automatic Versioning**: Versions are fetched from GitHub release tags (`v*` pattern)

## Best Practices

### Handle Errors Gracefully

Network requests can fail, files may be missing, and binaries can crash. Rather than letting unhandled exceptions disrupt the user's workflow, catch errors and show helpful notifications.

```javascript
async function activate() {
  sigma.commands.registerCommand(
    { id: 'fetch-data', title: 'Fetch Data' },
    async () => {
      try {
        const data = await fetchFromAPI();
        sigma.ui.showNotification({
          title: 'Data loaded',
        subtitle: `Loaded ${data.length} items`,
          type: 'success'
        });
      } catch (error) {
        sigma.ui.showNotification({
          title: 'Failed to fetch data',
          subtitle: error.message || 'Check your network connection and try again',
          type: 'error'
        });
      }
    }
  );
}

module.exports = { activate };
```

**Key principles:**
- Wrap all async operations in `try/catch` blocks
- Show a notification with `type: 'error'` instead of silently failing
- Provide actionable error messages so users know what went wrong
- Use `sigma.storage` as a cache fallback when network requests fail:

```javascript
async function loadItems() {
  try {
    const freshData = await fetchFromAPI();
    await sigma.storage.set('cachedItems', freshData);
    return freshData;
  } catch (error) {
    const cached = await sigma.storage.get('cachedItems');
    if (cached) {
      sigma.ui.showNotification({
        title: 'Using cached data',
        subtitle: 'Could not fetch latest data. Showing previously loaded results.',
        type: 'warning'
      });
      return cached;
    }
    throw error;
  }
}
```

---

### Handle Runtime Dependencies

If your extension depends on external binaries (e.g., FFmpeg, yt-dlp), use `sigma.binary.ensureInstalled` to download them automatically. Always check availability before running commands, and show a clear message if a dependency is missing.

```javascript
const TOOL_ID = 'my-tool';

async function getToolPath() {
  const isInstalled = await sigma.binary.isInstalled(TOOL_ID);

  if (isInstalled) {
    return sigma.binary.getPath(TOOL_ID);
  }

  return sigma.binary.ensureInstalled(TOOL_ID, {
    name: 'my-tool',
    downloadUrl: (platform) => {
      const ext = platform === 'windows' ? '.exe' : '';
      return `https://example.com/releases/my-tool-${platform}${ext}`;
    },
    version: '2.1.0'
  });
}
```

**Key principles:**
- Use `sigma.binary.ensureInstalled` for automatic download and setup
- If only some commands require the binary, don't block activation — check availability inside the specific command handler
- Use `sigma.platform.os` to provide platform-specific download URLs
- Use `sigma.binary.getInfo` to check installed versions and offer updates

---

### Show Progress for Long Operations

When a command performs work that takes more than a second or two, use `sigma.ui.withProgress` to show a progress indicator. This keeps users informed and lets them cancel if needed.

```javascript
sigma.commands.registerCommand(
  { id: 'process-files', title: 'Process Files' },
  async () => {
    const files = sigma.context.getSelectedEntries().filter(entry => entry.isFile);

    if (files.length === 0) {
      sigma.ui.showNotification({
        title: 'No files selected',
        message: 'Select one or more files to process',
        type: 'warning'
      });
      return;
    }

    await sigma.ui.withProgress(
      { title: 'Processing files...', cancellable: true },
      async (progress, token) => {
        for (let index = 0; index < files.length; index++) {
          if (token.isCancellationRequested) {
            sigma.ui.showNotification({
              title: 'Cancelled',
              message: `Processed ${index} of ${files.length} files`,
              type: 'info'
            });
            return;
          }

          progress.report({
            message: files[index].name,
            increment: 100 / files.length
          });

          await processFile(files[index]);
        }
      }
    );

    sigma.ui.showNotification({
      title: 'Complete',
      message: `Processed ${files.length} files`,
      type: 'success'
    });
  }
);
```

**Key principles:**
- Use `cancellable: true` for any operation that takes more than a few seconds
- Check `token.isCancellationRequested` at each iteration or step
- Report progress with `message` (current item name) and `increment` (percentage points)
- Show a summary notification when the operation completes or is cancelled

---

### Validate Input Before Acting

When accepting user input (via command arguments, modals, or prompts), validate it before proceeding. Show clear feedback when validation fails.

#### Command Arguments

If a command declares `arguments` in the manifest, the Command Palette will prompt the user before execution. Always validate the received values:

```javascript
sigma.commands.registerCommand(
  {
    id: 'open-url',
    title: 'Open URL',
    arguments: [
      { name: 'url', type: 'text', placeholder: 'https://...', required: true }
    ]
  },
  async (args) => {
    const providedArgs = args && typeof args === 'object' ? args : {};
    const url = typeof providedArgs.url === 'string' ? providedArgs.url.trim() : '';

    if (!url) {
      sigma.ui.showNotification({
        title: 'URL required',
        message: 'Please provide a URL',
        type: 'warning'
      });
      return;
    }

    if (!url.startsWith('http://') && !url.startsWith('https://')) {
      sigma.ui.showNotification({
        title: 'Invalid URL',
        message: 'URL must start with http:// or https://',
        type: 'error'
      });
      return;
    }

    await openUrl(url);
  }
);
```

#### Modal Forms

For richer forms, validate in the `onSubmit` callback before processing:

```javascript
const modal = sigma.ui.createModal({
  title: 'Create Project',
  width: 400,
  content: [
    sigma.ui.input({ id: 'name', label: 'Project Name', placeholder: 'my-project' }),
    sigma.ui.select({
      id: 'template',
      label: 'Template',
      options: [
        { value: 'basic', label: 'Basic' },
        { value: 'advanced', label: 'Advanced' }
      ],
      value: 'basic'
    })
  ],
  buttons: [
    { id: 'cancel', label: 'Cancel', variant: 'secondary' },
    { id: 'create', label: 'Create', variant: 'primary' }
  ]
});

modal.onSubmit(async (values, buttonId) => {
  if (buttonId !== 'create') return;

  const name = typeof values.name === 'string' ? values.name.trim() : '';
  if (!name) {
    sigma.ui.showNotification({
      title: 'Name required',
      message: 'Enter a project name',
      type: 'warning'
    });
    return;
  }

  if (!/^[a-z0-9-]+$/.test(name)) {
    sigma.ui.showNotification({
      title: 'Invalid name',
      message: 'Use only lowercase letters, numbers, and hyphens',
      type: 'error'
    });
    return;
  }

  await createProject(name, values.template);
});
```

---

### Write Platform-Aware Code

If your extension behaves differently across operating systems, use `sigma.platform` to adapt. If your extension only works on certain platforms, declare the `platforms` field in `package.json`:

```json
{
  "platforms": ["windows", "macos"]
}
```

When omitted, the extension is considered cross-platform. The app will prevent installation on unsupported platforms and show a warning in the marketplace.

For platform-specific logic at runtime:

```javascript
async function getConfigPath() {
  if (sigma.platform.isWindows) {
    return sigma.platform.joinPath(process.env.APPDATA, 'MyTool', 'config.json');
  }
  if (sigma.platform.isMacos) {
    return sigma.platform.joinPath(process.env.HOME, 'Library', 'Application Support', 'MyTool', 'config.json');
  }
  return sigma.platform.joinPath(process.env.HOME, '.config', 'mytool', 'config.json');
}
```

---

### Clean Up Resources

Registration methods (commands, context menu items, event listeners) return `Disposable` objects. These are automatically cleaned up when the extension is disabled or uninstalled, but if you create resources that need explicit cleanup (timers, event listeners, temporary files), do so in `deactivate`:

```javascript
let pollingInterval = null;

async function activate() {
  pollingInterval = setInterval(checkForUpdates, 60000);

  sigma.commands.registerCommand(
    { id: 'check-now', title: 'Check for Updates' },
    checkForUpdates
  );
}

function deactivate() {
  if (pollingInterval) {
    clearInterval(pollingInterval);
    pollingInterval = null;
  }
}

module.exports = { activate, deactivate };
```

---

## Submitting to Registry

### Prerequisites

1. Public GitHub repository
2. Valid `package.json` in repository root
3. At least one release tag (e.g., `v1.0.0`)
4. README with usage instructions

### Submission Steps

1. **Fork this repository**

2. **Add entry to `registry.json`:**
   ```json
   {
     "id": "your-username.extension-name",
     "name": "Your Extension",
     "description": "Clear description of what it does",
     "publisher": "Your Name",
     "publisherUrl": "https://github.com/your-username",
     "repository": "https://github.com/your-username/your-extension",
     "featured": false,
     "categories": ["Productivity"],
     "tags": ["keyword1", "keyword2"],
     "releaseMetadata": {
       "1.0.0": {
         "integrity": "sha256:<release-zip-sha256-hex>"
       }
     }
   }
   ```

3. **Submit a Pull Request**

### Submission Guidelines

- ID format: `publisher.extension-name` (lowercase, hyphens allowed)
- Repository must be publicly accessible
- No malicious code or unauthorized data collection
- Provide useful, working functionality
- Include clear documentation
- Provide `releaseMetadata.<version>.integrity` for every published version so installers can verify release archive integrity
- Make sure `engines.sigmaFileManager` in the extension `package.json` matches the real compatibility range, because installs now enforce it

### Available Categories

| Category | Description |
|----------|-------------|
| `File Management` | File operations, organization |
| `Media` | Image, video, audio processing |
| `Security` | Encryption, privacy tools |
| `Backup & Sync` | Backup, synchronization |
| `Productivity` | Workflow automation |
| `Search & Filter` | Advanced search, filtering |
| `Appearance` | Themes, visual customization |
| `Integration` | Third-party services |
| `Developer Tools` | Development utilities |
| `Example` | Demo extensions |
| `Other` | Miscellaneous |

---

## Examples

### Example 1: Simple Context Menu Action

```javascript
async function activate() {
  sigma.contextMenu.registerItem(
    {
      id: 'copy-name',
      title: 'Copy File Name',
      icon: 'Copy',
      group: 'extensions',
      when: { selectionType: 'single' }
    },
    async (context) => {
      const name = context.selectedEntries[0].name;
      await navigator.clipboard.writeText(name);
      sigma.ui.showNotification({
        title: 'Copied',
        message: name,
        type: 'success'
      });
    }
  );
}

module.exports = { activate };
```

### Example 2: Command with Settings

```javascript
async function activate() {
  sigma.commands.registerCommand(
    { id: 'greet', title: 'Greet User' },
    async () => {
      const greeting = await sigma.settings.get('greeting');
      const result = await sigma.ui.showDialog({
        title: greeting,
        message: 'What is your name?',
        type: 'prompt'
      });
      
      if (result.confirmed) {
        sigma.ui.showNotification({
          title: greeting,
          message: `Hello, ${result.value}!`,
          type: 'success'
        });
      }
    }
  );
}

module.exports = { activate };
```

### Example 3: File Processing with Progress

```javascript
async function activate() {
  sigma.commands.registerCommand(
    { id: 'process-files', title: 'Process Selected Files' },
    async () => {
      const files = sigma.context.getSelectedEntries()
        .filter(e => e.isFile);
      
      if (files.length === 0) {
        sigma.ui.showNotification({
          title: 'No files selected',
          message: 'Please select some files first',
          type: 'warning'
        });
        return;
      }
      
      await sigma.ui.withProgress(
        { title: 'Processing...', cancellable: true },
        async (progress, token) => {
          for (let i = 0; i < files.length; i++) {
            if (token.isCancellationRequested) break;
            
            progress.report({
              message: `Processing ${files[i].name}`,
              increment: 100 / files.length
            });
            
            // Your processing logic here
            await processFile(files[i]);
          }
        }
      );
      
      sigma.ui.showNotification({
        title: 'Complete',
        message: `Processed ${files.length} files`,
        type: 'success'
      });
    }
  );
}

module.exports = { activate };
```

### Example 4: External Binary Usage

```javascript
const TOOL_ID = 'my-tool';

async function activate() {
  sigma.commands.registerCommand(
    { id: 'run-tool', title: 'Run External Tool' },
    async () => {
      // Ensure binary is installed
      const toolPath = await sigma.binary.ensureInstalled(TOOL_ID, {
        name: 'my-tool',
        downloadUrl: (platform) => {
          const ext = platform === 'windows' ? '.exe' : '';
          return `https://example.com/releases/my-tool-${platform}${ext}`;
        }
      });
      
      // Run with progress streaming
      const { result, cancel } = await sigma.shell.runWithProgress(
        toolPath,
        ['--input', sigma.context.getCurrentPath()],
        (payload) => {
          console.log(payload.line);
        }
      );
      
      const output = await result;
      
      if (output.code === 0) {
        sigma.ui.showNotification({
          title: 'Success',
          message: 'Tool completed successfully',
          type: 'success'
        });
      } else {
        sigma.ui.showNotification({
          title: 'Error',
          message: output.stderr,
          type: 'error'
        });
      }
    }
  );
}

module.exports = { activate };
```

### Example 5: Custom Modal Form

```javascript
async function activate() {
  sigma.commands.registerCommand(
    { id: 'create-item', title: 'Create New Item' },
    async () => {
      return new Promise((resolve) => {
        const modal = sigma.ui.createModal({
          title: 'Create New Item',
          width: 400,
          content: [
            sigma.ui.input({ id: 'name', label: 'Name', placeholder: 'Enter name' }),
            sigma.ui.select({
              id: 'type',
              label: 'Type',
              options: [
                { value: 'file', label: 'File' },
                { value: 'folder', label: 'Folder' }
              ]
            }),
            sigma.ui.checkbox({ id: 'open', label: 'Open after creation', checked: true })
          ],
          buttons: [
            { id: 'cancel', label: 'Cancel', variant: 'secondary' },
            { id: 'create', label: 'Create', variant: 'primary' }
          ]
        });
        
        modal.onSubmit(async (values, buttonId) => {
          if (buttonId === 'create' && values.name) {
            // Create the item
            await createItem(values.name, values.type);
            
            if (values.open) {
              await sigma.commands.executeCommand(
                'sigma.navigator.openPath',
                values.name
              );
            }
            
            resolve(values);
          } else {
            resolve(null);
          }
        });
        
        modal.onClose(() => resolve(null));
      });
    }
  );
}

module.exports = { activate };
```

---

## Badge System

| Badge | Meaning |
|-------|---------|
| **Official** | Created by the Sigma File Manager team (`sigma-hub` organization) |
| **Community** | Third-party extension |
| **Featured** | Highlighted in the marketplace |

---

## Schema Validation

Both `registry.json` and `featured.json` have JSON schemas for validation:

- `registry.schema.json` - Validates registry entries
- `featured.schema.json` - Validates featured list

Use these schemas in your editor for autocomplete and validation.

---

## Resources

- [Documentation Wiki](https://github.com/sigma-hub/sfm-extensions/wiki) - Extension guides and API docs
- [Example Extension](https://github.com/sigma-hub/sfm-extension-example) - Full-featured demo
- [Video Downloader](https://github.com/sigma-hub/sfm-extension-video-downloader) - Real-world example
- [SDK Types](https://raw.githubusercontent.com/aleksey-hoffman/sigma-file-manager/v2/src/modules/extensions/sdk/sigma-extension.d.ts) - TypeScript definitions
- [Package Schema](https://raw.githubusercontent.com/aleksey-hoffman/sigma-file-manager/v2/src/modules/extensions/sdk/manifest.schema.json) - JSON Schema
- [Lucide Icons](https://lucide.dev/icons/) - Icon reference

---

## License

MIT License - see [LICENSE](LICENSE) for details.
