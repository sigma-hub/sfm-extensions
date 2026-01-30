# Sigma File Manager Extensions Registry

This repository contains the official registry of extensions for [Sigma File Manager](https://github.com/aleksey-hoffman/sigma-file-manager).

## Browsing Extensions

Extensions can be browsed and installed directly from the Sigma File Manager app via the **Extensions** page in the sidebar.

## For Extension Developers

### Creating an Extension

1. **Create a new repository** for your extension

2. **Download the SDK types** from the main repo:
   ```bash
   curl -O https://raw.githubusercontent.com/aleksey-hoffman/sigma-file-manager/v2/src/modules/extensions/sdk/sigma-extension.d.ts
   ```

3. **Create a `manifest.json`** in your repository root:
   ```json
   {
     "$schema": "https://raw.githubusercontent.com/aleksey-hoffman/sigma-file-manager/v2/src/modules/extensions/sdk/manifest.schema.json",
     "id": "your-username.extension-name",
     "name": "My Extension",
     "version": "1.0.0",
     "description": "What your extension does",
     "author": {
       "name": "Your Name",
       "url": "https://github.com/your-username"
     },
     "repository": "https://github.com/your-username/your-extension",
     "license": "MIT",
     "tags": ["utility"],
     "type": "api",
     "main": "index.js",
     "permissions": ["contextMenu", "notifications"],
     "engines": {
       "sigmaFileManager": ">=2.0.0"
     }
   }
   ```

4. **Implement your extension** in the main entry file (e.g., `index.js`)

5. **Tag a release** with semantic versioning (e.g., `v1.0.0`)

### Submitting Your Extension

1. Fork this repository

2. Add your extension entry to `registry.json`:
   ```json
   {
     "id": "your-username.extension-name",
     "repository": "https://github.com/your-username/your-extension",
     "official": false,
     "verified": false,
     "featured": false,
     "versions": ["1.0.0"],
     "latestVersion": "1.0.0"
   }
   ```

3. Submit a Pull Request

### Submission Guidelines

- Extension ID must be in format `publisher.extension-name` (lowercase, hyphens allowed)
- Repository must be publicly accessible on GitHub
- Must have a valid `manifest.json` in the repository root
- Must have at least one tagged release
- Extension should provide useful functionality
- No malicious code or data collection without user consent
- Include a README with usage instructions

### Extension Types

| Type | Description |
|------|-------------|
| `api` | JavaScript extension with direct API access |
| `iframe` | Sandboxed iframe with postMessage communication |
| `webview` | Separate Tauri webview window |

### Permissions

| Permission | Description |
|------------|-------------|
| `contextMenu` | Add items to file browser context menu |
| `sidebar` | Add pages to sidebar navigation |
| `toolbar` | Add dropdown menus to window toolbar |
| `commands` | Register executable commands |
| `fs.read` | Read files in user-approved directories |
| `fs.write` | Write files in user-approved directories |
| `notifications` | Show notification messages |
| `dialogs` | Show dialog windows |

## Badge System

| Badge | Meaning |
|-------|---------|
| **Official** | Created by the Sigma File Manager team |
| **Verified** | Reviewed and approved by maintainers |
| **Unverified** | Third-party extension, use at your own risk |

## Registry Schema

### registry.json

```json
{
  "schemaVersion": "1.0.0",
  "extensions": [
    {
      "id": "publisher.extension-name",
      "name": "My Extension",
      "description": "What the extension does",
      "author": "Author Name",
      "authorUrl": "https://github.com/author-username",
      "repository": "https://github.com/user/repo",
      "official": false,
      "verified": false,
      "featured": false,
      "categories": ["Utility", "Productivity"],
      "versions": ["1.0.0", "1.1.0"],
      "latestVersion": "1.1.0"
    }
  ]
}
```

### featured.json

```json
{
  "schemaVersion": "1.0.0",
  "featured": ["publisher.extension-name"]
}
```

## License

MIT License - see [LICENSE](LICENSE) for details.
