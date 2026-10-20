# Insamling

## Källan: trelleborg.se

Kommunens webbplats körs på WordPress och exponerar WordPress REST API under `https://www.trelleborg.se/wp-json/wp/v2/`. Detta är inte en tjänst kommunen lovat att hålla igång, och därför är insamlaren isolerad.

Kontrollerat (oktober 2026):

- `robots.txt` förbjuder `/wp-admin/` och `/app/uploads/`. `/wp-json/` är tillåtet.
- Innehållstyper: `pages`, `project`, `bygglov`, `anslagstavla`.
- Parametern `modified_after` fungerar. Fältet `modified_gmt` ska användas, inte `modified` (lokal tid).
- Högst 100 träffar per sida (`per_page=100`). Totalt antal i svarshuvudena `X-WP-Total` och `X-WP-TotalPages`.
- Sidan "Detaljplaner" har undersidorna "Planbesked" och "Pågående detaljplaner". Det finns ingen avdelning för antagna planer.
- PDF-länkar finns inte i API:ets `content` utan i den renderade HTML-sidan.
- `sitemap.xml` innehåller bara omkring 500 sidor och saknar många detaljplaner. Den är ingen pålitlig reserv.
- Servern svarade en gång med något som inte var giltig JSON vid tätt upprepade anrop. Hantera det som ett fel.

## Bevakningslista

`config/bevakning.yaml`. Slugs och ID:n verifieras i byggsteg 2 och slås upp via `slug` vid start i stället för att hårdkodas.

```yaml
kommun: trelleborg
bas_url: https://www.trelleborg.se
grupper:
  - namn: pagaende_detaljplaner
    kalltyp: wp_sida
    forelder_slug: pagaende-detaljplaner
    projekttyp: detaljplan
    hamta_html_for_dokument: true
  - namn: planbesked
    kalltyp: wp_sida
    forelder_slug: planbesked
    projekttyp: detaljplan
  - namn: stadsutveckling
    kalltyp: wp_sida
    forelder_slug: stadsutvecklingsprojekt
    projekttyp: stadsutveckling
    uteslut_under_slug: [teknisk-handbok]   # verifiera slug
  - namn: planarbete_ovrigt
    kalltyp: wp_sida
    forelder_slug: planarbete
    inkludera_titlar: [Vägplan, Planprogram, Översiktsplan]
  - namn: vatten_infrastruktur
    kalltyp: wp_sida
    sidor: []   # nytt vattenverk, Sjöviksverket: identifiera i steg 2
    projekttyp: infrastruktur
  - namn: anslagstavla_plan
    kalltyp: wp_anslag
    filter_nyckelord: [detaljplan, planprogram, vägplan, samråd, granskning, antagande, laga kraft]
```

## Körningens flöde

### hamta_nya

1. Läs `robots.txt` och avbryt om bevakade sökvägar har blivit förbjudna.
2. För varje bevakningsgrupp:
   a. Hämta **hela listan** över undersidor (en förfrågan per grupp, med sidning). Jämför med kända källor för att hitta nya och försvunna sidor.
   b. Hämta det som ändrats sedan `korningsmarkor.senast_lyckad` minus en dag, med `modified_after`.
3. Hämta innehållet för nya och ändrade källor via API:et. För grupper med `hamta_html_for_dokument` hämtas även den renderade HTML-sidan för att hitta PDF-länkar.
4. Normalisera text (ta bort skript, menyer, blanksteg och dynamiska delar), räkna SHA-256 och jämför med `kalla.aktuell_hash`. Bara verkligt ändrat innehåll blir en ny `raversion` med status `andrad`.
5. Kör sanity checks (se nedan).
6. Kör regeltolkaren på nytt och ändrat.
7. Räkna fram vad som behöver AI-tolkas och uppskatta kostnaden. Sätt jobbet i `vantar_bekraftelse`.
8. Flytta fram `korningsmarkor.senast_lyckad` **bara** om alla steg lyckats.

### Försvunna sidor

En källa som inte längre finns i listan får `status = forsvunnen` och projektet får händelsen `lamnade_listan`. Det tolkas **inte** som laga kraft. Fasen blir `laga_kraft` först när en kungörelse om antagande eller laga kraft på anslagstavlan har matchats till projektet. Om sidan dyker upp igen skapas `atervande_till_listan`.

### fullstandig_genomsokning

Som `hamta_nya` men utan `modified_after`. Hashjämförelsen gör att bara verkligt ändrat innehåll går vidare till AI.

## Regeltolkning av PDF-länkar

Filnamnen följer i regel mönstret:

```
dp-262-planbeskrivning_samrad_pl-2024-73_2026-10-06.pdf
dp-<nr>-<dokumenttyp>[-<fas>]_<fas>_pl-<år>-<nr>_<åååå-mm-dd>.pdf
```

Ur dem läses plannummer, dokumenttyp, fas, diarienummer och datum ut. Filer som inte följer mönstret (t.ex. utredningar) får `dokumenttyp = utredning` eller `ovrigt`. Mönstren ligger som reguljära uttryck med testfall i fixtures.

En dokumentfas som är senare än projektets nuvarande fas skapar en `fasandring` med `ursprung = regel`.

PDF:er laddas **aldrig** ner automatiskt. Länken sparas med `lage = kand`. Nedladdning sker bara när användaren trycker på "Hämta".

## Robusthet

- Allt som inte är ett giltigt, förväntat svar är ett fel: HTTP-fel, ogiltig JSON, oväntad struktur. Försök igen med exponentiell väntetid (t.ex. 5 s, 15 s, 45 s), därefter misslyckas jobbet.
- **Sanity checks** efter varje körning: listan över pågående detaljplaner får inte vara tom, antalet får inte ha minskat med mer än hälften, och förväntade fält måste finnas. Om en kontroll fallerar markeras jobbet som misslyckat och systemstatus visar en varning. Ingenting rapporteras som "inga ändringar".
- Jobb är återupptagbara. Varje källa som behandlats markeras i jobbets förlopp.

## Artighet

- User-Agent: `ByggplanerTrelleborg/<version> (privat; kontakt: <e-post från konfiguration>)`.
- En förfrågan i taget, minst 2 sekunders paus (konfigurerbart).
- Vid 429 eller 503: respektera `Retry-After`, annars backa och avbryt efter upprepade fel.
- Tidsgräns per förfrågan: 30 sekunder.

## Manuell nedladdning av dokument

Jobbet `hamta_dokument` laddar ner en enskild fil som användaren valt. Krav:

- URL:en måste tillhöra en tillåten domän (`www.trelleborg.se`). Andra domäner avvisas.
- Innehållstypen ska vara PDF, och storleken får inte överstiga gränsen (standard 50 MB).
- Filen sparas under sin hash: `<datakatalog>/dokument/<hash[0:2]>/<hash>.pdf`. Samma fil sparas bara en gång.
- Om filen vid en senare nedladdning har annan hash än tidigare skapas en ny version och en händelse.
