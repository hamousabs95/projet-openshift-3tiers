# 🖥️ Projet OpenShift 3-tiers + KubeVirt

## Description générale

Ce projet met en place une architecture 3-tiers sur OpenShift avec une partie virtualisée (KubeVirt) et une partie conteneurisée :
- t1 : **VM routeur/firewall** (`vm1-pfsense`) via KubeVirt (iptables/NAT)
- t2 : **VM base de données MySQL** (`vm2-mysql`) via KubeVirt
- t3 : **Application web** conteneurisée dans un `Deployment` OpenShift Nginx + Node.js

Namespace utilisé : `hamousabs-dev`

Objectif : démontrer une infrastructure hybride (VMs + pods), routes réseau, services internes, et pipeline CI/CD.

---

## Arborescence du dépôt

```
projet-openshift-3tiers/
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions CI/CD
├── conteneur/
│   └── nginx/
│       ├── deployment.yaml     # Deployment Nginx + Node.js
│       └── mysql-service.yaml  # Service ClusterIP pour MySQL VM
├── network/
│   ├── multus-networks.yaml   # NetworkAttachmentDefinition pour DMZ/WAN
│   └── networkpolicy.yaml     # NetworkPolicy (allow-all-ingress)
├── vms/
│   ├── vm1-pfsense/
│   │   └── virtualmachine.yaml # VM routeur/NAT iptables
│   └── vm2-mysql/
│       └── virtualmachine.yaml # VM MySQL + setup DB
└── README.md
```

---

## 1) VM Routeur / Firewall (vm1-pfsense)

Chemin : `vms/vm1-pfsense/virtualmachine.yaml`

- Sur KubeVirt : machine Fedora
- 1 CPU, 2Gi mémoire
- Cloud-init : installation `iptables-services`, activation NAT, `net.ipv4.ip_forward=1`
- Rôle : point d’agrégation WAN/DMZ, NAT de sortie, filtrage basique

---

## 2) VM MySQL (vm2-mysql)

Chemin : `vms/vm2-mysql/virtualmachine.yaml`

- Sur KubeVirt : machine Fedora
- 1 CPU, 2Gi mémoire
- Cloud-init : install + config MySQL
  - root = `RootPass123!`
  - DB = `appdb`
  - user app = `appuser` / `AppPass123!`
  - table `test_connection` et donnée d’initialisation
  - configuration `bind-address=0.0.0.0` pour accès réseau

---

## 3) Application web (pod OpenShift)

Chemin : `conteneur/nginx/deployment.yaml`

Containers :
- `nodejs-app` (image `openshift/nodejs:18-ubi8`)
  - exécute `npm install express mysql2`, lit `app.js` via ConfigMap
  - variables d’environnement DB pointent vers `vm2-mysql-service`
- `nginx` (image `bitnami/nginx`) 
  - expose port 8080
  - configuration serveur via ConfigMap mount `nginx-config`

Service MySQL interne :
- `conteneur/nginx/mysql-service.yaml`
- `ClusterIP` `vm2-mysql-service` -> cible `vm2-mysql` (selector `kubevirt.io/domain=vm2-mysql`)

---

## 4) Réseau OpenShift

- `network/multus-networks.yaml` : deux réseaux Multus (`dmz-network`, `wan-network`) de type `ovn-k8s` layer2.
- `network/networkpolicy.yaml` : politique `allow-all-ingress` pour tous les pods (dev / test). À durcir en production.

---

## 5) CI/CD

- `.github/workflows/deploy.yml` : pipeline déclenché sur `push`.
- Actions : build, tests, déploiement vers cluster.

---

## Commandes utiles

- `git checkout main && git pull`
- `oc get vm,pod,svc,netpol -n hamousabs-dev`
- `oc get service vm2-mysql-service -n hamousabs-dev`
- `oc logs deploy/nginx-web -n hamousabs-dev`
- `curl http://<route-openshift>/`

---

## Notes de sécurité et évolutions

- Changer les mots de passe et utiliser des secrets OpenShift (au lieu de variables en dur).
- Remplacer `allow-all-ingress` par des policies strictes.
- Ajouter surveil & alerting (Prometheus/Grafana).
- Ajouter HA (réplicas, backup DB, sauvegarde d’images).

---

## Diagramme logique

```
[Internet]
   ↓
[Route OpenShift] ---> [Pod nginx + Nodejs]
                          ↓
                 [Service vm2-mysql-service]
                          ↓
                      [VM MySQL]

[VM Routeur (pfsense)]  (segment réseau DMZ/WAN)
```

---

