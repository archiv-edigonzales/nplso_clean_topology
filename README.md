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
)
SELECT 
    *
FROM
    overlaps_to_gaps
```