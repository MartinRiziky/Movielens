# Movielens
School project


Krátke vysvetlenie témy projektu MovieLens, typ dát a účel analýzy:
Projekt MovieLens predstavuje rozsiahlu databázu filmových hodnotení vytvorených používateľmi. Obsahuje dáta o filmoch, používateľoch a ich hodnoteniach (ratings), prípadne aj tagy, či časové údaje o hodnoteniach. Tieto údaje sú typicky využívané na tvorbu odporúčacích systémov (recommendation systems), analýzy preferencií používateľov, skúmanie popularít žánrov, filmov a pod.

Typ dát:
Hlavne textové atribúty (názvy filmov, žánre, mená používateľov a ich charakteristiky), číselné (hodnotenia, ID, roky, dátumy), dátovo-časové (čas/ dátum hodnotenia).
Účel analýzy: cieľom je získať prehľad o tom, aké filmy sa najviac hodnotia, aké žánre sú obľúbené, akí používatelia ich hodnotia, v akom čase prebieha najviac hodnotení či ako sa hodnotenia menia naprieč demografickými ukazovateľmi.



Základný popis každej tabuľky zo zdrojových dát a ich význam:

Movies 
Hlavné stĺpce: movieId, title, genres
Význam: Zahŕňa informácie o filmoch, ich názvy a žánre. Pre analýzy môžeme filmovú entitu spájať s konkrétnymi hodnoteniami.

Ratings
Hlavné stĺpce: userId, movieId, rating, timestamp
Význam: Zaznamenávajú hodnotenia filmov jednotlivými používateľmi na škále od 0,5 po 5 (podľa verzie datasetu). timestamp predstavuje čas vytvorenia hodnotenia.

Tags
Hlavné stĺpce: userId, movieId, tag, timestamp
Význam: Umožňujú používateľom priraďovať k filmom tagy (krátke textové popisy, napr. “thriller”, “based on a true story”), ktoré vystihujú film z pohľadu používateľa.

Users
Hlavné stĺpce: userId, prípadne demografické informácie (pohlavie, veková skupina, povolanie, PSČ a pod.) – to závisí od verzie datasetu.
Význam: Obsahuje informácie o demografii a iných atribútoch používateľov, na základe ktorých môžeme analyzovať rozdiely v preferenciách.

Time/Date
Vo väčšine datasetov MovieLens čas nie je v samostatnej tabuľke, ale je uložený v stĺpci timestamp. Pri potrebe detailných analýz (napr. dňa v týždni, času počas dňa) sa zvyčajne extrahuje do samostatnej tabuľky.
(Názvy tabuliek sa môžu mierne líšiť podľa konkrétnej verzie datasetu, ale princíp ostáva rovnaký.)

ERD diagram pôvodnej štruktúry zdrojových dát

Nižšie je príklad jednoduchej schémy vzťahov (pôvodná štruktúra MovieLens datasetu). Vzťahy sú:

Ratings je prepojovacia tabuľka medzi Users a Movies (1 používateľ môže ohodnotiť viac filmov, 1 film môže byť ohodnotený viacerými používateľmi).
Tags je obdobne prepojovacia tabuľka medzi Users a Movies (1 používateľ môže pridávať tagy k viacerým filmom a 1 film môže mať viacero tagov).

![alt text](image.png)

Prípadne, ak existuje samostatná tabuľka Time/Date, tak Ratings a Tags budú mať cudzie kľúče na túto tabuľku.



Návrh dimenzionálneho modelu typu hviezda (Hviezdička)

V dimenzionálnom modeli zvyčajne zoskupíme číselné a merateľné údaje do jednej faktovej tabuľky a opisné údaje do dimenzných tabuliek.

Faktová tabuľka:
fact_ratings (obsahuje všetky merateľné údaje: rating, počet hodnotení, dátum hodnotenia, a kľúče na dimenzie)

Dimenzie:
dim_users 
dim_movies 
dim_date 
dim_tags 

ERD dimenzionálneho modelu (Hviezdička):
![alt text](image-1.png)


Popis tabuliek v dimenzionálnom modeli

Faktová tabuľka: fact_ratings

Kľúče:
id (primárny kľúč faktu)
user_id (cudzie kľúče do dim_users)
movie_id (cudzie kľúče do dim_movies)
date_id (cudzie kľúče do dim_date)
time_id (cudzie kľúče do dim_time)

Hlavné metriky:
rating – hodnotenie od 0,5 do 5 (napr. “hviezdičky”).

Účel:
Umožňuje analyzovať, ako sa ratingy menia podľa používateľa, filmu, času, dňa v týždni a ďalších atribútov z dimenzií.

Dimenzná tabuľka: dim_users
Kľúč:
id 

Atribúty:
age_group
gender
occupation
zip_code
usercol


Dimenzná tabuľka: dim_movies
Kľúč:
id

Atribúty:
title
release_year
genres


Dimenzná tabuľka: dim_date
Kľúč:
id

Atribúty:
full_date
day, month, year, quarter, week, timestamp


Dimenzná tabuľka: dim_time
Kľúč:
id

Atribúty:
hour, timestamp, ampm

ETL proces v nástroji Snowflake

Extrakcia:

Načítanie zdrojových súborov (CSV predtým importovane do myphpadmin a nasledne exportovane) z MovieLens do staging zóny v Snowflake.
Hlavný SQL príkaz(príklad): COPY INTO movies (id, title, release_year)
FROM @MOVIELENS_MOVIELENS/movies.csv
FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);
SELECT * FROM movies LIMIT 10; -- na testovanie či príkaz prebehol správne a načítalo údaje
Účel: zabezpečiť, aby dáta boli fyzicky “nahrané” do Snowflake.



Transfor (Transformácia dát)
Vytvorenie dimenzných tabuliek
Dimenzia dim_users
Transformácia používateľov, rozdelenie veku do kategórií a obohatenie údajov:

CREATE TABLE dim_users AS
SELECT DISTINCT
    u.id AS dim_user_id,
    CASE 
        WHEN u.age < 18 THEN 'Under 18'
        WHEN u.age BETWEEN 18 AND 24 THEN '18-24'
        WHEN u.age BETWEEN 25 AND 34 THEN '25-34'
        WHEN u.age BETWEEN 35 AND 44 THEN '35-44'
        WHEN u.age BETWEEN 45 AND 54 THEN '45-54'
        WHEN u.age >= 55 THEN '55+'
        ELSE 'Unknown'
    END AS age_group,
    u.gender,
    o.name AS occupation
FROM users u
LEFT JOIN occupations o ON u.occupation_id = o.id;


Na základe poskytnutého príkladu a tvojich požiadaviek som pripravil SQL príkazy na transformáciu dát pre tvoju tému filmov (MovieLens):

Transformácia dát
Vytvorenie dimenzných tabuliek
Dimenzia dim_users
Transformácia používateľov, rozdelenie veku do kategórií a obohatenie údajov:


CREATE TABLE dim_users AS
SELECT DISTINCT
    u.id AS dim_user_id,
    CASE 
        WHEN u.age < 18 THEN 'Under 18'
        WHEN u.age BETWEEN 18 AND 24 THEN '18-24'
        WHEN u.age BETWEEN 25 AND 34 THEN '25-34'
        WHEN u.age BETWEEN 35 AND 44 THEN '35-44'
        WHEN u.age BETWEEN 45 AND 54 THEN '45-54'
        WHEN u.age >= 55 THEN '55+'
        ELSE 'Unknown'
    END AS age_group,
    u.gender,
    o.name AS occupation
FROM users u
LEFT JOIN occupations o ON u.occupation_id = o.id;

Dimenzia dim_movies
Transformácia údajov o filmoch:

CREATE TABLE dim_movies AS
SELECT DISTINCT
    m.id AS dim_movie_id,
    m.title,
    m.release_year
FROM movies m;
Dimenzia dim_genres
Žánre sú už pripravené. Môžeš ich skontrolovať:


SELECT * FROM genres;
Dimenzia dim_age_group
Transformácia kategórií veku (ak je potrebné):


CREATE TABLE dim_age_group AS
SELECT 
    id AS dim_age_group_id,
    name AS age_group
FROM age_group;
Dimenzia dim_occupations
Transformácia zamestnaní:


CREATE TABLE dim_occupations AS
SELECT 
    id AS dim_occupation_id,
    name AS occupation
FROM occupations;

Vytvorenie faktovej tabuľky fact_ratings
Transformácia faktov o hodnoteniach, ktoré obsahujú odkazy na dimenzné tabuľky:

sql
Kopírovať kód
CREATE TABLE fact_ratings AS
SELECT 
    r.id AS fact_rating_id,
    r.rating,
    r.rated_at AS timestamp,
    u.dim_user_id AS user_id,
    m.dim_movie_id AS movie_id
FROM ratings r
JOIN dim_users u ON r.user_id = u.dim_user_id
JOIN dim_movies m ON r.movie_id = m.dim_movie_id;

Načítanie a čistenie staging tabuliek
Odstránenie staging tabuliek po načítaní dát:

DROP TABLE IF EXISTS age_group;
DROP TABLE IF EXISTS genres;
DROP TABLE IF EXISTS genres_movies;
DROP TABLE IF EXISTS movies;
DROP TABLE IF EXISTS occupations;
DROP TABLE IF EXISTS ratings;
DROP TABLE IF EXISTS tags;
DROP TABLE IF EXISTS users;

Popis transformácie
Dimenzia dim_users:
Zahŕňa používateľov s kategorizovanými vekovými skupinami a zamestnaniami.
Dimenzia dim_movies:
Obsahuje údaje o filmoch, ako sú názov a rok vydania.
Faktová tabuľka fact_ratings:
Obsahuje všetky hodnotenia s odkazmi na dimenzie.

