# Guide d'Implémentation Technique - TOTP pour Kelio RH

## 1. Architecture Technique

### 1.1 Composants Système

```
┌─────────────────────────────────────────────────────────────┐
│                     Application Kelio RH                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐      ┌──────────────┐    ┌─────────────┐ │
│  │   Frontend   │◄────►│   Backend    │◄──►│  Database   │ │
│  │  (React/Vue) │      │  (API REST)  │    │ (PostgreSQL)│ │
│  └──────────────┘      └──────────────┘    └─────────────┘ │
│         │                      │                            │
│         │                      │                            │
│         │              ┌───────▼────────┐                   │
│         │              │ Service TOTP   │                   │
│         │              │  - Génération  │                   │
│         │              │  - Validation  │                   │
│         │              │  - QR Code     │                   │
│         │              └────────────────┘                   │
│         │                                                    │
│  ┌──────▼─────────────────────────────────────────────┐     │
│  │       Applications Authenticator (Client)          │     │
│  │  Google Auth / Microsoft Auth / Authy / etc.       │     │
│  └────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Schéma de Base de Données

```sql
-- Table principale des secrets TOTP
CREATE TABLE user_totp_secrets (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    encrypted_secret VARCHAR(512) NOT NULL,
    encryption_key_version INTEGER NOT NULL,
    is_active BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    activated_at TIMESTAMP,
    last_used_at TIMESTAMP,
    UNIQUE(user_id)
);

-- Table des codes de récupération
CREATE TABLE user_recovery_codes (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    code_hash VARCHAR(255) NOT NULL,
    used_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_unused (user_id, used_at)
);

-- Table des appareils de confiance
CREATE TABLE trusted_devices (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    device_token VARCHAR(255) NOT NULL UNIQUE,
    device_fingerprint VARCHAR(255),
    user_agent TEXT,
    ip_address VARCHAR(45),
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_used_at TIMESTAMP,
    INDEX idx_token (device_token),
    INDEX idx_user_active (user_id, expires_at)
);

-- Table des logs d'authentification
CREATE TABLE totp_authentication_logs (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    event_type VARCHAR(50) NOT NULL, -- 'success', 'failure', 'recovery_code_used'
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    details JSONB,
    INDEX idx_user_events (user_id, event_type, created_at),
    INDEX idx_created (created_at)
);

-- Table de configuration globale
CREATE TABLE totp_settings (
    id SERIAL PRIMARY KEY,
    setting_key VARCHAR(100) NOT NULL UNIQUE,
    setting_value TEXT NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 2. Implémentation Backend

### 2.1 Exemple Python (Flask/Django)

```python
import pyotp
import qrcode
import io
import base64
from cryptography.fernet import Fernet
from datetime import datetime, timedelta
import secrets
import hashlib

class TOTPService:
    """Service de gestion TOTP"""
    
    def __init__(self, encryption_key):
        self.cipher = Fernet(encryption_key)
        self.issuer_name = "Kelio RH"
    
    def generate_secret(self):
        """Génère un nouveau secret TOTP"""
        return pyotp.random_base32(length=32)
    
    def encrypt_secret(self, secret):
        """Chiffre le secret pour stockage"""
        return self.cipher.encrypt(secret.encode()).decode()
    
    def decrypt_secret(self, encrypted_secret):
        """Déchiffre le secret"""
        return self.cipher.decrypt(encrypted_secret.encode()).decode()
    
    def generate_qr_code(self, user_email, secret):
        """Génère un QR code pour l'activation"""
        totp = pyotp.TOTP(secret)
        provisioning_uri = totp.provisioning_uri(
            name=user_email,
            issuer_name=self.issuer_name
        )
        
        # Génération du QR code
        qr = qrcode.QRCode(version=1, box_size=10, border=5)
        qr.add_data(provisioning_uri)
        qr.make(fit=True)
        
        img = qr.make_image(fill_color="black", back_color="white")
        
        # Conversion en base64 pour affichage web
        buffer = io.BytesIO()
        img.save(buffer, format='PNG')
        img_str = base64.b64encode(buffer.getvalue()).decode()
        
        return f"data:image/png;base64,{img_str}", provisioning_uri
    
    def verify_totp(self, secret, code, window=1):
        """
        Vérifie un code TOTP
        window=1 permet une tolérance de ±30 secondes
        """
        totp = pyotp.TOTP(secret)
        return totp.verify(code, valid_window=window)
    
    def generate_recovery_codes(self, count=10):
        """Génère des codes de récupération"""
        codes = []
        for _ in range(count):
            code = secrets.token_hex(8)  # 16 caractères hexadécimaux
            codes.append(code)
        return codes
    
    def hash_recovery_code(self, code):
        """Hash un code de récupération pour stockage"""
        return hashlib.sha256(code.encode()).hexdigest()
    
    def verify_recovery_code(self, code, code_hash):
        """Vérifie un code de récupération"""
        return hashlib.sha256(code.encode()).hexdigest() == code_hash
    
    def generate_device_token(self):
        """Génère un token pour un appareil de confiance"""
        return secrets.token_urlsafe(32)


# Exemple d'API Flask
from flask import Flask, request, jsonify
from flask_login import login_required, current_user

app = Flask(__name__)
totp_service = TOTPService(encryption_key=b'your-encryption-key')

@app.route('/api/totp/setup', methods=['POST'])
@login_required
def setup_totp():
    """Initialise TOTP pour l'utilisateur"""
    # Générer le secret
    secret = totp_service.generate_secret()
    encrypted_secret = totp_service.encrypt_secret(secret)
    
    # Sauvegarder en base (non actif tant que non vérifié)
    db.execute(
        "INSERT INTO user_totp_secrets (user_id, encrypted_secret, is_active) "
        "VALUES (?, ?, ?)",
        (current_user.id, encrypted_secret, False)
    )
    
    # Générer QR code
    qr_code, uri = totp_service.generate_qr_code(current_user.email, secret)
    
    # Générer codes de récupération
    recovery_codes = totp_service.generate_recovery_codes()
    
    # Sauvegarder les codes de récupération (hashés)
    for code in recovery_codes:
        code_hash = totp_service.hash_recovery_code(code)
        db.execute(
            "INSERT INTO user_recovery_codes (user_id, code_hash) VALUES (?, ?)",
            (current_user.id, code_hash)
        )
    
    return jsonify({
        'qr_code': qr_code,
        'secret': secret,  # Pour saisie manuelle
        'recovery_codes': recovery_codes
    })

@app.route('/api/totp/verify-setup', methods=['POST'])
@login_required
def verify_totp_setup():
    """Vérifie et active TOTP"""
    code = request.json.get('code')
    
    # Récupérer le secret
    row = db.query(
        "SELECT encrypted_secret FROM user_totp_secrets WHERE user_id = ?",
        (current_user.id,)
    ).fetchone()
    
    if not row:
        return jsonify({'error': 'TOTP non initialisé'}), 400
    
    secret = totp_service.decrypt_secret(row['encrypted_secret'])
    
    # Vérifier le code
    if totp_service.verify_totp(secret, code):
        # Activer TOTP
        db.execute(
            "UPDATE user_totp_secrets SET is_active = ?, activated_at = ? "
            "WHERE user_id = ?",
            (True, datetime.now(), current_user.id)
        )
        return jsonify({'success': True})
    else:
        return jsonify({'error': 'Code invalide'}), 400

@app.route('/api/totp/validate', methods=['POST'])
def validate_totp_login():
    """Valide le code TOTP lors de la connexion"""
    user_id = request.json.get('user_id')
    code = request.json.get('code')
    is_recovery_code = request.json.get('is_recovery_code', False)
    
    # Récupérer le secret
    row = db.query(
        "SELECT encrypted_secret FROM user_totp_secrets "
        "WHERE user_id = ? AND is_active = ?",
        (user_id, True)
    ).fetchone()
    
    if not row:
        return jsonify({'error': 'TOTP non activé'}), 400
    
    if is_recovery_code:
        # Vérifier code de récupération
        codes = db.query(
            "SELECT id, code_hash FROM user_recovery_codes "
            "WHERE user_id = ? AND used_at IS NULL",
            (user_id,)
        ).fetchall()
        
        for code_row in codes:
            if totp_service.verify_recovery_code(code, code_row['code_hash']):
                # Marquer comme utilisé
                db.execute(
                    "UPDATE user_recovery_codes SET used_at = ? WHERE id = ?",
                    (datetime.now(), code_row['id'])
                )
                log_authentication(user_id, 'recovery_code_used', True)
                return jsonify({'success': True, 'warning': 'Reconfigurer TOTP'})
        
        log_authentication(user_id, 'recovery_code_failed', False)
        return jsonify({'error': 'Code de récupération invalide'}), 400
    else:
        # Vérifier code TOTP
        secret = totp_service.decrypt_secret(row['encrypted_secret'])
        
        if totp_service.verify_totp(secret, code):
            # Mise à jour dernière utilisation
            db.execute(
                "UPDATE user_totp_secrets SET last_used_at = ? WHERE user_id = ?",
                (datetime.now(), user_id)
            )
            log_authentication(user_id, 'totp_success', True)
            return jsonify({'success': True})
        else:
            log_authentication(user_id, 'totp_failure', False)
            return jsonify({'error': 'Code invalide'}), 400

def log_authentication(user_id, event_type, success):
    """Log les tentatives d'authentification"""
    db.execute(
        "INSERT INTO totp_authentication_logs "
        "(user_id, event_type, ip_address, user_agent) "
        "VALUES (?, ?, ?, ?)",
        (user_id, event_type, request.remote_addr, request.user_agent.string)
    )
```

### 2.2 Exemple Node.js (Express)

```javascript
const speakeasy = require('speakeasy');
const QRCode = require('qrcode');
const crypto = require('crypto');

class TOTPService {
    constructor(encryptionKey) {
        this.encryptionKey = Buffer.from(encryptionKey, 'hex');
        this.issuer = 'Kelio RH';
    }

    generateSecret() {
        return speakeasy.generateSecret({
            name: this.issuer,
            length: 32
        });
    }

    async generateQRCode(userEmail, secret) {
        const otpauthUrl = speakeasy.otpauthURL({
            secret: secret.base32,
            label: userEmail,
            issuer: this.issuer,
            encoding: 'base32'
        });

        const qrCodeDataUrl = await QRCode.toDataURL(otpauthUrl);
        return { qrCodeDataUrl, otpauthUrl };
    }

    verifyToken(secret, token, window = 1) {
        return speakeasy.totp.verify({
            secret: secret,
            encoding: 'base32',
            token: token,
            window: window
        });
    }

    encryptSecret(secret) {
        const iv = crypto.randomBytes(16);
        const cipher = crypto.createCipheriv('aes-256-cbc', this.encryptionKey, iv);
        let encrypted = cipher.update(secret, 'utf8', 'hex');
        encrypted += cipher.final('hex');
        return iv.toString('hex') + ':' + encrypted;
    }

    decryptSecret(encryptedSecret) {
        const parts = encryptedSecret.split(':');
        const iv = Buffer.from(parts[0], 'hex');
        const encrypted = parts[1];
        const decipher = crypto.createDecipheriv('aes-256-cbc', this.encryptionKey, iv);
        let decrypted = decipher.update(encrypted, 'hex', 'utf8');
        decrypted += decipher.final('utf8');
        return decrypted;
    }

    generateRecoveryCodes(count = 10) {
        const codes = [];
        for (let i = 0; i < count; i++) {
            codes.push(crypto.randomBytes(8).toString('hex'));
        }
        return codes;
    }

    hashRecoveryCode(code) {
        return crypto.createHash('sha256').update(code).digest('hex');
    }
}

// Routes Express
const express = require('express');
const router = express.Router();
const totpService = new TOTPService(process.env.ENCRYPTION_KEY);

router.post('/totp/setup', async (req, res) => {
    try {
        const userId = req.user.id;
        
        // Générer secret
        const secret = totpService.generateSecret();
        const encryptedSecret = totpService.encryptSecret(secret.base32);
        
        // Sauvegarder en base
        await db.query(
            'INSERT INTO user_totp_secrets (user_id, encrypted_secret, is_active) VALUES ($1, $2, $3)',
            [userId, encryptedSecret, false]
        );
        
        // Générer QR code
        const { qrCodeDataUrl } = await totpService.generateQRCode(req.user.email, secret.base32);
        
        // Générer codes de récupération
        const recoveryCodes = totpService.generateRecoveryCodes();
        
        // Sauvegarder codes de récupération
        for (const code of recoveryCodes) {
            const codeHash = totpService.hashRecoveryCode(code);
            await db.query(
                'INSERT INTO user_recovery_codes (user_id, code_hash) VALUES ($1, $2)',
                [userId, codeHash]
            );
        }
        
        res.json({
            qrCode: qrCodeDataUrl,
            secret: secret.base32,
            recoveryCodes: recoveryCodes
        });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

router.post('/totp/verify-setup', async (req, res) => {
    try {
        const userId = req.user.id;
        const { code } = req.body;
        
        // Récupérer secret
        const result = await db.query(
            'SELECT encrypted_secret FROM user_totp_secrets WHERE user_id = $1',
            [userId]
        );
        
        if (result.rows.length === 0) {
            return res.status(400).json({ error: 'TOTP non initialisé' });
        }
        
        const secret = totpService.decryptSecret(result.rows[0].encrypted_secret);
        
        // Vérifier code
        if (totpService.verifyToken(secret, code)) {
            await db.query(
                'UPDATE user_totp_secrets SET is_active = $1, activated_at = $2 WHERE user_id = $3',
                [true, new Date(), userId]
            );
            res.json({ success: true });
        } else {
            res.status(400).json({ error: 'Code invalide' });
        }
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

module.exports = router;
```

## 3. Implémentation Frontend

### 3.1 Exemple React

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TOTPSetup = () => {
    const [step, setStep] = useState(1);
    const [qrCode, setQrCode] = useState('');
    const [secret, setSecret] = useState('');
    const [recoveryCodes, setRecoveryCodes] = useState([]);
    const [verificationCode, setVerificationCode] = useState('');
    const [error, setError] = useState('');

    const initializeTOTP = async () => {
        try {
            const response = await axios.post('/api/totp/setup');
            setQrCode(response.data.qrCode);
            setSecret(response.data.secret);
            setRecoveryCodes(response.data.recoveryCodes);
            setStep(2);
        } catch (err) {
            setError('Erreur lors de l\'initialisation');
        }
    };

    const verifySetup = async () => {
        try {
            await axios.post('/api/totp/verify-setup', {
                code: verificationCode
            });
            setStep(3);
        } catch (err) {
            setError('Code invalide. Veuillez réessayer.');
        }
    };

    return (
        <div className="totp-setup">
            {step === 1 && (
                <div>
                    <h2>Activer l'authentification à deux facteurs</h2>
                    <p>Renforcez la sécurité de votre compte avec TOTP</p>
                    <button onClick={initializeTOTP}>Commencer</button>
                </div>
            )}

            {step === 2 && (
                <div>
                    <h2>Scanner le QR Code</h2>
                    <img src={qrCode} alt="QR Code TOTP" />
                    <p>Scannez ce code avec votre application d'authentification</p>
                    <details>
                        <summary>Saisie manuelle</summary>
                        <code>{secret}</code>
                    </details>
                    
                    <h3>Vérification</h3>
                    <input
                        type="text"
                        maxLength="6"
                        placeholder="Code à 6 chiffres"
                        value={verificationCode}
                        onChange={(e) => setVerificationCode(e.target.value)}
                    />
                    <button onClick={verifySetup}>Vérifier</button>
                    {error && <p className="error">{error}</p>}
                </div>
            )}

            {step === 3 && (
                <div>
                    <h2>Codes de récupération</h2>
                    <p>Conservez ces codes en lieu sûr. Chaque code ne peut être utilisé qu'une fois.</p>
                    <ul>
                        {recoveryCodes.map((code, index) => (
                            <li key={index}><code>{code}</code></li>
                        ))}
                    </ul>
                    <button onClick={() => {/* télécharger codes */}}>
                        Télécharger
                    </button>
                    <button onClick={() => {/* terminer */}}>
                        J'ai sauvegardé mes codes
                    </button>
                </div>
            )}
        </div>
    );
};

const TOTPLogin = ({ onSuccess }) => {
    const [code, setCode] = useState('');
    const [useRecoveryCode, setUseRecoveryCode] = useState(false);
    const [trustDevice, setTrustDevice] = useState(false);
    const [error, setError] = useState('');

    const handleSubmit = async (e) => {
        e.preventDefault();
        try {
            await axios.post('/api/totp/validate', {
                code: code,
                is_recovery_code: useRecoveryCode,
                trust_device: trustDevice
            });
            onSuccess();
        } catch (err) {
            setError('Code invalide');
        }
    };

    return (
        <form onSubmit={handleSubmit} className="totp-login">
            <h2>Authentification à deux facteurs</h2>
            {!useRecoveryCode ? (
                <>
                    <p>Entrez le code à 6 chiffres de votre application</p>
                    <input
                        type="text"
                        maxLength="6"
                        pattern="[0-9]{6}"
                        value={code}
                        onChange={(e) => setCode(e.target.value)}
                        autoFocus
                    />
                </>
            ) : (
                <>
                    <p>Entrez un code de récupération</p>
                    <input
                        type="text"
                        value={code}
                        onChange={(e) => setCode(e.target.value)}
                        autoFocus
                    />
                </>
            )}
            
            <label>
                <input
                    type="checkbox"
                    checked={trustDevice}
                    onChange={(e) => setTrustDevice(e.target.checked)}
                />
                Se souvenir de cet appareil pendant 30 jours
            </label>

            {error && <p className="error">{error}</p>}
            
            <button type="submit">Valider</button>
            
            <button
                type="button"
                onClick={() => setUseRecoveryCode(!useRecoveryCode)}
            >
                {useRecoveryCode ? 'Utiliser un code TOTP' : 'Utiliser un code de récupération'}
            </button>
        </form>
    );
};

export { TOTPSetup, TOTPLogin };
```

## 4. Sécurité et Bonnes Pratiques

### 4.1 Checklist Sécurité
- [ ] Secrets TOTP chiffrés en base de données (AES-256)
- [ ] Codes de récupération hashés (SHA-256 minimum)
- [ ] HTTPS obligatoire pour toutes les communications
- [ ] Rate limiting sur les endpoints de validation
- [ ] Logs d'authentification complets
- [ ] Protection CSRF sur toutes les requêtes
- [ ] Sanitization des inputs utilisateur
- [ ] Gestion sécurisée des clés de chiffrement (KMS recommandé)

### 4.2 Tests à Effectuer
- [ ] Test de génération de secret unique
- [ ] Test de validation avec fenêtre de tolérance
- [ ] Test de codes de récupération
- [ ] Test de rate limiting
- [ ] Test de chiffrement/déchiffrement
- [ ] Test de compatibilité multi-applications
- [ ] Test de désynchronisation temporelle
- [ ] Test de charge (1000+ validations/s)

### 4.3 Monitoring
- Métriques à surveiller :
  - Taux de succès/échec des validations
  - Temps de réponse des endpoints
  - Nombre d'utilisateurs actifs avec TOTP
  - Codes de récupération utilisés
  - Tentatives suspectes

---

**Version** : 1.0  
**Date** : Octobre 2025  
**Type** : Guide d'implémentation technique
