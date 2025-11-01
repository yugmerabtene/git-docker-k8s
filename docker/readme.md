## Présentation du cours

**Description générale.** Étude structurée et exhaustive de l’écosystème Docker : images (modèle OCI), exécution des conteneurs, stockage, réseau, construction (BuildKit), orchestration locale (Docker Compose), gestion de registry et distribution, sécurité/durcissement (incluant hardening du démon), observabilité/diagnostic, performance/optimisation, CI/CD et exploitation « prod-ready » sans orchestrateur, avec un pont optionnel vers Swarm/Kubernetes.
**Prérequis généraux.** Bases Linux (shell, permissions, processus), notions réseau (IP/CIDR, ports, DNS), Git + VS Code, Docker installé (Engine/Desktop).
**Compétences visées (extraits).** Construire et publier des images reproductibles et traçables, opérer des conteneurs en conditions réelles (sécurité, réseau, stockage), organiser une supply-chain d’images (registry, immutabilité, signatures, SBOM/provenance), concevoir et exploiter une stack multi-services (Compose) en contexte de production.

---

# Chapitre-00 — Installation & Environnement

**Objectifs d’apprentissage.**

* Installer et vérifier un environnement Docker fonctionnel (Windows/WSL2, Linux, macOS).
* Configurer le démon (`daemon.json`) et le client (`~/.docker/config.json`), proxys & CAs internes.
* Poser les bases de hardening du démon (accès/API).

**Contenu**

* Vérifications : VT-x/AMD-V, RAM/CPU, disque, BIOS/UEFI.
* Installation : Windows/WSL2 (activation, noyau), Docker Desktop (ressources, intégration WSL) ; Linux (dépôts officiels, Engine/CLI, systemd, groupe `docker`) ; macOS (Docker Desktop).
* Contrôles : `docker info`, `docker version`, `docker system df`.
* `daemon.json` : `data-root`, `storage-driver`, `log-driver`/`log-opts`, `registry-mirrors`, `insecure-registries` (à proscrire hors lab), `default-address-pools`, `bip`, `features`, `iptables`/`ip6tables`, `live-restore`.
* `~/.docker/config.json` : `credsStore`, `auths`, entêtes HTTP, préférences CLI.
* Proxys d’entreprise : `HTTP(S)_PROXY`, `NO_PROXY` (env + drop-in systemd).
* PKI & trust stores : ajout de CAs internes côté hôte et pour registres privés.
* API Docker sécurisée (aperçu) : exposition TLS, limitation de la socket (`docker-socket-proxy`), RBAC via sudoers (pointe vers Chap. 08).

---

# Chapitre-01 — Images Docker

**Objectifs d’apprentissage.**

* Maîtriser le modèle OCI (layers, manifest, digest, index multi-arch).
* Construire, taguer, pousser/tirer des images reproductibles.
* Inspecter et réduire l’empreinte (taille, couches).

**Contenu**

* Nommage : `[registry]/[namespace]/repo:tag` et `@sha256:…`.
* `docker images|image ls` (filtres/format, digests), `docker pull` (`--platform`, `--all-tags`, digest).
* `docker build` (BuildKit), `.dockerignore`, labels OCI (`org.opencontainers.image.*`), pinning des versions.
* Multi-stage (intro) & cache de build.
* `docker tag`, stratégies de versionning (SemVer, canaux `rc/stable`, politique de `latest`).
* `docker push` (bonnes pratiques de publication).
* `docker inspect`, `docker history` (analyse des couches).
* Portabilité : `docker save/load` vs `export/import`.
* Hygiène : `image prune`, `system df/prune`.
* Gouvernance du cycle de vie (aperçu) : choix d’images de base, suivi CVE amont, cadence d’actualisation (renvoi Chap. 07 & Annexe C).

---

# Chapitre-02 — Conteneurs (cycle de vie & exécution)

**Objectifs d’apprentissage.**

* Gérer le cycle de vie complet et les options d’exécution.
* Instrumenter : logs, exec, stats, healthchecks.
* Appliquer limites de ressources et terminaison propre.

**Contenu**

* Cycle de vie : `create/run/start/stop/restart/kill/rm`, `--rm`, `--init`.
* Modes : interactif/TTY (`-it`) vs détaché (`-d`), `--name`, `--restart`.
* Environnement & contexte : `-e`, `--env-file`, `-u`, `--workdir`, override ENTRYPOINT/CMD.
* Observabilité : `logs` (tail/since/follow), `stats`, `top`, `port`, `inspect`, `diff`, `events`.
* Limites : `--cpus`, `--cpu-shares`, `--cpuset-cpus`, `--memory`, `--pids-limit`, `--ulimit`.
* Healthchecks : `--health-cmd/*` (états `healthy/unhealthy`).
* Signaux et PID1, délais d’arrêt (`stop -t`), `docker update`.
* Accès matériel & sécurité de base : `--device`, éviter `--privileged`, principe de moindre privilège (renvoi Chap. 08).

---

# Chapitre-03 — Storage (volumes, bind mounts, tmpfs)

**Objectifs d’apprentissage.**

* Choisir et configurer bind/volume/tmpfs selon les cas.
* Gérer permissions (UID/GID) et SELinux.
* Concevoir sauvegarde/restauration.

**Contenu**

* Rappel overlay2 et writable layer.
* Volumes nommés : `volume create/ls/inspect/rm/prune`, pré-population.
* Bind mounts vs `--mount`, options `ro`, propagation (`rprivate/rshared`).
* Tmpfs (mémoire) pour runtime/caches.
* SELinux `:Z/:z`, gestion UID/GID, `-u`.
* NFS/SMB : driver local vs montage hôte.
* Sauvegarde/restauration (archives tar via conteneur utilitaire), plan de rotation.
* Chiffrement & quotas (approches) : FS chiffré (LUKS/ZFS), quotas FS/plugin.
* Durcissement : `--read-only` + `tmpfs` pour `/run` et `/tmp`.

---

# Chapitre-04 — Réseau Docker

**Objectifs d’apprentissage.**

* Configurer des réseaux user-defined et l’IPAM.
* Publier ports TCP/UDP, maîtriser le NAT.
* Segmenter (multi-réseaux) et employer alias/IP statiques.
* Contrôler l’egress/ingress et intégrer IPv6/mTLS (notions).

**Contenu**

* `docker0` vs bridges user-defined (isolation, DNS interne).
* `network ls/create/inspect/connect/disconnect/rm`.
* IPAM : `--subnet`, `--gateway`, `--ip-range`, IPs statiques.
* Publication de ports : `-p hostIP:hostPort:containerPort[/proto]`, `-P`.
* Modes : `host`, `none`, réseaux `--internal`.
* Alias : `--network-alias` ; concepts `macvlan/ipvlan` (aperçu).
* Contrôles egress/ingress (hors orchestrateur) : iptables/nftables par réseau, micro-segmentation, deny-by-default.
* mTLS & chiffrement interne (notions) : terminaison TLS côté proxy + mTLS pour services sensibles.
* IPv6 & dual-stack : activation, plan d’adressage, MTU, hairpin NAT, DNS de repli.
* Diagnostic : `docker port`, `ss/netstat`, iptables/nftables, hairpin NAT, MTU, netshoot.

---

# Chapitre-05 — Dockerfile & Build (BuildKit avancé)

**Objectifs d’apprentissage.**

* Écrire des Dockerfiles multi-stage optimisés et reproductibles.
* Exploiter `RUN --mount` (cache/secret/ssh) et `buildx` multi-arch.
* Normaliser métadonnées (labels OCI), SBOM & provenance.

**Contenu**

* Instructions : `FROM`, `RUN`, `COPY/ADD`, `ENV/ARG`, `USER`, `WORKDIR`, `EXPOSE`, `ENTRYPOINT/CMD`, `HEALTHCHECK`.
* Multi-stage : séparation builder/runtime, réduction de surface d’attaque.
* BuildKit : `--mount=type=cache`, `type=secret`, `type=ssh`.
* `.dockerignore`, pinning versions/digests, `USER` non-root.
* Labels OCI (source, version, revision…).
* `buildx` : builder, `--platform`, cache exporter/importer, `--provenance`, `--sbom`.
* Standards SBOM : SPDX & CycloneDX (formats, stockage comme artefacts OCI).
* Qualité : hadolint (lint), `dive` (analyse de couches).
* Compliance licences (notion) et politiques d’import d’images tierces.

---

# Chapitre-06 — Docker Compose v2 (multi-services)

**Objectifs d’apprentissage.**

* Décrire une stack : services, réseaux, volumes, secrets/configs.
* Gérer environnements, profils, overrides, dépendances.
* Opérer avec les commandes Compose.

**Contenu**

* `compose.yaml` : `services`, `networks`, `volumes`, `secrets`, `configs`.
* Services : `image/build`, `command`, `environment/env_file`, `ports`, `restart`, `healthcheck`, `depends_on`.
* Réseaux/volumes : déclarations, `external`, IPAM, ségrégation front/back.
* Secrets/configs : sources, montages, portée (pas de secrets dans les layers).
* Profils (`profiles`) et overrides (`-f` multiples).
* Dev vs prod : bind-mounts vs images, logs, ressources.
* Commandes : `compose up/down/ps/logs/exec/config/watch`.

---

# Chapitre-07 — Registry & Distribution

**Objectifs d’apprentissage.**

* Authentifier, publier et tirer des images (public/privé).
* Déployer un registry privé sécurisé (TLS+auth), proxy cache, GC.
* Gérer gouvernance & politiques (immutabilité, rétention, conformité).
* Intégrer SBOM/provenance et signatures d’images.

**Contenu**

* OCI registry : repo, tag, digest, manifest list (multi-arch).
* Authentification : `docker login`, PAT, tokens CI.
* Stratégies de tags : SemVer, canaux, digest en prod.
* Registry privé : `registry:2` + reverse-proxy (TLS), Basic Auth.
* Proxy cache Docker Hub, miroirs côté client.
* Suppression & garbage-collect, rétention, sauvegarde backend.
* Gouvernance cycle de vie : immutabilité des tags, rétention, EOL, nomenclature repositories (cf. Annexe C).
* Compliance & licences : scans de licences, règles d’import/export d’images tierces.
* Harbor/Nexus/Artifactory (aperçu) : RBAC, scans, immutabilité, réplication, admission par signature.
* SBOM & provenance (buildx), signatures cosign, vérification/politiques (gates registry/CI).

---

# Chapitre-08 — Sécurité & Durcissement

**Objectifs d’apprentissage.**

* Appliquer le moindre privilège au runtime et durcir le démon & les accès.
* Sécuriser builds/images (scans, SBOM, signatures).
* Situer les pratiques dans les référentiels (CIS, ISO).

**Contenu**

* Modèle de menace et gestion des secrets (Vault/SOPS/Secret Manager, pas de secret dans les layers, rotation/révocation, `tmpfs`).
* Runtime : `-u`, `--cap-drop/--cap-add`, `--read-only`, `tmpfs`, `--security-opt` (seccomp/AppArmor/SELinux), `no-new-privileges`.
* Hardening du démon & des accès : API TLS client/serveur, restriction de la socket (`docker-socket-proxy`), sudoers/RBAC, journalisation/audit des commandes.
* Rootless & userns-remap (principes/limites).
* Scans : Trivy/Docker Scout, SBOM (SPDX/CycloneDX), plan de remédiation priorisé.
* Immutabilité des tags, signatures cosign, contrôle d’admission (policies).
* Exposition contrôlée : reverse-proxy, egress/ingress (lien Chap. 04), journaux d’accès.
* Référentiels : CIS Docker Benchmark, ISO/IEC 27001 (Annex A).
* Forensic (notion) : à détailler Annexe B (captures, gel des preuves).

---

# Chapitre-09 — Observabilité & Diagnostic

**Objectifs d’apprentissage.**

* Centraliser logs, collecter métriques, initier le tracing.
* Diagnostiquer CPU/MEM/IO/réseau et incidents applicatifs.
* Formaliser runbooks de diagnostic rapide.

**Contenu**

* Logs : drivers (`local`, `json-file`), rotation, centralisation (Fluent Bit/ELK).
* Métriques : cAdvisor/Node-Exporter → Prometheus/Grafana (aperçu).
* Tracing : principes OpenTelemetry/Jaeger (patterns sidecar).
* Outils : `docker events`, `stats`, `inspect`, netshoot, `tcpdump`, `strace`, `nsenter`.
* Incidents fréquents : OOMKill, throttling CPU, FDs, DNS, conflits de ports.
* Runbooks : symptômes, commandes minimales, critères de succès.
* Forensic (rappel) : où récupérer journaux, images/layers, volumes (cf. Annexe B).

---

# Chapitre-10 — Performance & Optimisation

**Objectifs d’apprentissage.**

* Réduire la taille des images et accélérer builds/démarrages.
* Optimiser I/O overlay2 et trafic réseau.
* Mettre en place caches de build et miroirs de registry.
* Approfondir CPU/NUMA & pinning.

**Contenu**

* Choix d’images de base (distroless/alpine/slim), glibc vs musl.
* Ordre des instructions, cache layers, multi-stage agressif.
* Overlay2 : patterns d’écriture, options de montage, hot paths en `tmpfs`.
* Démarrage : pre-pull, warm/cold start, parallélisme de pulls.
* `buildx` : cache exporter/importer (local/remote), QEMU/binfmt, multi-arch.
* Réseau : MTU, buffers, tuning kernel (aperçu).
* CPU pinning & NUMA : `cpuset`, affinité, contention, isolation CPU.
* Nettoyage & rétention : images/volumes, pull-through cache côté registry.

---

# Chapitre-11 — CI/CD avec Docker

**Objectifs d’apprentissage.**

* Automatiser build/push/test/scan/signature & publication d’artefacts.
* Produire SBOM/provenance et promouvoir par digest.
* Sécuriser secrets et accès registre en CI.
* Introduire SLSA/in-toto & Rekor (policies).

**Contenu**

* Pipelines (GitHub Actions/GitLab/Jenkins) : structure type.
* Buildx multi-arch, caches de build, tests, scans.
* SBOM/provenance (artefacts OCI), signatures cosign (clé/Keyless OIDC).
* SLSA / in-toto / journal de transparence (Rekor) : concepts & intégration de politiques/gates.
* Stratégies de release : tags immuables, canaux, promotion digest.
* Secrets en CI : moindre privilège, rotation, OIDC vs PAT.
* Compliance licences en pipeline (scans, rapports).

---

# Chapitre-12 — Exploitation en Production (sans orchestrateur)

**Objectifs d’apprentissage.**

* Opérer des stacks Compose en production de manière sécurisée.
* Assurer TLS/renouvellement et redéploiements maîtrisés.
* Organiser sauvegardes/rollback, supervision et gestion de flotte.
* Capacité & PRA léger.

**Contenu**

* Compose « prod » : images versionnées/digest, healthchecks, ressources.
* Reverse-proxy TLS (Nginx/Traefik), ACME/Let’s Encrypt, renouvellement.
* Zéro-downtime basique (blue/green), canary minimal, ordonnancement.
* Sauvegardes/rotation/restauration de volumes (planification).
* Intégration `systemd` (unit files), journaux système, démarrage au boot.
* Gestion de flotte : systemd templated units, Ansible, stratégies d’updates, watchtower (avantages/risques), fenêtres de maintenance.
* Plan de capacité : `/var/lib/docker` (stockage), croissance du registry, compute CI (runners).
* PRA léger : procédures de reprise, documentation d’exploitation, jeu de preuves (lien Annexe B).

---

# Chapitre-13 — Swarm (optionnel) & Passerelle vers Kubernetes

**Objectifs d’apprentissage.**

* Découvrir Swarm (stacks, secrets/configs, rolling updates).
* Cartographier Compose → Kubernetes.
* Préparer une migration progressive.

**Contenu**

* Swarm : init, managers/workers (concepts), `stack deploy`.
* Services : réplicas, update config, healthchecks, contraintes de placement.
* Secrets/configs Swarm, networks overlay (chiffrement : aperçu).
* Cartographie : Compose → Deployments/Services/Ingress/Secrets/ConfigMaps.
* Limites/bonnes pratiques, critères de bascule vers Kubernetes.

---

## Annexes

### Annexe A — Air-gapped & mirroring (environnements contraints)

**Objectifs.** Exploiter Docker sans Internet : miroirs/pull-through cache, synchronisation d’artefacts OCI, outils alternatifs.

**Contenu**

* Registry proxy/cache (registry:2, Harbor) + configuration clients (mirrors).
* Synchronisation offline : `skopeo`, `crane`, `oras` (images & artefacts OCI : SBOM/attestations).
* Politique de publication interne (référentiel unique de vérité).
* Sécurité : signatures & vérification offline, rotation d’artefacts.

### Annexe B — Forensic & réponse à incident (IR)

**Objectifs.** Capturer, préserver et analyser les preuves en environnement conteneurisé.

**Contenu**

* Capture d’artefacts : `docker save` (image), export de couches, journaux bruts (`/var/lib/docker/containers/*.log`), FS des volumes (archives tar), horodatage/NTP.
* Bonnes pratiques : éviter `docker commit`, gel/duplication des preuves, chaîne de possession.
* Outillage : netshoot/tcpdump, `strace`, `nsenter`, `docker events` historisés.
* Cartographie ISO 27001 (preuves d’audit), intégration avec runbooks (Chap. 09) et PRA (Chap. 12).

### Annexe C — Politique de nommage, immutabilité & rétention d’images

**Objectifs.** Normaliser la gouvernance des images.

**Contenu**

* Conventions de nommage : `[org]/[projet]/[service]`, préfixes d’environnements.
* Versionning : SemVer, canaux (`dev/rc/stable`), interdiction de dé-tagger des versions prod.
* Immutabilité : tags critiques immuables, déploiement par digest.
* Rétention : délais par environnement, GC planifié, EOL & archivage.
* Conformité : revue licences, provenance, evidence SBOM/signatures stockées.

