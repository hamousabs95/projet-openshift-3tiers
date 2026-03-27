# 🖥️ TP OpenShift Virtualization
## Architecture Hybride : 2 VMs Ubuntu + 1 Conteneur Serveur Web

> **Namespace :** `hamousabs-dev`  
> **Plateforme :** Red Hat OpenShift Developer Sandbox

---

## 📐 Architecture

```text
                        INTERNET
                            │
                   ┌────────▼────────┐
                   │  VM1 - Router   │  ← VirtualMachine OpenShift
                   │  Ubuntu Jammy   │
                   │  iptables       │
                   │  (Firewall/NAT) │
                   └────────┬────────┘
                            │
              ┌─────────────┼──────────────┐
              │             │              │
     ┌────────▼──────┐     │    ┌─────────▼────────┐
     │  VM3 - DB     │     │    │  Conteneur Web   │
     │  MySQL 8.0    │◄────┘    │  + Node.js/Nginx │
     │  (Base de DB) │          │  (Serveur Web)   │
     │ VirtualMachine│          │  Pod OpenShift   │
     └───────────────┘          └──────────────────┘
                                        │
                                 Route publique
                                 (HTTPS OpenShift)


projet-openshift-3tiers/
├── .github/
│   └── workflows/
│       └── deploy.yml                  ← Pipeline CI/CD
├── network/
│   └── networkpolicy.yaml              ← Règles réseau (policies)
├── vms/
│   ├── vm1-router/
│   │   └── virtualmachine.yaml         ← VM Ubuntu + iptables
│   └── vm3-db/
│       ├── pvc.yaml                    ← Volume persistant MySQL
│       └── virtualmachine.yaml         ← VM Ubuntu + MySQL
├── k8s-web/
│   ├── deployment.yaml                 ← Nginx + Node.js
│   └── service.yaml                    ← Service interne (ClusterIP)
└── README.md