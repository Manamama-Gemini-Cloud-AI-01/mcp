### Report: Git Push Permission Error and Resolution

#### Executive Summary
The user encountered a `git push` error: `ERROR: Permission to Manamama-Gemini-Cloud-AI-01/mcp.git denied to 6Y3GRwDjtGVo4nAe2`, indicating that SSH authentication was using a key tied to the `6Y3GRwDjtGVo4nAe2` GitHub account, which lacked write access to the `Manamama-Gemini-Cloud-AI-01/mcp.git` repository. A tactical one-line fix (`git remote set-url origin https://github.com/Manamama-Gemini-Cloud-AI-01/mcp.git`) resolved the issue by switching to HTTPS, leveraging the correct `Manamama-Gemini-Cloud-AI-01` token. This report details the root cause, why the tactical fix worked, and how to permanently resolve the SSH issue.

#### What Was Wrong
The error stemmed from an SSH key mismatch:
- **SSH Key Issue**: The `git push origin main` command used SSH (`git@github.com:Manamama-Gemini-Cloud-AI-01/mcp.git`), and the SSH agent offered the key `/home/zezen/.ssh/codespaces.auto` (ED25519, SHA256:wH4VDGSVKK66/YtuS4BAIsZeA6tNYzoZLOu+HeTtHHo), which was tied to the `6Y3GRwDjtGVo4nAe2` GitHub account. This account lacked write permissions for the `Manamama-Gemini-Cloud-AI-01/mcp.git` repository, causing the permission error.
- **Key Source**: The `codespaces.auto` key was likely created by GitHub Codespaces and associated with `6Y3GRwDjtGVo4nAe2` during a previous authentication setup (e.g., via `gh auth login` or Codespaces configuration).
- **Multiple Accounts**: The user had three GitHub accounts configured in the GitHub CLI (`gh`): `Manamama-Gemini-Cloud-AI-01` (active, with `repo` scope), `Manamama`, and `6Y3GRwDjtGVo4nAe2`. The SSH agent ignored the active `gh` account and used the `codespaces.auto` key, leading to authentication as the wrong account.
- **Missing Key**: No SSH key in `~/.ssh` (e.g., `codespaces.auto`, `google_compute_engine`, `Mutatis_Mutandis.pem`) was explicitly tied to `Manamama-Gemini-Cloud-AI-01`, preventing correct SSH authentication.
- **Previous Functionality**: The user noted that SSH pushes “used to work,” suggesting a prior key for `Manamama-Gemini-Cloud-AI-01` may have been removed, replaced, or overridden by the `codespaces.auto` key.

Additional factors:
- The `~/.ssh` directory contained other keys (`google_compute_engine`, RSA, likely from Google Cloud SDK; `Mutatis_Mutandis.pem`, possibly from AWS), but only `codespaces.auto` was offered during SSH authentication.
- Attempts to use `gh auth login` with SSH failed due to an `HTTP 422: key is already in use` error, indicating `codespaces.auto` was already registered to `6Y3GRwDjtGVo4nAe2`.

#### Tactical Fix: Why One Line Worked
The command:
```bash
git remote set-url origin https://github.com/Manamama-Gemini-Cloud-AI-01/mcp.git
```
resolved the issue by changing the repository’s remote URL from SSH (`git@github.com:Manamama-Gemini-Cloud-AI-01/mcp.git`) to HTTPS (`https://github.com/Manamama-Gemini-Cloud-AI-01/mcp.git`). This:
- Bypassed SSH authentication, avoiding the `codespaces.auto` key tied to `6Y3GRwDjtGVo4nAe2`.
- Used the GitHub CLI’s stored token (`gho_...`) for `Manamama-Gemini-Cloud-AI-01`, which has `repo` scope, allowing write access to the repository.
- Leveraged Git’s credential helper (likely configured by `gh auth login`), which automatically provided the correct token, making the push seamless without manual credential input.

The push succeeded:
```
To https://github.com/Manamama-Gemini-Cloud-AI-01/mcp.git
   9db12f2..3c1e4eb  main -> main
```

#### Permanent Fix: Restore SSH Authentication
To restore SSH functionality and avoid reliance on HTTPS, the following steps address the root cause by ensuring the correct key is used for `Manamama-Gemini-Cloud-AI-01`.

1. **Generate a New SSH Key for `Manamama-Gemini-Cloud-AI-01`**:
   Since no existing key in `~/.ssh` is tied to `Manamama-Gemini-Cloud-AI-01`, create a new one:
   ```bash
   ssh-keygen -t ed25519 -C "email_for_manamama@example.com" -f ~/.ssh/id_ed25519_manamama
   ```
   - Replace `email_for_manamama@example.com` with the email associated with `Manamama-Gemini-Cloud-AI-01`.
   - This creates `~/.ssh/id_ed25519_manamama` (private) and `~/.ssh/id_ed25519_manamama.pub` (public).

2. **Add Key to GitHub**:
   Copy the public key:
   ```bash
   cat ~/.ssh/id_ed25519_manamama.pub
   ```
   Log in to GitHub as `Manamama-Gemini-Cloud-AI-01`, go to **Settings > SSH and GPG keys > New SSH key**, paste the key, title it (e.g., “Manamama Key”), and save.

3. **Configure SSH to Use the Correct Key**:
   Edit the SSH configuration:
   ```bash
   sudo nano ~/.ssh/config
   ```
   Add:
   ```
   Host github.com
       HostName github.com
       User git
       IdentityFile ~/.ssh/id_ed25519_manamama
       IdentitiesOnly yes
   ```
   - `IdentitiesOnly yes` ensures only the specified key is used, preventing `codespaces.auto` from being offered.
   Set permissions:
   ```bash
   sudo chmod 600 ~/.ssh/config
   ```

4. **Manage SSH Agent**:
   Clear existing keys:
   ```bash
   ssh-add -D
   ```
   Add the new key:
   ```bash
   ssh-add ~/.ssh/id_ed25519_manamama
   ```
   Verify:
   ```bash
   ssh-add -l
   ```
   Ensure only the new key’s fingerprint appears.

5. **Test SSH Authentication**:
   Run:
   ```bash
   ssh -T git@github.com
   ```
   Confirm it outputs:
   ```
   Hi Manamama-Gemini-Cloud-AI-01! You've successfully authenticated...
   ```
   If it shows `6Y3GRwDjtGVo4nAe2`, check with:
   ```bash
   ssh -vT git@github.com 2>&1 | grep "Offering public key"
   ```

6. **Switch Remote to SSH and Push**:
   Set the remote back to SSH:
   ```bash
   git remote set-url origin git@github.com:Manamama-Gemini-Cloud-AI-01/mcp.git
   ```
   Verify:
   ```bash
   git remote -v
   ```
   Push:
   ```bash
   git push origin main
   ```

7. **Optional: Remove or Reassign `codespaces.auto`**:
   To prevent future issues with `codespaces.auto`:
   - Log in to GitHub as `6Y3GRwDjtGVo4nAe2`, go to **Settings > SSH and GPG keys**, and delete the `codespaces.auto` key if it’s not needed.
   - Alternatively, move it out of `~/.ssh`:
     ```bash
     sudo mv ~/.ssh/codespaces.auto ~/.ssh/codespaces.auto.bak
     sudo mv ~/.ssh/codespaces.auto.pub ~/.ssh/codespaces.auto.pub.bak
     ```

#### Verification
- Confirm repository access: Ensure `Manamama-Gemini-Cloud-AI-01` has write or admin access to `https://github.com/Manamama-Gemini-Cloud-AI-01/mcp` (Settings > Collaborators and teams).
- If SSH fails, revert to HTTPS:
  ```bash
  git remote set-url origin https://github.com/Manamama-Gemini-Cloud-AI-01/mcp.git
  git push origin main
  ```

#### Conclusion
The permission error occurred because the `codespaces.auto` key, likely created by GitHub Codespaces, was tied to `6Y3GRwDjtGVo4nAe2`, which lacked access to the target repository. The tactical HTTPS fix worked by using the `Manamama-Gemini-Cloud-AI-01` token. The permanent SSH fix involves creating a new key, configuring `~/.ssh/config`, and ensuring only the correct key is used. This restores the user’s preferred SSH workflow, matching the previous functionality.

**Date**: August 2, 2025, 09:10 PM CEST
