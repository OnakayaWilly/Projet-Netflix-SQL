# Netlifx Movies and TV shows (SQL)


## Objectifs 

- Analyser le catalogue Netflix via des requêtes SQL pour en extraire des insights pertinents.
- Explorer les tendances de contenu (films vs séries, genres, pays, années, réalisateurs, acteurs...).
- Répondre à des questions métiers concrètes en utilisant des jointures, agrégations et filtres SQL.

# Dataset 
 - **Dataset Link:** [Netflix Dataset](https://www.kaggle.com/datasets/shivamb/netflix-shows?resource=download)
## Schéma

'''sql
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
'''
