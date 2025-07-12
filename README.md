# Shibafu - Educational Container Runtime & PaaS

**Shibafu** is an educational Platform-as-a-Service system designed for learning container runtime technologies from the ground up.

## What is Shibafu?

Shibafu is a **learning-focused project** that guides you through building a complete container platform, starting from basic `runc` usage and progressing to a full-featured PaaS. Unlike production systems, Shibafu prioritizes educational value and hands-on understanding.

## Learning Journey (Phase 0-8)

| Phase | Topic | Core Learning |
|-------|-------|---------------|
| **0** | runc Hello World | Container fundamentals, OCI bundles |
| **1** | libcontainer Implementation | Programmatic container management |
| **2** | Container Networking | Linux networking, bridges, veth pairs |
| **3** | OverlayFS & Layers | Efficient storage, Copy-on-Write |
| **4** | Port Mapping & Load Balancing | Service exposure, iptables, reverse proxy |
| **5** | OCI Registry | Image distribution, content-addressable storage |
| **6** | DNS & Service Discovery | Internal networking, name resolution |
| **7** | Integrated Runtime | System integration, monitoring, CLI |
| **8** | PaaS API | Application deployment automation |

## What You'll Build

After completing all phases, you will have created:

### Technical Implementation
- Complete container runtime (Docker-equivalent functionality)
- Multi-network container orchestration
- OCI-compliant image registry
- HTTP/DNS-based service discovery
- RESTful PaaS API for application deployment
- Management CLI tools (with prototype web dashboard HTML)

### Production-Ready Features
- Automatic application builds from source code
- Blue-green deployments with rollback
- SSL certificate management (Let's Encrypt)
- Environment variable & secrets management
- Real-time logging and metrics collection
- Auto-scaling based on resource usage

## Learning Outcomes

Upon completion, you will understand:

### Container Technology Fundamentals
- Linux namespaces, cgroups, and container isolation
- OCI specifications and image formats
- Container networking and storage internals

### System Programming Skills
- Go-based system programming
- Linux network programming (netlink, iptables)
- File system operations (OverlayFS, mount)

### Distributed Systems Design
- Service discovery and load balancing
- API design and event-driven architecture
- Monitoring, logging, and observability

### Platform Engineering
- Container orchestration patterns
- CI/CD pipeline integration
- Infrastructure automation

## Who Should Use Shibafu?

- **Platform Engineers** wanting to understand container internals
- **DevOps Engineers** seeking deep Kubernetes/Docker knowledge
- **System Programmers** interested in Linux container technology
- **Students** learning distributed systems and cloud computing

## Quick Start

```bash
# Start with Phase 0: Basic container execution
cd docs/plan/
cat 01-helloworld.md

# Each phase builds upon the previous one
# Follow the step-by-step learning plans
```

## Educational Philosophy

Shibafu emphasizes **understanding over usage**. Instead of teaching how to use existing tools, it guides you through building them yourself, ensuring deep comprehension of:

- Why container technology works the way it does
- How modern PaaS platforms are architected
- What challenges arise in distributed container systems

By the end, you'll have built your own mini-Heroku and understand every component from the kernel up to the application layer.

---

**Note**: Shibafu is designed for education. For production use, consider established platforms like Kubernetes, Docker, or cloud PaaS services.