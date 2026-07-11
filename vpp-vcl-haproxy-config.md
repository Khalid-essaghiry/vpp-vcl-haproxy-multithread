# Configuration finale — VPP + VCL + HAProxy

## Architecture validée

```
curl (host) → GigabitEthernet7/0/0 → VPP session layer → VCL → HAProxy (nbthread 1) → VCL → backend HTTP
```

⚠️ **Point critique validé par les tests** : `nbthread > 1` dans HAProxy n'est **pas supporté** avec VCL
(erreur `migrate NOT supported`). La configuration stable utilise `nbthread 1`.

---

## 1. `/etc/vpp/startup.conf`

```
unix {
  nodaemon
  log /var/log/vpp/vpp.log
  full-coredump
  cli-listen /run/vpp/cli.sock
  gid vpp
}

api-trace {
  on
}

api-segment {
  gid vpp
}

socksvr {
  default
}

cpu {
  # main-core 0
  # workers 1
}

session {
  enable
  use-app-socket-api
}

plugins {
  plugin default { enable }
}
```

---

## 2. `/etc/vpp/vcl.conf`

```
vcl {
  rx-fifo-size 4000000
  tx-fifo-size 4000000
  app-scope-local
  app-scope-global
  app-socket-api /run/vpp/app_ns_sockets/default
}
```

---

## 3. `/etc/haproxy-vcl/haproxy.cfg`

```
global
    log 127.0.0.1 local0 info
    maxconn 4096
    nbthread 1
    stats socket /tmp/haproxy.sock mode 660 level admin

defaults
    mode http
    retries 3
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    timeout check 10s

frontend http_in
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
    server web1 127.0.0.1:8080 check inter 1s rise 1 fall 3
```

---

## 4. Ordre de démarrage (commandes)

### a) Créer le conteneur VPP

```bash
docker run -itd --name vpp \
  --privileged \
  --network host \
  -v /dev/hugepages:/dev/hugepages \
  -v /run/vpp:/run/vpp \
  -v /etc/vpp-docker/startup.conf:/etc/vpp/startup.conf \
  ligato/vpp-base:26.02-rc0.213-g9c0e54a81
```

### b) Entrer dans le conteneur

```bash
docker exec -it vpp bash
```

### c) Configurer l'interface réseau (à refaire à chaque redémarrage du conteneur)

```bash
vppctl show interface
vppctl set interface state GigabitEthernet7/0/0 up
vppctl set interface ip address GigabitEthernet7/0/0 192.168.122.145/24
vppctl ip route add 0.0.0.0/0 via 192.168.122.140 GigabitEthernet7/0/0
```

### d) Vérifier la session layer

```bash
vppctl show session
vppctl show app
```

### e) Exporter VCL_CONFIG — OBLIGATOIRE avant tout LD_PRELOAD

```bash
export VCL_CONFIG=/etc/vpp/vcl.conf
```

### f) Démarrer le backend (VCL)

```bash
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libvcl_ldpreload.so python3 -m http.server 8080 &
```

Vérifier qu'aucune erreur `couldn't connect to VPP` n'apparaît, puis :

```bash
vppctl show app
```

→ doit afficher `python3-ldp-xxx`.

### g) Démarrer HAProxy (VCL) — dans un nouveau terminal du même conteneur

```bash
export VCL_CONFIG=/etc/vpp/vcl.conf
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libvcl_ldpreload.so haproxy -db -f /etc/haproxy-vcl/haproxy.cfg
```

→ ne doit afficher aucun `Server web_servers/web1 is DOWN`.

### h) Vérifier l'état du backend HAProxy

```bash
echo "show servers state" | socat stdio /tmp/haproxy.sock
```

→ `srv_op_state` doit valoir `2` (UP).

### i) Test final depuis l'hôte

```bash
curl -v http://192.168.122.145:80/
```

→ doit retourner `HTTP/1.0 200 OK`.

---

## Erreurs rencontrées et causes

| Erreur | Cause |
|---|---|
| `Failed with "nbthread 4 (more than 1)"` | HAProxy multi-thread (nbthread > 1) non supporté avec VCL |
| `vls_mt_session_migrate: ERROR migrate NOT supported` | Un backend/proxy tente de migrer une session VCL entre threads OS (ex: `ThreadingMixIn` Python) |
| `vcl_bapi_init: ERROR couldn't connect to VPP!` | `VCL_CONFIG` non exporté avant le lancement de l'application avec `LD_PRELOAD` |
| `Server web1 is DOWN, reason: Layer4 timeout/connection problem` | Le backend visé (127.0.0.1:8080) n'écoute pas réellement via VCL (crash silencieux ou non démarré) |

---

## Notes

- Toute la configuration réseau de l'interface VPP (IP, route) est perdue à chaque redémarrage du conteneur — elle n'est pas persistée dans `startup.conf`. Pour l'automatiser, ajouter un fichier de config exécuté au boot ou un script d'init.
- Pour un vrai parallélisme (au-delà d'un seul thread HAProxy), il faut envisager plusieurs processus HAProxy séparés (chacun en `nbthread 1`) plutôt que le multi-threading interne.
