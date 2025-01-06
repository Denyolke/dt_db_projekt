# **ETL Proces pre dataset IMDB**

Tento repozitár obsahuje implementáciu ETL procesu v Snowflake pre analýzu dát z **IMDB** datasetu. Projekt umožňuje hĺbkovú analýzu hodnotení IMDB v rámci série. 
Výsledný dátový model umožňuje multidimenzionálnu analýzu a vizualizáciu kľúčových metrik.

---

## **1. Úvod a popis zdrojových dát**

Cieľom projektu bolo analyzovať hodnotenia seriálov na IMDB v priebehu rokov

Hlavná tabuľka **(imdb_top_100)** súboru údajov pochádza z kaggle [tu](https://www.kaggle.com/datasets/harshitshankhdhar/imdb-dataset-of-top-1000-movies-and-tv-shows), ale je tiež prerobená pre jednoduchšiu demonštráciu a pohodlnejšie používanie, ostatné ktoré som vytvoril ja sú tieto tabuľky:
- `imdb_top_1000`
- `directors`
- `education`
- `raters`
- `ratings`


Účelom ETL procesu bolo tieto dáta pripraviť, transformovať a optimalizovať pre viacdimenzionálnu analýzu.

---

### **1.1 Dátová architektúra**

### **ERD diagram**

Surové dáta sú usporiadané v relačnom modeli, ktorý je znázornený na **entitno-relačnom diagrame (ERD)**:

<p align="center">
  <img src="https://github.com/Denyolke/dt_db_projekt/blob/main/imdb_erd_schema.png" alt="ERD Diagram">
  <br>
  <em>Obrázok 1: Entitno-relačná schéma datasetu IMDB</em>
</p>

---

## **2. Dimenzionálny model**

Navrhnutý bol **hviezdicový model (star schema)**, ktorý efektívne podporuje analýzu. Centrálnym bodom je faktová tabuľka **`fact_ratings`**, ktorá je prepojená s nasledujúcimi dimenziami:
- **`dim_raters`**: Obsahuje informácie o filmových kritikov, ktorí film hodnotili.
- **`dim_imdb_top_1000`**: Obsahuje informácie o filmoch, ako je dátum vydania, príjem, ktorý zarobil, žáner a doba trvania(atď).
- **`dim_date`**: Zahrňuje informácie o dátumoch hodnotení (deň, mesiac, rok, deň v týždni).
- **`dim_time`**: Obsahuje podrobné časové údaje (hodina, AM/PM).

Štruktúra hviezdicového modelu je znázornená na diagrame nižšie. Diagram ukazuje prepojenia medzi faktovou tabuľkou a dimenziami, čo zjednodušuje analýzu a implementáciu.

<p align="center">
  <img src="https://github.com/Denyolke/dt_db_projekt/blob/main/imdb_star_schema.png" alt="Star Schema">
  <br>
  <em>Obrázok 2: Schéma hviezdy pre IMDB dataset</em>
</p>

---

## **3. ETL proces v Snowflake**

ETL proces pozostával z troch hlavných fáz: `Extract` (extrahovanie dát), `Transform` (transformácia dát) a `Load` (načítanie dát). Tento proces bol implementovaný v Snowflake na prípravu zdrojových dát z staging vrstvy do viacdimenzionálneho modelu vhodného na analýzu.

---

### **3.1 Extract (Extrahovanie dát)**

Dáta zo zdrojového datasetu (vo formáte `.csv`) boli najprv nahraté do Snowflake prostredníctvom interného stage úložiska s názvom `COYOTE_IMDB_STAGE`. Stage v Snowflake slúži ako dočasné úložisko pre import alebo export dát.

#### Príklad kódu:
```sql
CREATE OR REPLACE STAGE COYOTE_IMDB_STAGE;
```

Súbory obsahujúce údaje o filmoch, hodnoteniach, používateľoch a žánroch boli nahraté do stage a následne importované do staging tabuliek pomocou príkazu `COPY INTO`. Napríklad:
```sql
COPY INTO directors
FROM @COYOTE_IMDB_STAGE/directors.csv
FILE_FORMAT = (
  TYPE = 'CSV'
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  SKIP_HEADER = 1
  ESCAPE_UNENCLOSED_FIELD = '\\' 
  FIELD_DELIMITER = ','
)
ON_ERROR = 'CONTINUE';
```

### **3.2 Transform (Transformácia dát)**

Počas tejto fázy boli dáta zo staging tabuliek vyčistené, transformované a obohatené. Hlavným cieľom bolo pripraviť dimenzie a faktovú tabuľku na efektívnu analýzu.

#### Dimenzné tabuľky

- **`dim_raters`**: 
```sql
CREATE OR REPLACE TABLE dim_raters AS
SELECT 
  DISTINCT
  rater_id,
  first_name,
  last_name,
  gender
FROM raters;
```

- **`dim_imdb_top_1000`**: 
```sql
CREATE OR REPLACE TABLE dim_imdb_top_1000 AS
SELECT 
  DISTINCT
  Series_Id,
  Series_Title,
  Released_Year,
  Certificate,
  Runtime,
  Genre,
  Overview,
  Meta_score,
  Gross
FROM imdb_top_1000;
```

- **`dim_date`**: 
```sql
CREATE OR REPLACE TABLE dim_date AS
SELECT 
  DISTINCT
  ROW_NUMBER() OVER (ORDER BY year, month, day) AS dim_date_id,
  year,
  month,
  day
FROM (
  SELECT 
    EXTRACT(YEAR FROM CURRENT_TIMESTAMP) AS year,
    EXTRACT(MONTH FROM CURRENT_TIMESTAMP) AS month,
    EXTRACT(DAY FROM CURRENT_TIMESTAMP) AS day
);
```

- **`dim_time`**: 
```sql
CREATE OR REPLACE TABLE dim_time AS
SELECT 
  DISTINCT
  ROW_NUMBER() OVER (ORDER BY hour, minute, seconds) AS dim_time_id,
  hour,
  minute,
  seconds
FROM (
  SELECT 
    EXTRACT(HOUR FROM CURRENT_TIMESTAMP) AS hour,
    EXTRACT(MINUTE FROM CURRENT_TIMESTAMP) AS minute,
    EXTRACT(SECOND FROM CURRENT_TIMESTAMP) AS seconds
);
```

#### Faktová tabuľka

- **`fact_ratings`**: 
```sql
CREATE OR REPLACE TABLE fact_ratings AS
WITH time_data AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY CURRENT_TIMESTAMP) AS dim_time_id,  
        EXTRACT(HOUR FROM CURRENT_TIMESTAMP) AS hour,
        EXTRACT(MINUTE FROM CURRENT_TIMESTAMP) AS minute,
        EXTRACT(SECOND FROM CURRENT_TIMESTAMP) AS seconds
    FROM ratings
    LIMIT 1  
),
date_data AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY CURRENT_DATE) AS dim_date_id,       
        EXTRACT(YEAR FROM CURRENT_DATE) AS year,
        EXTRACT(MONTH FROM CURRENT_DATE) AS month,
        EXTRACT(DAY FROM CURRENT_DATE) AS day
    FROM ratings
    LIMIT 1  
)
SELECT 
    i.Series_Title AS Series_Title,          
    r.IMDB_Rating AS IMDB_Rating,           
    d.rater_id AS dim_raters_rater_id,   
    t.dim_time_id AS dim_time_id,           
    da.dim_date_id AS dim_date_id            
FROM ratings r
LEFT JOIN dim_raters d
  ON r.rater_id = d.rater_id
CROSS JOIN time_data t 
CROSS JOIN date_data da 
LEFT JOIN dim_imdb_top_1000 i
  ON r.series_id = i.Series_Id               
ORDER BY i.Series_Title; 
```

---

### **3.3 Load (Načítanie dát)**

Po úspešnom vytvorení dimenzií a faktovej tabuľky boli dáta nahraté do finálnej štruktúry. Na záver boli staging tabuľky odstránené, aby sa optimalizovalo využitie úložiska:
```sql
DROP TABLE IF EXISTS ratings;
DROP TABLE IF EXISTS raters;
DROP TABLE IF EXISTS imdb_top_1000;
DROP TABLE IF EXISTS education;
DROP TABLE IF EXISTS directors;
```

ETL proces v Snowflake umožnil spracovanie pôvodných dát z `.csv` formátu do viacdimenzionálneho modelu typu hviezda. Tento proces zahŕňal čistenie, obohacovanie a reorganizáciu údajov, čo umožňuje efektívnu analýzu.

---

## **4 Vizualizácia dát**

Dashboard obsahuje niektoré z vizualizovaných údajov hodnotení, trendov a lepšie pochopenie samotných používateľov.

<p align="center">
  <img src="https://github.com/Denyolke/dt_db_projekt/blob/main/imdb_dashboard.png" alt="Dashboard">
  <br>
  <em>Obrázok 3: Dashboard IMDB datasetu</em>
</p>

---

### **Graf 1: Priemerné hodnotenie podľa žánrov**
Vizualizácia zobrazuje variácie hodnotenia naprieč žánrami.
```sql
SELECT 
    i.Genre, 
    i.Released_Year, 
    AVG(f.IMDB_Rating) AS Avg_Rating
FROM fact_ratings f
JOIN dim_imdb_top_1000 i
  ON f.Series_Title = i.Series_Title
GROUP BY i.Genre, i.Released_Year
ORDER BY i.Genre, i.Released_Year
;
```

---

### **Graf 2: Počet hodnotení podľa rokov**
Vizualizácia zobrazuje ako sú hodnotenia rozdelené vo všetkých sériách.
```sql
SELECT 
    ROUND(IMDB_Rating, 1) AS Rating_Bucket, 
    COUNT(*) AS Frequency
FROM fact_ratings
GROUP BY Rating_Bucket
ORDER BY Rating_Bucket;
```

### **Graf 3: Počet hodnotení podľa rokov**
Vizualizácia zobrazuje podiel hodnotení od rôznych hodnotiteľov.
```sql
SELECT 
    d.first_name || ' ' || d.last_name AS Rater_Name, 
    COUNT(*) AS Total_Ratings
FROM fact_ratings f
JOIN dim_raters d
  ON f.dim_raters_rater_id = d.rater_id
GROUP BY Rater_Name
ORDER BY Total_Ratings DESC
limit 10;
```

### **Graf 4: Počet hodnotení podľa rokov**
Vizualizácia zobrazuje hodnotenia IMDB pre každú sériu pomocou ID série.
```sql
SELECT 
    Series_Title, 
    AVG(IMDB_Rating) AS Avg_Rating
FROM fact_ratings
GROUP BY Series_Title
ORDER BY Avg_Rating DESC

limit 10;
```

### **Graf 5: Počet hodnotení podľa rokov**
Vizualizácia zobrazuje trend vydávania sérií v priebehu rokov
```sql
SELECT 
    i.Released_Year AS Release_Year, 
    COUNT(*) AS Total_Series
FROM fact_ratings f
JOIN dim_imdb_top_1000 i
  ON f.Series_Title = i.Series_Title
GROUP BY Release_Year
ORDER BY Release_Year;
```

**Autor:** Dániel Tátyi