# Analyse Fonctionnelle : Authentification TOTP pour Kelio Suite RH

## 1. Contexte et Objectifs

### 1.1 Contexte
La suite RH Kelio nécessite un renforcement de son système d'authentification pour garantir la sécurité des données des clients. L'implémentation d'une authentification multifacteur (MFA) via TOTP (Time-based One-Time Password) permettra d'ajouter une couche de sécurité supplémentaire.

### 1.2 Objectifs
- Renforcer la sécurité de l'authentification des clients
- Implémenter une solution TOTP conforme aux standards (RFC 6238)
- Assurer une expérience utilisateur fluide lors de l'activation et de l'utilisation
- Permettre une gestion administrative efficace des authentifications

## 2. Périmètre Fonctionnel

### 2.1 Utilisateurs Concernés
- **Clients de la suite RH Kelio** : Utilisateurs finaux nécessitant un accès sécurisé
- **Administrateurs** : Gestion et supervision des authentifications
- **Support technique** : Assistance aux utilisateurs

### 2.2 Modules Impactés
- Module d'authentification
- Interface utilisateur (portail client)
- Espace d'administration
- Base de données utilisateurs
- Système de logs et d'audit

## 3. Spécifications Fonctionnelles

### 3.1 Activation du TOTP

#### 3.1.1 Processus d'Inscription
1. **Accès au paramétrage**
   - L'utilisateur accède à son profil/paramètres de sécurité
   - Option "Activer l'authentification à deux facteurs" visible

2. **Génération du secret TOTP**
   - Le système génère un secret unique pour l'utilisateur
   - Affichage d'un QR code contenant :
     - Secret encodé en base32
     - Nom de l'application (Kelio RH)
     - Identifiant de l'utilisateur (email/login)

3. **Configuration de l'application d'authentification**
   - L'utilisateur scanne le QR code avec une application compatible (Google Authenticator, Microsoft Authenticator, Authy, etc.)
   - Affichage alternatif du secret en texte pour saisie manuelle

4. **Vérification initiale**
   - L'utilisateur saisit le code TOTP généré par son application
   - Le système valide le code
   - En cas de succès : activation confirmée
   - En cas d'échec : possibilité de réessayer (max 3 tentatives)

5. **Codes de récupération**
   - Génération de 10 codes de récupération à usage unique
   - Affichage et téléchargement obligatoire avant finalisation
   - Stockage sécurisé côté serveur (hashés)

### 3.2 Processus de Connexion avec TOTP

#### 3.2.1 Flux d'authentification
1. **Étape 1 : Identifiants classiques**
   - Saisie de l'email/login et mot de passe
   - Validation des identifiants

2. **Étape 2 : Code TOTP**
   - Si TOTP activé pour l'utilisateur :
     - Affichage d'un champ pour saisir le code à 6 chiffres
     - Durée de validité : 30 secondes (standard TOTP)
     - Fenêtre de tolérance : ±1 période (90 secondes au total)
   
3. **Validation**
   - Vérification du code TOTP
   - Maximum 3 tentatives par période de 5 minutes
   - Blocage temporaire après échecs répétés

4. **Option "Se souvenir de cet appareil"**
   - Case à cocher optionnelle
   - Si cochée : création d'un token de confiance (validité 30 jours)
   - Stockage sécurisé du token (cookie httpOnly, secure)
   - TOTP non demandé sur cet appareil pendant la période de validité

### 3.3 Utilisation des Codes de Récupération

#### 3.3.1 Scénario d'utilisation
- Perte/changement de téléphone
- Application d'authentification non disponible
- Impossibilité de générer un code TOTP

#### 3.3.2 Processus
1. Lien "Utiliser un code de récupération" sur l'écran TOTP
2. Saisie d'un code de récupération valide
3. Le code est consommé (usage unique)
4. Connexion réussie
5. Recommandation de reconfigurer TOTP

### 3.4 Désactivation du TOTP

#### 3.4.1 Par l'utilisateur
- Accès via les paramètres de sécurité
- Confirmation par :
  - Mot de passe actuel
  - Code TOTP actuel OU code de récupération
- Suppression du secret TOTP
- Invalidation des codes de récupération
- Invalidation des tokens d'appareil de confiance

#### 3.4.2 Par un administrateur
- Accès via l'interface d'administration
- Recherche de l'utilisateur
- Fonction "Réinitialiser TOTP"
- Journalisation de l'action
- Notification à l'utilisateur

### 3.5 Gestion Administrative

#### 3.5.1 Tableau de bord administrateur
- Statistiques :
  - Nombre d'utilisateurs avec TOTP activé
  - Taux d'adoption
  - Nombre d'échecs d'authentification
  - Codes de récupération utilisés

#### 3.5.2 Actions disponibles
- Forcer l'activation TOTP pour certains rôles/utilisateurs
- Réinitialiser TOTP d'un utilisateur
- Consulter l'historique des authentifications
- Configurer les paramètres globaux :
  - Période de validité du token d'appareil
  - Nombre de tentatives autorisées
  - Durée de blocage après échecs

## 4. Spécifications Techniques

### 4.1 Standard et Algorithme
- **Protocole** : TOTP (RFC 6238)
- **Algorithme de hachage** : SHA-1 (standard), SHA-256 (option)
- **Longueur du code** : 6 chiffres
- **Période** : 30 secondes
- **Fenêtre de tolérance** : ±1 période

### 4.2 Génération du Secret
- **Longueur** : 160 bits minimum (20 octets)
- **Encodage** : Base32
- **Générateur** : Cryptographiquement sécurisé (CSPRNG)

### 4.3 Stockage Sécurisé
- **Secret TOTP** : Chiffré en base de données (AES-256)
- **Codes de récupération** : Hashés (bcrypt ou Argon2)
- **Logs d'authentification** : Horodatés et signés

### 4.4 Format du QR Code
```
otpauth://totp/Kelio%20RH:utilisateur@example.com?secret=SECRET&issuer=Kelio%20RH&algorithm=SHA1&digits=6&period=30
```

## 5. Exigences Non-Fonctionnelles

### 5.1 Sécurité
- Chiffrement des secrets en base de données
- Protection contre les attaques par force brute
- Limitation du taux de tentatives
- Journalisation complète des événements
- Invalidation des sessions en cas de détection d'anomalie

### 5.2 Performance
- Temps de réponse < 500ms pour la validation TOTP
- Support de la charge : validation simultanée pour 10 000+ utilisateurs
- Mise en cache des secrets déchiffrés (durée limitée)

### 5.3 Disponibilité
- Mécanisme de basculement en cas de défaillance
- Codes de récupération toujours disponibles
- Interface administrative accessible 24/7

### 5.4 Compatibilité
- Support des applications d'authentification standard :
  - Google Authenticator
  - Microsoft Authenticator
  - Authy
  - 1Password
  - Bitwarden
- Compatibilité navigateurs : Chrome, Firefox, Safari, Edge (3 dernières versions)
- Support mobile : iOS 12+, Android 8+

### 5.5 Accessibilité
- Conformité RGAA (niveau AA minimum)
- Support des lecteurs d'écran
- Navigation au clavier complète
- Textes alternatifs pour QR codes

## 6. Parcours Utilisateur (User Stories)

### 6.1 En tant qu'utilisateur client
**US1** : "Je veux activer l'authentification à deux facteurs pour sécuriser mon compte"
- Accès facile depuis les paramètres
- Instructions claires étape par étape
- QR code lisible
- Confirmation de l'activation

**US2** : "Je veux me connecter rapidement avec TOTP"
- Champ de saisie clair et intuitif
- Indication du temps restant
- Option pour ne pas redemander sur cet appareil

**US3** : "J'ai changé de téléphone et je ne peux plus générer de codes"
- Accès simple aux codes de récupération
- Instructions pour reconfigurer TOTP
- Support accessible

### 6.2 En tant qu'administrateur
**US4** : "Je veux suivre l'adoption du TOTP dans mon organisation"
- Tableau de bord avec statistiques
- Export des données
- Filtres et recherche

**US5** : "Je veux aider un utilisateur qui a perdu l'accès à son TOTP"
- Fonction de réinitialisation accessible
- Vérification d'identité de l'utilisateur
- Traçabilité de l'action

## 7. Gestion des Erreurs et Cas Limites

### 7.1 Erreurs Utilisateur
| Erreur | Message | Action |
|--------|---------|--------|
| Code TOTP incorrect | "Code incorrect. Veuillez réessayer." | Autoriser 3 tentatives |
| Code expiré | "Ce code a expiré. Veuillez saisir le code actuel." | Nouvelle saisie |
| Trop de tentatives | "Trop de tentatives. Réessayez dans 5 minutes." | Blocage temporaire |
| Code de récupération invalide | "Code de récupération invalide." | Nouvelle saisie ou contact support |

### 7.2 Scénarios Exceptionnels
- **Désynchronisation temporelle** : Fenêtre de tolérance ±1 période
- **Panne système** : Mode de secours sans TOTP avec notification
- **Attaque par force brute détectée** : Blocage du compte + notification administrateur
- **Secret compromis** : Procédure de révocation et réémission

## 8. Migration et Déploiement

### 8.1 Phase de Déploiement
1. **Phase 1 - Pilote** (Semaine 1-2)
   - Activation pour un groupe test de 50 utilisateurs
   - Collecte de feedback
   - Ajustements nécessaires

2. **Phase 2 - Déploiement progressif** (Semaine 3-6)
   - Activation par vagues de 500 utilisateurs
   - Communication et documentation
   - Support renforcé

3. **Phase 3 - Généralisation** (Semaine 7+)
   - Activation pour tous les nouveaux utilisateurs
   - Incitation pour les utilisateurs existants
   - Option d'obligation pour certains profils

### 8.2 Communication
- Email de notification avant activation
- Guide utilisateur détaillé
- Vidéos tutorielles
- FAQ dédiée
- Support chat/téléphone renforcé

### 8.3 Formation
- Formation des équipes support
- Documentation technique pour les développeurs
- Guide administrateur
- Webinaires pour les clients

## 9. Métriques de Succès

### 9.1 Indicateurs Techniques
- Taux de disponibilité : 99.9%
- Temps de réponse moyen < 500ms
- Taux d'erreur < 0.1%

### 9.2 Indicateurs Métier
- Taux d'adoption : objectif 80% à 6 mois
- Satisfaction utilisateur : score > 4/5
- Réduction des incidents de sécurité liés aux accès
- Taux de support concernant TOTP < 5%

### 9.3 Indicateurs de Sécurité
- Nombre de tentatives de connexion suspectes bloquées
- Nombre de comptes protégés par TOTP
- Temps moyen de détection d'anomalie

## 10. Conformité et Réglementations

### 10.1 RGPD
- Consentement explicite pour l'activation
- Droit de désactivation à tout moment
- Stockage sécurisé des données biométriques (si applicable)
- Durée de conservation des logs : 12 mois maximum

### 10.2 Standards de Sécurité
- ISO 27001 : Gestion de la sécurité de l'information
- RFC 6238 : Spécification TOTP
- OWASP : Bonnes pratiques d'authentification

## 11. Risques et Mitigation

| Risque | Impact | Probabilité | Mitigation |
|--------|--------|-------------|------------|
| Rejet utilisateur | Moyen | Moyen | Communication proactive, UX optimisée |
| Perte codes de récupération | Élevé | Faible | Procédure de réinitialisation sécurisée |
| Attaque MITM sur QR code | Élevé | Très faible | HTTPS obligatoire, vérification du code |
| Surcharge support | Moyen | Moyen | Documentation complète, FAQ, chatbot |
| Problème de compatibilité | Faible | Faible | Tests sur multiples applications |

## 12. Annexes

### 12.1 Bibliothèques Recommandées
- **Backend** :
  - Python : pyotp, qrcode
  - Node.js : otplib, qrcode
  - Java : GoogleAuth (Wdullaer)
  - PHP : OTPHP

- **Frontend** :
  - qrcode.js pour génération QR codes
  - Material-UI ou Bootstrap pour composants

### 12.2 Ressources
- RFC 6238 : https://tools.ietf.org/html/rfc6238
- OWASP Authentication Cheat Sheet
- Google Authenticator documentation

### 12.3 Glossaire
- **TOTP** : Time-based One-Time Password
- **MFA** : Multi-Factor Authentication
- **QR Code** : Quick Response Code
- **CSPRNG** : Cryptographically Secure Pseudo-Random Number Generator
- **Base32** : Encodage pour représenter des données binaires
- **RFC** : Request for Comments (standard Internet)

---

**Version** : 1.0  
**Date** : Octobre 2025  
**Statut** : Document de spécifications fonctionnelles  
**Auteur** : Équipe technique Kelio RH
