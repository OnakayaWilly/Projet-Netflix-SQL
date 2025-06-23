# Netlifx Movies and TV shows (SQL)


## Objectifs 

- Analyser le catalogue Netflix via des requêtes SQL pour en extraire des insights pertinents.
- Explorer les tendances de contenu (films vs séries, genres, pays, années, réalisateurs, acteurs...).
- Répondre à des questions métiers concrètes en utilisant des jointures, agrégations et filtres SQL.

# Dataset 
 - **Dataset Link:** [Netflix Dataset](https://www.kaggle.com/datasets/shivamb/netflix-shows?resource=download)
## Schéma

```sql
DROP TABLE IF EXISTS netflix;
CREATE TABLE netflix 
(
	show_id VARCHAR(6),
	type VARCHAR(10),
	title VARCHAR(150),
	director VARCHAR(208),
	casts VARCHAR(1000),	
	country VARCHAR(150),
	date_added	VARCHAR(50),
	release_year INT,
	rating	VARCHAR(10),
	duration VARCHAR(15),
	listed_in VARCHAR(100),
	description VARCHAR(250)
);
```


### 1. Count the Number of Movies vs TV Shows

```sql
SELECT 
	type,  -- 'Movie ou 'TV show'
	COUNT(*) as total_count -- Nombre total de contenu de type
FROM netflix 
GROUP BY type -- Regroupement par type
```

### 2. Find the Most Common Rating for Movies and TV Shows
```sql
		    -- Comptage du nombre d'occurrences pour chaque combinaison de type et d'évaluation
with RatingCount as (
	SELECT
		type, -- 'Movie' ou 'TV show'
		rating,  -- Classification 
		COUNT(*) as rating_count --  nombre d'occurrences pour de cette classification
	FROM Netflix
	WHERE rating IS NOT NULL -- On vérifie bien que la colonne ne contient pas de valeurs nulles
	GROUP BY type, rating -- 
RankedRatings as (
	SELECT 
		rating, -- Classification 
		type, -- 'Movie' ou 'TV show'
		RANK() OVER(PARTITION BY type ORDER BY rating_count DESC) as rank_rating -- Windows function pour faire un classement regroupé par type et trié par le nb d'évalution  du plus petit au plus grand
	FROM RatingCount -- Table temporaire contenant le regroupement par type puis rating
)
SELECT
	type,
	rating
FROM RankedRatings -- table temporaire contenant le classement selon le nb d'évaluations
WHERE rank_rating = 1 -- On prend uniquement l'évaluation la plus comptée pour chaque type
```

### 3. List All Movies Released in a Specific Year (e.g., 2020)
```sql
SELECT * FROM netflix
WHERE type = 'Movie' AND release_year = 2020
```

### 4. Find the Top 5 Countries with the Most Content on Netflix
```sql
SELECT
	TRIM(unnest(string_to_array(country, ','))) as country, -- On découpe la colonne country (chaque pays séparé par une virgule) puis on déplie le tableau sur plusieurs lignes / TRIM() pour supprimer les espaces
	COUNT(*) as content_displayed 					-- nombre d'occurences de chaque pays
FROM netflix
WHERE country IS NOT NULL							 -- On exclut les lignes de la colonnes contenant la valeur nulle
GROUP BY 1 											-- Regroupement par pays
ORDER BY content_displayed DESC 					-- Tri par le nb d'occurences du plus grand au plus petit
LIMIT 5 						-- On affiche les 5 premiers ( = 5 pays avec le plus de contenus sur netflix)
```

### 5. Identify the Longest Movie
```sql
SELECT 
    *
FROM netflix
WHERE type = 'Movie' AND duration IS NOT NULL 				-- Type 'Movie' et on exclut les lignes contenant une valeur nulle pour la colonne 'duration'
ORDER BY SPLIT_PART(duration, ' ', 1)::INT DESC 			--  On garde seulement la valeur numérique et on la convertit en INT (ex : '90 min' -> 90)
LIMIT 1; 			-- On garde le plus long
```

### 6. Find Content Added in the Last 5 Years
```sql
SELECT * FROM netflix
WHERE date_added IS NOT NULL -- On exclut les lignes contenant la valeur null pour la colonne date_added
	AND
	  TO_DATE(date_added, 'Month DD, YYYY')  >= CURRENT_DATE - INTERVAL '5 Years' 			-- Conversion de la date en format adéquat pour faire une comparaison
```

### 7. Find All Movies/TV Shows by Director 'Rajiv Chilaka'

```sql
with director_q as (
	SELECT 
		*,
		TRIM(UNNEST(string_to_array(director, ','))) as director_name  -- Découpage de la colonne director (string_to_array()) séparé par une ',' puis dépliage sur plusieurs lignes (UNNEST())
	FROM netflix
	WHERE director IS NOT NULL
)
SELECT
	*
FROM director_q 			
WHERE director_name = 'Rajiv Chilaka'
```

### 8. List All TV Shows with More Than 5 Seasons
```sql
SELECT 
	*,
	SPLIT_PART(duration, ' ', 1)::int as nb_seasons
FROM Netflix
WHERE type = 'TV Show' AND duration IS NOT NULL AND SPLIT_PART(duration, ' ', 1)::int >= 5 			-- Selection du type 'TV show' et filtrage de la durée (NON NULL)
```

### 9. Count the Number of Content Items in Each Genre

```sql
SELECT 
	TRIM(UNNEST(string_to_array(listed_in, ','))) as unique_gender,  		-- On découpe à nouveau via le séparateur ',', genre identifié de manière séparé
	COUNT(*) as nb_content
FROM Netflix
GROUP BY UNNEST(string_to_array(listed_in, ','))
ORDER by nb_content DESC
```

### 10.Find each year and the average numbers of content release in India on netflix.

```sql
SELECT 
    EXTRACT(YEAR FROM TO_DATE(date_added, 'Month DD, YYYY')) AS year, -- Extraction de l'année à partir de la date d'ajout
    COUNT(*) AS nb_content,                                           -- Nombre de contenus ajoutés cette année-là
    ROUND(
        COUNT(*)::numeric / (
            SELECT COUNT(*) 
            FROM netflix 
            WHERE country = 'India'
        ) * 100, 2
    ) AS pct_total_content_india                                     -- Pourcentage par rapport au total des contenus indiens
FROM netflix
WHERE country = 'India' AND date_added IS NOT NULL                                       -- Exclusion des lignes avec date manquante
GROUP BY year
ORDER BY year
```

### 11. List All Movies that are Documentaries
```sql
SELECT
	show_id,
	type,
	title,
	listed_in
FROM Netflix
WHERE type = 'Movie' AND 'Documentaries' = ANY(string_to_array(listed_in, ',')) 		-- Filtrage pour garder que les 'Movie' /  On recherche 'Documentaries' dans le array ou le séparateur est ',' 
																					-- ex :  Documentaries, Horror ->{Documentaries, Horror}  (ligne conservé)
```

 
### 12 Find All Content Without a Director
```sql
SELECT
	*
FROM netflix
WHERE director IS NULL
```
### 13 Find How Many Movies Actor 'Salman Khan' Appeared in the Last 10 Years
```sql
with film as (
	SELECT
		*
	FROM netflix
	WHERE 'Salman Khan' = ANY(string_to_array(casts, ',')) AND type = 'Movie' 					-- Filtrage dans une CTE pour selectionner l'acteur parti la liste d'acteur donnée ainsi que le type 
)
SELECT
	COUNT(*) as appeared_movies
FROM film
WHERE release_year >= EXTRACT(YEAR FROM CURRENT_DATE) - 10 			-- On prend seulement les apparitions lors des 10 dernières années
```

### 14. Find the Top 10 Actors Who Have Appeared in the Highest Number of Movies Produced in India

```sql
SELECT  
	UNNEST(string_to_array(casts, ',')) as actor,
	COUNT(*) as appearance
FROM Netflix
WHERE 'India' = ANY(string_to_array(country, ',')) AND type = 'Movie' 			-- On cherche le pays 'India' parmi le array crée ansi que le type 'movie'
GROUP BY actor							-- Regroupement par acteur
ORDER BY appearance DESC
LIMIT 10   				-- Les 10 acteurs qui ont le plus d'apparences
```

### 15 Categorize Content Based on the Presence of 'Kill' and 'Violence' Keywords

```sql
SELECT category,
		COUNT(*)
	-- Sous requête où s'il y a le mot kill ou violence dans la description on renvoie 'Bad' sinon 'good'
FROM (																													
	SELECT 	CASE 
		WHEN description ILIKE '%kill%' OR description ILIKE '%violence%' THEN 'Bad'
			ELSE 'Good'
			END AS category
	FROM netflix
)
GROUP BY category					-- Regroupement selon la category ('Bad' ou 'Good')
```

### 16. Identifier les réalisateurs les plus prolifiques (Top 5)
```sql
WITH sep_directors as (
	SELECT
		UNNEST(string_to_array(director, ',')) as director_name   			-- Séparation de chaque directeur séparé par une ','
	FROM netflix
	WHERE director IS NOT NULL
)
SELECT 
	TRIM(director_name) as director,  			-- On enlève les espaces sur les côtés
	COUNT(*) as film_count
FROM sep_directors
GROUP  BY director
ORDER BY film_count DESC
LIMIT 5							-- On identifie les 5 réalisateurs les plus prolifiques via le ORDER BY DESC / LIMIT 
```


### 17. Répartition du contenu par année et par type (Movies/TV Shows)

```sql
SELECT
	release_year,
	type,
	COUNT(*)
FROM Netflix
GROUP BY release_year, type
ORDER BY release_year DESC, type
```

### 18. Liste des paires acteur-actrice les plus fréquentes

```sql
with actors_list as (
	SELECT
		show_id,
		TRIM(UNNEST(string_to_array(casts, ','))) as actor 		-- Nettoyage des espaces + découpage des acteurs par virgule
	FROM Netflix
	WHERE casts IS NOT NULL
),
pair_actors as (
SELECT
	a1.show_id,
	LEAST(a1.actor, a2.actor) as actor_1,  			-- L'acteur dont le nom est alphabétiquement le plus petit
	GREATEST(a1.actor, a2.actor) as actor_2			-- L'autre acteur, pour éviter les doublons inversés
FROM actors_list a1
JOIN actors_list a2 ON a1.show_id = a2.show_id AND a1.actor < a2.actor 			-- Jointure avec la table elle-même and condition pour éviter les doublons et paires (X,X)
)
SELECT
	actor_1,
	actor_2,
	COUNT(*) as appeareances
FROM pair_actors
GROUP BY actor_1, actor_2
ORDER BY appeareances DESC
LIMIT 10
```


	
### 19. Durée moyenne des films par pays

```sql
with movies as (
	SELECT
		SPLIT_PART(duration, ' ', 1)::int AS duree,
		TRIM(country_name) AS sep_country
	FROM netflix,
	LATERAL UNNEST(string_to_array(country, ',')) AS country_name  		-- LATERAL UNNEST : Permet à cette opération d’accéder aux colonnes de la ligne principale (netflix) ligne par ligne
	WHERE type = 'Movie' AND duration IS NOT NULL AND country IS NOT NULL
)
SELECT 
	sep_country,
	ROUND(AVG(duree), 2) as mean_duration  -- Moyenne de la durée, arrondi avec 2 chiffres après la virgule
FROM movies
GROUP BY sep_country
ORDER BY mean_duration DESC
```

### 20. Top 5 catégories les plus courantes dans les documentaires

```sql
SELECT
	TRIM(UNNEST(string_to_array(listed_in, ','))) as genre, 			-- Découpage des catégories par une ','
	COUNT(*) as common_cat
FROM netflix
WHERE listed_in ILIKE '%Documentaries%' AND listed_in IS NOT NULL   -- Filtrage des contenues étant des 'Documentaries' et non nul
GROUP BY genre
ORDER by common_cat DESC
LIMIT 5
```

### 21. Évolution du nombre de sorties par genre au fil du temps 

```sql
with extract_movie as (
SELECT
	show_id,
	EXTRACT(YEAR FROM TO_DATE(date_added, 'Month DD, YYYY')) as years, -- Extraction de l'année
	TRIM(UNNEST(string_to_array(listed_in, ','))) as genre 	
FROM Netflix
WHERE date_added IS NOT NULL
)
SELECT
	years,
	genre,
	COUNT(show_id) as nb_releases	-- Comptage du nombre de sorties
FROM extract_movie
WHERE years = 2018
GROUP BY years, genre				-- Regroupement par année et genre
ORDER BY years DESC, nb_releases DESC
```


### 22 Durée moyenne des films par pays (avec > 50 films)

```sql
with Countries as (
SELECT
	show_id,
	TRIM(UNNEST(string_to_array(country, ','))) as country,			-- Nettoyage et Séparation des pays
	SPLIT_PART(duration, ' ', 1)::int as duree 				-- -- On extrait la durée numérique des films (ex : '90 min' → 90)
FROM netflix
WHERE country IS NOT NULL 
	AND type = 'Movie'
)
SELECT 
	country,
	ROUND(AVG(duree), 2) AVG_duration,
	COUNT(*) as total_movies
FROM Countries
GROUP BY country
HAVING COUNT(*) > 50
ORDER BY AVG_duration DESC							-- Regroupement par pays / Filtrage après group by en gardant les pays qui ont plus de 50 films / Tri DESC par durée moyenne
```


### 23 Identifier les contenus co-réalisés (plus d’un réalisateur)

```sql
with directors as (
SELECT 
	show_id,
	TRIM(UNNEST(string_to_array(director,','))) as director  			-- Nettoyage et séparation des directeurs, séparé par une ','
FROM Netflix
WHERE director IS NOT NULL
)
SELECT
	show_id,
	COUNT(DISTINCT director) as num_director
FROM directors
GROUP BY show_id
HAVING COUNT(DISTINCT director) >= 2
ORDER BY num_director DESC							-- Regroupement par show_id , aggrégation via HAVING pour garder ceux ayant 2 directeurs ou plus.
```

### 24. Détecter les contenus sans pays renseigné

```sql
SELECT
	*
FROM netflix
WHERE country IS NULL
```


### 25. Trouver les 10 meilleurs mois en nombre d’ajouts sur Netflix

```sql
with months as (
	SELECT
		show_id,
		title,
		TO_CHAR(TO_DATE(date_added, 'Month DD, YYYY'), 'Month') as date		-- Conversio de la date au format 'Month DD, YYYY', puis on conserve le mois
FROM 
netflix
WHERE date_added IS NOT NULL
)
SELECT
	date,
	COUNT(show_id) as content_added
FROM months
GROUP BY date						
ORDER BY content_added DESC
LIMIT 10								-- Regroupement par mois , Tri par contenu ajouté du plus grand nombre au plus petit / On garde les 10 plus grands
```
