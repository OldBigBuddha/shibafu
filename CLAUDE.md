# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## ⚠️ IMPORTANT: Educational Implementation Policy

**Claude Code MUST NOT implement any actual container runtime code outside of the `docs/` directory.**

This is a hands-on learning project where the developer must implement all functionality themselves to gain deep understanding of container technologies. Claude Code's role is strictly limited to:

### What Claude Code CAN Do:
- Read and explain existing documentation in `docs/plan/`
- Help understand concepts and answer theoretical questions
- Review and analyze code that has already been written by the developer
- Suggest debugging approaches for specific errors
- Explain Linux concepts (namespaces, cgroups, networking)
- Help with Go language syntax questions

### What Claude Code MUST NOT Do:
- Write any implementation code (`.go` files, shell scripts, etc.)
- Create project structure outside of documentation
- Implement container runtime functionality
- Generate boilerplate code for phases
- Write configuration files or build scripts
- Provide complete code solutions

## Project Overview

Shibafu is an educational Platform-as-a-Service system for learning container runtime technologies from the ground up. The developer must implement every component manually to understand how container platforms work internally.

## Learning Path (9 Phases)

The project follows a structured approach where each phase builds upon the previous:

| Phase | Focus | What You'll Build Yourself |
|-------|-------|---------------------------|
| **0** | runc Hello World | Manual OCI bundle creation, container execution |
| **1** | libcontainer | Go program using libcontainer APIs |
| **2** | Networking | Linux bridge, veth pairs, NAT rules |
| **3** | OverlayFS | Layer management, Copy-on-Write storage |
| **4** | Port Mapping | iptables rules, reverse proxy |
| **5** | OCI Registry | Image distribution server |
| **6** | DNS & Service Discovery | Internal DNS server |
| **7** | Integrated Runtime | Complete container runtime |
| **8** | PaaS API | Application deployment automation |

## Learning Approach

### Phase-by-Phase Implementation
1. **Read Documentation**: Study `docs/plan/XX-*.md` thoroughly
2. **Research Concepts**: Understand the underlying Linux technologies
3. **Implement Yourself**: Write all code from scratch
4. **Test and Debug**: Verify each component works before proceeding
5. **Understand Why**: Focus on the reasoning behind each design decision

### Key Learning Areas
- **Linux System Programming**: namespaces, cgroups, mount operations
- **Container Fundamentals**: OCI specs, image formats, runtime lifecycle
- **Network Programming**: netlink, iptables, bridge networking
- **Storage Systems**: OverlayFS, content-addressable storage
- **Distributed Systems**: service discovery, load balancing, APIs

## Educational Philosophy

**Build It Yourself to Understand It**

The only way to truly understand container technology is to implement it yourself. This project guides you through building your own:
- Container runtime (like Docker)
- Image registry (like Docker Hub)
- Orchestration system (like basic Kubernetes)
- PaaS platform (like Heroku)

By the end, you'll understand every layer from Linux kernel features up to application deployment.

## Getting Help from Claude Code

When working with Claude Code on this project:

```
❌ BAD: "Implement Phase 1 for me"
❌ BAD: "Write the container runtime code"
❌ BAD: "Create the project structure"

✅ GOOD: "Explain how libcontainer Factory works"
✅ GOOD: "What's the difference between CLONE_NEWPID and CLONE_NEWNET?"
✅ GOOD: "I'm getting this error in my code: [error]. What might be wrong?"
✅ GOOD: "Help me understand this documentation: [specific docs section]"
```

## Important Reminders

- **No Shortcuts**: The learning value comes from implementing everything yourself
- **Read First**: Always study the phase documentation before asking questions
- **Focus on Understanding**: Don't just make it work, understand why it works
- **Embrace Challenges**: The difficult parts are where the most learning happens

This is your journey to mastering container technology. Claude Code is here to guide your understanding, not to do the work for you.