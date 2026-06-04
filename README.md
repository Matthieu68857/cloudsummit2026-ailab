# Atelier Pratique (75 min) : Concevoir des Applications Agentiques avec AlloyDB, Gemini et MCP (Scénario Cinéma)

> [!NOTE]
> **Durée estimée :** 1h15 (75 minutes)  
> **Niveau :** Intermédiaire  
> **Objectif :** Apprendre à coupler les services de données de Google Cloud (**AlloyDB**) avec **Gemini** et le protocole ouvert **Model Context Protocol (MCP)** pour concevoir des agents IA intelligents, hautement sécurisés et performants.

---

## 🎯 Abstract de l'Atelier

Pour que vos agents IA soient réellement efficaces, vos bases de données doivent suivre le rythme. Cet atelier pratique vous montre comment coupler les services data de Google Cloud (notamment **AlloyDB**) avec **Gemini** et le **Model Context Protocol (MCP)** pour créer votre prochaine génération d'applications. Nous aborderons concrètement comment intégrer l'IA à la source avec **AlloyDB AI**, comment ouvrir des accès sécurisés à vos agents via **MCP**, et comment exploiter **QueryData** pour la traduction du langage naturel en SQL.

---

## ⏱️ Agenda de l'Atelier (75 minutes)

| Étape | Sujet | Durée |
| :--- | :--- | :--- |
| **Étape 1** | Configuration et Initialisation de la Base de Données | **15 min** |
| **Étape 2** | L'IA à la source avec AlloyDB AI Operators | **15 min** |
| **Étape 3** | Traduction sémantique prévisible avec QueryData | **15 min** |
| **Étape 4** | Sécurisation des accès avec MCP et l'Agent Sémantique (ADK) | **15 min** |
| **Étape 5** | Agent Conversational Analytics depuis la Console AlloyDB | **15 min** |

---

## 🛠️ Étape 1 : Configuration et Initialisation de la Base de Données (15 min)

### 1.1 Création de la base de données cinema_db
1. Dans la console Google Cloud, accédez à **AlloyDB > Clusters**, puis cliquez sur votre cluster **`alloydb-cinema-cluster`**.
2. Sur la page de votre instance principale **`alloydb-cinema-cluster-pr`**, cliquez sur **AlloyDB Studio** dans le menu de gauche.
3. Authentifiez-vous en utilisant l'authentification IAM avec les paramètres suivants :
   *   **Database** : `postgres`
   *   **Authentication Method** (Méthode d'authentification) : Sélectionnez `IAM database authentication` (Authentification de base de données IAM)
   *   **User** (Utilisateur) : Votre adresse email de compte Google (déjà sélectionnée par défaut si vous choisissez l'authentification IAM).
4. Cliquez sur le bouton **Authenticate** (S'authentifier).
5. Dans l'éditeur de requêtes d'AlloyDB Studio, exécutez la commande suivante pour créer notre base de données de travail :
   ```sql
   CREATE DATABASE cinema_db;
   ```
6. Une fois exécutée avec succès, cliquez sur le bouton **Switch user/database** (ou le bouton de changement d'utilisateur/base en haut à gauche) et reconnectez-vous à la base de données **`cinema_db`** en utilisant à nouveau l'**authentification IAM** avec votre adresse email Google.

### 1.2 Initialisation des Extensions, du Schéma et des Données
Copiez et exécutez le script SQL complet suivant dans l'éditeur de requêtes pour activer les extensions requises, créer le schéma des 9 tables et insérer notre jeu de données cinémato-graphique :

```sql
-- Activation des extensions requises
CREATE EXTENSION IF NOT EXISTS vector CASCADE;
CREATE EXTENSION IF NOT EXISTS google_ml_integration CASCADE;
CREATE EXTENSION IF NOT EXISTS alloydb_scann CASCADE;
CREATE EXTENSION IF NOT EXISTS rum CASCADE;

-- Activer le moteur d'IA sémantique embarqué d'AlloyDB pour notre base
ALTER DATABASE cinema_db SET google_ml_integration.enable_ai_query_engine = 'on';

-- Nettoyage préalable des tables
DROP TABLE IF EXISTS public.movie_reviews CASCADE;
DROP TABLE IF EXISTS public.tickets CASCADE;
DROP TABLE IF EXISTS public.showtimes CASCADE;
DROP TABLE IF EXISTS public.theaters CASCADE;
DROP TABLE IF EXISTS public.movie_cast CASCADE;
DROP TABLE IF EXISTS public.actors CASCADE;
DROP TABLE IF EXISTS public.movie_genres CASCADE;
DROP TABLE IF EXISTS public.genres CASCADE;
DROP TABLE IF EXISTS public.movies CASCADE;

-- 1. Table des Films (avec génération d'embeddings automatique)
CREATE TABLE public.movies (
    movie_id BIGINT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    release_year INT,
    duration_mins INT,
    -- Génération automatique de l'embedding textuel
    description_embedding vector(3072) GENERATED ALWAYS AS (embedding('gemini-embedding-001', description)) STORED,
    -- Génération automatique du tsvector en français pour le FTS
    description_tsvector tsvector GENERATED ALWAYS AS (to_tsvector('french', description)) STORED
);

-- 2. Table des Genres
CREATE TABLE public.genres (
    genre_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- 3. Table de liaison Films-Genres
CREATE TABLE public.movie_genres (
    movie_id BIGINT REFERENCES public.movies(movie_id) ON DELETE CASCADE,
    genre_id INT REFERENCES public.genres(genre_id) ON DELETE CASCADE,
    PRIMARY KEY (movie_id, genre_id)
);

-- 4. Table des Acteurs
CREATE TABLE public.actors (
    actor_id INT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    birth_year INT
);

-- 5. Table de liaison Casting (Films-Acteurs)
CREATE TABLE public.movie_cast (
    movie_id BIGINT REFERENCES public.movies(movie_id) ON DELETE CASCADE,
    actor_id INT REFERENCES public.actors(actor_id) ON DELETE CASCADE,
    role_name VARCHAR(255),
    PRIMARY KEY (movie_id, actor_id)
);

-- 6. Table des Salles de cinéma
CREATE TABLE public.theaters (
    theater_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    screen_type VARCHAR(50) NOT NULL, -- 'IMAX', 'Dolby Cinema', 'Standard'
    total_seats INT NOT NULL
);

-- 7. Table des Séances
CREATE TABLE public.showtimes (
    showtime_id BIGINT PRIMARY KEY,
    movie_id BIGINT REFERENCES public.movies(movie_id) ON DELETE CASCADE,
    theater_id INT REFERENCES public.theaters(theater_id) ON DELETE CASCADE,
    show_date DATE NOT NULL,
    start_time TIME NOT NULL,
    price NUMERIC(5,2) NOT NULL,
    remaining_seats INT NOT NULL
);

-- 8. Table des Billets/Réservations
CREATE TABLE public.tickets (
    ticket_id BIGINT PRIMARY KEY,
    showtime_id BIGINT REFERENCES public.showtimes(showtime_id) ON DELETE CASCADE,
    customer_name VARCHAR(255) NOT NULL,
    seats_reserved INT NOT NULL,
    booking_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 9. Table des Avis Clients
CREATE TABLE public.movie_reviews (
    review_id BIGINT PRIMARY KEY,
    movie_id BIGINT REFERENCES public.movies(movie_id) ON DELETE CASCADE,
    reviewer_name VARCHAR(255) NOT NULL,
    rating INT CHECK (rating >= 1 AND rating <= 5),
    review_text TEXT NOT NULL,
    review_date DATE DEFAULT CURRENT_DATE
);

-- Insertion des Genres
INSERT INTO public.genres (genre_id, name) VALUES
(1, 'Action'),
(2, 'Aventure'),
(3, 'Drame'),
(4, 'Science-Fiction'),
(5, 'Thriller'),
(6, 'Romance'),
(7, 'Animation'),
(8, 'Fantastique'),
(9, 'Crime'),
(10, 'Musical');

-- Insertion des Acteurs
INSERT INTO public.actors (actor_id, name, birth_year) VALUES
(1, 'Leonardo DiCaprio', 1974),
(2, 'Joseph Gordon-Levitt', 1981),
(3, 'Elliot Page', 1987),
(4, 'Matthew McConaughey', 1969),
(5, 'Anne Hathaway', 1982),
(6, 'Jessica Chastain', 1977),
(7, 'Christian Bale', 1974),
(8, 'Heath Ledger', 1979),
(9, 'Aaron Eckhart', 1968),
(10, 'Russell Crowe', 1964),
(11, 'Joaquin Phoenix', 1974),
(12, 'Keanu Reeves', 1964),
(13, 'Laurence Fishburne', 1961),
(14, 'Carrie-Anne Moss', 1967),
(15, 'Timothée Chalamet', 1995),
(16, 'Zendaya', 1996),
(17, 'Ryan Gosling', 1980),
(18, 'Emma Stone', 1988),
(19, 'Marlon Brando', 1924),
(20, 'Al Pacino', 1940),
(21, 'John Travolta', 1954),
(22, 'Samuel L. Jackson', 1948);

-- Insertion des Salles
INSERT INTO public.theaters (theater_id, name, screen_type, total_seats) VALUES
(1, 'Salle IMAX Luxe 1', 'IMAX', 200),
(2, 'Salle Dolby Atmos 2', 'Dolby Cinema', 150),
(3, 'Salle Standard 3', 'Standard', 100),
(4, 'Salle Standard 4', 'Standard', 100);

-- Insertion des Films
INSERT INTO public.movies (movie_id, title, description, release_year, duration_mins) VALUES
(1, 'Inception', 'Un voleur de secrets industriels spécialisé dans l''extraction de données à travers les rêves doit réaliser sa mission la plus périlleuse : implanter une idée dans le subconscient d''un héritier.', 2010, 148),
(2, 'Interstellar', 'Dans un futur proche où la Terre se meurt d''asphyxie, un groupe de pilotes et scientifiques voyage à travers un trou de ver pour explorer de nouvelles galaxies et trouver un refuge pour l''humanité.', 2014, 169),
(3, 'The Dark Knight', 'Batman affronte le criminel machiavélique connu sous le nom du Joker, un psychopathe imprévisible qui sème le chaos et teste les limites morales du chevalier noir à Gotham.', 2008, 152),
(4, 'Gladiator', 'Un général romain trahi par l''empereur usurpateur Commode voit sa famille assassinée et se bat comme gladiateur au Colisée pour assouvir sa vengeance.', 2000, 155),
(5, 'The Matrix', 'Un hacker informatique apprend que la réalité est une simulation virtuelle construite par des machines cybernétiques rebelles pour contrôler l''humanité, et se joint à la lutte de libération.', 1999, 136),
(6, 'Dune', 'L''héritier de la maison des Atréides, Paul, est envoyé sur Arrakis, une dangereuse planète désertique produisant la ressource la plus précieuse de l''univers : l''Épice de prescience.', 2021, 155),
(7, 'La La Land', 'À Los Angeles, une actrice en devenir et un pianiste de jazz tombent amoureux tout en poursuivant leurs carrières artistiques respectives, menacées par leurs ambitions personnelles.', 2016, 128),
(8, 'Le Parrain', 'Le patriarche vieillissant d''une dynastie du crime organisé en Amérique transfère le contrôle de son empire clandestin à son fils réticent.', 1972, 175),
(9, 'Pulp Fiction', 'Les vies de deux tueurs à gages, d''un boxeur, d''un gangster et de sa femme s''entremêlent dans quatre histoires de violence et de rédemption à Los Angeles.', 1994, 154),
(10, 'Avatar', 'Sur la lointaine planète de Pandora, un marine paraplégique envoyé en mission s''infiltre parmi les autochtones Na''vi et se retrouve tiraillé entre son devoir et l''amour.', 2009, 162),
(11, 'Le Voyage de Chihiro', 'Une jeune fille de dix ans s''égare dans le monde des esprits. Après la transformation de ses parents, elle doit travailler dans les bains d''une sorcière pour les libérer.', 2001, 125),
(12, 'Le Roi Lion', 'Un jeune lionceau, héritier du trône, doit surmonter la trahison de son oncle et la mort de son père pour revendiquer sa place légitime de roi de la Terre des Lions.', 1994, 88),
(13, 'Whiplash', 'Un jeune batteur de jazz prometteur est soumis à la méthode d''enseignement impitoyable et psychologiquement abusive de son chef d''orchestre dans un conservatoire prestigieux.', 2014, 106);

-- Liaisons Films-Genres
INSERT INTO public.movie_genres (movie_id, genre_id) VALUES
(1, 1), (1, 4), (1, 5),
(2, 2), (2, 3), (2, 4),
(3, 1), (3, 3), (3, 9),
(4, 1), (4, 2), (4, 3),
(5, 1), (5, 4),
(6, 2), (6, 4), (6, 3),
(7, 3), (7, 6), (7, 10),
(8, 9), (8, 3),
(9, 9), (9, 5),
(10, 2), (10, 4),
(11, 7), (11, 8),
(12, 7), (12, 2), (12, 10),
(13, 3), (13, 10);

-- Casting des Films
INSERT INTO public.movie_cast (movie_id, actor_id, role_name) VALUES
(1, 1, 'Cobb'), (1, 2, 'Arthur'), (1, 3, 'Ariadne'),
(2, 4, 'Cooper'), (2, 5, 'Brand'), (2, 6, 'Murph'),
(3, 7, 'Bruce Wayne / Batman'), (3, 8, 'The Joker'), (3, 9, 'Harvey Dent'),
(4, 10, 'Maximus'), (4, 11, 'Commodus'),
(5, 12, 'Neo'), (5, 13, 'Morpheus'), (5, 14, 'Trinity'),
(6, 15, 'Paul Atreides'), (6, 16, 'Chani'),
(7, 17, 'Sebastian'), (7, 18, 'Mia'),
(8, 19, 'Vito Corleone'), (8, 20, 'Michael Corleone'),
(9, 21, 'Vincent Vega'), (9, 22, 'Jules Winnfield');

-- Insertion des Séances
INSERT INTO public.showtimes (showtime_id, movie_id, theater_id, show_date, start_time, price, remaining_seats) VALUES
(1, 1, 1, CURRENT_DATE, '18:00:00', 15.50, 150),
(2, 1, 1, CURRENT_DATE, '21:00:00', 15.50, 4),
(3, 2, 1, CURRENT_DATE, '20:30:00', 15.50, 80),
(4, 3, 2, CURRENT_DATE, '19:30:00', 13.00, 2),
(5, 3, 2, CURRENT_DATE + 1, '21:30:00', 13.00, 120),
(6, 5, 2, CURRENT_DATE, '22:30:00', 13.00, 148),
(7, 6, 1, CURRENT_DATE, '15:00:00', 15.50, 12),
(8, 7, 3, CURRENT_DATE, '17:30:00', 9.50, 3),
(9, 8, 2, CURRENT_DATE, '14:00:00', 13.00, 45),
(10, 9, 3, CURRENT_DATE, '21:00:00', 9.50, 85),
(11, 10, 1, CURRENT_DATE, '11:00:00', 15.50, 190);

-- Insertion des Réservations (Tickets)
INSERT INTO public.tickets (ticket_id, showtime_id, customer_name, seats_reserved) VALUES
(1, 2, 'Alice Martin', 2),
(2, 2, 'Bob Durand', 4),
(3, 4, 'Charlie Lemoine', 3),
(4, 8, 'Diana Prince', 2),
(5, 7, 'Edward Nygma', 5);

-- Insertion des Avis Clients
INSERT INTO public.movie_reviews (review_id, movie_id, reviewer_name, rating, review_text) VALUES
(1, 1, 'Critique Masqué', 5, 'Inception est absolument fascinant et hallucinant ! Un chef-d''œuvre absolu de science-fiction cérébrale.'),
(2, 1, 'Bob Le Râleur', 2, 'Trop complexe et prétentieux. Je n''ai rien compris aux rêves dans les rêves.'),
(3, 2, 'Stellaire99', 5, 'Interstellar est visuellement époustouflant et émotionnellement intense. La musique de Zimmer est divine !'),
(4, 3, 'GothamFan', 5, 'La prestation de Heath Ledger en Joker est légendaire. Le film est sombre, tendu et parfait.'),
(5, 5, 'CyberNéo', 5, 'Révolutionnaire pour l''époque ! L''action et la philosophie cyberpunk s''accordent parfaitement.'),
(6, 6, 'ArrakisTraveler', 5, 'Une adaptation grandiose de l''œuvre d''Herbert. Denis Villeneuve a fait un travail titanesque.'),
(7, 7, 'Mélomane', 5, 'La La Land m''a mis des étoiles dans les yeux. Ryan Gosling et Emma Stone sont formidables.'),
(8, 3, 'BatmanLover', 5, 'Le meilleur film de super-héros de tous les temps. Une écriture brillante et une tension incroyable.'),
(9, 5, 'MatrixFan', 4, 'Des effets spéciaux révolutionnaires et un scénario très intelligent qui fait réfléchir.');
```

---

## 🛠️ Étape 2 : L'IA à la source avec AlloyDB AI Operators (15 min)

Plutôt que de ramener toutes les données côté application pour les traiter avec un LLM, **AlloyDB AI** permet d'appliquer des fonctions intelligentes directement au cœur du moteur SQL.

### 2.1 Génération d'embeddings et recherche sémantique simple
AlloyDB AI expose des fonctions intégrées directement en SQL pour appeler les modèles d'embedding de Vertex AI. Pour calculer l'embedding d'une phrase en temps réel et trouver les films les plus similaires, exécutez la requête suivante :
```sql
SELECT title, description,
  description_embedding <=> embedding('gemini-embedding-001', 'un voyage spatial intersidéral pour sauver l''humanité')::vector AS distance
FROM public.movies
ORDER BY distance ASC
LIMIT 5;
```
> [!NOTE]
> L'opérateur `<=>` calcule la distance cosinus entre deux vecteurs. Une distance proche de 0 indique une forte similarité sémantique. Ici, **Interstellar** ressortira en première position sans avoir besoin de correspondance exacte par mot-clé.

### 2.2 Filtrage Sémantique avec `ai.if` (Optimisé)
> [!WARNING]
> **Avertissement de Performance :** Les fonctions comme `ai.if` interrogent un grand modèle de langage (LLM) sous-jacent. L'exécuter sur une table entière sans filtre force la base de données à appeler l'API de Vertex AI pour *chaque* ligne de la table, ce qui est extrêmement lent.
>
> Pour garantir de bonnes performances, vous devez **toujours combiner l'opérateur IA avec des clauses de filtrage classiques** (par exemple en limitant la recherche aux IDs de films ou aux années de sortie récentes) pour restreindre l'appel de l'IA à un très petit nombre de lignes.

Essayons avec une requête ciblée sur un sous-ensemble de films :
```sql
SELECT title, description
FROM public.movies
WHERE ai.if(
    prompt => 'Parmi ces descriptions de films, renvoie VRAI uniquement pour ceux qui traitent de voyages dans l''espace ou de l''exploration du cosmos : ' || description
  );
```

### 2.3 Jointures Sémantiques
De la même manière, nous pouvons réaliser des jointures intelligentes en limitant le périmètre pour préserver les performances :
```sql
SELECT m.title, r.reviewer_name, r.review_text
FROM public.movies m
JOIN public.movie_reviews r ON m.movie_id = r.movie_id
WHERE m.movie_id = 1 -- Filtrage sur le film Inception uniquement
  AND google_ml.if(
    prompt => 'Est-ce que l''avis critique suivant exprime de la déception ou de l''incompréhension face au film ? L''avis : ' || r.review_text
  );
```

### 2.4 Recherche Hybride de Haute Performance avec RRF
Combinons la précision de la recherche par mot-clé (FTS) et la sémantique de la recherche vectorielle à l'aide de l'index combiné d'AlloyDB.

1. Créez des index haute performance sur la table `movies` :
```sql
-- Index RUM pour la recherche plein texte (FTS) sur le tsvector
CREATE INDEX IF NOT EXISTS movies_tsvector_idx ON public.movies USING RUM (description_tsvector rum_tsvector_ops);

-- Index ScaNN pour la recherche vectorielle rapide (Cosinus de similarité)
CREATE INDEX IF NOT EXISTS movies_vector_idx ON public.movies USING scann (description_embedding cosine) WITH (num_leaves=10);
```

2. Activez les fonctions de préversion de l'IA et exécutez la recherche hybride.
> [!IMPORTANT]
> **Résolution du problème de typage (BIGINT) :** Par défaut, `ai.hybrid_search` retourne une colonne d'ID de type `TEXT`. Si vos clés primaires sont de type `BIGINT` (comme `movie_id`), la jointure générera une erreur d'incompatibilité de type. Pour résoudre ce problème, spécifiez explicitement le paramètre `id_type => NULL::BIGINT` pour forcer le transtypage automatique dans le bon format.

```sql
SET google_ml_integration.enable_preview_ai_functions = true;

SELECT m.title, m.description, search_results.score
FROM public.movies m
JOIN ai.hybrid_search(
  search_inputs => ARRAY[
      $$ {
        "data_type": "vector",
        "table_name": "movies",
        "key_column": "movie_id",
        "vec_column": "description_embedding",
        "distance_operator": "public.<=>",
        "limit": 3,
        "query_vector": "ai.embedding('gemini-embedding-001', 'thriller sur les rêves et l''inconscient')::vector"
      } $$::JSONB,
      $$ {
        "data_type": "text",
        "table_name": "movies",
        "key_column": "movie_id",
        "text_column": "description_tsvector",
        "limit": 3,
        "ranking_function": "<=>",
        "query_text_input": "Inception"
      } $$::JSONB
  ],
  id_type => NULL::BIGINT
) AS search_results ON m.movie_id = search_results.id;
```

---

## 🛠️ Étape 3 : Traduction sémantique prévisible avec QueryData (15 min)

Bien que les LLM soient doués pour écrire du SQL, ils peuvent halluciner des tables ou des colonnes inexistantes. **QueryData** apporte de la prévisibilité en guidant le LLM grâce à des templates pré-approuvés définissant comment mapper des requêtes en langage naturel vers des clauses SQL strictes.

### 3.1 Création du Fichier de Contexte QueryData
Nous allons concevoir un fichier JSON (`querydata_cinema_contextset.json`) qui apprend au modèle Gemini Data Analytics comment interpréter des questions complexes nécessitant des jointures sur plusieurs tables (films, séances, salles, acteurs, billets, etc.) et comment appliquer des filtres intelligents ou faire correspondre des termes saisis :
1. Trouver les séances disponibles dans un certain type de salle (ex: IMAX) pour un genre donné.
2. Lister la distribution (acteurs et rôles) d'un film.
3. Afficher les billets d'un client.
4. Calculer la note moyenne des avis pour les films dans lesquels joue un certain acteur.
5. Appliquer une règle métier personnalisée complexe : trouver les films "incontournables" d'un genre (définis par au moins 2 avis et une note moyenne de 4.5 ou plus).
6. Détecter des conflits logistiques complexes : identifier les séances qui se chevaucheent dans la même salle à la même date (nécessitant une auto-jointure et des calculs d'intervalles temporels).
7. Filtrer les films par durée ou par note minimale d'avis clients (via les **facets**).
8. Associer sans ambiguïté des mots-clés de recherche en langage naturel aux valeurs de la base, comme les noms de films, d'acteurs, de genres et de types de salle (via les **value_searches**).

Créez un fichier nommé `querydata_cinema_contextset.json` sur votre ordinateur local et collez-y le contenu suivant :
```json
{
  "templates": [
    {
      "nlQuery": "Séances de cinéma disponibles aujourd'hui dans une salle IMAX pour les films de Science-Fiction",
      "sql": "SELECT s.showtime_id, m.title, t.name AS theater_name, s.start_time, s.remaining_seats FROM public.showtimes s INNER JOIN public.movies m ON s.movie_id = m.movie_id INNER JOIN public.movie_genres mg ON m.movie_id = mg.movie_id INNER JOIN public.genres g ON mg.genre_id = g.genre_id INNER JOIN public.theaters t ON s.theater_id = t.theater_id WHERE g.name = 'Science-Fiction' AND t.screen_type = 'IMAX' AND s.show_date = CURRENT_DATE AND s.remaining_seats > 0 ORDER BY s.start_time ASC",
      "intent": "Trouver les séances de cinéma pour un genre spécifique et un type d'écran donnés aujourd'hui",
      "manifest": "Trouver les séances disponibles aujourd'hui pour un genre et un type de salle",
      "parameterized": {
        "parameterized_intent": "Séances disponibles aujourd'hui dans une salle $1 pour les films de $2",
        "parameterized_sql": "SELECT s.showtime_id, m.title, t.name AS theater_name, s.start_time, s.remaining_seats FROM public.showtimes s INNER JOIN public.movies m ON s.movie_id = m.movie_id INNER JOIN public.movie_genres mg ON m.movie_id = mg.movie_id INNER JOIN public.genres g ON mg.genre_id = g.genre_id INNER JOIN public.theaters t ON s.theater_id = t.theater_id WHERE g.name = $2 AND t.screen_type = $1 AND s.show_date = CURRENT_DATE AND s.remaining_seats > 0 ORDER BY s.start_time ASC"
      }
    },
    {
      "nlQuery": "Distribution (acteurs) du film Inception",
      "sql": "SELECT a.name AS actor_name, mc.role_name FROM public.movie_cast mc INNER JOIN public.actors a ON mc.actor_id = a.actor_id INNER JOIN public.movies m ON mc.movie_id = m.movie_id WHERE m.title = 'Inception'",
      "intent": "Trouver la liste des acteurs et leurs rôles pour un film spécifique comme Inception",
      "manifest": "Trouver la distribution d'un film spécifique",
      "parameterized": {
        "parameterized_intent": "Distribution (acteurs) du film $1",
        "parameterized_sql": "SELECT a.name AS actor_name, mc.role_name FROM public.movie_cast mc INNER JOIN public.actors a ON mc.actor_id = a.actor_id INNER JOIN public.movies m ON mc.movie_id = m.movie_id WHERE m.title = $1"
      }
    },
    {
      "nlQuery": "Détails des réservations sous le nom de Alice Martin",
      "sql": "SELECT t.ticket_id, m.title, s.show_date, s.start_time, th.name AS theater, t.seats_reserved FROM public.tickets t INNER JOIN public.showtimes s ON t.showtime_id = s.showtime_id INNER JOIN public.movies m ON s.movie_id = m.movie_id INNER JOIN public.theaters th ON s.theater_id = th.theater_id WHERE t.customer_name = 'Alice Martin'",
      "intent": "Récupérer l'historique et les détails des réservations de billets pour un client spécifique comme Alice Martin",
      "manifest": "Récupérer les détails des réservations d'un client",
      "parameterized": {
        "parameterized_intent": "Détails des réservations sous le nom de $1",
        "parameterized_sql": "SELECT t.ticket_id, m.title, s.show_date, s.start_time, th.name AS theater, t.seats_reserved FROM public.tickets t INNER JOIN public.showtimes s ON t.showtime_id = s.showtime_id INNER JOIN public.movies m ON s.movie_id = m.movie_id INNER JOIN public.theaters th ON s.theater_id = th.theater_id WHERE t.customer_name = $1"
      }
    },
    {
      "nlQuery": "Note moyenne des avis pour les films avec Leonardo DiCaprio",
      "sql": "SELECT m.title, AVG(r.rating) AS avg_rating, COUNT(r.review_id) AS total_reviews FROM public.movie_reviews r INNER JOIN public.movies m ON r.movie_id = m.movie_id INNER JOIN public.movie_cast mc ON m.movie_id = mc.movie_id INNER JOIN public.actors a ON mc.actor_id = a.actor_id WHERE a.name = 'Leonardo DiCaprio' GROUP BY m.title",
      "intent": "Calculer la note moyenne et le nombre total d'avis critiques pour les films dans lesquels joue un acteur particulier comme Leonardo DiCaprio",
      "manifest": "Note moyenne des avis pour un acteur spécifique",
      "parameterized": {
        "parameterized_intent": "Note moyenne des avis pour les films avec $1",
        "parameterized_sql": "SELECT m.title, AVG(r.rating) AS avg_rating, COUNT(r.review_id) AS total_reviews FROM public.movie_reviews r INNER JOIN public.movies m ON r.movie_id = m.movie_id INNER JOIN public.movie_cast mc ON m.movie_id = mc.movie_id INNER JOIN public.actors a ON mc.actor_id = a.actor_id WHERE a.name = $1 GROUP BY m.title"
      }
    },
    {
      "nlQuery": "Trouver les films incontournables du genre Science-Fiction",
      "sql": "SELECT m.title, AVG(r.rating) AS avg_rating, COUNT(r.review_id) AS total_reviews FROM public.movies m INNER JOIN public.movie_reviews r ON m.movie_id = r.movie_id INNER JOIN public.movie_genres mg ON m.movie_id = mg.movie_id INNER JOIN public.genres g ON mg.genre_id = g.genre_id WHERE g.name = 'Science-Fiction' GROUP BY m.title, m.movie_id HAVING COUNT(r.review_id) >= 2 AND AVG(r.rating) >= 4.5",
      "intent": "Identifier les films incontournables d'un genre spécifique (définis par au moins 2 avis et une note moyenne >= 4.5)",
      "manifest": "Trouver les films incontournables d'un genre spécifique",
      "parameterized": {
        "parameterized_intent": "Trouver les films incontournables du genre $1",
        "parameterized_sql": "SELECT m.title, AVG(r.rating) AS avg_rating, COUNT(r.review_id) AS total_reviews FROM public.movies m INNER JOIN public.movie_reviews r ON m.movie_id = r.movie_id INNER JOIN public.movie_genres mg ON m.movie_id = mg.movie_id INNER JOIN public.genres g ON mg.genre_id = g.genre_id WHERE g.name = $1 GROUP BY m.title, m.movie_id HAVING COUNT(r.review_id) >= 2 AND AVG(r.rating) >= 4.5"
      }
    },
    {
      "nlQuery": "Détecter les conflits de planification dans les salles de cinéma",
      "sql": "SELECT s1.showtime_id AS showtime1_id, m1.title AS movie1_title, s2.showtime_id AS showtime2_id, m2.title AS movie2_title, t.name AS theater_name, s1.show_date, s1.start_time AS start1, s2.start_time AS start2 FROM public.showtimes s1 INNER JOIN public.movies m1 ON s1.movie_id = m1.movie_id INNER JOIN public.showtimes s2 ON s1.theater_id = s2.theater_id AND s1.show_date = s2.show_date AND s1.showtime_id < s2.showtime_id INNER JOIN public.movies m2 ON s2.movie_id = m2.movie_id INNER JOIN public.theaters t ON s1.theater_id = t.theater_id WHERE (s2.start_time >= s1.start_time AND s2.start_time < s1.start_time + (m1.duration_mins || ' minutes')::interval) OR (s1.start_time >= s2.start_time AND s1.start_time < s2.start_time + (m2.duration_mins || ' minutes')::interval)",
      "intent": "Détecter les chevauchements de séances planifiées dans la même salle le même jour",
      "manifest": "Détecter les conflits de planification de séances"
    }
  ],
  "facets": [
    {
      "sql_snippet": "m.duration_mins > 120",
      "intent": "films de plus de 2 heures",
      "manifest": "Films ayant une durée supérieure à un nombre de minutes donné",
      "parameterized": {
        "parameterized_intent": "films de plus de $1 minutes",
        "parameterized_sql_snippet": "m.duration_mins > $1"
      }
    },
    {
      "sql_snippet": "r.rating >= 4",
      "intent": "bons avis",
      "manifest": "Avis critiques avec une note supérieure ou égale à un seuil",
      "parameterized": {
        "parameterized_intent": "avis avec une note supérieure ou égale à $1",
        "parameterized_sql_snippet": "r.rating >= $1"
      }
    }
  ],
  "value_searches": [
    {
      "query": "SELECT DISTINCT T.title as value, 'public.movies.title' as columns, 'Movie Title' as concept_type, 0 as distance, '{}'::text as context FROM public.movies T WHERE T.title ILIKE $value",
      "concept_type": "Movie Title",
      "description": "Recherche de titres de films"
    },
    {
      "query": "SELECT DISTINCT T.name as value, 'public.actors.name' as columns, 'Actor Name' as concept_type, 0 as distance, '{}'::text as context FROM public.actors T WHERE T.name ILIKE $value",
      "concept_type": "Actor Name",
      "description": "Recherche de noms d'acteurs"
    },
    {
      "query": "SELECT DISTINCT T.name as value, 'public.genres.name' as columns, 'Genre Name' as concept_type, 0 as distance, '{}'::text as context FROM public.genres T WHERE T.name ILIKE $value",
      "concept_type": "Genre Name",
      "description": "Recherche de genres de films"
    },
    {
      "query": "SELECT DISTINCT T.screen_type as value, 'public.theaters.screen_type' as columns, 'Screen Type' as concept_type, 0 as distance, '{}'::text as context FROM public.theaters T WHERE T.screen_type ILIKE $value",
      "concept_type": "Screen Type",
      "description": "Recherche de types de salles de projection"
    }
  ]
}
```

### 3.2 Charger le Context Set dans AlloyDB
1. Dans la console GCP d'AlloyDB, ouvrez **AlloyDB Studio** (assurez-vous d'être connecté à la base de données `cinema_db` en utilisant l'**authentification IAM**).
2. Dans la barre latérale de gauche, faites défiler vers le bas jusqu'à **Context sets** (Ensembles de contexte).
3. Cliquez sur le bouton d'action (les trois petits points) et sélectionnez **Create context set** (Créer un ensemble de contexte).
4. Remplissez les champs :
   *   **Name** : `cinema_context`
   *   **Description** : `Contexte QueryData avancé pour la gestion du cinéma`
   *   **Upload context file** : Importez le fichier `querydata_cinema_contextset.json` créé localement.
5. Cliquez sur **Save**.

### 3.3 Valider le comportement de QueryData
Une fois enregistré :
1. Cliquez sur les trois points à côté de votre nouveau contexte `cinema_context` et sélectionnez **Test context set**.
2. Essayez de saisir différentes questions complexes en variant les paramètres :
   *   *« Trouver les films incontournables du genre Action »* (Déclenche la règle métier personnalisée avec agrégation et filtrage HAVING : note moyenne >= 4.5 et au moins 2 avis).
   *   *« Détecter les conflits de planification dans les salles de cinéma »* (Déclenche une auto-jointure complexe sur les séances avec arithmétique d'intervalles de temps que le LLM seul ne saurait générer).
   *   *« Quels films de plus de 150 minutes ont de bons avis ? »* (Combine les facettes de durée et de qualité).
3. Constatez la capacité de QueryData à générer instantanément la bonne requête SQL paramétrée et à récupérer fidèlement les données correspondantes !

---

## 🛠️ Étape 4 : Sécurisation des accès avec MCP et l'Agent Sémantique (15 min)

Pour s'assurer que les agents IA accèdent de manière sécurisée et contrôlée aux bases de données, nous utilisons le **Model Context Protocol (MCP)** et le serveur **MCP Toolbox for databases**.

[MCP Toolbox for databases](https://github.com/googleapis/mcp-toolbox) est un serveur MCP open source conçu pour simplifier l'accès des agents IA à vos bases de données. Il fait office de passerelle d'API sécurisée et standardisée :
* **Développement simplifié** : permet de déclarer des requêtes SQL ou APIs sous forme d'outils ("tools") documentés en YAML, prêts à être consommés par le LLM.
* **Performances & Sécurité** : gère le pooling de connexions, l'authentification, et utilise des requêtes préparées/paramétrées sous le capot pour empêcher toute injection SQL.
* **Observabilité** : propose des métriques de suivi intégrées et un traçage prêt à l'emploi.

### 4.1 Télécharger et Installer MCP Toolbox
Dans votre terminal Cloud Shell, récupérez le binaire de **MCP Toolbox for databases** :
```bash
export VERSION=1.1.0
curl -L -o toolbox https://storage.googleapis.com/mcp-toolbox-for-databases/v$VERSION/linux/amd64/toolbox
chmod +x toolbox
```

### 4.2 Configurer le fichier d'outils MCP
Créez le fichier `tools.yaml` pour le serveur MCP local. Remplissez-le avec les identifiants de votre base de données en remplaçant les variables de projet et région :

```bash
export PROJECT_ID=$(gcloud config get-value project)

cat << EOF > tools.yaml
# 1. Source SQL classique pour exécuter des requêtes prédéfinies
kind: source
name: cinema-sql-source
type: alloydb-postgres
project: "$PROJECT_ID"
region: "us-central1"
cluster: "alloydb-cinema-cluster"
instance: "alloydb-cinema-cluster-pr"
ipType: "public"
database: "cinema_db"
user: "postgres"
password: "BuildWithGemini2026"
---
# 2. Source GDA pour la traduction sémantique NL2SQL
kind: source
name: gda-api-source
type: cloud-gemini-data-analytics
projectId: "$PROJECT_ID"
---
# 3. Outil NL2SQL s'appuyant sur QueryData
kind: tool
name: querydata_cinema_tool
type: cloud-gemini-data-analytics-query
source: gda-api-source
location: "us-central1"
description: Outil sémantique. Permet d'interroger la base de données cinéma en langage naturel (films, genres, séances, avis clients, etc.).
context:
  datasourceReferences:
    alloydb:
      databaseReference:
        projectId: "$PROJECT_ID"
        region: "us-central1"
        clusterId: "alloydb-cinema-cluster"
        instanceId: "alloydb-cinema-cluster-pr"
        databaseId: "cinema_db"
      agentContextReference:
        contextSetId: "projects/$PROJECT_ID/locations/us-central1/contextSets/cinema_context"
generationOptions:
  generateQueryResult: true
  generateNaturalLanguageAnswer: true
---
# 4. Outil transactionnel fixe (sans NL2SQL) pour réserver des billets de cinéma
kind: tool
name: book_movie_ticket
type: postgres-sql
source: cinema-sql-source
description: Enregistre une réservation de places de cinéma sous un nom de client donné pour une séance spécifique.
parameters:
  - name: showtime_id
    type: integer
    description: L'ID de la séance de cinéma.
  - name: customer_name
    type: string
    description: Le nom du client réalisant la réservation.
  - name: seats_reserved
    type: integer
    description: Le nombre de places à réserver.
statement: |
  INSERT INTO public.tickets (ticket_id, showtime_id, customer_name, seats_reserved)
  VALUES ((SELECT COALESCE(MAX(ticket_id), 0) + 1 FROM public.tickets), $1, $2, $3)
  RETURNING ticket_id, customer_name, seats_reserved;
---
# 5. Ensemble d'outils (Toolset) pour charger tous les outils cinéma d'un coup
kind: toolset
name: cinema_toolset
tools:
  - querydata_cinema_tool
  - book_movie_ticket
EOF
```

Lancez le serveur local sur le port par défaut :
```bash
./toolbox --config tools.yaml
```
> [!NOTE]
> Le serveur doit indiquer `Server ready to serve!` dans les journaux, confirmant l'exposition des outils dynamiques et statiques.

### 4.3 Créer l'Agent IA Python (ADK Framework)
Nous allons configurer un agent conversationnel sémantique en Python avec le framework **Google ADK**.

1. Ouvrez un nouvel onglet de terminal Cloud Shell.
2. Créez le script de l'agent `agent.py` :
```bash
cat << 'EOF' > agent.py
from google.adk.agents import Agent
from toolbox_core import ToolboxSyncClient

# 1. Connexion au serveur MCP local
MCP_SERVER_URL = "http://127.0.0.1:5000"
toolbox = ToolboxSyncClient(MCP_SERVER_URL)

# 2. Chargement dynamique du toolset configuré pour le cinéma
cinema_tools = toolbox.load_toolset('cinema_toolset')

# 3. Configuration de l'Agent
root_agent = Agent(
    name='cinema_agent',
    model="gemini-2.5-flash",
    description='Assistant cinéma intelligent pour la gestion des séances et des avis.',
    instruction="""
    Vous êtes l'assistant virtuel du cinéma. Aidez les clients à trouver des séances et à réserver des places.
    Utilisez uniquement les outils MCP à votre disposition pour récupérer et enregistrer des informations.
    Ne donnez que des informations basées sur les données de la base 'cinema_db' d'AlloyDB.
    """,
    tools=cinema_tools,
)
EOF
```

3. Créez un environnement virtuel Python, activez-le, installez les dépendances, configurez l'accès à Vertex AI et lancez le serveur web ADK :
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install google-adk toolbox-core

# Configurer les variables d'environnement pour cibler Vertex AI (Google Cloud)
export GOOGLE_CLOUD_PROJECT=$(gcloud config get-value project)
export GOOGLE_CLOUD_LOCATION=us-central1
export GOOGLE_GENAI_USE_VERTEXAI=True

adk web --allow_origins "*"
```

> [!IMPORTANT]
> **Gestion des ports et proxy Cloud Shell :**
> * L'option `--allow_origins "*"` est indispensable dans Cloud Shell pour contourner les restrictions d'origines (CORS) du proxy sécurisé de Cloud Shell.
> * Vous avez maintenant deux serveurs en parallèle : **MCP Toolbox (port 5000)** et **ADK Web (port 8000)**.
> * Pour utiliser l'agent conversationnel, cliquez sur **Aperçu sur le Web** (en haut à droite de Cloud Shell), sélectionnez **Changer de port** et entrez le port **`8000`**.

### 4.4 Démonstration Pratique
Posez des questions sémantiques ou donnez des ordres d'action à votre agent dans l'interface ADK et observez quel type d'outil est sélectionné par Gemini en arrière-plan :

*   *« Trouve s'il y a des séances qui entrent en conflit dans nos salles de cinéma. »*
    *   **Outil déclenché :** `querydata_cinema_tool`. Gemini traduit la requête complexe en s'appuyant sur le template de conflit de planification QueryData (ce qui effectue une auto-jointure complexe et de l'arithmétique temporelle qui mettrait en échec un LLM standard).
*   *« Réserve-moi 3 places pour la séance 2 sous le nom de Bob Durand. »*
    *   **Outil déclenché :** `book_movie_ticket`. Gemini comprend le besoin d'action et appelle l'outil statique/transactionnel direct en extrayant les paramètres (`showtime_id: 2`, `customer_name: "Bob Durand"`, `seats_reserved: 3`) sans passer par de la traduction NL2SQL dynamique.

---

## 🛠️ Étape 5 : Création d'un Agent Conversational Analytics depuis la Console AlloyDB (15 min)

En plus de l'agent ADK en Python, la console Google Cloud intègre un outil d'analyse puissant appelé **Conversational Analytics** permettant de générer des agents conversationnels intelligents directement depuis l'interface d'AlloyDB. Nous allons créer un agent graphique pour notre base de données `cinema_db` en y réutilisant le contexte QueryData créé à l'étape 3.

### 5.1 Accéder à l'onglet Agents d'AlloyDB
1. Dans la console Google Cloud, retournez sur la page de votre cluster **`alloydb-cinema-cluster`**.
2. Dans la barre de navigation latérale de gauche, sous le menu AlloyDB, cliquez sur l'onglet **« Agents »**.
3. Vous êtes maintenant dans l'interface de gestion des agents sémantiques intégrée à AlloyDB.

### 5.2 Créer l'Agent d'Analyse Cinéma
1. Cliquez sur le bouton **Nouvel agent** (ou **New Agent**).
2. Remplissez les informations de configuration :
   *   **Nom de l'agent** : `cinema-analytics-agent`
   *   **Description** : `Agent conversationnel pour l'analyse des films et séances`
3. L'agent étant lancé directement au sein de votre espace de travail AlloyDB, la source de données est pré-configurée. Sélectionnez simplement la base de données **`cinema_db`** dans la liste déroulante.

### 5.3 Lier votre Context Set QueryData
Associons notre contexte pour guider l'intelligence de l'agent :
1. Dans la section de configuration du contexte de l'agent, cochez l'option pour **Utiliser un ensemble de contexte existant** (Use Context Set).
2. Choisissez le contexte **`cinema_context`** que vous avez créé et téléversé à l'étape 3.
3. Grâce à cette association, l'agent d'analyse d'AlloyDB va directement hériter de vos filtres sémantiques complexes et de vos requêtes SQL paramétrées pour répondre de manière fiable et prévisible.
4. Cliquez sur **Créer** (Create) ou **Sauvegarder**.

### 5.4 Tester l'Agent
1. Sur le panneau de droite dans la console d'administration de l'agent, utilisez le panneau de chat interactif de test.
2. Saisissez en langage naturel :
   *   *« Trouve s'il y a des séances de cinéma qui se chevauchent ou entrent en conflit dans les salles. »*
   *   *« Quels sont les films incontournables du genre Drame ? »*
   *   *« Est-ce qu'il y a des films de plus de 150 minutes avec de bons avis ? »*
3. L'agent d'AlloyDB applique instantanément vos templates QueryData, génère le SQL exact de manière déterministe en effectuant les jointures nécessaires, et affiche une réponse claire et formatée !

---

## 🏆 Conclusion et Résumé des Acquis

Félicitations pour avoir complété cet atelier pratique de **1h15** ! Vous n'avez pas besoin de nettoyer votre projet ni de supprimer le cluster AlloyDB ; les organisateurs se chargeront du nettoyage global de l'infrastructure à la fin de l'événement.

Au cours de ce lab, vous avez appris à :
1.  **IA à la source** : Utiliser les filtres sémantiques (`google_ml.if`) et de score (`google_ml.rank`) directement dans le moteur de requête AlloyDB.
2.  **Recherche Avancée** : Implémenter un index de recherche hybride combinant FTS textuel (RUM) et vectoriel (ScaNN) via Vertex AI.
3.  **Predictive SQL** : Mettre en œuvre **QueryData** pour cadrer de manière stricte les interprétations NL2SQL et éviter les hallucinations.
4.  **Sécurité via MCP** : Déployer une passerelle sécurisée basée sur le **Model Context Protocol (MCP)** connectée à un agent intelligent en Python (framework Google ADK).
5.  **Conversational Analytics UI** : Créer un agent analytique conversationnel graphique directement dans la console Google Cloud, réutilisant vos règles QueryData pour interroger visuellement vos bases de données.
