# Secrets and Security

Keep your API keys, tokens, and credentials safe when using AI terminal tools.

> **The core problem:** Every AI tool in this guide (Claude Code, Gemini CLI, Codex, opencode) runs with your user's full permissions. They can read `.env` files, access your keychain, and run any command you can.

## Platform Compatibility

| Technique | macOS | Linux | Windows |
|-----------|-------|-------|---------|
| envchain (Keychain secrets) | Native | Use `pass` instead | Use Credential Manager |
| sandbox-exec (process isolation) | Native (deprecated but works) | Use `firejail` / `bubblewrap` | Use Windows Sandbox |
| gitleaks (pre-commit scanning) | Yes | Yes | Yes |
| Claude Code hooks | Yes | Yes | No (WSL only) |
| .gitignore / .env patterns | Yes | Yes | Yes |

---

## 1. The envchain Pattern (macOS)

Store secrets in macOS Keychain instead of `.env` files:

```bash
brew install envchain

# Store secrets (prompts for value)
envchain -s myproject API_KEY
envchain -s myproject DATABASE_URL

# Run a command WITH secrets
envchain myproject npm run dev

# Start a shell WITH secrets
envchain myproject zsh
```

**The key insight:** Start your AI tool in a SEPARATE terminal WITHOUT envchain:

```bash
# Terminal 1 — AI tool (no secrets in environment)
claude        # or gemini, codex, opencode

# Terminal 2 — development (with secrets)
envchain myproject zsh
npm run dev
```

The AI tool literally cannot access secrets that aren't in its environment.

### Linux Alternative: pass

```bash
# Install
sudo apt install pass
gpg --gen-key
pass init "your-gpg-id"

# Store
pass insert myproject/API_KEY

# Use in a command
API_KEY=$(pass myproject/API_KEY) npm run dev
```

---

## 2. sandbox-exec (macOS)

Go further by sandboxing the AI tool's process:

```bash
# Create sandbox profile: ~/claude-sandbox.sb
cat > ~/claude-sandbox.sb << 'EOF'
(version 1)
(allow default)

;; Block Keychain access
(deny file-read* file-write*
  (subpath "/Users/YOU/Library/Keychains")
  (subpath "/Library/Keychains"))

;; Block security command
(deny mach-lookup
  (global-name "com.apple.securityd")
  (global-name "com.apple.SecurityServer"))

;; Block envchain binary
(deny process-exec
  (literal "/opt/homebrew/bin/envchain"))

;; Block common secret files
(deny file-read*
  (regex #"\.env($|\..*)$")
  (subpath "/Users/YOU/.ssh")
  (subpath "/Users/YOU/.aws"))
EOF

# Create alias
echo 'alias claude-safe="sandbox-exec -f ~/claude-sandbox.sb claude"' >> ~/.zshrc
source ~/.zshrc

# Use it
claude-safe   # sandboxed — can't read Keychain, .env, .ssh
```

> **Note:** `sandbox-exec` is deprecated since macOS 10.15 but still works through macOS Sequoia (2026). Monitor on OS upgrades. Linux users can achieve similar results with `firejail`.

---

## 3. Claude Code Hooks

Claude Code (see [04-claude-code.md](04-claude-code.md)) supports hooks that run before and after tool use:

**PreToolUse hook** — blocks reading secret files:
```bash
#!/bin/bash
# .claude/hooks/block-secrets.sh
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // .tool_input.path // ""')
BASENAME=$(basename "$FILE_PATH")

if [[ "$BASENAME" =~ ^\.env($|\.[a-zA-Z]+$) ]] || \
   [[ "$BASENAME" =~ \.(pem|p8|key)$ ]]; then
    echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny","permissionDecisionReason":"Blocked: sensitive file"}}'
    exit 0
fi
exit 0
```

**PostToolUse hook** — scans output for leaked secrets (JWT tokens, API keys, PEM blocks).

> **Important:** Only Claude Code has hooks. Gemini CLI, Codex, and opencode have no equivalent. For those tools, rely on `.gitignore` and gitleaks.

---

## 4. gitleaks — Universal Safety Net

The ONE security measure that works for every tool on every platform:

```bash
# Install
brew install gitleaks                    # macOS
# Or: go install github.com/gitleaks/gitleaks/v8@latest  # any platform

# Add to .git/hooks/pre-commit:
if command -v gitleaks &> /dev/null; then
    echo "Running gitleaks..."
    gitleaks protect --staged --no-banner || {
        echo "Secrets detected in staged files!"
        exit 1
    }
fi
```

Regardless of which AI tool generated the code, everything passes through git. gitleaks catches it there.

---

## 5. AI Doc Rules That Work

All tools in this guide support context files (see [07-context-files.md](07-context-files.md)). Add security rules to yours:

```markdown
## Secrets (MANDATORY)
- NEVER read .env files, API keys, tokens, or passwords
- NEVER include secrets in output or explanations
- Reference secrets via environment variables only
- You have NO access to secret values
```

**Why this format works:**
- Short imperatives — LLMs follow "NEVER X" better than long explanations
- CAPS for emphasis — signals high priority to the model
- Max 5 rules — more gets ignored
- Same rules in every context file (CLAUDE.md, .gemini, .cursorrules)

---

## 6. The Practical Security Stack

Layer your defenses:

```
Layer 1: envchain / pass       — secrets never in files
Layer 2: sandbox-exec          — AI tool can't access Keychain (macOS)
Layer 3: Claude Code hooks     — blocks reading .env/.key files (Claude only)
Layer 4: AI doc rules          — "NEVER read secrets" in context files
Layer 5: gitleaks pre-commit   — catches secrets before they enter git
Layer 6: .gitignore            — .env, *.pem, *.key excluded
```

---

## 7. The ADHD Rule

> If a security measure requires daily manual effort, it will be abandoned within two weeks.

| Measure | Daily effort | Survives 30 days? |
|---------|-------------|-------------------|
| envchain (set once) | None | Yes |
| gitleaks pre-commit hook | None | Yes |
| .gitignore | None | Yes |
| AI doc rules (write once) | None | Yes |
| Manual log review | Must remember | No |
| File encryption with gpg | Decrypt/encrypt each session | No |
| Separate OS accounts per tool | Switch accounts constantly | No |

The best security is invisible security. Set it up once, forget about it, and it keeps working.

---

**Next:** [FAQ](16-faq.md) | **Previous:** [Troubleshooting](15-troubleshooting.md)
