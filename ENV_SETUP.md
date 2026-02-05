# OpenCode Configuration Security Setup

## Overview

This document explains how to securely configure API keys in your OpenCode installation using environment variables. This prevents sensitive API keys from being accidentally committed to GitHub.

## What Was Changed

1. ✅ **Updated `opencode.json`** - Replaced hardcoded API key with `{env:VARIABLE_NAME}` syntax
2. ✅ **Updated `.gitignore`** - Added `.env` files to prevent accidental commits
3. ✅ **Created `.env.example`** - Template showing required environment variables

## Setup Instructions

### Step 1: Copy the Environment Template

Create your personal `.env` file by copying the template:

```bash
cp .env.example .env
```

### Step 2: Add Your API Keys

Edit the `.env` file and replace the placeholder values with your actual API keys:

```bash
# Using your favorite editor
nano .env
# or
vim .env
# or
code .env
```

Update the file with your actual API keys:

```env
# Context7 MCP Server API Key
CONTEXT7_API_KEY=ctx7sk-af98b8f2-4b9f-40e6-a6af-8369db18ff03
```

**Note:** If you lost your original API key, you can retrieve it from:
- Context7: https://context7.com/

### Step 3: Verify the Configuration

OpenCode will automatically load environment variables from the `.env` file. To verify it's working:

1. Restart OpenCode if it's currently running
2. The MCP services should connect successfully
3. No API keys should appear in your `opencode.json` file

### Step 4: Verify Git Security

Make sure your `.env` file is not tracked by Git:

```bash
git status
```

You should NOT see `.env` in the list of tracked files. If you do, remove it from tracking:

```bash
git rm --cached .env
git commit -m "Remove .env from tracking"
```

## Adding New API Keys

When you need to add new API keys in the future:

1. **For headers in MCP configurations:**
   ```json
   "headers": {
     "NEW_API_KEY": "{env:NEW_API_KEY}"
   }
   ```

2. **For provider options:**
   ```json
   "provider": {
     "anthropic": {
       "options": {
         "apiKey": "{env:ANTHROPIC_API_KEY}"
       }
     }
   }
   ```

3. **Update `.env.example`** to document the new variable:
   ```env
   # New Service API Key
   NEW_API_KEY=your_new_api_key_here
   ```

4. **Update your `.env` file** with the actual value (do not commit this file)

## Security Best Practices

1. **Never commit `.env` files** - They are now excluded in `.gitignore`
2. **Share `.env.example`** - This template helps team members know what keys they need
3. **Rotate compromised keys** - If a key was accidentally committed, revoke it immediately and generate a new one
4. **Use different keys for different environments** - Dev, staging, and production should use separate API keys

## Troubleshooting

### MCP Services Not Connecting

If MCP services fail to connect after these changes:

1. Verify the `.env` file exists in the same directory as `opencode.json`
2. Check that the environment variable names match exactly (case-sensitive)
3. Ensure there are no quotes around the values in the `.env` file
4. Restart OpenCode completely

### Environment Variables Not Loading

OpenCode uses the `{env:VARIABLE_NAME}` syntax. Make sure:
- The syntax is exactly `{env:VAR_NAME}` (including the curly braces)
- The variable name matches what's in your `.env` file
- There are no extra spaces in the syntax

## Important Reminder

⚠️ **The old hardcoded API key that was in `opencode.json` has been exposed in your Git history.** If this was a production key:

1. **Revoke the exposed key immediately** at the provider's dashboard
2. **Generate a new key** 
3. **Update your `.env` file** with the new key
4. **Consider rotating other keys** if they were in the same repository

---

For more information about OpenCode configuration, visit: https://opencode.ai/
