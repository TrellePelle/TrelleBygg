# Gränssnitt

Webbsidan är en React-applikation som bara pratar med det interna API:et. Alla vyer har egna adresser så att bakåtknappen och bokmärken fungerar.

## Adresser

| Adress | Vy |
|---|---|
| `/` | Startsida: karta och nyhetsflöde |
| `/?projekt=<id>` | Startsida med projektpanelen öppen |
| `/projekt` | Listvy med alla projekt |
| `/projekt/<id>` | Projektsida |
| `/granska` | Att granska |
| `/korningar` | Körlogg och kostnader |
| `/installningar` | Inställningar |
| `/status` | Systemstatus |
| `/logga-in` | Inloggning |

## Sidhuvud (alla vyer)

Titel, "Senast hämtat" och antal projekt, knappen **Hämta nya** (den enda framträdande knappen), samt en meny med: Fullständig genomsökning, Tolka om, Alla projekt, Att granska, Körlogg, Inställningar, Systemstatus, Logga ut. Medan ett jobb pågår visas förloppet i sidhuvudet. Ett jobb som väntar på kostnadsbekräftelse visas tydligt med knapparna "Kör AI-tolkning" och "Avbryt".

## Startsida

Två kolumner: karta (cirka 60 %) och nyhetsflöde (cirka 40 %). På mobil staplas de med flikar för att växla.

**Karta**
- Filter ovanför kartan: fasgrupp, projekttyp, ort.
- Nålar färgade efter fasgrupp. `avslutad` är dold som standard.
- Precision syns: skarp nål eller blek cirkel.
- Förklaring under kartan, samt källhänvisning till OpenStreetMap.
- Klick på nål öppnar projektpanelen.

**Nyhetsflöde: "Sedan ditt senaste besök"**
- Händelser som användaren inte sett, nyast först. Baslinjehändelser visas inte.
- Etiketter: Öppet för synpunkter, Nytt projekt, Fasändring, Osäker, Lämnade listan.
- Projekt med öppen tidsfrist för synpunkter visas överst.
- Hovring markerar motsvarande nål på kartan. Klick öppnar projektpanelen.
- "Tidigare händelser" fäller ut äldre.
- Händelser markeras som sedda när panelen öppnas eller via "Markera alla som sedda".

## Projektpanel

Ersätter nyhetsflödet, med tillbakalänk till nyheterna. Kartan ligger kvar med vald nål markerad.

Innehåll: namn, typ och ort; fasetikett (med osäkerhetsmarkering vid behov); fassteg för projekttypen; AI-sammanfattning märkt som AI; rader som bara visas när de är relevanta (synpunkter senast, senaste händelse, dokument kända/nedladdade, platsens precision); knapparna **Öppna projektet** och länk till kommunens sida.

## Projektsida

1. **Sidhuvud:** namn, typ, fas, osäkerhetsmarkering, länk till kommunens sida.
2. **Fakta** i två kolumner: plannummer, diarienummer, fastigheter, ort, stadsdel, antal bostäder, överordnat projekt. Varje uppgift visar ursprung (regel, AI, du) och säkerhet, och kan rättas direkt. En rättad uppgift låses.
3. **Karta** med platsen och knapparna "Fel plats" och "Placera själv".
4. **Tidslinje** med alla händelser, nyast överst, med källa och ursprung.
5. **Dokument** grupperade efter fas. Knappar efter läge: "Hämta" (känd), "Öppna" och "Tolka med AI" (nedladdad), "Visa tolkning" (tolkad).
6. **Relaterade projekt:** överordnat projekt och delprojekt.
7. **Åtgärder:** Slå ihop med…, Dela projekt, Lägg till anteckning, Sätt fas manuellt.

## Listvy

Tabell med alla projekt: namn, typ, fas, ort, senaste händelse, plats (ja/nej/ungefärlig). Fritextsökning, filter (fasgrupp, typ, ort, osäker, saknar plats) och sortering. Klick öppnar projektsidan.

## Att granska

Listor med åtgärdsknappar: osäkra uppgifter (bekräfta / rätta), osäkra matchningar (bekräfta / flytta / nytt projekt), projekt som lämnat listan utan bekräftad fas, projekt utan plats eller med ungefärlig plats, och ogiltiga AI-svar.

## Körlogg

Varje jobb med tid, typ, status, vad som hittades, antal AI-anrop och kostnad. Summa för innevarande månad och kvar till månadstaket.

## Inställningar

Bevakningslista, pauser mellan förfrågningar, AI-modell, kostnadstak per körning och per månad, kontaktadress i User-Agent. Nycklar visas aldrig, bara om de är konfigurerade.

## Systemstatus

Worker igång eller inte, senaste lyckade körning per bevakningsgrupp, varningar från sanity checks, senaste säkerhetskopia och provåterställning, diskutrymme för datakatalogen.

## Design

- Ljust och mörkt läge.
- Färger per fasgrupp: planeras (lila), synpunkter (bärnsten), beslutad (grönblå), byggs (korall), avslutad (grå). Färg är aldrig enda bäraren av information; etiketter och text finns alltid.
- Svenska genomgående, svensk datumvisning.
- All text från kommunen och AI visas som text, aldrig som HTML.
