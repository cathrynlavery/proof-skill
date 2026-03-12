# proof-skill

A Claude Code skill for working with [Proof](https://proofeditor.ai) documents over HTTP — create, read, edit, comment, suggest, and track presence.

## Install

Copy the skill file into your project's Claude Code skills directory:

```bash
mkdir -p .claude/skills
curl -sS -o .claude/skills/proof.SKILL.md \
  https://raw.githubusercontent.com/cathrynlavery/proof-skill/main/proof.SKILL.md
```

Or clone and copy manually:

```bash
git clone https://github.com/cathrynlavery/proof-skill.git
cp proof-skill/proof.SKILL.md your-project/.claude/skills/
```

## What it enables

Once installed, your Claude Code agent can:

- **Create** shared documents on Proof
- **Read** documents via content negotiation (JSON or markdown)
- **Edit** with block-level operations (stable refs, revision locking)
- **Comment** on specific text passages
- **Suggest** replacements with accept/reject workflow
- **Track presence** and poll for document events

## Works with

- **[proofeditor.ai](https://proofeditor.ai)** — hosted Proof (default, no setup needed)
- **Self-hosted Proof SDK** — any deployment running the open-source Proof SDK

## Self-hosted configuration

The skill defaults to `https://proofeditor.ai`. To point at a self-hosted instance, set the base URL before using Proof commands:

```bash
export BASE_URL="https://your-proof-instance.example"
```

## Links

- [proofeditor.ai](https://proofeditor.ai) — hosted product
- [proof-sdk](https://github.com/cathrynlavery/proof-sdk) — open-source SDK
- [REFERENCE.md](./REFERENCE.md) — advanced API reference
