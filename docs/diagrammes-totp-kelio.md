# Diagrammes de Flux - TOTP Kelio RH

## 1. Flux d'Activation TOTP

```mermaid
sequenceDiagram
    actor User as Utilisateur
    participant UI as Interface Web
    participant API as API Backend
    participant DB as Base de données
    participant Auth as App Authenticator

    User->>UI: Accède aux paramètres de sécurité
    UI->>User: Affiche option "Activer 2FA"
    User->>UI: Clique sur "Activer 2FA"
    UI->>API: POST /api/totp/setup
    API->>API: Génère secret TOTP (32 caractères)
    API->>API: Chiffre le secret
    API->>DB: Sauvegarde secret chiffré (is_active=false)
    API->>API: Génère QR code
    API->>API: Génère 10 codes de récupération
    API->>DB: Sauvegarde codes hashés
    API-->>UI: Retourne QR code + secret + codes récupération
    UI->>User: Affiche QR code
    User->>Auth: Scanne le QR code
    Auth->>Auth: Enregistre le secret
    Auth-->>User: Affiche code TOTP (6 chiffres)
    User->>UI: Saisit le code TOTP
    UI->>API: POST /api/totp/verify-setup {code}
    API->>DB: Récupère secret chiffré
    API->>API: Déchiffre le secret
    API->>API: Vérifie le code TOTP
    alt Code valide
        API->>DB: UPDATE is_active=true, activated_at=NOW()
        API-->>UI: {success: true}
        UI->>User: Affiche codes de récupération
        User->>User: Télécharge/sauvegarde les codes
        UI->>User: Confirmation "2FA activée"
    else Code invalide
        API-->>UI: {error: "Code invalide"}
        UI->>User: Message d'erreur + réessayer
    end
```


## 2. Flux de Connexion avec TOTP

```mermaid
sequenceDiagram
    actor User as Utilisateur
    participant UI as Interface Web
    participant API as API Backend
    participant DB as Base de données
    participant Auth as App Authenticator

    User->>UI: Saisit email + mot de passe
    UI->>API: POST /api/auth/login {email, password}
    API->>DB: Vérifie identifiants
    alt Identifiants valides
        API->>DB: Vérifie si TOTP activé pour user
        alt TOTP activé
            API-->>UI: {totp_required: true, user_id: xxx}
            UI->>User: Affiche champ code TOTP
            User->>Auth: Consulte le code actuel
            Auth-->>User: Affiche code TOTP
            User->>UI: Saisit le code TOTP
            UI->>API: POST /api/totp/validate {user_id, code}
            API->>DB: Récupère secret chiffré
            API->>API: Déchiffre le secret
            API->>API: Vérifie code TOTP (fenêtre ±30s)
            alt Code valide
                API->>DB: Log événement (totp_success)
                API->>DB: UPDATE last_used_at
                alt Option "Appareil de confiance" cochée
                    API->>API: Génère token d'appareil
                    API->>DB: Sauvegarde token (expires_at = +30j)
                    API-->>UI: {success: true, device_token: xxx}
                    UI->>UI: Stocke token (cookie sécurisé)
                else
                    API-->>UI: {success: true}
                end
                UI->>User: Connexion réussie
            else Code invalide
                API->>DB: Log événement (totp_failure)
                API-->>UI: {error: "Code invalide"}
                UI->>User: Message d'erreur (2 essais restants)
            end
        else TOTP non activé
            API-->>UI: {success: true, token: xxx}
            UI->>User: Connexion réussie
        end
    else Identifiants invalides
        API-->>UI: {error: "Identifiants invalides"}
        UI->>User: Message d'erreur
    end
```


## 3. Flux d'Utilisation d'un Code de Récupération

```mermaid
sequenceDiagram
    actor User as Utilisateur
    participant UI as Interface Web
    participant API as API Backend
    participant DB as Base de données

    User->>UI: Clique "Utiliser code de récupération"
    UI->>User: Affiche champ code récupération
    User->>UI: Saisit un code de récupération
    UI->>API: POST /api/totp/validate {user_id, code, is_recovery_code: true}
    API->>DB: SELECT codes non utilisés WHERE user_id=xxx
    API->>API: Vérifie hash du code
    alt Code valide et non utilisé
        API->>DB: UPDATE used_at=NOW() pour ce code
        API->>DB: Log événement (recovery_code_used)
        API-->>UI: {success: true, warning: "Reconfigurer TOTP"}
        UI->>User: Connexion réussie + Alerte reconfiguration
        UI->>User: Affiche bannière "Reconfigurer 2FA recommandé"
    else Code invalide ou déjà utilisé
        API->>DB: Log événement (recovery_code_failed)
        API-->>UI: {error: "Code invalide"}
        UI->>User: Message d'erreur
        UI->>User: Option "Contacter le support"
    end
```


## 4. Flux de Désactivation TOTP

```mermaid
sequenceDiagram
    actor User as Utilisateur
    participant UI as Interface Web
    participant API as API Backend
    participant DB as Base de données

    User->>UI: Accède aux paramètres de sécurité
    UI->>User: Affiche "Désactiver 2FA"
    User->>UI: Clique "Désactiver 2FA"
    UI->>User: Affiche formulaire confirmation
    User->>UI: Saisit mot de passe + code TOTP actuel
    UI->>API: POST /api/totp/disable {password, totp_code}
    API->>DB: Vérifie mot de passe
    API->>DB: Récupère secret TOTP
    API->>API: Vérifie code TOTP
    alt Vérifications réussies
        API->>DB: DELETE FROM user_totp_secrets WHERE user_id=xxx
        API->>DB: DELETE FROM user_recovery_codes WHERE user_id=xxx
        API->>DB: DELETE FROM trusted_devices WHERE user_id=xxx
        API->>DB: Log événement (totp_disabled)
        API-->>UI: {success: true}
        UI->>User: "2FA désactivée avec succès"
    else Vérification échouée
        API-->>UI: {error: "Mot de passe ou code invalide"}
        UI->>User: Message d'erreur
    end
```


## 5. Diagramme d'Architecture Système


```mermaid
graph TB
    subgraph PRESENTATION["COUCHE PRÉSENTATION"]
        Web["Interface Web<br/>(React/Vue)"]
        AppAuth["Application<br/>Authenticator"]
        Mobile["API Mobile<br/>(iOS/Android)"]
    end

    subgraph APPLICATION["COUCHE APPLICATION"]
        Gateway["API REST / GraphQL Gateway<br/>(Express.js / Flask / Spring Boot)"]
        
        subgraph SERVICES["SERVICES MÉTIER"]
            ServiceAuth["Service Auth<br/>- Login<br/>- Logout<br/>- Session<br/>- Permissions"]
            ServiceTOTP["Service TOTP<br/>- Génération<br/>- Validation<br/>- QR Code<br/>- Recovery Codes"]
            ServiceUser["Service Utilisateur<br/>- Profil<br/>- Paramètres<br/>- Admin"]
        end
    end

    subgraph DATA["COUCHE DONNÉES"]
        DB[("PostgreSQL/MySQL<br/>- Users<br/>- TOTP Secrets<br/>- Recovery Codes<br/>- Logs")]
        Cache[("Redis Cache<br/>- Sessions<br/>- Rate Limit<br/>- Tokens")]
        KMS[("KMS (Clés)<br/>- Encryption<br/>- Secrets")]
    end

    subgraph EXTERNAL["SERVICES EXTERNES"]
        Email["Service d'email"]
        Monitor["Service de monitoring"]
        Logs["Service de logs"]
        Alert["Service d'alerte"]
    end

    Web --> Gateway
    AppAuth --> Gateway
    Mobile --> Gateway
    
    Gateway --> SERVICES
    
    SERVICES --> DB
    SERVICES --> Cache
    SERVICES --> KMS
    
    SERVICES -.-> Email
    SERVICES -.-> Monitor
    SERVICES -.-> Logs
    SERVICES -.-> Alert
```

┌─────────────────────────────────────────────────────────────────────┐
│                         COUCHE PRÉSENTATION                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  Interface Web   │  │  Application     │  │  API Mobile      │  │
│  │  (React/Vue)     │  │  Authenticator   │  │  (iOS/Android)   │  │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘  │
│           │                     │                     │             │
└───────────┼─────────────────────┼─────────────────────┼─────────────┘
            │                     │                     │
            │         HTTPS/TLS   │                     │
            └─────────────────────┼─────────────────────┘
                                  │
┌─────────────────────────────────┼─────────────────────────────────┐
│                         COUCHE APPLICATION                         │
├─────────────────────────────────┼─────────────────────────────────┤
│                                 │                                  │
│  ┌──────────────────────────────▼──────────────────────────────┐  │
│  │              API REST / GraphQL Gateway                     │  │
│  │              (Express.js / Flask / Spring Boot)             │  │
│  └──────────────────────────────┬──────────────────────────────┘  │
│                                 │                                  │
│  ┌──────────────────────────────▼──────────────────────────────┐  │
│  │                    SERVICES MÉTIER                          │  │
│  ├──────────────────┬──────────────────┬──────────────────────┤  │
│  │ Service Auth     │ Service TOTP     │ Service Utilisateur  │  │
│  │ - Login          │ - Génération     │ - Profil             │  │
│  │ - Logout         │ - Validation     │ - Paramètres         │  │
│  │ - Session        │ - QR Code        │ - Admin              │  │
│  │ - Permissions    │ - Recovery Codes │                      │  │
│  └──────────────────┴──────────────────┴──────────────────────┘  │
│                                                                    │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
┌────────────────────────────────▼───────────────────────────────────┐
│                         COUCHE DONNÉES                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │
│  │ PostgreSQL/MySQL│  │  Redis Cache    │  │  KMS (Clés)     │   │
│  │ - Users         │  │  - Sessions     │  │  - Encryption   │   │
│  │ - TOTP Secrets  │  │  - Rate Limit   │  │  - Secrets      │   │
│  │ - Recovery Codes│  │  - Tokens       │  │                 │   │
│  │ - Logs          │  │                 │  │                 │   │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      SERVICES EXTERNES                              │
├─────────────────────────────────────────────────────────────────────┤
│  - Service d'email (notifications)                                  │
│  - Service de monitoring (Datadog, New Relic)                       │
│  - Service de logs (ELK Stack, Splunk)                              │
│  - Service d'alerte (PagerDuty)                                     │
└─────────────────────────────────────────────────────────────────────┘


## 6. Flux de Sécurité - Rate Limiting

```mermaid
flowchart TD
    A[Requête validation TOTP] --> B{Vérifier compteur tentatives}
    B -->|< 3 tentatives| C[Traiter la requête]
    B -->|≥ 3 tentatives| D{Vérifier délai depuis dernière tentative}
    D -->|< 5 minutes| E[Bloquer - HTTP 429]
    D -->|≥ 5 minutes| F[Réinitialiser compteur]
    F --> C
    C --> G{Code valide?}
    G -->|Oui| H[Réinitialiser compteur]
    G -->|Non| I[Incrémenter compteur]
    H --> J[Succès - Créer session]
    I --> K{Compteur ≥ 3?}
    K -->|Oui| L[Bloquer pendant 5 min]
    K -->|Non| M[Permettre nouvelle tentative]
    E --> N[Log tentative suspecte]
    L --> N
```

## 7. Diagramme d'États - Compte Utilisateur TOTP

```mermaid
stateDiagram-v2
    [*] --> SansTOTP: Compte créé
    SansTOTP --> InitialisationEnCours: Début activation
    InitialisationEnCours --> SansTOTP: Abandon
    InitialisationEnCours --> TOTPActif: Vérification réussie
    TOTPActif --> TOTPSuspendu: Trop d'échecs
    TOTPSuspendu --> TOTPActif: Réactivation admin
    TOTPActif --> SansTOTP: Désactivation utilisateur
    TOTPActif --> ReconfigurationRequise: Code récupération utilisé
    ReconfigurationRequise --> InitialisationEnCours: Reconfiguration
    ReconfigurationRequise --> TOTPActif: Délai expiré (30j)
```

## 8. Flux de Gestion Admin

```mermaid
sequenceDiagram
    actor Admin as Administrateur
    participant UI as Interface Admin
    participant API as API Backend
    participant DB as Base de données
    participant Email as Service Email

    Admin->>UI: Accède au tableau de bord TOTP
    UI->>API: GET /api/admin/totp/stats
    API->>DB: Requête statistiques
    DB-->>API: Données agrégées
    API-->>UI: Statistiques
    UI->>Admin: Affiche dashboard

    Note over Admin,Email: Cas: Réinitialisation TOTP utilisateur

    Admin->>UI: Recherche utilisateur "user@example.com"
    UI->>API: GET /api/admin/users?email=user@example.com
    API->>DB: SELECT user
    DB-->>API: Données utilisateur
    API-->>UI: Profil utilisateur
    UI->>Admin: Affiche profil + statut TOTP
    Admin->>UI: Clique "Réinitialiser TOTP"
    UI->>Admin: Demande confirmation + raison
    Admin->>UI: Confirme + saisit raison
    UI->>API: POST /api/admin/totp/reset {user_id, reason}
    API->>DB: DELETE secrets, codes, tokens
    API->>DB: Log action admin
    API->>Email: Envoie notification utilisateur
    API-->>UI: {success: true}
    UI->>Admin: Confirmation "TOTP réinitialisé"
```

---

**Note** : Ces diagrammes peuvent être visualisés avec des outils compatibles Mermaid comme :
- GitHub (rendu automatique des blocs mermaid)
- Mermaid Live Editor (https://mermaid.live)
- Extensions VSCode (Mermaid Preview)
- GitLab, Notion, Confluence

**Version** : 1.0  
**Date** : Octobre 2025
