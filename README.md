# VPP + VCL + HAProxy — Multi-thread Configuration

Documentation et configuration de référence pour faire fonctionner **HAProxy** au-dessus de **VPP** via **VCL** (VPP Comms Library), incluant les limitations rencontrées avec le mode multi-thread (`nbthread`).

📄 Configuration complète : [`vpp-vcl-haproxy-config.md`](./vpp-vcl-haproxy-config.md)

---

## Architecture

```mermaid
flowchart LR
    A[Client externe<br/>curl / navigateur] -->|TCP :80| B[Interface physique<br/>GigabitEthernet]
    B --> C[VPP<br/>Session Layer]
    C --> D[VCL<br/>libvcl_ldpreload.so]
    D --> E[HAProxy<br/>nbthread 1]
    E --> F[VCL<br/>libvcl_ldpreload.so]
    F --> G[Backend HTTP<br/>:8080]

    style A fill:#e8f4fd,stroke:#1a73e8
    style C fill:#fff3cd,stroke:#e6a700
    style D fill:#d4edda,stroke:#28a745
    style E fill:#f8d7da,stroke:#dc3545
    style F fill:#d4edda,stroke:#28a745
    style G fill:#e8f4fd,stroke:#1a73e8
```

**Points clés validés :**
- `nbthread 1` est **obligatoire** côté HAProxy — le multi-thread interne (`nbthread > 1`) échoue avec `vls_mt_session_migrate: ERROR migrate NOT supported`, car VCL ne supporte pas la migration de session entre threads OS.
- Chaque application (HAProxy, backend) doit exporter `VCL_CONFIG` **avant** son lancement via `LD_PRELOAD`, sinon elle ne peut pas s'enregistrer auprès de VPP.
- Pour obtenir un vrai parallélisme, il faut lancer **plusieurs processus HAProxy séparés** (chacun en `nbthread 1`), plutôt que du multi-threading interne.

---

## Erreurs rencontrées

| Erreur | Cause |
|---|---|
| `Failed with "nbthread 4 (more than 1)"` | HAProxy multi-thread (`nbthread > 1`) non supporté avec VCL |
| `vls_mt_session_migrate: ERROR migrate NOT supported` | Migration de session VCL entre threads OS (ex: `nbthread > 1` ou backend threadé) |
| `vcl_bapi_init: ERROR couldn't connect to VPP!` | `VCL_CONFIG` non exporté avant le lancement avec `LD_PRELOAD` |
| `Server web1 is DOWN, reason: Layer4 timeout` | Le backend ciblé n'écoute pas réellement via VCL |

---

## Fichiers

- [`vpp-vcl-haproxy-config.md`](./vpp-vcl-haproxy-config.md) — `startup.conf`, `vcl.conf`, `haproxy.cfg`, ordre de démarrage complet et tableau de diagnostic.

---

## Discussion liée

[FDio/vpp Discussion #77 — Assistance with VPP VCL and HAProxy Multi-thread Configuration Issue](https://github.com/FDio/vpp/discussions/77)
