# Sovrizon V2 – Projet 3A

**Sovrizon V2** est un système de partage d'images chiffrées avec **chiffrement côté serveur**, développé dans le cadre de notre 3e année à Centrale Lyon. Cette version corrige une faille de sécurité majeure identifiée dans la V1 : **les clés de chiffrement ne quittent jamais le serveur**.

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.11+-3776AB.svg)](https://www.python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.109.0-009688.svg)](https://fastapi.tiangolo.com)
[![React](https://img.shields.io/badge/React-18.0+-61DAFB.svg)](https://reactjs.org)

---

##  Problématique et innovation

### Faille de sécurité identifiée (V1)

Dans la version précédente de Sovrizon, les clés de chiffrement étaient **transmises au client** (navigateur ou extension Chrome) pour permettre le chiffrement et déchiffrement des images côté client. Cette approche créait plusieurs risques :

-  **Interception des clés** pendant le transit (attaque Man-in-the-Middle)
-  **Extraction des clés** depuis le cache ou la mémoire du navigateur
-  **Sauvegarde non contrôlée** des images déchiffrées en local
-  **Dépendance à une extension** Chrome (pas de support mobile)

### Solution apportée (V2)

Sovrizon V2 adopte une approche radicalement différente : **tout le traitement cryptographique est effectué côté serveur** dans un composant isolé appelé **Tiers de Confiance**. Les clés de chiffrement ne circulent jamais sur le réseau et ne sont jamais accessibles au client.

**Principe de fonctionnement :**

1. **Upload** : Le client envoie l'image en clair → Le Tiers de Confiance chiffre côté serveur (AES-256-GCM) et applique un tatouage invisible → Stockage dans Secugram
2. **Consultation** : Le Tiers de Confiance déchiffre côté serveur → Envoie l'image temporairement au client (60 secondes) → Pas de cache local possible
3. **Sécurité** : Les clés restent dans le Tiers de Confiance, jamais transmises, jamais exposées

---

##  Dépôts principaux

Le projet est organisé en **3 repositories indépendants** (un par composant) :

###  [Tiers de Confiance](https://github.com/Marwanagr/Serveur-TDC)

Cœur de sécurité du système. Responsable de :
- **Authentification** : Génération et validation de tokens JWT expirables (1 heure)
- **Gestion des clés** : Création et stockage sécurisé des clés AES-256 (jamais transmises)
- **Chiffrement/Déchiffrement** : Traitement côté serveur uniquement
- **Tatouage numérique** : Application de filigranes invisibles pour traçabilité
- **Contrôle d'accès** : Vérification des autorisations avant déchiffrement
- **Traçabilité** : Logs détaillés de tous les accès (qui, quand, IP)
- **API REST** : 11 endpoints pour communication avec le frontend

**Technologies** : Python, FastAPI, MongoDB, Cryptography, OpenCV  
**Base de données** : 5 collections MongoDB (users, encryption_keys, images_metadata, tokens_actifs, logs_tracabilite)

---

###  [Backend Secugram](https://github.com/MalekChammakhi1/Secugram)

Système de stockage pur. Responsable de :
- **Stockage** : Sauvegarde des images chiffrées uniquement (aucune image en clair)
- **Métadonnées** : Informations sur les images (dates, propriétaire, format)
- **Récupération** : API pour que le Tiers de Confiance récupère les images chiffrées

**Principe de sécurité** : Secugram ne connaît **PAS** les clés de chiffrement. Il ne peut pas déchiffrer les images qu'il stocke.

**Technologies** : Python, FastAPI, MongoDB  
**Base de données** : 1 collection MongoDB (photos chiffrées)

---

###  [Frontend](https://github.com/khalfaECL/secugram_front_kh.git)

Interface utilisateur cross-platform. Responsable de :
- **Authentification** : Pages de connexion et inscription
- **Upload** : Interface d'upload avec formulaire d'autorisation (qui peut voir l'image ?)
- **Galerie** : Visualisation des images (chiffrées ou dévoilées)
- **Consultation éphémère** : Bouton "Dévoiler" pour affichage temporaire (60 secondes)
- **Gestion** : Modification des autorisations, suppression définitive

**Contraintes de sécurité** :
- Images affichées uniquement en mémoire (Blob URL)
- Pas de cache navigateur (headers `Cache-Control: no-store`)
- Token stocké en sessionStorage (expire à la fermeture du navigateur)
- Désactivation du clic droit et de l'enregistrement

**Technologies** : React, TypeScript, Axios  
**Support** : Web, iOS (React Native), Android (React Native)

---

##  Démarrage rapide

### Prérequis

- Python 3.11+
- Node.js 18+
- MongoDB 7.0+ (local ou MongoDB Atlas)

### Installation

```bash
# 1. Créer un dossier pour le projet
mkdir sovrizon && cd sovrizon

# 2. Cloner les 3 repositories
git clone https://github.com/votre-org/sovrizon-tiers.git tiers-de-confiance
git clone https://github.com/votre-org/sovrizon-secugram.git backend-secugram
git clone https://github.com/votre-org/sovrizon-frontend.git frontend

# 3. Installer le Tiers de Confiance
cd tiers-de-confiance
python -m venv venv
source venv/bin/activate  # Linux/Mac (ou venv\Scripts\activate sur Windows)
pip install -r requirements.txt
cp .env.example .env
# Éditer .env avec vos paramètres MongoDB et clés secrètes
cd ..

# 4. Installer le Backend Secugram
cd backend-secugram
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# Éditer .env avec vos paramètres MongoDB
cd ..

# 5. Installer le Frontend
cd frontend
npm install
cp .env.example .env
# Éditer .env avec l'URL de l'API (http://localhost:8300)
cd ..
```

### Lancement

**Ouvrir 3 terminaux** :

**Terminal 1 – Tiers de Confiance :**
```bash
cd tiers-de-confiance
source venv/bin/activate
uvicorn app.main:app --reload --port 8300
```

**Terminal 2 – Backend Secugram :**
```bash
cd backend-secugram
source venv/bin/activate
uvicorn app.main:app --reload --port 8000
```

**Terminal 3 – Frontend :**
```bash
cd frontend
npm start
```

### Accès

| Service | URL | Documentation |
|---------|-----|---------------|
| **Frontend** | http://localhost:3000 | Interface utilisateur |
| **API (Tiers de Confiance)** | http://localhost:8300/docs | Swagger interactif |
| **Backend Secugram** | http://localhost:8000/docs | Swagger interactif |

---

##  Garanties de sécurité

### Architecture à 3 tiers

```
┌──────────────────────────────────────────────────────┐
│                FRONTEND (Web/Mobile)                 │
│         Interface utilisateur cross-platform         │
└────────────────────┬─────────────────────────────────┘
                     │ HTTPS
                     ▼
┌──────────────────────────────────────────────────────┐
│         TIERS DE CONFIANCE + API : Coeur de Sécurité │
│   Chiffrement/Déchiffrement (AES-256-GCM)            │
│   Gestion des clés (jamais transmises)               │
│   Tatouage numérique invisible                       │
│   Contrôle d'accès et autorisations                  │
│   Traçabilité complète                               │
└────────────────────┬─────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────┐
│              BACKEND SECUGRAM                        │
│   Stockage images chiffrées uniquement               │
│   Ne connaît PAS les clés                            │
└──────────────────────────────────────────────────────┘
```

### Principes de sécurité implémentés

1. **Clés jamais transmises** : Les clés de chiffrement AES-256 sont générées et stockées uniquement dans le Tiers de Confiance. Le client ne les voit jamais.

2. **Chiffrement côté serveur** : Toutes les opérations cryptographiques (chiffrement, déchiffrement) sont effectuées dans un environnement serveur isolé et sécurisé.

3. **Tatouage numérique invisible** : Chaque image reçoit un filigrane invisible contenant l'identité du propriétaire, permettant la traçabilité en cas de fuite.

4. **Images éphémères** : Les images dévoilées sont affichées temporairement (60 secondes) en mémoire uniquement, sans possibilité de cache local.

5. **Tokens JWT expirables** : Les sessions utilisateur sont limitées à 1 heure. Reconnexion obligatoire après expiration.

6. **Traçabilité complète** : Tous les accès aux images sont enregistrés avec l'identité de l'utilisateur, la date, l'heure et l'adresse IP.

7. **Suppression définitive** : La suppression d'une image la détruit dans les 2 bases de données (Tiers + Secugram) et aucune copie locale n'existe.

---

##  Objectifs du projet

- **Corriger la faille V1** : Éliminer le risque d'interception des clés de chiffrement
- **Garantir la confidentialité** : Assurer que seul le serveur a accès aux clés
- **Contrôle total** : Donner au propriétaire le contrôle absolu sur qui peut voir ses images
- **Traçabilité** : Permettre l'audit des accès en cas de problème
- **Cross-platform** : Rendre le système accessible sur toutes les plateformes (Web, iOS, Android)

---

##  Technologies

**Backend (Tiers de Confiance + Secugram)** :
- Python 3.11+
- FastAPI 0.109.0 (framework web moderne et performant)
- MongoDB 7.0+ (base de données NoSQL)
- Cryptography 42.0.2 (algorithmes AES-256-GCM)
- OpenCV, NumPy, Pillow (traitement d'images et watermarking)
- PyJWT, python-jose (tokens JWT)

**Frontend** :
- React 18+
- TypeScript
- Axios (client HTTP)
- React Native (pour mobile iOS/Android)

**Déploiement** :
- Render / Railway (backend)
- Vercel / Netlify (frontend)
- MongoDB Atlas (base de données cloud)

---

##  Documentation détaillée

Chaque composant possède sa propre documentation complète :

- **[Tiers de Confiance - README](https://github.com/Marwanagr/Serveur-TDC/blob/main/README.md)** : Architecture interne, endpoints API, schémas de base de données, modules de sécurité
- **[Backend Secugram - README](https://github.com/MalekChammakhi1/Secugram/blob/main/README.md)** : API de stockage, schéma base de données
- **[Frontend - README](https://github.com/khalfaECL/secugram_front_kh/blob/main/README.md)** : Composants React, services API, gestion de l'éphémérité

---

##  Présentation académique

### Innovation principale

**Chiffrement côté serveur avec Tiers de Confiance isolé** : Contrairement aux solutions existantes (incluant notre V1) qui chiffrent côté client et transmettent les clés, Sovrizon V2 garantit que les clés de chiffrement **ne quittent jamais le serveur**, éliminant ainsi le vecteur d'attaque principal.

### Technologies et concepts mis en œuvre

- **Cryptographie** : AES-256-GCM (chiffrement authentifié), JWT HS256 (sessions)
- **Traitement d'images** : Watermarking invisible (LSB/DWT)
- **Architecture distribuée** : 4 tiers avec séparation des responsabilités
- **Sécurité applicative** : HTTPS, CORS, protection CSRF, expiration de sessions
- **Base de données** : MongoDB avec 6 collections (5 Tiers + 1 Secugram)
- **API REST** : 11 endpoints documentés avec Swagger

### Comparatif V1 vs V2

| Aspect | V1 | V2 |
|--------|----|----|
| Transmission des clés |  Oui (client) |  Jamais |
| Chiffrement | Client |  Serveur |
| Déchiffrement | Client |  Serveur |
| Tatouage |  Non |  Oui |
| Images éphémères |  Non |  60 secondes |
| Tokens | Sans expiration |  1 heure |
| Traçabilité | Limitée |  Complète |
| Support mobile |  Extension Chrome uniquement |  API REST universelle |

---

##  Équipe

**Projet académique – Centrale Lyon 2025**

- Marwane AGREBI
- Malek CHAMMAKHI
- Youssef KHALFA
- Amani KRID

---

##  License

MIT License – voir fichier [LICENSE](LICENSE)

---

##  Liens rapides

- 🔐 [Tiers de Confiance - Repository](https://github.com/Marwanagr/Serveur-TDC)
- 📦 [Backend Secugram - Repository](https://github.com/MalekChammakhi1/Secugram)
- 🎨 [Frontend - Repository](https://github.com/khalfaECL/secugram_front_kh.git)

---

© 2026 Sovrizon V2 – Tous droits réservés.

