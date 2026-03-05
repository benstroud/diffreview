# 🔍 Zsh helpers for LLM Git diff review

> **Pipe any Git diff range into Claude Code or GitHub Copilot CLI — with a single command.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Shell: Zsh](https://img.shields.io/badge/Shell-Zsh-blue.svg)](https://www.zsh.org/)
[![Works with: Claude Code](https://img.shields.io/badge/Works%20with-Claude%20Code-blueviolet)](https://code.claude.com)
[![Works with: GitHub Copilot](https://img.shields.io/badge/Works%20with-GitHub%20Copilot-black)](https://github.com/features/copilot/cli)
[![BuyMeACoffee](https://raw.githubusercontent.com/pachadotdev/buymeacoffee-badges/main/bmc-donate-yellow.svg)](https://www.buymeacoffee.com/benstroud)

---

## Usage Examples

Review Git commits easily with Claude Code. Check out the branch you want to
review, then try these examples from the repo root:

```zsh
# Review the last commit of your feature branch
claudiff "HEAD^..HEAD" "code review"
claudiff "HEAD^..HEAD" "generate git commit message"
claudiff "HEAD^..HEAD" "generate pull request description"

# Review the last 3 commits of your feature branch
claudiff "HEAD~3..HEAD" "suggest code cleanups"
claudiff "HEAD~3..HEAD" "produce gherkin acceptance criteria"

# Review working tree changes (uncommitted)
claudiff HEAD "Suggest code cleanups"

# Review staged changes
claudiff --staged "review as if you were Linus Torvalds"

# Review a pull request branch before merging
claudiff "origin/main..HEAD" "review as if you were Anders Hejlsberg"

# Since last tag
claudiff "$(git describe --tags --abbrev=0)..HEAD" "Generate release notes"
```

```diff
+ Or do the same as above with Github Copilot CLI using `copdiff` instead of `claudiff`

- Only use trusted or sanitized prompts
```

## Git Commit Review: General workflow

Several AI coding harnesses now provide special agentic code review modes. These
modes can be effective but they often don’t provide a great deal of control,
delegating to agent tool calls in ways that can be unpredictable or slow. Here’s
a general fine grained workflow for when you want more precise control.

* In your local dev environment, check out the branch you want to review.
* Determine the Git diff range expression for change set that you want to review.
* Now use `git` to produce a diff output, e.g., `git diff HEAD~3..HEAD`. Capture that output in some way, depending on how you like to run Git (CLI, GUI, etc.). Basically isolate the text of the diff you want to review to provide to the LLM.
* Now simply ask the LLM, in its “Ask” mode if available, i.e., tools disabled, to review the diff text, but while it *also has access to the context of the codebase*. This also will be different depending on your preferred tool/workflow. The prompt doesn't need to be very special these days with the latest models, it can be simply "Code review" to get very good results.
* The same pattern of providing the diff text can also be used with other prompts to good effect (see examples below for ideas).

## Zsh code to install

The code below takes the steps above and streamlines into CLI commands by
defining Zsh functions. One named `claudiff` for Claude Code and one named
`copdiff` for GitHub Copilot CLI. These functions take a Git diff range and a
prompt, produce the diff, and feed it to the LLM CLI with tools disabled so that
the model can only "read" the diff text and respond to the prompt.

### `claudiff`: Claude Code CLI zsh function

If you have installed [Claude Code](https://code.claude.com/docs/en/overview), `claude`, add the following shell function to your .zshrc:

```zsh
# Usage:
#   claudiff "HEAD~3..HEAD" "Code Review"
claudiff() {
  local range="$1"; shift
  local prompt="$*"

  if [[ -z "$range" || -z "$prompt" ]]; then
    print -u2 'Usage: claudiff "<git-range>" "<prompt text>"'
    return 2
  fi

  local tmpfile
  tmpfile="$(mktemp -t claudiff.XXXXXX.patch)" || return 1

  {
    git diff "$range" > "$tmpfile" || return 1

    # Pipe diff to Claude; disable all tools so it can only "read" stdin and answer.
    # Update model with what you prefer.
    cat "$tmpfile" | claude -p "$prompt" \
      --tools "" \
      --no-chrome \
      --model opus \
      --disable-slash-commands
  } always {
    rm -f "$tmpfile"
  }
}
```

### `copdiff`: GitHub Copilot CLI zsh function

If you have installed [GitHub Copilot CLI](https://github.com/features/copilot/cli), `copilot`, add the following shell function to your .zshrc:

```zsh
# Usage:
#   copdiff "HEAD~3..HEAD" "Code Review"
copdiff() {
  local range="$1"; shift
  local prompt="$*"

  if [[ -z "$range" || -z "$prompt" ]]; then
    print -u2 'Usage: copdiff "<git-range>" "<prompt text>"'
    return 2
  fi

  local tmpfile
  tmpfile="$(mktemp -t copdiff.XXXXXX.patch)" || return 1

  {
    git diff "$range" > "$tmpfile" || return 1

    # Update with the model you prefer
    copilot \
      --model gpt-5.3-codex \
      --disable-builtin-mcps \
      --deny-tool shell \
      --deny-tool url \
      --deny-tool write \
      --deny-tool memory \
      -sp "${prompt} @${tmpfile}"
  } always {
    rm -f "$tmpfile"
  }
}
```

## 💡 Example Prompts

Try these example prompts with the diff review functions above, or come up with your own!

| Category | Prompt |
|----------|--------|
| **Documentation** | `"Produce Gherkin Acceptance Criteria"` |
| **Git workflow** | `"Generate Git commit message"` |
| **Git workflow** | `"Generate Bitbucket pull request description"` |
| **Git workflow** | `"Suggest how these commits should be squashed and organized"` |
| **Security** | `"Perform a security review of this diff"` |
| **Security** | `"Identify potential security vulnerabilities, including injection, auth bypass, or secret exposure"` |
| **Risk** | `"Analyze backward compatibility risks"` |
| **Risk** | `"Identify breaking API or behavioral changes"` |
| **Risk** | `"Identify potential regressions or risky changes introduced by this diff"` |
| **Architecture** | `"Explain how this change affects system architecture"` |
| **Architecture** | `"Explain runtime or infrastructure impact of this change"` |
| **Testing** | `"Identify missing test coverage for this change"` |
| **Testing** | `"Suggest unit tests that should accompany this change"` |
| **Release** | `"Generate release notes for these changes"` |
| **Performance** | `"Identify performance issues introduced by this change"` |
| **Observability** | `"Suggest logging, metrics, and tracing improvements"` |
| **Onboarding** | `"Explain this change as if onboarding a new engineer"` |
| **Code quality** | `"Identify code smells or design problems introduced by this diff"` |
| **Code quality** | `"Suggest refactoring opportunities"` |

## Git diff range expressions

| Expression | Description | When useful |
|------------|-------------|-------------|
| `HEAD^..HEAD` | The single most recent commit | Quick review of last commit |
| `HEAD~3..HEAD` | The last N commits (here: 3) | Feature branch review before PR |
| `HEAD` | All uncommitted working-tree changes | Reviewing in-progress work |
| `origin/main..HEAD` | All commits on the current branch not yet in `origin/main` | Pre-merge / pre-PR review |
| `$(git describe --tags --abbrev=0)..HEAD` | Everything since the most recent tag | Generating release notes |
| `v1.2.0..v1.3.0` | All changes between two named tags | Migration notes between releases |
| `f8097ec67^!` | Exactly one specific commit (shorthand for `SHA^..SHA`) | Auditing or re-reviewing a single commit |
| `main...HEAD` | Commits unique to the current branch since it diverged from `main` (three-dot symmetric diff) | Branch review when the base branch has moved on since you branched |
| `ORIG_HEAD..HEAD` | Changes introduced by the most recent merge or rebase (`ORIG_HEAD` is set automatically by Git) | Post-merge or post-rebase review |
| `HEAD@{1}..HEAD` | Everything since the last time `HEAD` moved (reflog-based) | Reviewing what just landed after a `git pull` |
| `$(git hash-object -t tree /dev/null)..HEAD` | Every commit from the very beginning of the repo | Full project audit or initial review |

---

## License

MIT
