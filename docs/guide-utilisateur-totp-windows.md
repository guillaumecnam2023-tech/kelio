# Guide Utilisateur : Activer l'Authentification TOTP sur Kelio HR
## Solutions Sans Smartphone pour PC Windows

---

## 🎯 Objectif
Sécuriser votre compte Kelio HR avec une authentification à deux facteurs (2FA), même sans smartphone.

---

## 📋 Quelle Solution Choisir ?

### Vous avez un smartphone ? 📱
➡️ **Utilisez** : Google Authenticator ou Microsoft Authenticator  
✅ **Avantages** : Simple, rapide, mobile  
📥 **Téléchargement** :
- Google Authenticator : App Store / Google Play
- Microsoft Authenticator : App Store / Google Play

### Vous n'avez PAS de smartphone ? 💻
➡️ **Utilisez** : WinAuth (PC Windows)  
✅ **Avantages** : Gratuit, simple, pas de cloud  
📥 **Téléchargement** : https://winauth.github.io/winauth/

### Vous travaillez sur plusieurs appareils ? 🔄
➡️ **Utilisez** : Authy Desktop  
✅ **Avantages** : Synchronisation automatique  
📥 **Téléchargement** : https://authy.com/download/

---

## 🔧 Installation WinAuth (PC Windows)

### Étape 1 : Téléchargement
1. Aller sur : https://github.com/winauth/winauth/releases
2. Télécharger : **WinAuth-X.X.X.zip** (dernière version)
3. Extraire le fichier ZIP dans un dossier (ex: C:\WinAuth)

### Étape 2 : Premier Lancement
1. Double-cliquer sur **WinAuth.exe**
2. À la première ouverture, WinAuth est vide (normal)

### Étape 3 : Configuration Mot de Passe (Recommandé)
1. Menu : **WinAuth** → **Protect WinAuth**
2. Cocher : **Password protect WinAuth**
3. Entrer un mot de passe sécurisé
4. Cliquer **OK**

✅ **WinAuth est maintenant installé et sécurisé !**

---

## ✨ Activation TOTP sur Kelio HR avec WinAuth

### Étape 1 : Activer TOTP sur Kelio HR
1. Se connecter à Kelio HR (identifiants normaux)
2. Aller dans **Mon Profil** → **Sécurité**
3. Cliquer sur **Activer l'authentification à deux facteurs**
4. **Kelio HR affiche un QR code** → **NE PAS FERMER CETTE PAGE**

### Étape 2 : Ajouter Kelio HR dans WinAuth
1. Ouvrir **WinAuth**
2. Cliquer sur **Add** (en bas à gauche)
3. Choisir : **Authenticator** (ou **Google**)
4. Trois options s'offrent à vous :

#### Option A : Scan QR Code (Si vous avez une webcam)
1. Cliquer sur **Scan QR Code**
2. Autoriser l'accès à la webcam
3. Présenter le QR code de Kelio HR devant la webcam
4. WinAuth capture automatiquement le secret

#### Option B : Capture d'Écran
1. Faire une capture d'écran du QR code Kelio HR
2. Dans WinAuth, cliquer sur **Scan QR Code** → **Load Image**
3. Sélectionner votre capture d'écran
4. WinAuth lit le QR code

#### Option C : Saisie Manuelle (Si pas de webcam)
1. Sur Kelio HR, cliquer **Saisir manuellement**
2. Kelio affiche le code secret (ex: JBSWY3DPEHPK3PXP)
3. Dans WinAuth, coller ce code dans **Secret Code**

### Étape 3 : Nommer le Compte
1. Dans **Name**, entrer : **Kelio HR**
2. Cliquer **Verify Authenticator**
3. WinAuth affiche un code à 6 chiffres (ex: 123456)

### Étape 4 : Validation sur Kelio HR
1. Retourner sur la page Kelio HR
2. Entrer le code à 6 chiffres affiché par WinAuth
3. Cliquer **Vérifier**
4. Si le code est correct : **TOTP activé !** ✅

### Étape 5 : Sauvegarder les Codes de Récupération
1. Kelio HR affiche **10 codes de récupération**
2. **IMPORTANT** : Télécharger ou imprimer ces codes
3. Les conserver en lieu sûr (en cas de perte de WinAuth)
4. Cliquer **J'ai sauvegardé mes codes**

✅ **Configuration terminée !**

---

## 🔐 Connexion Quotidienne avec TOTP

### Scénario : Je me connecte à Kelio HR

#### Étape 1 : Connexion Classique
1. Aller sur Kelio HR
2. Entrer **email** et **mot de passe**
3. Cliquer **Se connecter**

#### Étape 2 : Code TOTP Demandé
1. Kelio affiche : **"Entrez votre code d'authentification"**
2. Ouvrir **WinAuth** sur votre PC
3. Double-cliquer sur **Kelio HR** dans la liste
4. Le code est **automatiquement copié** dans le presse-papier

#### Étape 3 : Validation
1. Retourner sur Kelio HR
2. **Coller** (Ctrl+V) le code dans le champ
3. Cliquer **Valider**
4. ✅ **Connecté !**

### ⏱️ Important : Validité du Code
- Chaque code est valide **30 secondes**
- Si le code a expiré, attendez le code suivant dans WinAuth
- WinAuth affiche une barre de progression (temps restant)

---

## 💾 Sauvegarde WinAuth (IMPORTANT)

### Pourquoi faire une sauvegarde ?
- En cas de réinstallation Windows
- En cas de changement de PC
- En cas de suppression accidentelle

### Comment sauvegarder ?
1. Ouvrir WinAuth
2. Menu : **WinAuth** → **Export**
3. Choisir un emplacement sécurisé (clé USB, cloud personnel)
4. Le fichier est chiffré avec votre mot de passe WinAuth
5. Nommer : **WinAuth_Backup_YYYYMMDD.xml**

### Comment restaurer ?
1. Installer WinAuth sur le nouveau PC
2. Menu : **WinAuth** → **Import**
3. Sélectionner le fichier de sauvegarde
4. Entrer le mot de passe
5. ✅ Tous vos comptes sont restaurés

---

## 🆘 Dépannage

### ❌ "Code invalide" sur Kelio HR

**Causes possibles** :
1. **Horloge PC désynchronisée**
   - Solution : Synchroniser l'horloge Windows
   - Clic droit sur l'heure → Ajuster la date/l'heure
   - Activer "Définir l'heure automatiquement"

2. **Code expiré**
   - Solution : Attendre le code suivant (30 secondes)

3. **Mauvais compte sélectionné**
   - Solution : Vérifier que c'est bien "Kelio HR" dans WinAuth

### ❌ J'ai perdu WinAuth (PC cassé, réinstallation...)

**Solution 1** : Utiliser un code de récupération
1. Sur l'écran TOTP de Kelio HR, cliquer **Utiliser un code de récupération**
2. Entrer un des 10 codes sauvegardés lors de l'activation
3. ✅ Connecté
4. **Immédiatement** : Reconfigurer TOTP avec un nouveau WinAuth

**Solution 2** : Contacter le support IT
1. Contacter votre support Kelio HR
2. Demander une **réinitialisation TOTP**
3. Le support désactivera TOTP
4. Reconfigurer TOTP depuis le début

### ❌ WinAuth ne s'ouvre plus

**Solutions** :
1. Vérifier que WinAuth.exe n'est pas bloqué par l'antivirus
2. Réinstaller WinAuth (télécharger nouvelle version)
3. Restaurer depuis une sauvegarde

### ❌ Je n'ai plus de codes de récupération

**Solution** :
1. Se connecter à Kelio HR (avec TOTP fonctionnel)
2. Aller dans **Mon Profil** → **Sécurité**
3. Cliquer **Générer de nouveaux codes de récupération**
4. ⚠️ Les anciens codes sont invalidés
5. Sauvegarder les nouveaux codes

---

## 🔄 Alternative : Authy Desktop

### Quand utiliser Authy ?
- Vous avez plusieurs PC
- Vous voulez synchroniser avec votre smartphone
- Vous acceptez le cloud sécurisé

### Installation Rapide
1. Télécharger : https://authy.com/download/
2. Installer Authy Desktop
3. Créer un compte (email + téléphone)
4. Sur Kelio HR : Scanner le QR code avec Authy
5. Le compte est synchronisé entre tous vos appareils Authy

### Avantages Authy
- ✅ Synchronisation automatique
- ✅ Backup cloud chiffré
- ✅ Interface moderne
- ✅ Disponible sur smartphone + PC

---

## 📊 Comparaison Rapide

| Critère | WinAuth | Authy Desktop | Google/Microsoft Auth |
|---------|---------|---------------|---------------------|
| **Coût** | Gratuit | Gratuit | Gratuit |
| **Plateforme** | Windows uniquement | Windows/Mac/Linux | Smartphone uniquement |
| **Stockage** | Local (PC) | Cloud chiffré | Smartphone |
| **Synchronisation** | ❌ Non | ✅ Oui | ❌/⚠️ (selon app) |
| **Sauvegarde** | Manuelle | Automatique | Dépend app |
| **Facilité** | 🟢 Simple | 🟢 Simple | 🟢 Très simple |
| **Sécurité** | 🟢 Élevée | 🟢 Élevée | 🟢 Élevée |

---

## 📞 Support

### Besoin d'aide ?

**Documentation** :
- FAQ TOTP Kelio HR : [lien interne]
- Vidéo tutorielle WinAuth : [lien]
- Guide complet TOTP : [lien]

**Contact Support IT Kelio** :
- Email : support@kelio.com
- Téléphone : XX XX XX XX XX
- Chat : [lien]

**Horaires Support** :
- Lundi - Vendredi : 9h - 18h
- Assistance TOTP prioritaire

---

## ✅ Checklist Activation

- [ ] Choisir ma solution (WinAuth / Authy / Smartphone)
- [ ] Télécharger et installer l'application
- [ ] Me connecter à Kelio HR
- [ ] Activer TOTP dans Sécurité
- [ ] Scanner ou saisir le code secret
- [ ] Vérifier le premier code TOTP
- [ ] **Sauvegarder les 10 codes de récupération** ⚠️
- [ ] Faire une sauvegarde WinAuth (si applicable)
- [ ] Tester une connexion complète
- [ ] Conserver les codes de récupération en lieu sûr

---

**Document** : Guide Utilisateur TOTP PC Windows  
**Version** : 1.0  
**Date** : Octobre 2025  
**Public** : Utilisateurs Kelio HR
