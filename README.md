# nplso_clean_topology

```
docker run --rm --name nplso-clean-topology -p 54321:5432 --hostname primary \
-e PG_DATABASE=oereb -e PG_LOCALE=de_CH.UTF-8 -e PG_PRIMARY_PORT=5432 -e PG_MODE=primary \
-e PG_USER=admin -e PG_PASSWORD=admin \
-e PG_PRIMARY_USER=repl -e PG_PRIMARY_PASSWORD=repl \
-e PG_ROOT_PASSWORD=secret \
-e PG_WRITE_USER=gretl -e PG_WRITE_PASSWORD=gretl \
-e PG_READ_USER=ogc_server -e PG_READ_PASSWORD=ogc_server \
sogis/oereb-db:latest
```

```
java -jar /Users/stefan/apps/ili2pg-4.3.0/ili2pg-4.3.0.jar --dbhost localhost --dbport 54321 --dbdatabase oereb --dbusr admin --dbpwd admin --disableValidation --strokeArcs --nameByTopic --defaultSrsCode 2056 --doSchemaImport --dbschema etziken --import exp1_npletztst03_20191104C_NoVal.xtf
```

```

```
Drei Polygone müssen manuell entfernt, weil doppelt vorhanden:

```
DELETE FROM 
    etziken.nutzungsplanung_grundnutzung 
WHERE  
    t_ili_tid IN ('82f9eb41-1dd2-479c-911c-76852a9388d3','255e5503-a79a-4ae0-9d04-454f96109852','14afac84-7747-4246-8756-a37736e5b229')
```
Eines ist wohl falsch. Man müsste aber zuerst noch das grössere aufschneiden und einer anderen Art zuweisen.


see: https://github.com/sogis/gretljobs/blob/master/agi_hoheitsgrenzen_pub/agi_hoheitsgrenzen_pub_sogis_hoheitsgrenzen_gemeindegrenze.sql


```
WITH overlaps_to_gaps AS (
    SELECT
        t_ili_tid,
        COALESCE(
            ST_Difference(geometrie,
                (
                    SELECT 
                        ST_Union(grundnutzung_1.geometrie)
                    FROM 
                        etziken.nutzungsplanung_grundnutzung AS grundnutzung_1
                    WHERE 
                        ST_Intersects(grundnutzung_2.geometrie, grundnutzung_1.geometrie)
                        AND 
                        grundnutzung_2.t_ili_tid != grundnutzung_1.t_ili_tid
                )
            ),
            grundnutzung_2.geometrie
        ) AS geometrie
    FROM 
        etziken.nutzungsplanung_grundnutzung AS grundnutzung_2
),
grundnutzung_multipolygon_gaps AS (
    SELECT
        geometrie
    FROM (
        SELECT
            ST_Union(geometrie) AS geometrie
        FROM
            overlaps_to_gaps
        ) AS query
),
perimetergeometrie AS (
    SELECT
        ST_Union(geometrie) AS geometrie
    FROM
        (
            SELECT
                ST_MakePolygon(ST_ExteriorRing(subquery.geometrie)) AS geometrie,
                1 AS id
            FROM
                (
                    SELECT
                        (ST_Dump(ST_Union(geometrie))).geom AS geometrie
                    FROM 
                        etziken.nutzungsplanung_grundnutzung
                ) AS subquery
            GROUP BY
                id,
                subquery.geometrie
        ) AS query
),
gaps AS (
    SELECT
        geometrie,
        ROW_NUMBER() OVER() AS id
    FROM (
        SELECT DISTINCT
            (ST_Dump(differenz.differenz_geometrie)).geom AS geometrie
        FROM 
            perimetergeometrie, (
                SELECT DISTINCT
                    ST_Difference(perimetergeometrie.geometrie, grundnutzung_multipolygon_gaps.geometrie) AS differenz_geometrie
                FROM 
                    perimetergeometrie,
                    grundnutzung_multipolygon_gaps
            ) AS differenz
        ) AS query
),
flaechenmass_grundnutzung AS (
    SELECT
        t_ili_tid,
        geometrie,
        ST_Area(geometrie) AS flaechemass
    FROM
        overlaps_to_gaps
),
area AS (
    SELECT DISTINCT
        ST_Intersects(gaps.geometrie, flaechenmass_grundnutzung.geometrie),
        t_ili_tid,
        flaechemass,
        gaps.geometrie,
        gaps.id
    FROM
        flaechenmass_grundnutzung,
        gaps
    WHERE
        ST_Intersects(gaps.geometrie, flaechenmass_grundnutzung.geometrie) = True
),
zugehoerigkeit AS (
    SELECT DISTINCT
        CASE
            WHEN area_a.flaechemass > area_b.flaechemass
                THEN area_a.t_ili_tid
            WHEN area_a.flaechemass < area_b.flaechemass
                THEN area_b.t_ili_tid
            WHEN area_a.flaechemass = area_b.flaechemass
                THEN area_a.t_ili_tid
        END AS groesser,
        area_a.geometrie
    FROM
        area AS area_a,
        area AS area_b
    WHERE 
        area_a.id = area_b.id
        AND 
        area_a.t_ili_tid <> area_b.t_ili_tid
),
gaps_multipolygon AS (
    SELECT DISTINCT
        ST_Collect(geometrie) AS geometrie,
        groesser AS t_ili_tid
    FROM 
        zugehoerigkeit
    GROUP BY
        groesser
),
corrected_polygons AS (
    SELECT
        overlaps_to_gaps.t_ili_tid,
        ST_Union(gaps_multipolygon.geometrie, overlaps_to_gaps.geometrie) as geometrie
    FROM 
        gaps_multipolygon,
        overlaps_to_gaps
    WHERE 
        gaps_multipolygon.t_ili_tid = overlaps_to_gaps.t_ili_tid
)
SELECT
    orig.t_id,
    orig.t_ili_tid,
    geometry.geometrie
FROM
(
    SELECT
        t_ili_tid,
        ST_Multi(ST_Union(geometrie)) AS geometrie
    FROM
        overlaps_to_gaps
    WHERE
        t_ili_tid NOT IN (
            SELECT
                t_ili_tid
            FROM
                corrected_polygons)
    GROUP BY
        t_ili_tid
    
    UNION
    
    SELECT
        t_ili_tid AS gemeindename,
        ST_Multi(ST_Union(geometrie)) AS geometrie
    FROM
        corrected_polygons
    GROUP BY
        t_ili_tid
) AS geometry
LEFT JOIN etziken.nutzungsplanung_grundnutzung AS orig
ON orig.t_ili_tid = geometry.t_ili_tid
;
```