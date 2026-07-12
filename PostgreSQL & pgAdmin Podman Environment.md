# Complete Guide: Enterprise-Grade PostgreSQL & pgAdmin using Podman

---

### Purpose
This guide adapts our fail-safe PostgreSQL and pgAdmin architecture to run entirely on **Podman**. By migrating to Podman, we leverage a daemonless, rootless container engine for enhanced security, while maintaining full compatibility across Linux, macOS (Apple Silicon/Intel), and Windows (WSL2 via Podman Desktop/Machine). It retains automated user provisioning, baseline seeding, and the universal backup/restore utility.

---

### Design Decisions

*   **Rootless Execution by Default:** Podman runs containers as a non-root user, significantly reducing security risks. We rely on unprivileged ports (`5432` and `5050`) to avoid root-binding restrictions.
*   **SELinux Contexts (`:z` flags):** Linux distributions using SELinux (like Fedora, RHEL, CentOS) strictly block containers from accessing host directories by default. We append the `:z` flag to our bind mounts to automatically apply the correct SELinux labels, preventing silent permission failures.
*   **Named Volumes for Persistence:** Named volumes seamlessly handle the User Namespace (UserNS) UID/GID mapping that rootless Podman requires, completely sidestepping `chown` headaches on host file systems.
*   **Podman Compose Compatibility:** Podman natively supports Docker Compose specifications. We use standard `compose.yaml` syntax, allowing execution via `podman compose` or `podman-compose`.

---

### Architecture Diagram

```text
Host Machine (Mac/Win via podman machine, Linux native)
│
├── [ Podman Network: rootless internal bridge ]
│   │
│   ├── Container: production_postgres (Port 5432)
│   │   ├── Volume: postgres_data (Native DB Files mapped to UserNS)
│   │   ├── Mount: ./init-scripts (SELinux :z,ro labeled)
│   │   └── Mount: ./backups (SELinux :z labeled)
│   │
│   └── Container: production_pgadmin (Port 5050)
│       ├── Volume: pgadmin_data (Saves DB connections & UI settings)
│       └── Connects internally to 'production_postgres:5432'
│
└── [ Utility Scripts ]
    └── scripts/db-tool.sh (Triggers podman exec/run and S3 uploads)