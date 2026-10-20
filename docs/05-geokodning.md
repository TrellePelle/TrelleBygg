# Geokodning

Geokodaren omvandlar platsangivelser till koordinater. Den är inte AI och svaren cachas, så den får köras automatiskt för nya platser i samband med `hamta_nya`.

## Kedja

Prova i ordning och stanna vid första resultat med precision `exakt`. Annars behålls bästa resultat.

1. **Gatuadress** ur projektuppgifterna → Nominatim.
2. **`platsbeskrivning` från AI-tolkningen** → Nominatim.
3. **Fastighetsbeteckning** → Lantmäteriet. *Inte i första versionen.* Licens och åtkomst utreds senare. Implementeras som ännu en geokodare bakom samma protokoll.
4. **Ortens mittpunkt** → precision `ort`.
5. **Manuell placering** av användaren → precision `manuell`, låst.

Testat: gatuadresser ger träff på gatunivå, fastighetsbeteckningar ger ingen träff, och "Maglarp 28:20" gav byn Maglarp. Ett resultat kan alltså se ut som en träff men vara ungefärligt. Precisionen är därför obligatorisk.

## Nominatim

- Högst en förfrågan per sekund, identifierande User-Agent, svar cachas (krav enligt Nominatims användarvillkor).
- `countrycodes=se` och `viewbox` + `bounded=1` med kommunens gränsruta.
- Träffar utanför kommunens gränsruta avvisas.
- Precision från svarets typ: husnummer → `exakt`; väg eller gata → `gata`; by, ort, stadsdel → `ort`.
- Källhänvisning till OpenStreetMap visas på kartan.

## Cache

Tabellen `plats` nycklas på normaliserad söktext (gemener, trimmade blanksteg, ", Trelleborg" tillagt om ort saknas). Samma söktext skickas aldrig två gånger till en extern tjänst.

## Felaktiga platser

- "Fel plats" på kartan sätter kopplingen `projekt_plats.status = avvisad`. Platsen raderas inte, eftersom en ny fråga skulle ge samma felaktiga svar.
- En avvisad plats kopplas aldrig igen till samma projekt. Geokodaren går vidare i kedjan, eller lämnar projektet utan plats.
- "Placera själv" låter användaren klicka på kartan och skapar en låst plats med precision `manuell`.
- Projekt utan plats syns i listvyn och på granskningssidan.

## Visning

- `exakt` och `manuell`: skarp nål.
- `gata`: nål med lätt markering.
- `ort`: blek cirkel.

## Framtid

`projekt.geometri` finns redan för planområden. När databasen byts till PostGIS kan områden visas som ytor.
