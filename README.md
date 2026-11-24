# Global Capitals Weather Analytics — End-to-End Microsoft Fabric & Power BI Project

Pipeline end-to-end di Data Engineering e Business Intelligence sviluppata interamente su **Microsoft Fabric**. Il progetto estrae 365 giorni di rilevazioni meteorologiche orarie per 29 capitali mondiali tramite API REST, modella i dati in un Lakehouse con architettura **Star/Snowflake Schema** su tabelle **Delta Lake** e li espone in un report interattivo **Power BI** a due pagine.

---

## Architettura del Progetto

---

## Stack Tecnologico

* **Data Ingestion & Transformation**: Python, PySpark, Pandas, REST API (`archive-api.open-meteo.com`)
* **Storage & Lakehouse**: Microsoft Fabric Lakehouse, Delta Lake (`MERGE` idempotente, schema enforcement)
* **Data Modeling & Analytics**: Fabric Semantic Model, DAX (calcoli contestuali e gerarchici)
* **Data Visualization**: Power BI Service

---

## Data Pipeline (ETL)

Il notebook PySpark (`NB_Meteo`) gestisce l'intero ciclo di estrazione, trasformazione e caricamento:

1. **Estrazione API**: scarica 365 giorni di dati orari (`temperature_2m`, `relative_humidity_2m`, `wind_speed_10m`) per 29 capitali distribuite su 6 continenti, con gestione dei retry e rate-limiting.
2. **Normalizzazione e Chiavi Surrogate**: genera identificativi deterministici tramite hash MD5 (`continent_id`, `country_id`, `city_id`) e chiavi temporali intere (`timestamp_id` nel formato `yyyyMMddHH`).
3. **Caricamento Incrementale Idempotente**: utilizza `DeltaTable.merge()` per aggiornare i record esistenti e inserire solo i nuovi dati senza duplicazioni (`city_id` + `timestamp_id`).
4. **Audit & Observability**: registra ogni esecuzione nella tabella `etl_run_log` tracciando `run_id` (UUID), stato (`SUCCESS`/`FAILED`), numero di righe elaborate, eventuali città fallite e timestamp.

---

## Modello Dati (Star / Snowflake Schema)

* **`fact_weather_hourly`**: tabella dei fatti a granularità oraria per città (`city_id`, `timestamp_id`, `temperature`, `humidity`, `wind_speed`).
* **`dim_city`**: anagrafica delle 29 capitali con coordinate geografiche (`latitude`, `longitude`) e fuso orario (`timezone`).
* **`dim_countries`**: paesi collegati ai rispettivi continenti.
* **`dim_continents`**: macro-aree geografiche (Europa, Asia, Nord America, Sud America, Africa, Oceania).
* **`dim_time`**: dimensione temporale oraria (`date`, `year`, `month`, `day`, `quarter`, `hour`, `day_of_week`).

---

## Principali Misure DAX

* **`Temperatura Media (°C)`**: media aritmetica delle rilevazioni orarie nel contesto selezionato.
* **`Temperatura Ultima Ora (°C)`**: temperatura media rilevata nell'ultimo `timestamp_id` disponibile nel dataset.
* **`Escursione Termica (°C)`**: differenza tra temperatura massima e minima assoluta (`MAX - MIN`) nel contesto attivo.
* **`Temp Media Continente (°C)`**: calcola la media del continente di appartenenza rimuovendo i filtri su città e paese tramite `REMOVEFILTERS(dim_countries, dim_city)`.
* **`Delta vs Media Continente (°C)`**: scostamento termico della singola città o paese rispetto alla media del proprio continente, con gestione `ISBLANK` per evitare combinazioni cartesiane vuote nelle gerarchie.

---

## Struttura del Report Power BI

1. **Pagina 1 — Panoramica Globale**
   * **Andamento Mensile**: grafico a linee con la curva termica media sui 12 mesi per ciascuna capitale.
   * **KPI Summary**: 5 schede riepilogative con valori medi e picchi annuali (Temperatura Ultima Ora, Escursione Termica, Temperatura Media, Vento Medio, Umidità Media).
   * **Ranking Città**: grafico a barre orizzontali ordinato per temperatura media annua.

2. **Pagina 2 — Dettaglio Geografico**
   * **Filtro Temporale**: selettore orizzontale a riquadri per filtrare l'analisi per singolo mese (`1–12`).
   * **Matrice Gerarchica (`Continente > Paese > Città`)**: esplorazione drill-down delle metriche climatiche con formattazione condizionale su `Delta vs Media Continente (°C)` per evidenziare immediatamente le anomalie termiche positive (rosso) e negative (blu).

---

## Come Replicare il Progetto

1. Creare un **Lakehouse** (`LH_Meteo`) in un workspace Microsoft Fabric.
2. Importare il notebook PySpark e collegarlo al Lakehouse come destinazione predefinita.
3. Eseguire il notebook per popolare le tabelle Delta e `etl_run_log`.
4. Creare il **Modello Semantico** collegando le tabelle secondo le chiavi primarie/esterne (`city_id`, `country_id`, `continent_id`, `timestamp_id`) e aggiungere le misure DAX.
5. Aprire o importare il report Power BI collegandolo al modello semantico.


