# Cahier des Charges – Application Psy2Bib

## Projet : Challenge Startup – Application de prise de rendez-vous psychologiques sécurisée et Zero-Knowledge  
**Version : 1.0**

---

# 1. Présentation générale

## 1.1. Mission de Psy2Bib
Psy2Bib vise à révolutionner l’accès aux psychologues en offrant une plateforme sécurisée, confidentielle et fluide pour réserver des séances en cabinet ou en visioconférence, tout en garantissant un modèle Zero-Knowledge Privacy : aucune donnée sensible ne quitte jamais le navigateur en clair.

## 1.2. Proposition de valeur disruptive
Psy2Bib introduit un concept unique :  
- Lors des séances en visioconférence, le vrai visage du patient n’est jamais transmis.  
- L’utilisateur choisit un avatar animé (2D cartoon, 3D stylisé ou silhouette expressive), et la vidéo de la webcam est transformée localement en temps réel en avatar.  
- Le flux envoyé via WebRTC ne contient aucun élément biométrique.

## 1.3. Acteurs du système
- **Patient** : crée un compte, chiffre ses données, prend rendez-vous, participe aux séances (cabinet/visio).  
- **Psychologue** : gère ses disponibilités, reçoit les rendez-vous, assure la séance.  
- **Administrateur technique (DevOps / monitoring)** : supervise l’infra sans jamais accéder aux données sensibles (ZKP).

---

# 2. Spécifications fonctionnelles

## 2.1. Inscription / Connexion (E2EE)
- Création de compte avec e-mail + mot de passe.  
- Génération locale d’une clé AES-GCM dérivée du mot de passe via PBKDF2.  
- Stockage serveur :  
  - profil chiffré AES  
  - salt PBKDF2  
  - hash du mot de passe (Argon2id côté backend, indépendant de la clé E2EE)  
- Connexion : re-dérivation de la clé + vérification du hash.

## 2.2. Recherche de psychologues
Filtres : spécialité, créneaux disponibles, langue parlée, séance cabinet/visio.  
Résultats fournis par backend (aucune donnée sensible).

## 2.3. Réservation de rendez-vous
- Choix du créneau sur un calendrier synchronisé.  
- Modes : cabinet (adresse + plan d’accès) ou visioconférence (lien WebRTC sécurisé).  
- Détails stockés chiffrés côté serveur.

## 2.4. Gestion de calendrier
**Vue patient** : historique chiffré, prochaines séances.  
**Vue psychologue** : gestion des disponibilités, validation/refus des créneaux.

## 2.5. Visio sécurisée avec avatar
Voir section 4.

## 2.6. Messagerie patient/psy chiffrée
- Envoi/réception E2EE.  
- Serveur = relais de blobs chiffrés.

---

# 3. Architecture Zero-Knowledge

## 3.1. Schéma de circulation des clés
```
        +------------------------+
        |   Navigateur Client   |
        +------------------------+
                  |
     PW ---> PBKDF2(salt) ---> AES Key
                  |
                  v
       Chiffrement local (AES-GCM)
                  |
        Envoi blobs chiffrés
                  |
                  v
         +------------------+
         | Serveur NestJS  |
         | (aveugle total) |
         +------------------+
```

## 3.2. Génération de la clé
- Entrée : mot de passe  
- PBKDF2 (100k itérations / SHA-256 / salt aléatoire)  
- Sortie : clé AES-GCM 256 bits  
- Jamais transmise au backend

## 3.3. Données chiffrées
Identité du patient, notes, historique, messages, métadonnées sensibles.

## 3.4. Serveur aveugle (NestJS)
Stocke uniquement des blobs chiffrés, gère JWT, ne lit aucune donnée sensible.

## 3.5. Justification cryptographique
Le backend ne possède aucune information permettant de dériver la clé → Zero-Knowledge garanti.

---

# 4. Module Visio-Avatar

## 4.1. Pipeline technique
```
Webcam ---> Face Landmarks (MediaPipe)
            |
            v
     Avatar Rigging (Canvas/WebGL)
            |
            v
      Animation temps réel
            |
            v
        Encodage vidéo
            |
            v
         WebRTC Stream
```

## 4.2. Technologies
MediaPipe, Canvas/WebGL, WebRTC, WebAssembly (optionnel).

## 4.3. Sécurité
Le flux WebRTC contient uniquement l’avatar, aucun pixel réel de webcam.

## 4.4. Modes
- Cabinet : pas d’avatar  
- Visio : avatar obligatoire

---

# 5. Backend (NestJS)

## 5.1. Endpoints
### Auth
POST /auth/register  
POST /auth/login  
POST /auth/refresh  

### Patients
GET /patient/me  
PUT /patient/me  

### Psychologues
GET /psy/list  
GET /psy/:id  

### Rendez-vous
POST /appointments  
GET /appointments  
DELETE /appointments/:id  

### Messagerie
POST /messages  
GET /messages/thread/:id  

### WebRTC
POST /webrtc/offer  
POST /webrtc/answer  

## 5.2. DTOs stricts
Validation `class-validator`.  
Prévention des injections.

## 5.3. Authentification
JWT access (15 min), refresh (7 jours), rotation.

## 5.4. Logs
Sans données sensibles : uniquement IDs, timestamps, erreurs.

---

# 6. Base de données (PostgreSQL + RLS)

## 6.1. Modèle
```
users(id, email, password_hash, salt, encrypted_profile)
psychologists(id, public_profile)
appointments(id, patient_id, psy_id, encrypted_details, date)
messages(id, sender_id, receiver_id, encrypted_body, timestamp)
```

## 6.2. RLS
- users : accès limité à soi-même  
- appointments : patient ou psy  
- messages : sender ou receiver

## 6.3. Blocage cross-user
Toute violation → 403.

---

# 7. Infrastructure & DevOps

## 7.1. Docker
Containers : frontend, api, postgres+RLS, coturn, nginx.

## 7.2. Réseau
VLAN interne Docker, accès restreint.

## 7.3. CI/CD
npm audit, ESLint, Prettier, Gitleaks, Trivy, tests, déploiement staging.

---

# 8. Risques & Menaces

| Risque | Description | Mesures |
|--------|-------------|---------|
| XSS | Injection | CSP strict, sanitization |
| CSRF | Requêtes malveillantes | SameSite cookies |
| IDOR | Accès non autorisé | RLS + vérifs |
| Bruteforce | Attaque mot de passe | Rate limiting |
| MITM | Interception WebRTC | DTLS + SRTP |
| Fuite de clés | Mauvaise gestion locale | WebCrypto |

---

### Rôles
Frontend / API / DevOps.

---

# 9. Critères de validation (Demo Day)
- Admin incapable de lire les données  
- Visio-avatar opérationnel  
- RLS fonctionnel  
- Logs sécurisés  
- Aucun IDOR

