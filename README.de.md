
In den letzten Jahren werden in Deutschland, gerade im ländlichen Raum, sogenannte Mitfahrbänke aufgestellt.
Wer auf diesen sitzt, signalisiert, mancherorts über ein zusätzlich ausgeklapptes Schild über der Bank, seinen Mitfahrwunsch.
Vorbeikommende AutofahrerInnen sollen so einfach erkennen können, ob sie das gleiche Ziel haben und im Besten Fall anhalten und den/die Mitfahrwilligen mitnehmen.

Ob diese Mitfahrbänke im Alltag allerdings wirklich genutzt werden, ist unsicher. Mehrere Berichte stimmen skeptisch. Aber vielleicht liegt es auch nur daran, dass Menschen unsicher sind, ob sie im Zweifel tatsächlich in absehbarer Zeit mitgenommen würden? Was wäre, wenn für verschiedene Tageszeiten Erfahrungswerte vorlägen, wie lange man im Schnitt warten muss, bis man mitgenommen wird? Dann könnte man in einem Routenplaner das Mitfahren mit Schätzzeiten als eine Reiseoption anzeigen. Umgekehrt würde eine Einbindung in Routenplaner einen konkreten Anreiz schaffen, solche Wartezeiten auch zu erheben.

In diesem Blogpost zeigen wir, wie Mitnahmeverkehre in den OpenSource-Routenplaner OpenTripPlanner (OTP) integriert werden können und damit solche Reiseauskünfte möglich werden.

Dazu erstellen wir einen hypothetischen Fahrplan für unsere Mitfahrbank und binden diesen in OTP ein.
OpenTripPlanner kann Fahplandaten im sogenannten General Transit Feed Specification (GTFS)-Format importieren.

Ein solcher Feed besteht aus mehreren Textdateien im CSV-Format, die in einer Zip-Datei zusammengefasst sind.
Wir legen die folgenden Dateien alle in einem Ordner ```gtfs``` an.

### stops.txt
Beginnen wir mit der stops.txt-Datei, in der die Lage unserer Mitfahrbank und möglicher Ziele erfassen. Die Koordinaten und die optionale stop_url übernehmen wir aus OpenStreetMap http://overpass-turbo.eu/s/IZW.

```
stop_id,stop_name,stop_url,stop_lat,stop_lon
ST1,Mitfahrbank Nersingen Seyboldt,https://commons.wikimedia.org/wiki/File:Mitfahrbank_Nersingen_Seybold_01.jpg,48.4289217,10.1220101
ST2,Mitfahrbank Oberfahlheim Brücke,https://commons.wikimedia.org/wiki/File:Mitfahrbank_Oberfahlheim_Brücke.jpg,48.4295964,10.1438063
ST3,Mitfahrbank Unterfahlheim Flurweg,https://commons.wikimedia.org/wiki/File:Mitfahrbank_Unterfahlheim_Flurweg_01.jpg,48.4289574,10.1603
```

### routes.txt
Die nächste erforrderliche Datei ist die routes.txt https://developers.google.com/transit/gtfs/reference/#routestxt. 
In ihr beschreiben wir die Mitfahrverbindungen zwischen unseren eben angelegten Stops. Für Mitfahren bzw. Trampen existiert derzeit
kein offizieller route_type, der allerdings verpflichtend anzugeben ist. Für MFDZ haben wir uns temporär damit beholfen, OTP zu patchen und
einen neuen Code 1700 für Fahrgemeinschaften einzuführen. Wer unsere OTP-Variante nutzt, kann also 1700 als route_type verwenden. Wer 
den offizielleen OTP nutzt, muss sich vorerst einmal mit einem noch nicht belegten Code der extended route types https://developers.google.com/transit/gtfs/reference/extended-route-types behelfen, der vom OpenTripPlanner dennoch ohne Code-Anpassung unterstützt wird, z.B. 718 in der Kategorie Bus.

```
route_id,route_long_name,route_type 
R1,Mitfahrverbindung Nersingen - Unterfahlheim,1700
```

### calendar.txt
Die calender.txt definiert, an welchen Wochentagen ein bestimmter Takt (Service) gilt. Für dieses Beispiel vereinfachen wir und unterstellen, dass alle von uns gleich definierten Wartezeiten ganzjährig in 2019 an allen Tagen der Woche identisch seien. Die service_id ist frei wählbar. Wir verwenden die sprechende ID ALL_DAYS.

```
service_id,monday,tuesday,wednesday,thursday,friday,saturday,sunday,start_date,end_date
ALL_DAYS,1,1,1,1,1,1,1,20190101,20191231
```

### trips.txt
Die trips.txt beschreibt regelmäßig in gleicher Form wiederkehrende Fahrten, das heißt, jeder Trip besitzt gleiche Eigenschaften (wie z.B. Fahrradmitnahme erlaubt/untersagt) sowie - an anderer Stelle definierte - gleiche Abfolgen angefahrener Haltestellen. Wir definieren Hin- und Rückfahrt der Route Nersingen - Unterfahlheim und untersagen die Fahrrad-Mitnahme.

```
route_id,service_id,trip_id,trip_headsign,direction,bikes_allowed
R1,ALL_DAYS,T1,Unterfahlheim,0,2
R1,ALL_DAYS,T2,Nersingen,1,2
```

### stop_times.txt
In der stop_times.txt werden die Halte-Folgen und zugehörigen Abfahrtszeiten einer Fahrt beschrieben.
Für unseren Frequenz-basierten Fahrplan reicht es aus, eine Fahrt anzugegen, deren Dauern zwischen den Halten dann für alle regelmäßigen Fahrten angenommen werden. Um die Dauern zu bestimmen, nutzen wir GraphHopper/OpenStreetMap https://graphhopper.com/maps/?point=48.428973%2C10.160336&point=Oberfahlheim%2C%2089278%2C%20Deutschland&point=48.428859%2C10.121841&locale=de-DE&vehicle=car&weighting=fastest&elevation=true&use_miles=false&layer=Omniscale. 

```
trip_id,service_id,arrival_time,departure_time,stop_id,stop_sequence,pickup_type,drop_off_type
T1,ALL_DAYS,6:00:00,6:00:00,ST1,1,0,0
T1,ALL_DAYS,6:02:00,6:02:00,ST2,1,0,0
T1,ALL_DAYS,6:04:00,6:04:00,ST3,1,0,0
T2,ALL_DAYS,6:00:00,6:00:00,ST3,1,0,0
T2,ALL_DAYS,6:02:00,6:02:00,ST2,1,0,0
T2,ALL_DAYS,6:04:00,6:04:00,ST1,1,0,0
```

### frequencies.txt
Und nun kommen wir zur Gretchenfrage: Wie lange muss jemand zu welcher Tageszeit warten, bis ein Auto hält und er/sie mitgenommen wird?
Für dieses Beispiel nehmen wir an, dass es morgens im Berufsverkehr im Schnitt 5 Minuten dauert, den Rest des Tages jedoch 15 Minuten. 
Wir nutzen das GTFS-Konstrukt der sogenannten "Frequenz-basierten" Fahrten in GTFS, das die Abbildung von getakteten Verkehren ohne feste Uhrzeiten ermöglicht. OpenTripPlanner plant exakt die angegebene Taktzeit als Wartezeit ein. Damit bleibt es nun uns überlassen, ob wir eher einen größeren Puffer und dafür aber vielleicht auch realistischere Anschlüsse bevorzugen, oder ob uns eine 50% Wahrscheinlichkeit ausreicht. Wir sind mal lieber vorsichtig und verdoppeln die mittlere Wartezeit. Aber das könnte ja mal untersucht werden...  

```
trip_id,start_time,end_time,headway_secs
T1,06:00,08:30,600
T1,08:30,19:00,1800
T2,06:00,08:30,600
T2,08:30,19:00,1800
```

### agency.txt
Zu guter letzt fordert die GTFS-Spezifikation Angaben zum Verkehrsverbund/-unternehmen, das die Fahrten anbietet. Bei unserer Mitfahrbank gibt es eigentlich keinen Betreiber, aber zumindest einen Zeitungsbericht mit allgemeinen Informationen zum Projekt können wir als agency_url angeben.

```
agency_id,agency_name,agency_url,agency_timezone
A1,Mitfahrbänke Nersingen,https://www.augsburger-allgemeine.de/neu-ulm/Per-Anhalter-durch-die-Gemeinde-id54321911.html,Europe/Berlin
```

Das alles packen wir in ein zip-Archiv unterhalb eines neu angelegten `data` Ordners:

```
mkdir data
zip -j data/mitfahrbank.gtfs.zip gtfs/*.txt
```

## OpenStreetMap-Daten
Zusätzlich brauchen wir noch OpenStreetMap-Daten, die uns das für das Fuß-, Rad- und PKW-Routing nötiige Straßennetz bereitstellen.

Dieses kann man z.B. über den Browser oder mittels wget oder curl herunterladen, z.B. so:

```
curl http://download.geofabrik.de/europe/germany/bayern/schwaben-latest.osm.pbf > schwaben.latest.osm.pbf
```

## OpenTripPlanner
Um den OpenTripPlanner zu installieren, kann man entweder das github-Repository auschecken und ihn selber bauen, oder ein von uns gebautes Docker-Image verwenden.

Wir verwenden ein fertiges Docker-Image der MFDZ-Variante des OpenTripPlanners und starten ihn mittels docker-compose. Unser docker-compose.yml sieht so aus:

version: '3'
services:
  otp:
    container_name: opentripplanner
    image: "mfdz/opentripplanner"
    ports:
     - "8080:8080"
    environment: 
      JAVA_MX: 2G
    volumes:
      - "$PWD/data:/var/otp/"
    command: otp --build /var/otp --preFlight
    
Bevor wir es starten, legen wir noch die eben erzeugte gtfs.zip und die heruntergeladene pbf-Datei in ein `data`-Verzeichnis.

 und rufen dann 
`docker-compose up` auf.

Recht bald nach dem Start sollten zwei OTP zwei Zeilen ausgeben:

[36mopentripplanner | [0m 19:59:42.659 INFO (GraphBuilder.java:202) Found OSM file /var/otp/schwaben.latest.osm.pbf
[36mopentripplanner | [0m 19:59:42.664 INFO (GraphBuilder.java:198) Found GTFS file /var/otp/mitfahrbank.gtfs.zip

OTP baut nun den GaphenNicht erschrecken ob der vielen Warnungen, die der OpenTripPlanner beim Bauen des Routing-Graphen so ausspuckt, die meisten sind nicht von Belang, aber das ist ein Thema für einen anderen Blogpost. 

Wenn uns kein Fehler bei der Formattierung der GTFS-Dateien unterlaufen ist, sollten wir nach 2-5 Minuten eine Ausgabe ähnlich der folgenden sehen:

```
[36mopentripplanner | [0m 20:17:31.363 INFO (GtfsModule.java:189) This Agency has the ID A1
[36mopentripplanner | [0m 20:17:31.363 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.Block
[36mopentripplanner | [0m 20:17:31.365 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.ShapePoint
[36mopentripplanner | [0m 20:17:31.371 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.Note
[36mopentripplanner | [0m 20:17:31.372 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.Area
[36mopentripplanner | [0m 20:17:31.373 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.Route
[36mopentripplanner | [0m 20:17:31.387 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.Stop
[36mopentripplanner | [0m 20:17:31.392 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.Trip
[36mopentripplanner | [0m 20:17:31.406 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.StopTime
[36mopentripplanner | [0m 20:17:31.415 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.ServiceCalendar
[36mopentripplanner | [0m 20:17:31.422 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.ServiceCalendarDate
[36mopentripplanner | [0m 20:17:31.423 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.FareAttribute
[36mopentripplanner | [0m 20:17:31.424 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.FareRule
[36mopentripplanner | [0m 20:17:31.425 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.Frequency
[36mopentripplanner | [0m 20:17:31.428 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.Pathway
[36mopentripplanner | [0m 20:17:31.430 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.Transfer
[36mopentripplanner | [0m 20:17:31.432 INFO (GtfsModule.java:180) reading entities: org.onebusaway.gtfs.model.FeedInfo
[36mopentripplanner | [0m 20:17:31.545 INFO (PatternHopFactory.java:419) Added 4 frequency-based and 0 single-trip timetable entries.
```

Nach zwei bis fünf Minuten sollte der Graph gebaut sein und der Server unter Port 8080 erreichbar sein.

http://localhost:8080/otp/routers/default/

http://localhost:8080/otp/routers/default/plan?fromPlace=48.428915792143506%2C10.162138938903809&toPlace=48.42857407392729%2C10.121197700500488&time=7%3A00am&date=05-22-2019&mode=TRANSIT%2CWALK&maxWalkDistance=2000&arriveBy=false&wheelchair=false&locale=en

## Troubleshooting
### Eine meiner Mitfahrbänke wird in der Karte nicht angezeigt und auch nicht bei der Verbindungssuche genutzt.
Nochmals sorgfältig kontrollieren, ob alle CSV-Dateien mit einem CR/CRLF abschließen, da sonst ggf. die letzte Zeile nicht erkannt wird.
