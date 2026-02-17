# Sigma File Manager Extensions Registry

This repository contains the official registry of extensions for [Sigma File Manager](https://github.com/aleksey-hoffman/sigma-file-manager).

## Table of Contents

- [Browsing Extensions](#browsing-extensions)
- [Quick Start](#quick-start)
- [Extension Architecture](#extension-architecture)
- [Extension Lifecycle](#extension-lifecycle)
- [Extension API Reference](#extension-api-reference)
- [Manifest Reference](#manifest-reference)
- [Submitting to Registry](#submitting-to-registry)
- [Examples](#examples)

---

## Browsing Extensions

Extensions can be browsed and installed directly from the Sigma File Manager app via the **Extensions** page in the sidebar.

---

## Quick Start

### 1. Create Your Extension Repository

```bash
mkdir my-extension
cd my-extension
git init
```

### 2. Download SDK Types (Optional, for TypeScript support)

```bash
curl -O https://raw.githubusercontent.com/aleksey-hoffman/sigma-file-manager/v2/src/modules/extensions/sdk/sigma-extension.d.ts
```

### 3. Create `manifest.json`

```json
{
  "$schema": "https://raw.githubusercontent.com/aleksey-hoffman/sigma-file-manager/v2/src/modules/extensions/sdk/manifest.schema.json",
  "id": "your-username.my-extension",
  "name": "My Extension",
  "version": "1.0.0",
  "repository": "https://github.com/your-username/my-extension",
  "license": "MIT",
  "type": "api",
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
        message: 'My first extension is working!',
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

### Registry vs Manifest

The extension system uses two complementary sources of metadata:

| Source | Location | Controller | Purpose |
|--------|----------|------------|---------|
| **Registry** | This repo (`registry.json`) | SFM maintainers | Curated marketplace metadata |
| **Manifest** | Extension repo (`manifest.json`) | Extension developer | Runtime configuration |

### Design Principles

1. **Trust & Curation**: Fields that affect user trust (author, description, categories) are controlled by the registry
2. **Developer Control**: Technical/runtime fields (permissions, entry point, version) are controlled by the manifest
3. **Automatic Versioning**: Versions are fetched from GitHub release tags (`v*` pattern)

## Extension Lifecycle

### Activation Events

Extensions can be activated by various events defined in `activationEvents`:

| Event | When Triggered |
|-------|----------------|
| `onStartup` | App starts (if extension is enabled) |
| `onInstall` | Extension is installed |
| `onUninstall` | Extension is being uninstalled |
| `onEnable` | Extension is enabled |
| `onDisable` | Extension is disabled |
| `onUpdate` | Extension is updated to a new version |
| `onCommand:commandId` | Specific command is executed |

### Lifecycle Functions

```javascript
async function activate(context) {
  // context.extensionPath - Absolute path to extension directory
  // context.storagePath - Path for extension-specific storage
  // context.activationEvent - Which event triggered activation
  
  // Register your handlers here
  sigma.commands.registerCommand(...);
  sigma.contextMenu.registerItem(...);
}

function deactivate() {
  // Cleanup code (optional)
  // Disposables are automatically cleaned up
}

module.exports = { activate, deactivate };
```

### Disposables

Most registration methods return a `Disposable` object. These are automatically cleaned up when the extension is disabled/uninstalled, but you can manually dispose them:

```javascript
const disposable = sigma.commands.registerCommand(...);
// Later, if needed:
disposable.dispose();
```

---

## Extension API Reference

Extensions access the API through the global `sigma` object. All methods are available after activation.

### `sigma.commands` - Command Registration

Register commands that appear in the Command Palette (`Ctrl+Shift+P`).

```javascript
// Register a command
sigma.commands.registerCommand(
  {
    id: 'my-command',           // Unique within your extension
    title: 'Do Something',      // Display name
    description: 'Optional description',
    icon: 'Zap',                // Lucide icon name (optional)
    shortcut: 'ctrl+shift+m'    // Default shortcut (optional)
  },
  async (...args) => {
    // Command handler
    return result; // Optional return value
  }
);

// Execute a command (yours or built-in)
await sigma.commands.executeCommand('sigma.quickView.open', filePath);

// List available built-in commands
const builtins = sigma.commands.getBuiltinCommands();
// Returns: [{ id, title, description }, ...]
```

**Built-in Commands:**
- `sigma.navigator.openPath` - Navigate to a path
- `sigma.quickView.open` - Open quick view for a file
- `sigma.dialog.openFile` - Open file picker dialog

---

### `sigma.contextMenu` - Context Menu Items

Add items to the file browser right-click menu.

```javascript
sigma.contextMenu.registerItem(
  {
    id: 'my-action',
    title: 'My Action',
    icon: 'FileText',           // Lucide icon name
    group: 'extensions',        // Menu group
    order: 1,                   // Sort order within group
    when: {                     // Conditional visibility (optional)
      selectionType: 'single',  // 'single' | 'multiple' | 'any'
      entryType: 'file',        // 'file' | 'directory' | 'any'
      fileExtensions: ['.txt', '.md']  // Filter by extension
    }
  },
  async (context) => {
    // context.selectedEntries - Array of selected items
    for (const entry of context.selectedEntries) {
      console.log(entry.path, entry.name, entry.isDirectory);
    }
  }
);
```

---

### `sigma.context` - App Context

Access current navigation state and selection.

```javascript
// Get current directory path
const currentPath = sigma.context.getCurrentPath();
// Returns: string | null

// Get selected files/folders
const selected = sigma.context.getSelectedEntries();
// Returns: ContextEntry[]
// Each entry: { path, name, isDirectory, isFile, size, extension, createdAt, modifiedAt }

// Get app version
const version = await sigma.context.getAppVersion();

// Get user's downloads directory
const downloadsDir = await sigma.context.getDownloadsDir();

// React to path changes
sigma.context.onPathChange((newPath) => {
  console.log('Navigated to:', newPath);
});

// React to selection changes
sigma.context.onSelectionChange((entries) => {
  console.log('Selected:', entries.length, 'items');
});
```

---

### `sigma.ui` - User Interface

Show notifications, dialogs, progress indicators, and custom modals.

#### Notifications

```javascript
sigma.ui.showNotification({
  title: 'Success!',
  message: 'Operation completed',
  type: 'success',    // 'info' | 'success' | 'warning' | 'error'
  duration: 4000      // milliseconds (optional)
});
```

#### Dialogs

```javascript
// Info dialog
await sigma.ui.showDialog({
  title: 'Information',
  message: 'This is important information',
  type: 'info',
  confirmText: 'OK'
});

// Confirmation dialog
const result = await sigma.ui.showDialog({
  title: 'Confirm',
  message: 'Are you sure?',
  type: 'confirm',
  confirmText: 'Yes',
  cancelText: 'No'
});
if (result.confirmed) { /* proceed */ }

// Prompt dialog
const result = await sigma.ui.showDialog({
  title: 'Enter Name',
  message: 'What is your name?',
  type: 'prompt',
  defaultValue: 'World',
  confirmText: 'Submit'
});
if (result.confirmed) {
  console.log('Name:', result.value);
}
```

#### Progress Indicator

```javascript
const result = await sigma.ui.withProgress(
  {
    title: 'Processing...',
    location: 'notification',  // 'notification' | 'statusBar'
    cancellable: true
  },
  async (progress, token) => {
    // Listen for cancellation
    token.onCancellationRequested(() => {
      console.log('User cancelled');
    });
    
    for (let i = 0; i < 100; i++) {
      if (token.isCancellationRequested) {
        return { cancelled: true };
      }
      
      progress.report({
        message: `Step ${i + 1} of 100`,
        increment: 1  // Percentage points
      });
      
      await doWork();
    }
    
    return { success: true };
  }
);
```

#### Custom Modals

Create rich forms with multiple input types:

```javascript
const modal = sigma.ui.createModal({
  title: 'Download Options',
  width: 480,
  content: [
    sigma.ui.input({
      id: 'url',
      label: 'URL',
      placeholder: 'Enter URL...',
      value: ''
    }),
    sigma.ui.select({
      id: 'quality',
      label: 'Quality',
      options: [
        { value: 'high', label: 'High' },
        { value: 'medium', label: 'Medium' },
        { value: 'low', label: 'Low' }
      ],
      value: 'high'
    }),
    sigma.ui.checkbox({
      id: 'notify',
      label: 'Notify when complete',
      checked: true
    }),
    sigma.ui.textarea({
      id: 'notes',
      label: 'Notes',
      placeholder: 'Optional notes...',
      rows: 3
    }),
    sigma.ui.separator(),
    sigma.ui.text('Additional information here')
  ],
  buttons: [
    { id: 'cancel', label: 'Cancel', variant: 'secondary' },
    { id: 'submit', label: 'Download', variant: 'primary' }
  ]
});

modal.onSubmit((values, buttonId) => {
  if (buttonId === 'submit') {
    console.log('URL:', values.url);
    console.log('Quality:', values.quality);
    console.log('Notify:', values.notify);
  }
});

modal.onClose(() => {
  console.log('Modal closed');
});

// Programmatic control
modal.updateElement('url', { disabled: true });
const currentValues = modal.getValues();
modal.close();
```

---

### `sigma.dialog` - Native Dialogs

Open native file/folder picker dialogs.

```javascript
// Open file dialog
const filePath = await sigma.dialog.openFile({
  title: 'Select a file',
  defaultPath: '/home/user',
  filters: [
    { name: 'Images', extensions: ['png', 'jpg', 'gif'] },
    { name: 'All Files', extensions: ['*'] }
  ],
  multiple: false,    // Allow multiple selection
  directory: false    // Select directories instead of files
});
// Returns: string | string[] | null

// Save file dialog
const savePath = await sigma.dialog.saveFile({
  title: 'Save as',
  defaultPath: '/home/user/document.txt',
  filters: [
    { name: 'Text Files', extensions: ['txt'] }
  ]
});
// Returns: string | null
```

---

### `sigma.fs` - File System

Read and write files in user-approved directories.

> **Security:** Extensions can only access directories the user has explicitly approved via the scoped directories system.

```javascript
// Read file as bytes
const data = await sigma.fs.readFile('/path/to/file.txt');
const text = new TextDecoder().decode(data);

// Write file
const content = new TextEncoder().encode('Hello, World!');
await sigma.fs.writeFile('/path/to/output.txt', content);

// List directory contents
const entries = await sigma.fs.readDir('/path/to/directory');
// Returns: DirEntry[]
// Each entry: { path, name, isDirectory, size?, modifiedAt? }

// Check if path exists
const exists = await sigma.fs.exists('/path/to/check');

// Download a file from URL
await sigma.fs.downloadFile(
  'https://example.com/file.zip',
  '/path/to/save/file.zip'
);
```

---

### `sigma.shell` - Shell Commands

Execute binaries and shell commands.

> **Requires:** `shell` permission in manifest

```javascript
// Run a command and wait for result
const result = await sigma.shell.run('/path/to/binary', ['arg1', 'arg2']);
console.log('Exit code:', result.code);
console.log('Output:', result.stdout);
console.log('Errors:', result.stderr);

// Run with streaming progress
const { taskId, result, cancel } = await sigma.shell.runWithProgress(
  '/path/to/binary',
  ['--verbose'],
  (payload) => {
    console.log(payload.line);       // Output line
    console.log(payload.isStderr);   // Is this stderr?
    console.log(payload.taskId);     // Task identifier
  }
);

// Cancel the running process
await cancel();

// Wait for completion
const finalResult = await result;
```

---

### `sigma.binary` - Binary Management

Manage external binaries your extension depends on.

```javascript
// Ensure a binary is installed (downloads if missing)
const binaryPath = await sigma.binary.ensureInstalled('my-tool', {
  name: 'my-tool',
  downloadUrl: (platform) => {
    // platform: 'windows' | 'macos' | 'linux'
    if (platform === 'windows') {
      return 'https://example.com/my-tool.exe';
    }
    return 'https://example.com/my-tool';
  },
  executable: 'my-tool',  // Executable name (optional)
  version: '1.0.0'        // Version string (optional)
});

// Check if binary is installed
const isInstalled = await sigma.binary.isInstalled('my-tool');

// Get binary path
const path = await sigma.binary.getPath('my-tool');

// Get binary info
const info = await sigma.binary.getInfo('my-tool');
// Returns: { id, path, version?, installedAt }

// Remove binary
await sigma.binary.remove('my-tool');
```

---

### `sigma.settings` - Extension Settings

Access user-configurable settings defined in your manifest.

First, define settings in `manifest.json`:

```json
{
  "contributes": {
    "configuration": {
      "title": "My Extension Settings",
      "properties": {
        "apiKey": {
          "type": "string",
          "default": "",
          "description": "Your API key"
        },
        "maxItems": {
          "type": "number",
          "default": 10,
          "minimum": 1,
          "maximum": 100,
          "description": "Maximum items to process"
        },
        "enableFeature": {
          "type": "boolean",
          "default": true,
          "description": "Enable the feature"
        },
        "theme": {
          "type": "string",
          "default": "auto",
          "enum": ["auto", "light", "dark"],
          "enumDescriptions": ["Follow system", "Light mode", "Dark mode"],
          "description": "Color theme"
        }
      }
    }
  }
}
```

Then access them in your code:

```javascript
// Get a setting value
const apiKey = await sigma.settings.get('apiKey');
const maxItems = await sigma.settings.get('maxItems');

// Set a setting value
await sigma.settings.set('maxItems', 50);

// Get all settings
const all = await sigma.settings.getAll();

// Reset to default
await sigma.settings.reset('maxItems');

// React to setting changes
sigma.settings.onChange('enableFeature', (newValue, oldValue) => {
  console.log(`Changed from ${oldValue} to ${newValue}`);
});
```

---

### `sigma.storage` - Extension Storage

Store arbitrary data that persists across sessions.

```javascript
// Store a value
await sigma.storage.set('lastRun', Date.now());
await sigma.storage.set('cache', { items: [...] });

// Retrieve a value
const lastRun = await sigma.storage.get('lastRun');
const cache = await sigma.storage.get('cache');

// Remove a value
await sigma.storage.remove('cache');
```

---

### `sigma.platform` - Platform Information

Detect the current operating system and architecture.

```javascript
// Properties (all read-only)
sigma.platform.os          // 'windows' | 'macos' | 'linux'
sigma.platform.arch        // 'x64' | 'arm64' | 'x86'
sigma.platform.pathSeparator  // '\\' on Windows, '/' elsewhere
sigma.platform.isWindows   // boolean
sigma.platform.isMacos     // boolean
sigma.platform.isLinux     // boolean

// Utility method
const fullPath = sigma.platform.joinPath('folder', 'subfolder', 'file.txt');
// Windows: 'folder\\subfolder\\file.txt'
// Others:  'folder/subfolder/file.txt'
```

---

## Manifest Reference

### Required Fields

```json
{
  "$schema": "https://raw.githubusercontent.com/aleksey-hoffman/sigma-file-manager/v2/src/modules/extensions/sdk/manifest.schema.json",
  "id": "publisher.extension-name",
  "name": "Extension Name",
  "version": "1.0.0",
  "repository": "https://github.com/publisher/extension-name",
  "license": "MIT",
  "type": "api",
  "main": "index.js",
  "permissions": [],
  "engines": {
    "sigmaFileManager": ">=2.0.0"
  }
}
```

### All Fields

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | ✅ | Unique ID: `publisher.extension-name` |
| `name` | string | ✅ | Display name |
| `version` | string | ✅ | Semantic version (e.g., `1.0.0`) |
| `repository` | string | ✅ | GitHub repository URL |
| `license` | string | ✅ | SPDX license identifier |
| `type` | string | ✅ | `api`, `iframe`, or `webview` |
| `main` | string | ✅ | Entry point file |
| `permissions` | array | ✅ | Required permissions |
| `engines` | object | ✅ | Version constraints |
| `previousName` | string | | Previous name (for rename support) |
| `author` | object | | `{ name, url? }` |
| `icon` | string | | Path to icon file |
| `banner` | string | | Path to banner image |
| `categories` | array | | Category strings |
| `tags` | array | | Search keywords |
| `activationEvents` | array | | When to activate |
| `contributes` | object | | Static contributions |

### Permissions

| Permission | Description | API Access |
|------------|-------------|------------|
| `contextMenu` | Add context menu items | `sigma.contextMenu` |
| `commands` | Register commands | `sigma.commands` |
| `sidebar` | Add sidebar pages | `sigma.sidebar` |
| `toolbar` | Add toolbar dropdowns | `sigma.toolbar` |
| `notifications` | Show notifications | `sigma.ui.showNotification` |
| `dialogs` | Show dialogs | `sigma.ui.showDialog`, `sigma.dialog` |
| `fs.read` | Read files | `sigma.fs.readFile`, `readDir`, `exists` |
| `fs.write` | Write files | `sigma.fs.writeFile`, `downloadFile` |
| `shell` | Execute commands | `sigma.shell`, `sigma.binary` |

### Extension Types

| Type | Description | Entry Point |
|------|-------------|-------------|
| `api` | JavaScript with full API access | `index.js` |
| `iframe` | Sandboxed iframe (postMessage) | `index.html` |
| `webview` | Separate Tauri window | `index.html` |

---

## Submitting to Registry

### Prerequisites

1. Public GitHub repository
2. Valid `manifest.json` in repository root
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
     "author": "Your Name",
     "authorUrl": "https://github.com/your-username",
     "repository": "https://github.com/your-username/your-extension",
     "featured": false,
     "categories": ["Productivity"],
     "tags": ["keyword1", "keyword2"]
   }
   ```

3. **Submit a Pull Request**

### Submission Guidelines

- ID format: `publisher.extension-name` (lowercase, hyphens allowed)
- Repository must be publicly accessible
- No malicious code or unauthorized data collection
- Provide useful, working functionality
- Include clear documentation

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

- [Example Extension](https://github.com/sigma-hub/sfm-extension-example) - Full-featured demo
- [Video Downloader](https://github.com/sigma-hub/sfm-extension-video-downloader) - Real-world example
- [SDK Types](https://raw.githubusercontent.com/aleksey-hoffman/sigma-file-manager/v2/src/modules/extensions/sdk/sigma-extension.d.ts) - TypeScript definitions
- [Manifest Schema](https://raw.githubusercontent.com/aleksey-hoffman/sigma-file-manager/v2/src/modules/extensions/sdk/manifest.schema.json) - JSON Schema
- [Lucide Icons](https://lucide.dev/icons/) - Icon reference

---

## License

MIT License - see [LICENSE](LICENSE) for details.
