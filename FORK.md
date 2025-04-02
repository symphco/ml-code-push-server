# Maintaining Your Code-Push-Server Fork

This document outlines the process for keeping your fork of the code-push-server repository in sync with the official Microsoft repository.

## Initial Setup (Do Once)

1. **Fork the repository**

   - Go to https://github.com/microsoft/code-push-server
   - Click the "Fork" button in the top-right corner
   - This creates a copy of the repository under your GitHub account

2. **Clone your fork to your local machine**

   ```
   git clone https://github.com/YOUR-USERNAME/code-push-server.git
   ```

   - This downloads your fork to your computer
   - Replace `YOUR-USERNAME` with your GitHub username

3. **Navigate to the repository directory**

   ```
   cd code-push-server
   ```

4. **Add the original repository as an "upstream" remote**

   ```
   git remote add upstream https://github.com/microsoft/code-push-server.git
   ```

   - `remote add`: Adds a new remote repository reference
   - `upstream`: Common name used for the original repository
   - The URL points to the original Microsoft repository

5. **Verify the remotes are set up correctly**
   ```
   git remote -v
   ```
   - Should show both `origin` (your fork) and `upstream` (original repo)

## Updating Your Fork (Do Periodically)

1. **Fetch all changes from the upstream repository**

   ```
   git fetch upstream
   ```

   - Downloads all changes from the original repository without merging them

2. **Switch to your main branch**

   ```
   git checkout main
   ```

   - Ensures you're on your main branch to receive the updates

3. **Merge the changes from upstream's main branch**

   ```
   git merge upstream/main
   ```

   - Integrates the original repository's changes into your local copy

4. **Push the updated main branch to your fork**
   ```
   git push origin main
   ```
   - Uploads the updated branch to your GitHub fork

## Working with Your Fork

- If you want to make your own changes, create a new branch from the updated main:

  ```
  git checkout -b your-feature-branch
  ```

- After making changes, push your feature branch to GitHub:

  ```
  git push origin your-feature-branch
  ```

- Always update your main branch from upstream before creating new feature branches to ensure you're working with the latest code.
