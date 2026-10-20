# Datamodell

Alla primärnycklar är UUID. Alla tidsstämplar lagras i UTC. Geometri lagras som GeoJSON så att databasen senare kan bytas mot PostGIS utan konvertering.

## Grundprinciper

1. **Projektet är navet.** Allt annat hänger på projekt.
2. **Fas lagras inte på projektet.** Aktuell fas härleds från den senaste fasändringen bland projektets händelser. Fas och tidslinje kan därför aldrig säga emot varandra.
3. **Varje uppgift har ett ursprung och en säkerhet.** Ursprung: `regel`, `ai`, `anvandare`. Säkerhet: `saker`, `osaker`.
4. **Rangordning vid konflikt:** användare (låst) > regel > AI. En låst uppgift skrivs aldrig över.
5. **Inget raderas som har betydelse.** Avvisade platser, försvunna källor och sammanslagna projekt markeras med status.

## Tabeller

### kommun
`id`, `namn`, `kod`, `bbox` (GeoJSON, används för att avvisa geokodningsträffar utanför kommunen).

### projekttyp
`id`, `kod` (`detaljplan`, `vagplan`, `stadsutveckling`, `program`, `infrastruktur`; senare `bygglov`), `namn`, `har_faser` (bool).

### fas
`id`, `projekttyp_id`, `kod`, `namn`, `ordning`, `fasgrupp`.
Laddas från `config/faser.yaml`. Se avsnittet om faser nedan.

### projekt
`id`, `kommun_id`, `projekttyp_id`, `foralder_id` (nullable, överordnat projekt), `status` (`aktiv`, `sammanslagen`, `borttagen`), `sammanslagen_till_id` (nullable), `geometri` (nullable, GeoJSON för framtida planområde), `skapad`, `uppdaterad`.

Själva sakuppgifterna (namn, plannummer osv.) ligger i `projektuppgift` så att varje uppgift har ursprung och historik. För snabb läsning får en cachekolumn `sammanfattning_json` finnas, som byggs om från `projektuppgift`.

### projektuppgift
`id`, `projekt_id`, `falt`, `varde` (JSON), `ursprung`, `sakerhet`, `last` (bool), `aktuell` (bool), `kalla_id` (nullable), `tolkning_id` (nullable), `skapad`.

Fält i första versionen: `namn`, `plannummer`, `diarienummer`, `ort`, `stadsdel`, `fastigheter` (lista), `sammanfattning`, `syfte`, `antal_bostader` (nullable), `platsbeskrivning`, `synpunkter_senast` (datum).

Ny uppgift för samma fält: den gamla får `aktuell = false`. Regler och AI får inte skapa en ny aktuell uppgift om den aktuella är `last`.

### handelse
`id`, `projekt_id`, `typ`, `datum` (händelsens datum enligt källan), `fas_id` (nullable, sätts vid fasändring), `beskrivning`, `kalla_id`, `dokument_id` (nullable), `ursprung`, `sakerhet`, `last`, `baslinje` (bool), `skapad`.

Typer: `upptackt`, `fasandring`, `kungorelse`, `nytt_dokument`, `lamnade_listan`, `atervande_till_listan`, `uppgift_andrad`, `manuell_anteckning`.

`baslinje = true` för händelser som skapas vid den allra första importen. De visas i tidslinjen men inte i nyhetsflödet.

### kalla
`id`, `kommun_id`, `kalltyp` (`wp_sida`, `wp_anslag`, senare `wp_bygglov` m.fl.), `extern_id` (t.ex. WordPress-ID), `url`, `bevakningsgrupp`, `status` (`aktiv`, `forsvunnen`), `senast_sedd`, `senast_andrad_hos_kalla` (UTC), `aktuell_hash`.

### projekt_kalla
`projekt_id`, `kalla_id`, `roll` (`huvudsida`, `kungorelse`, `relaterad`), `ursprung`, `sakerhet`.

### raversion
`id`, `kalla_id`, `hamtad`, `hash` (SHA-256 på normaliserad text), `innehall_sokvag` (i råarkivet), `rattext_sokvag`, `andringsstatus` (`ny`, `andrad`, `oforandrad`).
Råarkivet är facit: allt efterföljande ska kunna återskapas från det.

### dokument
`id`, `projekt_id` (nullable tills matchat), `kalla_id` (sidan där länken hittades), `url`, `filnamn`, `lanktext`, `dokumenttyp` (`planbeskrivning`, `plankarta`, `miljobedomning`, `foljebrev`, `samradsredogorelse`, `granskningsutlatande`, `utredning`, `ovrigt`), `fas_kod` (från filnamn, nullable), `datum` (från filnamn, nullable), `lage` (`kand`, `nedladdad`, `tolkad`), `hash` (vid nedladdning), `lagrad_sokvag` (hashbaserad), `storlek`, `forsta_sedd`, `senast_sedd`.

### tolkning
`id`, `typ` (`sida`, `dokument`), `raversion_id` eller `dokument_id`, `modell`, `promptversion`, `schemaversion`, `resultat` (JSON), `giltig` (bool), `valideringsfel` (nullable), `jobb_id`, `skapad`.
Gör det möjligt att köra om tolkning selektivt, t.ex. allt med äldre promptversion.

### plats
`id`, `kommun_id`, `soktext_normaliserad`, `lat`, `lon`, `precision` (`exakt`, `gata`, `ort`, `manuell`), `tjanst` (`nominatim`, `lantmateriet`, `manuell`), `ra_svar` (JSON), `skapad`.
Fungerar som cache: samma söktext frågas aldrig två gånger mot en extern tjänst.

### projekt_plats
`projekt_id`, `plats_id`, `status` (`aktiv`, `avvisad`), `ursprung`, `last`, `skapad`.
En avvisad koppling hindrar att samma plats kopplas till projektet igen.

### anvandare
`id`, `anvandarnamn`, `losenordshash` (argon2), `skapad`.

### sedd_handelse
`anvandare_id`, `handelse_id`, `sedd`.
Används för "Sedan ditt senaste besök". Kopplat till användare så att fler användare fungerar senare.

### jobb
`id`, `typ`, `status` (`koad`, `pagar`, `vantar_bekraftelse`, `klar`, `misslyckad`, `avbruten`, `pausad_kostnadstak`), `parametrar` (JSON), `forlopp` (JSON), `sammanfattning` (JSON), `uppskattad_kostnad`, `faktisk_kostnad`, `fel`, `startad_av`, `skapad`, `startad`, `klar`.

### ai_anrop
`id`, `jobb_id`, `tolkning_id`, `modell`, `tokens_in`, `tokens_ut`, `kostnad`, `tid_ms`, `skapad`.

### korningsmarkor
`kalltyp`, `bevakningsgrupp`, `senast_lyckad` (UTC).
Flyttas bara fram när ett jobb har slutförts utan fel.

### installning
`nyckel`, `varde` (JSON), `uppdaterad`.
Överskrivningar av standardvärden i `config/`. Aldrig hemligheter.

## Faser

Faser definieras per projekttyp i `config/faser.yaml` och laddas till tabellen `fas`. Varje fas hör till en fasgrupp som styr färg och filter på kartan.

| Fasgrupp | Betydelse | Standard på kartan |
|---|---|---|
| `planeras` | Arbete påbörjat, inget att tycka till om än | Visas |
| `synpunkter` | Öppet för synpunkter | Visas, framhävd |
| `beslutad` | Beslutad eller under prövning | Visas |
| `byggs` | Byggnation pågår | Visas |
| `avslutad` | Klar eller avbruten | Dold |

Detaljplan: `planuppdrag` (planeras), `samrad` (synpunkter), `granskning` (synpunkter), `antagen` (beslutad), `overklagad` (beslutad), `laga_kraft` (beslutad), `byggstart` (byggs), `under_byggnation` (byggs), `klar` (avslutad), `avbruten` (avslutad), `okand`.

Vägplan: `planeras`, `samrad`, `granskning`, `faststalld`, `byggs`, `klar`, `okand`.

Infrastruktur: `planeras`, `projektering`, `byggs`, `klar`, `okand`.

Program (t.ex. Kuststad Trelleborg): inga faser. Status visas genom delprojekten.

**Känd begränsning:** kommunens detaljplanssidor slutar vid laga kraft. Faserna i gruppen `byggs` och `klar` kommer därför oftast att sättas manuellt eller från stadsutvecklingssidor, tills bygglov och startbesked läggs till som källa.

## Sammanslagning och delning av projekt

- **Slå ihop A in i B:** alla händelser, dokument, källkopplingar, platser och uppgifter flyttas till B. Vid konflikt mellan uppgifter vinner låsta, därefter regel, därefter den nyaste. A får `status = sammanslagen` och `sammanslagen_till_id = B`. Länkar till A omdirigeras till B.
- **Dela A:** användaren väljer vilka händelser, dokument och källor som ska till ett nytt projekt. Båda projekten får en `manuell_anteckning`.
- Båda operationerna skapar matchningsregler så att matcharen inte gör om samma fel.
