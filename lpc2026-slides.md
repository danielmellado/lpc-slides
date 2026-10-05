class: title-slide

# Fedora eBPF SIG

## The Fedora 46 Change Proposal & Lessons Learned

### Daniel Mellado · Fedora eBPF SIG

Linux Plumbers Conference 2026 - eBPF Track

---

# The Fedora 46 Change Proposal

Ship `bpf-rbacd` - a daemon for **unprivileged, policy-controlled eBPF** via a single YAML policy:

.two-col[
.col[
### Capability Granting
Containers in **user namespaces**, BPF token delegation via the nsenter dance, **in-kernel LSM enforcement** through 3 hooks, per-namespace policy bitmaps
]
.col[
### Proxy Execution
Desktop users in **init_user_ns**, daemon calls `bpf()` on behalf of user, FDs back via `SCM_RIGHTS`, `SO_PEERCRED` for auth
]
]

### Policy: allow-list only, fails closed

Roles (`ebpf`, `ebpf-net`, `ebpf-admin`) mapped to groups - effective policy = **role ∩ system_policy**

.footnote[System-Wide Change - Fedora 46 - Opt-in, does not block release · [Fedora Forge: ebpf/bpf-rbacd](https://forge.fedoraproject.org/ebpf/bpf-rbacd)]

---

class: fill

# Issues We Found

| # | Issue | Impact |
|---|-------|--------|
| 1 | **BPF tokens don't work on the desktop** | `BPF_TOKEN_CREATE` returns `EOPNOTSUPP` in `init_user_ns` - proxy mode is the only path |
| 2 | **Rust/DWARF/BTF toolchain gap** | `rustc` doesn't emit DWARF for `extern` decls on BPF targets → kfuncs and open-coded iterators fail to load |
| 3 | **Two parallel LSM codebases** | C version has full features; Aya only has 3 core hooks - ship C for F46, migrate after `rustc` fix |
| 4 | **BPF not in Fedora's active LSM list** | `CONFIG_BPF_LSM=y` is set, but `bpf` not in `lsm=` - needs kernel team coordination, SELinux concerns |
| 5 | **Container discovery is open** | How does the daemon learn about new namespaces? cgroup events, CRI hooks, `PrivateBPF=`? |

---

# Upstream Work We're Driving

| PR | What | Status |
|----|------|--------|
| [rust-lang/rust#161390](https://github.com/rust-lang/rust/pull/161390) | `rustc` DWARF for BPF extern decls - **unblocks kfuncs for all Rust eBPF** | In review (compiler team) |
| [aya-rs/aya#1372](https://github.com/aya-rs/aya/pull/1372) | kfunc relocation support in Aya | In progress |
| [systemd/systemd#36134](https://github.com/systemd/systemd/pull/36134) | `PrivateBPF=` - automatic bpffs delegation for services | In progress |

### Phased delivery

**F46 (now):** C eBPF + Rust userspace - full feature set, not blocked by `rustc`

**After `rustc` fix:** migrate eBPF programs to Rust/Aya - drop-in replacement

**Target state:** pure Rust - single `cargo build`, shared `#[repr(C)]` types, memory safety for eBPF too

.footnote[Downstream target: RHEL · OpenShift (user namespaces GA in 4.21+) · CoreOS · K8s user namespaces GA in v1.36]

---

# Community Input Wanted

**LSM list ordering** - should `bpf` go before or after SELinux? What interactions?

**Container discovery** - best mechanism for learning about new user namespaces?

**Helper/kfunc policy** - bytecode walking can extract helper IDs; should this be a policy dimension?

**Revocation** - removing a map entry denies future ops, but existing FDs stay valid. Sufficient?

### Links

Code: [forge.fedoraproject.org/ebpf/bpf-rbacd](https://forge.fedoraproject.org/ebpf/bpf-rbacd) · [tohojo/bpf-rbac-lsm](https://github.com/tohojo/bpf-rbac-lsm)

SIG: [fedoraproject.org/wiki/SIGs/eBPF](https://fedoraproject.org/wiki/SIGs/eBPF) · Matrix: [#ebpf:fedoraproject.org](https://matrix.to/#/#ebpf:fedoraproject.org)

Talk: [lpc.events/event/20/contributions/2441](https://lpc.events/event/20/contributions/2441/)
