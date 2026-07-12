# Bytes, Bits & CPU Units — Reference

## 1. Bits and bytes

The base unit of memory is the **bit** (a 0 or a 1). Memory is almost always
measured in **bytes**:

```
1 byte = 8 bits
```

Bits show up in network speeds and rarely in storage; Kubernetes resource
fields are always bytes. If you see "memory: 100Mi" — that's 100 mebibytes,
not megabits.

---

## 2. The two scales — decimal (SI) vs binary (IEC)

There are **two** valid scales for "kilo" / "mega" / "giga", and
Kubernetes accepts both. They are not the same.

### Decimal (SI, base 10) — what marketing uses

| Prefix | Symbol | Value | Bytes |
|---|---|---|---|
| kilo | `K` | 10³ | 1,000 |
| mega | `M` | 10⁶ | 1,000,000 |
| giga | `G` | 10⁹ | 1,000,000,000 |
| tera | `T` | 10¹² | 1,000,000,000,000 |
| peta | `P` | 10¹⁵ | 10¹⁵ |
| exa  | `E` | 10¹⁸ | 10¹⁸ |

A 1 TB hard drive on a store shelf is 1 × 10¹² = 1,000,000,000,000 bytes.

### Binary (IEC, base 2) — what computers actually use

| Prefix | Symbol | Value | Bytes |
|---|---|---|---|
| kibi | `Ki` | 2¹⁰ | 1,024 |
| mebi | `Mi` | 2²⁰ | 1,048,576 |
| gibi | `Gi` | 2³⁰ | 1,073,741,824 |
| tebi | `Ti` | 2⁴⁰ | ~1.1 × 10¹² |
| pebi | `Pi` | 2⁵⁰ | ~1.13 × 10¹⁵ |
| exbi | `Ei` | 2⁶⁰ | ~1.15 × 10¹⁸ |

Computers count in powers of 2 because memory addresses are binary. A
"1 GiB" stick of RAM holds exactly 2³⁰ bytes.

### The naming convention to lock in

**The `bi` infix marks binary.**

- `K` → kilo (×1,000); `Ki` → **kibi** (×1,024)
- `M` → mega (×1,000,000); `Mi` → **mebi** (×1,048,576)
- `G` → giga (×10⁹); `Gi` → **gibi** (×2³⁰)
- `T` → tera; `Ti` → **tebi**

The `i` (or `bi` in the spelled-out names) is the binary tell. "Gigabyte"
vs "gibibyte" — the second one means base-2.

### The size difference matters

`1 G` vs `1 Gi`:

- `1 G` = 1,000,000,000 bytes
- `1 Gi` = 1,073,741,824 bytes
- Difference: ~7.4%

For 1 KB vs 1 KiB it's only 2.4%, but the gap widens with each step. At
`1 Ti` vs `1 T` it's ~10%. Confusing them in a memory limit is a real
operational problem.

### Historical note (why this is a mess)

Before ~1998 everyone just said "kilobyte" and "megabyte" and *meant*
base-2 even when calling it `K` or `M`. Hard drive manufacturers, who
preferred the larger-sounding base-10 numbers, started using SI strictly —
your "500 GB" drive is 500,000,000,000 bytes, not 500 × 2³⁰. The IEC
standardized `Ki/Mi/Gi` in 1998 to disambiguate. Now both notations
coexist. Kubernetes accepts both, so you have to know the difference.

---

## 3. Kubernetes memory units — what you'll actually write

Kubernetes memory values accept any of: `K`, `M`, `G`, `T`, `P`, `E`,
`Ki`, `Mi`, `Gi`, `Ti`, `Pi`, `Ei`, or plain bytes.

```yaml
memory: 128974848   # plain bytes — valid but unreadable
memory: 129M        # 129,000,000 bytes
memory: 128974848   # equivalent (close enough)
memory: 128.9M      # 128,900,000 bytes
memory: 128974848e0 # scientific notation, valid
memory: 128Mi       # 128 × 2²⁰ = 134,217,728 bytes
memory: "1Gi"       # 2³⁰ = 1,073,741,824 bytes — quote it
```

### Convention to follow

> **Use base-2 (`Ki`, `Mi`, `Gi`) for memory by default.**

Reasons:

- It matches how the OS and the container runtime actually account for
  memory.
- It matches what `kubectl top` reports (in `Mi`).
- The lecture's examples use it (`1Gi`, `2Gi`, `512Mi`).
- It's the convention in nearly all production manifests you'll read.

Base-10 (`G`, `M`) works but is unusual. If you see it in a manifest, the
author either wrote it from habit, or did so intentionally to align with a
billing dimension — both real, neither common.

### Quoting

Always safe to quote: `memory: "1Gi"`. Without quotes, YAML's parser
*usually* gets it right because `1Gi` isn't a valid number, but quoting is
a free habit and immunizes against weird edge cases.

---

## 4. CPU units — millicores

CPU is measured in **cores** (or more precisely, "compute units" — see
below). The unit you'll see constantly is `m`, for **milli** (one
thousandth):

```
1000m = 1 CPU core
500m  = 0.5 CPU core
100m  = 0.1 CPU core
1m    = 0.001 CPU core
```

So `cpu: 100m` means "0.1 of a CPU core." It's equivalent to `cpu: 0.1`,
but `m` notation is preferred because it avoids decimal-point ambiguities
in YAML.

```yaml
cpu: 1       # one full core
cpu: "1"     # same, quoted
cpu: 1000m   # same — millicores
cpu: 0.5     # half a core
cpu: 500m    # same — preferred
cpu: 100m    # one-tenth of a core
```

### What "1 CPU" actually means

Equivalents:

> 1 CPU  ≡  1 AWS vCPU  ≡  1 GCP Core  ≡  1 Azure Core  ≡  1 Hyperthread

The unifying definition: **1 CPU = the compute the kernel exposes as a
single schedulable logical processor**. On modern hyperthreaded x86 chips,
each physical core appears as 2 logical processors to the kernel; each of
*those* is "1 CPU" in Kubernetes terms.

Practical consequence: on an 8-core machine with hyperthreading enabled,
Linux sees 16 CPUs, and Kubernetes can allocate up to 16 to Pods on that
node. If you disabled hyperthreading, it would see 8.

### Sub-millicore is not allowed

`cpu: 0.5m` is invalid. The smallest unit is 1m (one millicore). In
practice, you'll rarely go below 10m for anything that does meaningful work.

### Typical sizing

Rough ranges, just for calibration:

| Workload | Typical CPU request | Typical memory request |
|---|---|---|
| Sidecar / proxy | 50m – 100m | 64Mi – 128Mi |
| Small web app | 100m – 250m | 128Mi – 256Mi |
| Mid web app | 500m – 1 | 512Mi – 1Gi |
| Database (small) | 1 – 2 | 1Gi – 4Gi |
| ML training | 2 – many | many Gi |

The right values are workload-specific and discovered empirically (load
test, measure, tune). These are starting points, not answers.

---

## 5. Quick cheat sheet

```
# Memory: prefer base-2 (Gi, Mi, Ki) — matches OS, kubectl top, conventions
memory: "256Mi"
memory: "1Gi"
memory: "2Gi"

# CPU: millicores for sub-1, integers for whole cores
cpu: "100m"     # 0.1 core
cpu: "500m"     # 0.5 core
cpu: "1"        # 1 core
cpu: "2"        # 2 cores

# Base-10 vs base-2 (memory):
1 K  =  1,000 bytes        1 Ki  =  1,024 bytes
1 M  =  1,000,000          1 Mi  =  1,048,576
1 G  =  1,000,000,000      1 Gi  =  1,073,741,824
                                  (the i = binary; "gibi" not "giga")

# CPU millicores:
1000m = 1 CPU
 500m = 0.5 CPU
 100m = 0.1 CPU
```

---

## 6. Why this matters operationally

Two real-world failure modes from getting units wrong:

1. **Specifying `1G` when you meant `1Gi`** — your container has 7.4% less
   memory than you think. Usually harmless; occasionally the cause of a
   maddening intermittent OOM near "full" usage that no one can explain.
2. **Specifying `1` when you meant `1m`** — your container reserves
   1,000× more CPU than intended. Either the Pod sits in `Pending` forever
   (no node fits a 1-core request when you intended 1 millicore) or it
   reserves a whole CPU on the node that the scheduler then can't give
   anyone else. Both wasteful, both common.

Reading units fluently is the cheapest insurance there is.
