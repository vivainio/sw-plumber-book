---
icon: lucide/shield
---

# Security

Security mechanisms answer different questions: who can read a key, how
stored data is encrypted, which resources a process can access, and what
happens when that process is compromised. Understanding the boundary each
mechanism enforces helps when combining them.

This section explores those building blocks, from cryptographic storage
to operating system access controls. Some are new; others have existed for
years but rarely appear in application-level documentation.

## Chapters

- [Linux kernel keyrings](linux-keyrings.md) — retaining encryption keys in
  the kernel, possession and permissions, process inheritance, and lifetime.
