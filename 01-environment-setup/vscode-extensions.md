# VS Code Extensions for Salesforce

**The extensions that make Salesforce development viable in VS Code.**

---

## Extension list

| Extension | Publisher | What it gives you | Required? |
|---|---|---|---|
| Salesforce Extension Pack | Salesforce | Apex syntax highlighting, code completion, LWC IntelliSense, org operations from the command palette, push/pull source without leaving the editor | Required |
| Prettier - Code Formatter | Prettier | Consistent formatting for Apex, LWC HTML, JavaScript, and JSON. Runs on save so your files are always clean before committing. | Required |
| ESLint | Microsoft | Static analysis for LWC JavaScript. Catches undefined variables, unused imports, and Lightning-specific anti-patterns before runtime. | Required |
| Apex PMD | chuckjonas | Runs PMD static analysis rules on Apex files as you type. Catches SOQL in loops, empty catch blocks, and other issues inline without a separate build step. | Recommended |
| Salesforce Package XML Generator | VignaeshMurugan | Generate a `package.xml` manifest by selecting metadata types in a UI rather than editing XML by hand. | Recommended |
| GitLens | GitKraken | Inline git blame, file history, and branch comparison. Useful for understanding who changed a trigger handler and when. | Recommended |
| Markdown Preview Mermaid | Matt Bierner | Renders Mermaid diagram code blocks in VS Code's Markdown preview pane. Useful for viewing architecture diagrams in this knowledge hub. | Recommended |

---

## Install the Salesforce Extension Pack

The Extension Pack installs multiple Salesforce extensions as a group. Search for it by the exact name in the VS Code marketplace:

1. Open VS Code.
2. Press `Ctrl+Shift+X` (Windows/Linux) or `Cmd+Shift+X` (macOS) to open the Extensions panel.
3. Search for `Salesforce Extension Pack`.
4. Click **Install** on the result published by **Salesforce**.

This installs Apex support, LWC support, SOQL editor, and org operations as a bundle.

---

## Recommended settings.json

Add these settings to your VS Code workspace or user settings (`Ctrl+Shift+P` > `Open User Settings (JSON)`):

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "[apex]": {
    "editor.defaultFormatter": "salesforce.salesforcedx-vscode-apex"
  },
  "[html]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "salesforcedx-vscode-apex.java.home": "/path/to/your/jdk",
  "eslint.validate": ["javascript", "html"],
  "editor.rulers": [120],
  "files.trimTrailingWhitespace": true
}
```

Replace `/path/to/your/jdk` with the actual path to your JDK installation. On Windows this is typically `C:\\Program Files\\Eclipse Adoptium\\jdk-11.x.x`. On macOS, run `java_home` in the terminal to find it.

---

## Prettier configuration for Apex

Prettier needs a config file to format Apex correctly. Create `.prettierrc` in your project root:

```json
{
  "trailingComma": "none",
  "printWidth": 120,
  "tabWidth": 4,
  "singleQuote": false,
  "apexInsertFinalNewline": true
}
```

And a `.prettierignore` to skip generated files:

```
.localdevserver
.sf
.sfdx
node_modules
```

---

## ESLint configuration for LWC

The Salesforce Extension Pack creates a default ESLint config. If it's missing, create `.eslintrc.json` in your project root:

```json
{
  "extends": ["@salesforce/eslint-config-lwc/recommended"],
  "rules": {
    "no-console": "warn",
    "no-unused-vars": "error"
  }
}
```

Install the ESLint config package:

```bash
npm install --save-dev @salesforce/eslint-config-lwc
```

---

## **Anti-Pattern**: skipping the extensions and writing Apex in a plain text editor

Without the Salesforce Extension Pack:

- No Apex IntelliSense -- you type field and method names from memory
- No inline error detection -- syntax errors only appear after a failed deploy
- No LWC component completion -- HTML attribute names are guessed, not suggested
- No org operations -- every push and pull requires a terminal command

You can technically write Salesforce code in Notepad. You'll catch every mistake at deploy time instead of as you type. For anything beyond a one-line script, install the Extension Pack.
