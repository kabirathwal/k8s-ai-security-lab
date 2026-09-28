# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Project: k8s-ai-security-lab

A local Kubernetes lab (kind) that serves a small LLM via Ollama plus a FastAPI
chat-api. Kustomize base + two overlays: vulnerable/ and hardened/.
Goal: portfolio project showing attack, hardening, CI scanning, monitoring.

## About me
I am learning Kubernetes. Explain what each manifest field does and why,
briefly, when you create or change it. Prefer simple, standard tools.

## Rules
- Ask before running any command that deletes clusters, volumes, or files.
- Never put real secrets in the repo. Use obviously fake values like FAKE-API-KEY.
- Keep the vulnerable overlay intentionally insecure; label each weakness
  with a comment `# VULN:` and a one-line explanation.
- In the hardened overlay, label each control with `# CONTROL:` and what it mitigates.
- Pin image versions (no :latest).
- After changes, tell me the exact commands to verify them.