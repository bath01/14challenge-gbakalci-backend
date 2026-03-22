# Gbakalci Backend

> **"Gbakal"** = ambiance / fête en nouchi (argot ivoirien). Backend d'une plateforme de musique ivoirienne — sans inscription ni connexion requise.

API REST construite avec **.NET 9**, **Entity Framework Core** et **MySQL**.

---

## Stack

| Élément | Technologie |
|---|---|
| Framework | ASP.NET Core 9 Web API |
| ORM | Entity Framework Core 9 |
| Base de données | MySQL (via Pomelo EF Core) |
| Documentation | Swagger / OpenAPI |
| Stockage fichiers | Système de fichiers local (`/uploads`) |

---

## Démarrage rapide

### Prérequis

- [.NET 9 SDK](https://dotnet.microsoft.com/download)
- MySQL 8+
- `dotnet-ef` : `dotnet tool install --global dotnet-ef`

### Configuration

Éditer `appsettings.json` :

```json
"ConnectionStrings": {
  "DefaultConnection": "Server=localhost;Port=3306;Database=gbakalci;User=root;Password=TONMOTDEPASSE;"
}
```

### Lancer

```bash
dotnet run
```

Les migrations sont appliquées automatiquement au démarrage. Les dossiers `uploads/audio` et `uploads/covers` sont créés si absents.

Swagger disponible sur : `http://localhost:PORT/swagger`

---

## Docker

```bash
# Build
docker build -t gbakalci-backend .

# Run
docker run -d \
  -p 8080:8080 \
  -v $(pwd)/uploads:/app/uploads \
  -e ConnectionStrings__DefaultConnection="Server=HOST_MYSQL;Port=3306;Database=gbakalci;User=root;Password=MOTDEPASSE;" \
  gbakalci-backend
```

> Le dossier `uploads` doit être monté en volume pour persister les fichiers entre les redémarrages du container.

---

## Modèle de données

```
Category
  id, name

Track
  id, categoryId (→ Category), artist, title, cover, duration (sec), filePath

Playlist
  id, name

PlaylistTrack  (jointure many-to-many)
  playlistId (→ Playlist), trackId (→ Track)
```

Un morceau peut appartenir à 0 ou plusieurs playlists. `TotalDuration` d'une playlist est calculé dynamiquement (somme des `duration` des morceaux).

---

## API Reference

### Catégories — `/api/categories`

| Méthode | Route | Description |
|---|---|---|
| GET | `/api/categories` | Liste toutes les catégories |
| GET | `/api/categories/{id}` | Détail d'une catégorie |
| POST | `/api/categories` | Créer une catégorie |
| PUT | `/api/categories/{id}` | Modifier une catégorie |
| DELETE | `/api/categories/{id}` | Supprimer une catégorie |

**Corps POST/PUT** (`application/json`) :
```json
{ "name": "Coupé-Décalé" }
```

**Réponse** :
```json
{ "id": 1, "name": "Coupé-Décalé", "trackCount": 12 }
```

---

### Morceaux — `/api/tracks`

| Méthode | Route | Description |
|---|---|---|
| GET | `/api/tracks` | Liste tous les morceaux |
| GET | `/api/tracks?categoryId={id}` | Morceaux filtrés par catégorie |
| GET | `/api/tracks/{id}` | Détail d'un morceau |
| GET | `/api/tracks/{id}/audio` | Stream du fichier audio |
| POST | `/api/tracks` | Ajouter un morceau (upload) |
| PUT | `/api/tracks/{id}` | Modifier un morceau |
| DELETE | `/api/tracks/{id}` | Supprimer un morceau |

**Corps POST** (`multipart/form-data`) :

| Champ | Type | Requis | Description |
|---|---|---|---|
| `categoryId` | int | oui | ID de la catégorie |
| `artist` | string | oui | Nom de l'artiste |
| `title` | string | oui | Titre du morceau |
| `duration` | int | oui | Durée en secondes |
| `audioFile` | file | oui | Fichier audio (mp3, wav, ogg…) |
| `coverFile` | file | non | Image de couverture |

**Corps PUT** (`multipart/form-data`) : tous les champs sont optionnels.

**Réponse** :
```json
{
  "id": 3,
  "categoryId": 1,
  "categoryName": "Coupé-Décalé",
  "artist": "DJ Arafat",
  "title": "Dosabado",
  "duration": 214,
  "coverUrl": "https://domaine.com/uploads/covers/abc123.jpg"
}
```

**`GET /api/tracks/{id}/audio`**
- Retourne le fichier audio directement (pas une URL)
- Supporte les **Range requests** (permet le seek dans un player)
- `Content-Type` détecté automatiquement selon l'extension

---

### Playlists — `/api/playlists`

| Méthode | Route | Description |
|---|---|---|
| GET | `/api/playlists` | Liste toutes les playlists |
| GET | `/api/playlists/{id}` | Détail d'une playlist avec ses morceaux |
| POST | `/api/playlists` | Créer une playlist |
| PUT | `/api/playlists/{id}` | Renommer une playlist |
| DELETE | `/api/playlists/{id}` | Supprimer une playlist |
| POST | `/api/playlists/{id}/tracks/{trackId}` | Ajouter un morceau à la playlist |
| DELETE | `/api/playlists/{id}/tracks/{trackId}` | Retirer un morceau de la playlist |

**Corps POST/PUT** (`application/json`) :
```json
{ "name": "Ma playlist du soir" }
```

**Réponse** :
```json
{
  "id": 2,
  "name": "Ma playlist du soir",
  "totalDuration": 642,
  "tracks": [
    {
      "id": 3,
      "categoryId": 1,
      "categoryName": "Coupé-Décalé",
      "artist": "DJ Arafat",
      "title": "Dosabado",
      "duration": 214,
      "coverUrl": "https://ton-domaine.com/uploads/covers/abc123.jpg"
    }
  ]
}
```

---

## Fichiers uploadés

Les fichiers sont servis comme ressources statiques :

```
https://ton-domaine.com/uploads/audio/{filename}   ← audio
https://ton-domaine.com/uploads/covers/{filename}  ← couvertures
```

Les noms de fichiers sont générés avec un UUID pour éviter les conflits. Lors de la suppression ou de la mise à jour d'un morceau, les anciens fichiers sont automatiquement supprimés du disque.
