Rapport de Patch de Sécurité

Vue d'ensemble
Ce document présente le travail complet de patch de sécurité effectué sur une application web. L'objectif était d'identifier, analyser et corriger les vulnérabilités de sécurité tout en documentant la méthodologie et les solutions apportées.

Méthodologie de détection des vulnérabilités

Processus d'analyse :

1. Analyse statique du code source

2. Recherche de patterns dangereux :
   - Concaténation de strings dans les requêtes SQL
   - Routes acceptant des entrées non validées
   - Logs exposant des données sensibles
   - Absence de validation des entrées utilisateur

3. Classification par criticité : Évaluation de l'impact potentiel de chaque vulnérabilité

Outils d'analyse utilisés :

Lecture manuelle du code
Recherche de patterns de sécurité anti-patterns

Analyse comparative du backend (server.js) :

Vulnérabilité 1 : Injection SQL dans la fonction insertRandomUsers
Code original (ligne 27) :
`INSERT INTO users (name, password) VALUES ('${fullName}', '${password}')`
Problème identifié : Concaténation directe des variables dans la requête SQL, permettant à un attaquant d'injecter du code SQL malveillant.

Exemple d'exploitation : Si fullName contient Robert'); DROP TABLE users;--, la requête devient destructrice.

Solution implémentée :
`INSERT INTO users (name, password) VALUES (?, ?)`, [fullName, password]

Explication : Utilisation de requêtes paramétrées. Les marqueurs ? sont remplacés par les valeurs du tableau, empêchant l'interprétation du contenu comme code SQL.

Vulnérabilité 2 : Route /query dangereusement permissive
Code original (lignes 47-50) :
app.post('/query', async (req, res) => {
  db.run(req.body)
  res.send('Inserted 3 users into database.');
});

Problème identifié : La route exécute directement le contenu du corps de la requête, permettant l'exécution de commandes SQL arbitraires.

Solution implémentée :
app.post('/query', async (req, res) => {
  res.status(403).send('Forbidden');
});

Explication : Désactivation complète de la route avec réponse HTTP 403 (Interdit).

Vulnérabilité 3 : Logs exposant des données sensibles
Code original (lignes 58-71) :
console.log(req.body);  // Affiche les données utilisateur
console.log('Query results:', rows);  // Affiche les résultats des requêtes

Problème identifié : Journalisation de données potentiellement sensibles (requêtes SQL, résultats de base de données).

Solutions implémentées :
1. Suppression des console.log sensibles

2. Remplacement des messages d'erreur détaillés par des messages génériques :
   - Avant : return res.status(500).json({ error: err.message });
   - Après : return res.status(500).json({ error: 'Database error' });

Vulnérabilité 4 : Absence de validation des entrées
Route /user originale :
app.post('/user', (req, res) => {
    console.log(req.body);
    db.all(req.body, [], (err, rows) => { ... });
});

Problèmes identifiés :
1. Pas de validation du type de données
2. Exécution directe du contenu de la requête

Solutions implémentées :
app.post('/user', (req, res) => {
    const userId = parseInt(req.body);
    if (isNaN(userId)) {
        return res.status(400).json({ error: 'Invalid user ID' });
    }
    
    db.all(`SELECT id, name FROM users WHERE id = ?`, [userId], (err, rows) => {
        if (err) {
            return res.status(500).json({ error: 'Database error' });
        }
        res.json(rows);
    });
});

Explication :

1. Conversion et validation de l'ID (doit être un nombre)
2. Utilisation de requête paramétrée au lieu d'exécution directe
3. Messages d'erreur génériques

Vulnérabilité 5 : Manque de validation des commentaires
Route /comment originale :
app.post('/comment', (req, res) => {
  const comment = req.body;
  db.all(`INSERT INTO comments (content) VALUES (?)`,[comment], (err) => { ... });
});

Solution implémentée :
app.post('/comment', (req, res) => {
  const comment = req.body;
  
  if (!comment || comment.length > 500) {
    return res.status(400).json({ error: 'Invalid comment' });
  }
  
  db.run(`INSERT INTO comments (content) VALUES (?)`, [comment], (err) => { ... });
});

Explication : Validation de la longueur des commentaires (maximum 500 caractères).

Amélioration structurelle : Organisation des tables
Problème original : La table comments était créée après les routes qui l'utilisaient.

Solution : Déplacement de la création de la table avant les routes :
// Déplacé des lignes 87-90 aux lignes 12-15
db.run(`CREATE TABLE IF NOT EXISTS comments (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  content TEXT NOT NULL
)`);
Explication : Garantit que la table existe avant que les routes ne soient définies.

Analyse comparative du frontend (App.js)
Vulnérabilité 1 : Construction de requêtes SQL côté client
Code original :
`SELECT id, name FROM users WHERE id = ${queryId}`

Problème identifié : Construction dynamique de requête SQL côté client, permettant l'injection indirecte.

Solution implémentée :
String(id)  // Envoi uniquement de l'ID

Explication : Le frontend envoie uniquement l'ID, la construction de la requête SQL est gérée de manière sécurisée par le backend.

Vulnérabilité 2 : Exposition des mots de passe dans l'interface
Code original :
<p key={u.id}>
  ID: {u.id} — Name: {u.name} — Password: {u.password}
</p>

Problème identifié : Affichage des mots de passe en clair.

Solution implémentée :
<p key={u.id}>
  ID: {u.id} — Name: {u.name}
</p>

Explication : Suppression de l'affichage des mots de passe.

Vulnérabilité 3 : Absence de validation côté client
Code original : Pas de validation avant l'envoi des données au serveur.

Solutions implémentées :
// Validation de l'ID
const id = parseInt(queryId);
if (isNaN(id) || id <= 0) {
  alert('Please enter a valid positive user ID');
  return;
}

// Validation des commentaires
if (!newComment.trim() || newComment.length > 500) {
  alert('Comment cannot be empty or too long (max 500 chars)');
  return;
}

Explication : Validation des entrées utilisateur avant l'envoi au serveur.

Amélioration : Externalisation de la configuration
Code original : URL hardcodée à plusieurs endroits :
'http://localhost:8000'

Solution implémentée :
const API_BASE = process.env.REACT_APP_API_BASE || 'http://localhost:8000';

Explication : Utilisation d'une variable d'environnement pour la configuration de l'URL API.

Dockerisation de l'application
Objectifs de la dockerisation
1/ Reproductibilité : Garantir que l'application s'exécute de manière identique sur tout environnement
2/ Isolation : Séparer les services frontend et backend dans des conteneurs distincts
3/ Portabilité : Faciliter le déploiement sur différentes plateformes
4/ Simplicité : Permettre le lancement de toute l'application avec une seule commande

Structure des fichiers Docker :
docker/backend/Dockerfile

FROM node:18-alpine
WORKDIR /app
COPY ../../backend/package*.json ./
RUN npm ci --only=production
COPY ../../backend/server.js .
EXPOSE 8000
CMD ["node", "server.js"]

Explication :

- Image de base légère avec Node.js 18
- Installation des dépendances de production uniquement
- Exposition du port 8000
- Commande de démarrage du serveur

docker/frontend/Dockerfile
FROM node:18-alpine as build
WORKDIR /app
COPY ../../frontend/my-app/package*.json ./
RUN npm ci
COPY ../../frontend/my-app/ .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

Explication :

- Build en deux étapes pour optimiser la taille
- Première étape : Construction de l'application React
- Deuxième étape : Service avec Nginx pour servir les fichiers statiques
- Exposition du port 80

docker/docker-compose.yml
version: '3.8'
services:
  backend:
    build:
      context: ../backend
      dockerfile: ../docker/backend/Dockerfile
    ports:
      - "8000:8000"
    environment:
      - NODE_ENV=production
    volumes:
      - ../backend/database.db:/app/database.db

  frontend:
    build:
      context: ../frontend/my-app
      dockerfile: ../docker/frontend/Dockerfile
    ports:
      - "3000:80"
    environment:
      - REACT_APP_API_BASE=http://backend:8000
    depends_on:
      - backend


Explication :

- Définition des deux services (backend et frontend)
- Configuration des ports et variables d'environnement
- Dépendance du frontend sur le backend
- Montage du volume pour la base de données

Avantages de la solution Dockerisée
1. Environnement contrôlé : Toutes les dépendances sont encapsulées
2. Facilité de déploiement : docker-compose up lance toute l'application
3. Consistance : Même environnement de développement et de production
4. Scalabilité : Possibilité de scaler les services indépendamment


Conclusion du travail de patch :

Ce travail a démontré une approche méthodique de sécurisation d'application web :

1. Analyse approfondie : Identification systématique des vulnérabilités
2. Corrections appropriées : Application des meilleures pratiques de sécurité
3. Modernisation infrastructure : Ajout de Docker pour la portabilité

L'application est désormais robuste contre les attaques courantes tout en restant maintenable et facile à déployer. Les corrections adressent à la fois la sécurité fonctionnelle et la sécurité infrastructurelle.
