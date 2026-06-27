# SysWarden — Déploiement Homelab

Documentation de l'intégration de [SysWarden](https://github.com/duggytuxy/syswarden) sur mon infrastructure homelab auto-hébergée. Ce dépôt ne contient pas le code source de SysWarden — il documente ma configuration de déploiement, les décisions d'intégration et la coexistence avec la stack existante.

> Basé sur SysWarden v0.39.x par [@duggytuxy](https://github.com/duggytuxy) — licence GPLv3.

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
   └── L7 HIPS (Fail2ban)  : 56+ jails sur services système (SSH, etc.)
   │
   ▼
[ nftables — pare-feu noyau ]
   │
   ▼
[ CrowdSec — couche applicative ]
   ├── Analyse logs NPM (conteneur)
   ├── Analyse logs Cowrie (honeypot)
   └── Bans comportementaux → ipset / nftables
   │
   ▼
[ Stack Docker — securehomelab ]
   ├── Nginx Proxy Manager  (reverse proxy / SSL)
   ├── Cowrie               (honeypot SSH :2222)
   ├── Metabase             (threat intel BI)
   └── Grafana / Prometheus (monitoring)
```

Les deux systèmes écrivent dans nftables sur des **tables distinctes** : SysWarden opère dans `syswarden_table` sur le hook `netdev` (ingress NIC), CrowdSec opère dans ses propres chaînes sur le hook `input`. Aucun conflit d'écriture possible.

## Décisions de déploiement

### CrowdSec + Fail2ban : périmètres séparés

SysWarden active automatiquement des jails Fail2ban pour les services détectés en écoute. Comme CrowdSec couvre déjà les services Docker (NPM, Cowrie), il y a un risque de double détection. La règle appliquée :

- Fail2ban de SysWarden : services **hôte** uniquement (SSH système, journald, kernel).
- CrowdSec : services **conteneurisés** (NPM, Cowrie, HTTP).

Pas de désactivation manuelle nécessaire — SysWarden scanne les services en écoute et n'active que les jails pertinentes. SSH système est sur un port non-standard, ce qui réduit naturellement le bruit.

### Docker : whitelisting des réseaux internes

`SYSWARDEN_USE_DOCKER="y"` est impératif. Sans ça, les règles Default-Deny de SysWarden bloquent le trafic inter-conteneurs sur les bridges Docker (`172.16.0.0/12`). SysWarden détecte automatiquement les interfaces `docker0` et les réseaux actifs et les exclut de ses règles de filtrage.

### GeoIP : liste ciblée, pas totale

Le géoblocage est activé sur les pays les plus représentés dans les logs du honeypot et les décisions CrowdSec, pas sur une liste exhaustive qui produirait des faux positifs. La liste est revue manuellement à chaque mise à jour des stats Metabase.

### WireGuard : accès admin cloisonné

SSH système n'écoute plus que sur l'interface `wg0` après activation de WireGuard. L'administration de l'hôte passe uniquement par le tunnel VPN — le port SSH physique n'est plus exposé publiquement. Le port WireGuard `:51820` reste ouvert sur l'IP publique.

## Installation

### Prérequis

- Debian 12 Bookworm (ARM64 — Freebox Ultra)
- Accès root
- Stack Docker active ([securehomelab](https://github.com/Jager-29/securehomelab) déployée)
- GitHub CLI `gh` si vérification de l'attestation souhaitée

### 1. Télécharger et vérifier SysWarden

```bash
# Télécharger le paquet .deb et son checksum depuis les releases officielles
wget https://github.com/duggytuxy/syswarden/releases/download/v0.39.3/syswarden_0.39.3_all.deb
wget https://github.com/duggytuxy/syswarden/releases/download/v0.39.3/SHA256SUMS.txt

# Vérifier l'intégrité
sha256sum -c SHA256SUMS.txt --ignore-missing
```

Pour les environnements qui exigent une vérification de la chaîne d'approvisionnement :

```bash
gh attestation verify syswarden_0.39.3_all.deb --owner duggytuxy
```

### 2. Installer le paquet

```bash
apt-get install -y ./syswarden_0.39.3_all.deb
```

### 3. Déployer la configuration

Copier le fichier de configuration de ce dépôt vers le chemin attendu par SysWarden :

```bash
# Éditer les variables sensibles avant de copier (IP admin, clé AbuseIPDB)
cp syswarden-auto.conf /opt/syswarden/syswarden-auto.conf
chmod 600 /opt/syswarden/syswarden-auto.conf

# Lancer l'installation non interactive
syswarden /opt/syswarden/syswarden-auto.conf
```

### 4. Vérifier le déploiement

```bash
# Vérifier que la table SysWarden est active dans nftables
nft list ruleset | grep syswarden_table

# Vérifier que les jails Fail2ban sont actives
fail2ban-client status

# Vérifier que les tables CrowdSec sont toujours intactes
nft list ruleset | grep crowdsec

# S'assurer que les conteneurs Docker communiquent toujours
docker compose -f /path/to/securehomelab/docker-compose.yaml ps
```

### 5. Vérifier WireGuard (si activé)

```bash
wg show
# Vérifier l'accès SSH via le tunnel avant de fermer la session courante
ssh -p <port> user@<wg_ip>
```

## Mises à jour

SysWarden se met à jour via le paquet `.deb`. Vérifier les releases upstream avant chaque mise à jour, notamment le changelog des règles Fail2ban et des changements de tables nftables qui pourraient impacter la coexistence avec CrowdSec.

```bash
# Vérifier la version installée
syswarden --version

# Mettre à jour (remplacer la version)
wget https://github.com/duggytuxy/syswarden/releases/latest/download/SHA256SUMS.txt
# ... puis répéter les étapes 1 à 3
```

## Licence

Ce dépôt (documentation et configuration) est sous licence MIT.
SysWarden lui-même est distribué sous [GPLv3](https://github.com/duggytuxy/syswarden/blob/main/LICENSE) par [@duggytuxy](https://github.com/duggytuxy).
