# Byggordning

Data före utseende. Varje steg ska fungera och vara testat innan nästa påbörjas. En gren per steg.

## Steg 1: Grund

- Repostruktur enligt `docs/01-arkitektur.md`, `.gitignore`, `.env.example`, pre-commit med gitleaks och ruff.
- Docker Compose med `api` och `worker`.
- Konfigurationsläsning (`config/` + miljövariabler + `installning`), datakatalog via `BYGGPLANER_DATA_DIR`.
- Databas, SQLAlchemy-modeller och första Alembic-migreringen för hela datamodellen.
- Laddning av `config/faser.yaml` och projekttyper.
- Inloggning, sessioner, CSRF, Host-kontroll, kommando för att sätta lösenord.
- Jobbkö och worker med ett testjobb.
- `deploy/server-setup.md`, `deploy.sh`, `backup.sh`.

**Klart när:** applikationen startar lokalt och på servern, inloggning fungerar, ett testjobb kan köras från ett API-anrop, och en säkerhetskopia har återställts.

## Steg 2: Insamling utan AI

- WordPress-insamlare för sidor och anslagstavla, med sidning, `modified_after` och `modified_gmt`.
- Verifiera bevakningslistans slugs och identifiera infrastrukturprojekten.
- Råarkiv, normalisering och hash, ändringsdetektor, försvunna sidor.
- Robusthet, sanity checks, återupptagbara jobb, korrekt hantering av `senast_lyckad`.
- Jobben `hamta_nya` och `fullstandig_genomsokning`.
- Fixtures och tester.
- Enkel tillfällig vy som listar källor och deras status.

**Klart när:** en körning mot riktiga sajten hämtar alla bevakade sidor, en andra körning hittar inga ändringar, och ett simulerat felaktigt svar får jobbet att misslyckas med varning i stället för att rapportera noll ändringar.

## Steg 3: Regeltolkning

- PDF-länkar ur HTML, filnamnsmönster, dokument med läge `kand`.
- Plannummer och diarienummer ur titlar och text.
- Preliminära projekt från regler, med baslinjemarkering vid första importen.
- Fasändringar från dokumentfaser.

**Klart när:** alla pågående detaljplaner finns som projekt med plannummer och dokumentlista, utan något AI-anrop.

## Steg 4: AI-tolkning och matchning

- Pydantic-schema, promptfil v1, AI-tolkare bakom protokollet.
- Kostnadsuppskattning, bekräftelsesteg, tak per körning och månad, `ai_anrop`.
- Matcharen med alla fem stegen.
- Osäkerhetsmarkering och regler för företräde och låsning.
- Jobbet `tolka_om`.
- Testsamling för kvalitetsmätning med facit.

**Klart när:** en körning ger projekt med sammanfattning och fas, osäkra uppgifter är markerade, inget AI-anrop sker utan bekräftelse, och testsamlingen ger en rapport.

## Steg 5: Geokodning

- Geokodarkedjan, Nominatim med cache, precision, avvisning utanför kommunen.
- Avvisade platser och manuell placering via API.

**Klart när:** de flesta projekt har en plats med angiven precision, och en avvisad plats inte kommer tillbaka vid nästa körning.

## Steg 6: Gränssnitt

- Generera TypeScript-typer från OpenAPI.
- Sidhuvud med jobbförlopp och kostnadsbekräftelse.
- Startsida med karta och nyhetsflöde, projektpanel, projektsida, listvy.
- Rättning och låsning av uppgifter, "Fel plats", "Placera själv".
- Mobilanpassning, ljust och mörkt läge.

**Klart när:** allt i `docs/06-granssnitt.md` för dessa vyer fungerar på dator och mobil.

## Steg 7: Dokumenthantering

- Jobben `hamta_dokument` och `tolka_dokument`, med domänkontroll, storleksgräns och hashlagring.
- Visning av tolkade dokument på projektsidan.

**Klart när:** ett dokument kan hämtas, öppnas och tolkas från projektsidan, och resultatet syns i projektet.

## Steg 8: Administration

- Att granska, inklusive sammanslagning och delning av projekt.
- Körlogg med kostnader.
- Inställningar.
- Systemstatus med varningar, säkerhetskopior och diskutrymme.

**Klart när:** alla administrativa vyer fungerar, och en felaktig matchning kan rättas utan att data går förlorad.

## Senare (inte i första versionen)

- Bygglov och startbesked som källa (ger faserna `byggs` och `klar`).
- Lantmäteriet som geokodare och planområden som ytor (PostGIS).
- Schemalagd körning av `hamta_nya` (utan AI).
- Aviseringar vid nya händelser i bevakade områden.
- Fler kommuner.
