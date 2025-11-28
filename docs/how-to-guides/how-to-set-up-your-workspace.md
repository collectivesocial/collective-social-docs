# How to Set Up Your VS Code Workspace

**Goal:** Configure your development environment to work efficiently across all Collective Social repositories

**Difficulty:** Beginner

**Prerequisites:**
- Visual Studio Code installed
- Git installed and configured
- Basic familiarity with VS Code

## Overview

The Collective Social framework consists of three repositories that work together:
- **collective-social-api** - The backend API server
- **collective-social-web** - The frontend React application
- **collective-social-docs** - Documentation (this repo)

This guide will help you set up a multi-root workspace in VS Code that gives you access to all three repositories simultaneously.

## Steps

### 1. Clone All Three Repositories

First, clone the repositories into the same parent directory:

```bash
cd ~/Documents/Code  # or your preferred development directory

# Clone the repositories
git clone https://github.com/collectivesocial/collective-social-api.git
git clone https://github.com/collectivesocial/collective-social-web.git
git clone https://github.com/collectivesocial/collective-social-docs.git
```

Your directory structure should look like:
```
Code/
├── collective-social-api/
├── collective-social-web/
└── collective-social-docs/
```

### 2. Open the Workspace File

The `collective-social.code-workspace` file is located in the `collective-social-docs` repository and provides a pre-configured workspace for all three projects.

**Option A: From VS Code**
1. Open VS Code
2. Go to `File → Open Workspace from File...`
3. Navigate to `collective-social-docs/collective-social.code-workspace`
4. Click "Open"

**Option B: From Command Line**
```bash
cd collective-social-docs
code collective-social.code-workspace
```

### 3. Verify Your Workspace Setup

Once opened, you should see three folders in the Explorer sidebar:
- 📚 **Docs** - Documentation repository
- 🔧 **API** - Backend API server
- ⚛️ **Web** - Frontend React app

Each folder can be expanded independently, and you can work across all three repos without switching windows.

### 4. Install Dependencies

Open an integrated terminal for each project and install dependencies:

**For the API:**
```bash
# In terminal, navigate to API folder or use terminal dropdown to select it
cd collective-social-api  # if not already there
npm install
```

**For the Web app:**
```bash
cd collective-social-web
npm install
```

### 5. Configure Environment Variables

**API Configuration:**
1. In the API folder, copy `.env.example` to `.env`:
   ```bash
   cp .env.example .env
   ```
2. Edit `.env` and update the values:
   - `DATABASE_URL` - Your PostgreSQL connection string
   - `COOKIE_SECRET` - Generate a random 32+ character string
   - Leave `SERVICE_URL` commented out for local development

**Web Configuration:**
The web app uses Vite's default configuration with `127.0.0.1:5173`, which is already set up.

### 6. Start Development Servers

With the workspace open, you can run multiple terminals simultaneously:

**Terminal 1 - API Server:**
```bash
cd collective-social-api
npm run docker:up    # Start PostgreSQL
npm run dev          # Start the API server
```

**Terminal 2 - Web App:**
```bash
cd collective-social-web
npm run dev          # Start the React dev server
```

The API will run on `http://127.0.0.1:3000` and the web app on `http://127.0.0.1:5173`.

## Workspace Features

### Multi-Root Benefits

- **Unified Search**: Search across all three repositories at once (Ctrl+Shift+F / Cmd+Shift+F)
- **Shared Settings**: Prettier formatting and linting configured for all projects
- **Quick Navigation**: Jump between repos using the folder tree or Quick Open (Ctrl+P / Cmd+P)
- **Integrated Git**: Manage all three repositories from the Source Control panel

### Recommended Extensions

The workspace includes recommendations for:
- **Prettier** - Code formatting
- **ESLint** - JavaScript/TypeScript linting
- **TypeScript** - Enhanced TypeScript support

VS Code will prompt you to install these when you open the workspace.

### Terminal Management

**Creating terminals for specific folders:**
1. Click the dropdown in the terminal panel
2. Select the "+" icon next to a folder name
3. This creates a terminal with that folder as the working directory

**Tip:** Name your terminals by right-clicking on them:
- "API Dev"
- "Web Dev"  
- "Database"

## Troubleshooting

### Workspace file shows incorrect paths

If you cloned the repositories to a different location, update the workspace file paths:

1. Open `collective-social.code-workspace` in a text editor
2. Update the `path` values in the `folders` array to match your directory structure
3. Paths can be absolute or relative to the workspace file location

Example for Windows:
```json
{
  "name": "🔧 API",
  "path": "C:/dev/collective-social-api"
}
```

Example for relative paths:
```json
{
  "name": "🔧 API",
  "path": "../collective-social-api"
}
```

### Extensions not activating

If Prettier or other extensions aren't working:
1. Open the Extensions panel (Ctrl+Shift+X / Cmd+Shift+X)
2. Search for "Prettier"
3. Click "Install"
4. Reload the window if prompted

### Git showing wrong repository

Make sure you select the correct folder in the Source Control panel dropdown when making commits. Each repository is tracked independently.

## Alternative: Individual Folders

If you prefer not to use a workspace file, you can open each repository individually:

```bash
code collective-social-api
code collective-social-web
code collective-social-docs
```

This creates separate VS Code windows for each project, which may be preferable on multi-monitor setups.

## See Also

- [API Configuration Reference](../reference/api-configuration.md)
- [Getting Started Tutorial](../tutorials/getting-started.md)
- [Understanding the Architecture](../explanation/architecture-overview.md)

---

[← Back to How-to Guides](./README.md)
