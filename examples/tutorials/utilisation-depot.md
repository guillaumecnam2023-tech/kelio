# Tutoriel : Utilisation du Dépôt

## Objectif

Apprendre à utiliser ce dépôt pour organiser et partager vos fichiers.

## Étape 1 : Comprendre la Structure

Le dépôt est organisé en plusieurs répertoires :
- `docs/` : Documentation
- `src/` : Code source
- `data/` : Données
- `examples/` : Exemples
- `resources/` : Ressources

## Étape 2 : Ajouter un Fichier

1. Identifiez le type de fichier que vous voulez ajouter
2. Placez-le dans le répertoire approprié
3. Ajoutez une description ou un README si nécessaire

### Exemple

Pour ajouter un script Python :

```bash
# Créez votre script
nano src/mon-script.py

# Ajoutez-le au dépôt
git add src/mon-script.py
git commit -m "ajout: script d'analyse de données"
git push
```

## Étape 3 : Documenter

Ajoutez toujours une documentation claire :
- Commentaires dans le code
- README dans les sous-répertoires
- Documentation technique si nécessaire

## Étape 4 : Partager

Une fois vos fichiers ajoutés et documentés, ils sont prêts à être utilisés pour répondre aux questions et effectuer des analyses.

## Conseils

- Utilisez des noms de fichiers descriptifs
- Organisez par catégorie ou par projet
- Mettez à jour régulièrement la documentation
- Utilisez le .gitignore pour exclure les fichiers temporaires
