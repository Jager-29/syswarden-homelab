# SysWarden - Déploiement Homelab

Documentation de l'intégration de [SysWarden](https://github.com/duggytuxy/syswarden) sur mon infrastructure homelab auto-hébergée. Ce dépôt ne contient pas le code source de SysWarden - il documente ma configuration de déploiement, les décisions d'intégration et la coexistence avec la stack existante.

> Basé sur SysWarden v3.10.2 par [@duggytuxy](https://github.com/duggytuxy) — licence GPLv3.

## Contexte

Ce déploiement s'inscrit dans une infrastructure plus large documentée dans [securehomelab](https://github.com/Jager-29/securehomelab). SysWarden y prend en charge la couche de défense réseau bas niveau (L2/L3/L4) sur l'hôte Debian, en complément de CrowdSec qui opère sur la couche applicative (L7) via les logs des conteneurs Docker.

### Architecture de défense en profondeur

```
Internet
   │
   ▼
[ SysWarden — couche hôte ]
   ├── L2/L3 ingress (NIC) : blocklists GeoIP + ASN + Data-Shield → drop avant conntrack
   ├── L4 stateful         : purification TCP, Default-Deny catch-all
   └── L7 WAF (Go engine)  : signatures MITRE ATT&CK, UDS socket, telemetry
   │
   ▼
[ nftables — pare-feu noyau ]
   │
   ▼
[ CrowdSec — couche applicative ]
   ├── Analyse logs NPM (conteneur)
   ├── Analyse logs Cowrie (honeypot)
   └── Bans comportementaux → nftables
   │
   ▼
[ Stack Docker — securehomelab ]
   ├── Nginx Proxy Manager  (reverse proxy / SSL)
   ├── Cowrie               (honeypot SSH :2222)
   ├── Metabase             (threat intel BI)
   └── Grafana / Prometheus (monitoring)
```

Les deux systèmes écrivent dans nftables sur des **tables distinctes** : SysWarden opère dans `syswarden_hw_drop` sur le hook `netdev` (ingress NIC), CrowdSec opère dans `CROWDSEC_CHAIN` sur le hook `input`. Aucun conflit d'écriture possible.

## Spécificité ARM64 - Compilation manuelle

Les releases officielles de SysWarden v3 ne fournissent que des binaires `amd64`. Cette infra tourne sur **Freebox Ultra (aarch64 / ARM64)** - les binaires doivent être compilés depuis les sources.

### Prérequis

- Debian 12+ / Ubuntu 24.04+ (ARM64)
- Go 1.22+ (`apt install golang-go`)
- Git (`apt install git`)
- Accès root

### Compilation des binaires ARM64

```bash
# Cloner les sources
cd /usr/local/bin
git clone https://github.com/duggytuxy/syswarden.git
cd syswarden

# Forcer la compilation ARM64 (le build.sh officiel cible amd64 en dur)
export GOOS=linux
export GOARCH=arm64
mkdir -p dist/bin

# Compiler les trois modules
cd src/core/syswarden-cli
go mod tidy && go build -ldflags="-s -w" -o ../../../dist/bin/syswarden-cli .
cd ../syswarden-core
go mod tidy && go build -ldflags="-s -w" -o ../../../dist/bin/syswarden-core .
cd ../syswarden-tui
go mod tidy && go build -ldflags="-s -w" -o ../../../dist/bin/syswarden-tui .

# Vérifier l'architecture des binaires produits
file /usr/local/bin/syswarden/dist/bin/*
# -> ELF 64-bit LSB executable, ARM aarch64 ✓
```

### Installation des binaires

```bash
# Copier dans /usr/local/bin pour le PATH
cp /usr/local/bin/syswarden/dist/bin/* /usr/local/bin/
chmod +x /usr/local/bin/syswarden-cli /usr/local/bin/syswarden-core /usr/local/bin/syswarden-tui

# Créer le symlink syswarden -> syswarden-cli
ln -s /usr/local/bin/syswarden-cli /usr/local/bin/syswarden

# Créer le dossier attendu par le service systemd
mkdir -p /opt/syswarden/bin
cp /usr/local/bin/syswarden/dist/bin/* /opt/syswarden/bin/
chmod +x /opt/syswarden/bin/*
```

## Déploiement

### 1. Préparer la configuration

```bash
mkdir -p /opt/syswarden
cp syswarden-auto.conf /opt/syswarden/syswarden-auto.conf
chmod 600 /opt/syswarden/syswarden-auto.conf

# Renseigner les variables sensibles
nano /opt/syswarden/syswarden-auto.conf
# -> SYSWARDEN_SSH_PORT, SYSWARDEN_WHITELIST_IPS
```

### 2. Lancer l'installation

```bash
/usr/local/bin/syswarden-cli install
```

### 3. Corriger le signatures.json (spécificité ARM64)

Le service `syswarden-core` requiert `/opt/syswarden/signatures.json` — non généré par l'installeur en mode compilation manuelle. L'extraire depuis l'archive de release :

```bash
wget https://github.com/duggytuxy/syswarden/releases/latest/download/syswarden-release.tar.gz
tar -xzOf syswarden-release.tar.gz ./signatures.json > /opt/syswarden/signatures.json
rm syswarden-release.tar.gz

systemctl restart syswarden-core
systemctl is-active syswarden-core   # -> active
```

### 4. Vérifier le déploiement

```bash
/opt/syswarden/bin/syswarden-cli audit
```

Résultat obtenu sur cette infra (7/7 PASS) :

```
Phase 1 : Cron Orchestration              [PASS]
Phase 2 : Log Routing & Anti-Injection    [PASS]
Phase 3 : Kernel Shield & Threat Intel    [PASS] (GeoIP + ASN + Data-Shield + Docker)
Phase 4 : Layer 7 WAF                     [PASS]
Phase 5 : DevSecOps Telemetry             [PASS]
Phase 6 : WireGuard                       [INFO] Disabled
Phase 7 : CSPM / Persistence              [PASS]
```

## Décisions de déploiement

### CrowdSec + WAF SysWarden : périmètres séparés

SysWarden WAF analyse les logs système via UDS socket. CrowdSec couvre les services conteneurisés (NPM, Cowrie). Les deux opèrent sur des surfaces distinctes sans conflit.

### Docker : whitelisting automatique

`SYSWARDEN_USE_DOCKER="y"` permet à SysWarden de détecter et whitelister automatiquement tous les bridges Docker actifs. Sur cette infra, 7 bridges ont été whitelistés automatiquement à l'installation.

### GeoIP : liste ciblée

Le géoblocage est calibré sur les pays les plus représentés dans les logs Metabase/CrowdSec — pas une liste exhaustive pour éviter les faux positifs.

### WireGuard : désactivé pour ce déploiement de test

`SYSWARDEN_ENABLE_WG="n"` - à activer sur l'infra de production pour cloisonner l'accès SSH derrière un tunnel VPN.

## Commandes utiles

```bash
# Dashboard TUI temps réel
/opt/syswarden/bin/syswarden-tui

# Audit complet
/opt/syswarden/bin/syswarden-cli audit

# Bloquer une IP manuellement
/opt/syswarden/bin/syswarden-cli block <IP>

# Whitelister une IP
/opt/syswarden/bin/syswarden-cli whitelist <IP>

# Mettre à jour les feeds de threat intelligence
/opt/syswarden/bin/syswarden-cli update-feeds

# Voir les tables nftables SysWarden
nft list ruleset | grep syswarden
```

## Mise à jour

SysWarden v3 étant en Go, une mise à jour = recompilation depuis les sources :

```bash
cd /usr/local/bin/syswarden
git pull origin main
export GOOS=linux GOARCH=arm64
# Recompiler les trois modules (voir section compilation)
cp dist/bin/* /opt/syswarden/bin/
cp dist/bin/* /usr/local/bin/
systemctl restart syswarden-core
```

## Licence

Ce dépôt (documentation et configuration) est sous licence MIT.
SysWarden est distribué sous [GPLv3](https://github.com/duggytuxy/syswarden/blob/main/LICENSE) par [@duggytuxy](https://github.com/duggytuxy).
