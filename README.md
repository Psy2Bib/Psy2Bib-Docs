# Psy2Bib – Whitepaper (Zero-Knowledge + Sécurité + Schémas)

## Auteurs 

- [@Ismail Ibrahimi](https://www.github.com/IsmailIbrahimi) - ismail.ibrahimi@efrei.net
- [@Nathan Radivoniuk](https://www.github.com/nathanrdvnk) - nathan.radivoniuk@efrei.net
- [@Christophe Cui](https://www.github.com/ZrChristophe) - christophe.cui@efrei.net
- [@Ismael Zaïdi](https://www.github.com/ismax26) - ismael.zaidi@efrei.net
- [@Hamza Nadifi](https://www.github.com/HamzaNADIFI07) - hamza.nadifi@efrei.net
 

 
## 1. Le Concept Métier – Killer App
 
**Psy2Bib : La thérapie en ligne 100% anonyme, protégée par le Zero-Knowledge et un avatar vidéo temps réel.**
 
- Le patient consulte un psychologue **sans jamais montrer son vrai visage**.
- Toutes les données psychologiques sont **chiffrées en local puis envoyé dans l'url du navigateur**.
- Le serveur NestJS est un **serveur aveugle** incapable de lire les données.
- Le flux WebRTC n’inclut **aucun pixel réel** : seulement un avatar animé généré localement.
 
---
 
## 2. La Donnée Critique – Ce que l’on protège
 
- Identité complète du patient  
- Notes du psychologue (diagnostic, commentaires, suivi)  
- Messages patient ↔ psy  
- Historique des rendez-vous  
- Informations psychologiques sensibles  
- Flux vidéo réel (jamais transmis)
 
**Tout ce qui touche à la santé mentale est traité comme secret absolu.**
 
---
 
## 3. Schéma Global du Chiffrement – Preuve que le serveur ne détient jamais la clé
 
```
                                [ Navigateur Client ]
                                +-------------------+
                Mot de passe -->|  PBKDF2(salt)     |--> Clé AES-256 locale
                                +-------------------+
                                        |
                                        | AES-GCM (chiffrement local)
                                        v
                                Blobs chiffrés (profil, notes, messages)
                                        |
                                        v
                                [ Serveur NestJS "aveugle" ]
                            Stocke uniquement des blobs chiffrés
```
 
Aucun secret ne quitte votre machine.
 
---
 
# 4. Fonctionnalité par Fonctionnalité  
## Avec schémas détaillés + risques + solutions + technologies utilisées
 
---
 
# 4.1. Authentification (Login / Register)
 
## Objectif
Séparer complètement :
- **l’authentification** (hash côté serveur),
- **le chiffrement des données** (clé dérivée côté client).
 
## Schéma du Flux (Auth)
 
```
[Client]                             [Serveur]
mot de passe -----> Argon2(hash) ----> stock hash_auth
   |  
   |--> PBKDF2(salt_e2ee) --> clé AES-256 (locale)
                 |
                 |-> chiffre profil -> encrypted_profile -> envoyé
```
 
Le serveur ne voit jamais :
- la clé AES
- le profil en clair
 
## Risques & Solutions
 
| Risque | Explication | Solution | Technologie |
|-------|-------------|----------|-------------|
| Bruteforce | Attaque sur les mots de passe | Argon2id côté serveur | Argon2id |
| Key guessing | Deviner clé E2EE | PBKDF2 + 100k itérations + salt | Web Crypto API |
| Vol du JWT | Token volé | Rotation + TTL court | NestJS JWT |
 
---
 
# 4.2. Données Patient – Profil, Dossier, Notes Psy
 
## Objectif
Stocker **toutes les données psychologiques** chiffrées côté client.
 
## Flux de Chiffrement
 
```
                [Browser]
profile.json ----> AES-GCM(key_e2ee)
                      |
                      v
encrypted_profile --------> [Serveur]
```
 
Le serveur voit :
- `encrypted_profile`
- `user_id`
- `timestamp`
 
Jamais :
- nom, prénom
- détails du dossier
 
## Risques & Solutions
 
| Risque | Solution | Technologie |
|--------|----------|-------------|
| Déchiffrement serveur | Impossible sans clé locale | E2EE |
| XSS injectant JS malveillant | CSP + React DOM pur | React + CSP |
| Fuite depuis logs | Logs sans données sensibles | NestJS Logger Override |
 
---
 
# 4.3. Messagerie Patient ↔ Psy (E2EE)
 
## Objectif
Messagerie chiffrée, type Signal simplifié.
 
## Schéma Flux
 
```
Patient
  message_plain
       |
       | AES-GCM(key_e2ee)
       v
  encrypted_message ----> Serveur ----> Psy
                                    |
                                    v
                             Déchiffrement local (key_e2ee)
```
 
## Risques & Solutions
 
| Risque | Solution |
|--------|----------|
| Interception MITM | HTTPS + signature messages |
| Attaques injection | Validation DTO Nest + sanitize |
| Lecture serveur | Chiffrement côté client |
 
---
 
# 4.4. Rendez-vous & Historique
 
Même modèle que données patient :
 
```
rendezvous.json -> AES-GCM(key_e2ee) -> serveur
```
 
Serveur ne voit que dates, IDs techniques.
 
---
 
# 4.5. Visio Avatar – Anonymisation Vidéo & Audio
 
## Objectif
Ne jamais transmettre le visage réel.
 
## Schéma du Pipeline Vidéo
 
```
webcam
   |
   v
[MediaPipe] --- landmarks ---> [Avatar Engine WebGL] ---> frames avatar
                                  |
                                  v
                          Encodage WebRTC
                                  |
                                  v
                          Canal chiffré DTLS-SRTP
```
 
Flux final = **avatar uniquement**, chiffré SRTP.
 
## Risques & Solutions
 
| Risque | Solution | Technologie |
|--------|----------|-------------|
| Fuite visage réel | Interdiction d’accéder au raw stream WebRTC | WebRTC + sandbox |
| Injection frames | Validation pipeline | Canvas/WebGL |
| Interception flux | SRTP + DTLS | WebRTC |
 
---
 
# 5. Risques Globaux & Mitigation
 
| Risque Général | Type | Mitigation | Techno |
|----------------|------|------------|--------|
| XSS | App Web | CSP + React | Content Security Policy |
| CSRF | App Web | Cookies SameSite | SameSite=Lax |
| IDOR | Accès illégitime | RLS PostgreSQL | PostgreSQL RLS |
| MITM | Réseau | HTTPS + DTLS-SRTP | WebRTC |
| Compromission serveur | Infra | Zero-Knowledge | AES-GCM client |
| Vol Base de donnée | Infra | Données chiffrées | PostgreSQL + blobs AES |
 
---
 
# 6. Preuve Formelle Zero-Knowledge
 
Le serveur **ne possède aucune** des clés nécessaires au déchiffrement :
 
- La clé AES-256 est dérivée côté client (**PBKDF2**).
- Les flux WebRTC sont chiffrés **SRTP**, clés négociées P2P.
- Les données en base (`BYTEA`) sont inutiles sans `key_e2ee`.
- Les notes, messages, profils, historiques sont illisibles pour le backend.
 
---
 
# 7. Conclusion Whitepaper
 
Psy2Bib n’est pas seulement une application de psychologie.  
C’est une **infrastructure cryptographique Zero-Knowledge**, couplée à une **visio avatar anonyme**, rendant impossible l’exposition des données sensibles.
 
 
