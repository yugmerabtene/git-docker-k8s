# **Chapitre 5 — Sécurité et durcissement du cluster Kubernetes**

*(RBAC, Service Accounts, Secrets, Policies, TLS, Network Security)*

---

## **1. Objectifs d’apprentissage**

À la fin de ce chapitre, l’apprenant sera capable de :

* Comprendre les **mécanismes de sécurité intégrés** à Kubernetes.
* Mettre en œuvre le **contrôle d’accès basé sur les rôles (RBAC)**.
* Gérer les **identités applicatives** via les **ServiceAccounts**.
* Sécuriser les **informations sensibles** avec **Secrets**.
* Mettre en place des **politiques réseau (Network Policies)**.
* Activer et configurer la **sécurité TLS**, la **journalisation des audits** et la **sécurisation des communications**.
* Poursuivre le **projet fil rouge** (Phase 4) en durcissant le cluster et les Pods de l’application web.

---

## **2. Principes fondamentaux de la sécurité Kubernetes**

### 2.1 Modèle de sécurité à plusieurs couches

Kubernetes applique une **sécurité en profondeur (Defense in Depth)** :

1. **Authentification (AuthN)** : identifier *qui* fait la requête.
2. **Autorisation (AuthZ)** : déterminer *ce qu’il peut faire*.
3. **Admission Control** : appliquer ou rejeter les actions (validation/mutation).
4. **Sécurité réseau** : limiter la communication entre Pods et Services.
5. **Sécurité des secrets** : protéger les données sensibles (mots de passe, clés).
6. **Audit et traçabilité** : journaliser les accès et modifications.

### 2.2 Enjeux du durcissement

Un cluster mal configuré expose des risques majeurs :

* Escalade de privilèges (Pods en mode root).
* Exposition de Secrets dans les images ou logs.
* Réseau non segmenté → mouvements latéraux entre namespaces.
* Accès direct à l’API sans authentification forte.

---

## **3. Contrôle d’accès RBAC (Role-Based Access Control)**

### 3.1 Principe

* RBAC contrôle les droits d’accès aux ressources Kubernetes.
* Les autorisations sont définies via des **objets YAML** :

  * **Role** (au sein d’un namespace).
  * **ClusterRole** (global à tout le cluster).
  * **RoleBinding / ClusterRoleBinding** (lient les rôles aux utilisateurs ou groupes).

### 3.2 Exemple logique de hiérarchie

```
Utilisateur/ServiceAccount
   │
   ▼
RoleBinding (namespace)
   │
   ▼
Role (autorisations sur objets spécifiques)
```

### 3.3 Commandes d’inspection

```bash
kubectl get roles,rolebindings -A
kubectl auth can-i list pods --as=system:serviceaccount:default:sa1
```

### 3.4 Bonnes pratiques RBAC

* Appliquer le **principe du moindre privilège**.
* Créer des **rôles spécifiques par namespace**.
* Utiliser des **ClusterRoles** uniquement pour les administrateurs.
* Auditer les droits régulièrement (`kubectl get clusterrolebindings`).

---

## **4. Service Accounts et identités applicatives**

### 4.1 Définition

* Un **ServiceAccount (SA)** est une identité utilisée par un Pod pour interagir avec l’API Kubernetes.
* Chaque namespace contient par défaut un `default` ServiceAccount.
* On peut créer un SA dédié pour isoler les permissions d’une application.

### 4.2 Fichiers associés

* Token JWT monté automatiquement dans `/var/run/secrets/kubernetes.io/serviceaccount/`.
* Autorisations définies via **RoleBinding**.

### 4.3 Commandes utiles

```bash
kubectl create serviceaccount webapp-sa
kubectl get serviceaccount
kubectl describe sa webapp-sa
```

---

## **5. Secrets et gestion sécurisée des données sensibles**

### 5.1 Utilité

* Les **Secrets** stockent des informations sensibles : mots de passe, clés API, certificats TLS.
* Ils sont encodés en **Base64** et peuvent être :

  * montés dans un Pod sous forme de fichier, ou
  * injectés comme variables d’environnement.

### 5.2 Commandes

```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=1234
kubectl get secrets
kubectl describe secret db-secret
```

### 5.3 Bonnes pratiques

* Activer le **chiffrement at-rest** dans la configuration du kube-apiserver :

  ```
  --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
  ```
* Interdire les accès en clair via `kubectl get secrets -o yaml`.
* Utiliser **Vault** ou **SealedSecrets** pour la gestion externe.

---

## **6. Network Policies (Politiques réseau)**

### 6.1 Rôle

* Les **Network Policies** définissent les règles de communication entre Pods.
* Par défaut, tous les Pods communiquent librement.
* Une politique permet de **restreindre le trafic entrant (Ingress)** et **sortant (Egress)**.

### 6.2 Structure YAML type

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### 6.3 Points importants

* Nécessite un **CNI compatible** (Calico, Cilium, Weave).
* Chaque règle s’applique au niveau des Pods sélectionnés.
* Il faut définir explicitement les flux autorisés.

### 6.4 Bonnes pratiques

* **Isolation stricte** entre namespaces (frontend ↔ backend).
* Autoriser uniquement les flux nécessaires (`allow from namespace=backend`).
* Documenter les politiques et les tester avec `netshoot`.

---

## **7. TLS, audit et sécurité du plan de contrôle**

### 7.1 Sécurité TLS

* Toutes les communications entre composants utilisent TLS :

  * API Server ↔ etcd
  * API Server ↔ kubelet
  * Clients ↔ API Server
* Certificats stockés dans `/etc/kubernetes/pki/`.

### 7.2 Vérification des certificats

```bash
kubeadm certs check-expiration
```

### 7.3 Audit des accès

* Activer la journalisation des actions API :

  ```
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml
  --audit-log-path=/var/log/kubernetes/audit.log
  ```
* Permet de retracer les actions utilisateurs (compliance ISO, RGPD).

---

## **8. LAB – Projet Fil Rouge (Phase 4)**

### Sécurisation complète de l’application web sur le cluster local

---

### 8.1 Objectif

Durcir le projet du **Chapitre 4** :

* Appliquer un modèle RBAC minimaliste.
* Isoler le trafic réseau entre frontend et backend.
* Protéger les secrets et restreindre les accès.

---

### 8.2 Prérequis

* Cluster Minikube opérationnel.
* Application du fil rouge déjà déployée :

  * Namespace : `projet-fil-rouge`
  * Deployments : `frontend`, `backend`
* CNI compatible (Flannel ou Calico).

---

### 8.3 Étape 1 — Créer un ServiceAccount dédié

```bash
kubectl create serviceaccount web-sa -n projet-fil-rouge
kubectl get sa -n projet-fil-rouge
```

Associer un rôle de lecture :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: projet-fil-rouge
  name: read-pods
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-read
  namespace: projet-fil-rouge
subjects:
- kind: ServiceAccount
  name: web-sa
  namespace: projet-fil-rouge
roleRef:
  kind: Role
  name: read-pods
  apiGroup: rbac.authorization.k8s.io
```

Appliquer :

```bash
kubectl apply -f rbac-web.yaml
```

---

### 8.4 Étape 2 — Créer un Secret sécurisé

```bash
kubectl create secret generic api-key --from-literal=token=AZERTY123 -n projet-fil-rouge
```

Montage dans le backend :

```yaml
env:
- name: API_TOKEN
  valueFrom:
    secretKeyRef:
      name: api-key
      key: token
```

---

### 8.5 Étape 3 — Ajouter une politique réseau

Créer `network-policy.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-backend
  namespace: projet-fil-rouge
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
  policyTypes:
  - Ingress
```

Appliquer :

```bash
kubectl apply -f network-policy.yaml
```

Vérifier :

```bash
kubectl describe netpol allow-frontend-backend -n projet-fil-rouge
```

---

### 8.6 Étape 4 — Vérification globale

```bash
kubectl get all -n projet-fil-rouge
kubectl get secrets -n projet-fil-rouge
kubectl get networkpolicy -n projet-fil-rouge
kubectl auth can-i get pods --as=system:serviceaccount:projet-fil-rouge:web-sa
```

**Résultats attendus :**

* L’application fonctionne normalement.
* Les flux sont limités au namespace et au couple frontend-backend.
* Les accès sont contrôlés par RBAC et SA.
* Les secrets ne sont visibles que pour les Pods autorisés.

---

## **9. Bonnes pratiques de sécurité**

* **Principe du moindre privilège** sur RBAC et ServiceAccounts.
* **Rotation des certificats et tokens**.
* **Activation du chiffrement at rest** pour les Secrets.
* **Audit logging activé** (traçabilité des actions).
* **Réseau cloisonné** via Network Policies.
* **Sécurité applicative** (images signées, scans automatiques, CI/CD).

---

## **10. Résumé pour diapo**

### 1. Objectifs

* Sécuriser le cluster Kubernetes local.
* Protéger les identités, secrets et flux réseau.
* Mettre en place RBAC, SA, TLS et politiques réseau.

### 2. Mécanismes clés

* **RBAC** : contrôle d’accès basé sur les rôles.
* **ServiceAccount** : identité applicative.
* **Secrets** : gestion sécurisée des données sensibles.
* **Network Policies** : isolation du trafic.
* **TLS / Audit logs** : sécurité du plan de contrôle.

### 3. LAB – Phase 4 du projet fil rouge

* Création d’un **ServiceAccount dédié** et rôle restreint.
* Ajout d’un **Secret** pour le backend.
* Définition d’une **NetworkPolicy** limitant les flux frontend → backend.
* Résultat : cluster **durci, cloisonné et conforme aux bonnes pratiques**.
