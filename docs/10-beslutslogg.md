# Beslutslogg

Varje beslut: vad, varför och vilka alternativ som valdes bort. Nya beslut läggs till sist. Ändrade beslut markeras som ersatta med hänvisning.

**B01 Pipeline med isolerade moduler.** Modulerna kommunicerar bara via sparad data. Gör att en källa kan bytas eller läggas till utan att resten påverkas. Bortvalt: en sammanhållen "agent" som gör allt.

**B02 AI bara där omdöme krävs.** Insamling, ändringsdetektion och filnamnstolkning är kod. Billigare, snabbare och förutsägbart. Bortvalt: AI-agenter för hämtning.

**B03 AI-anrop bara via knapp.** Kostnadskontroll. Förhandsbesked och tak per körning och månad.

**B04 WordPress REST API som primär källa.** Ger ändringsdatum och ren text med få förfrågningar. Bortvalt: genomsökning av hela sajten (belastning, risk för spärr) och sitemap (ofullständig).

**B05 Råarkivet är facit.** Allt efterföljande kan återskapas. Möjliggör omtolkning och skyddar mot att kommunen tar bort sidor.

**B06 Ändringsdatum styr hämtning, hash styr AI.** Ändringsdatum kan ändras utan innehållsändring.

**B07 PDF:er hämtas bara manuellt.** `robots.txt` förbjuder automatisk hämtning från `/app/uploads/`. Användarstyrd nedladdning av enstaka filer motsvarar vanlig webbläsaranvändning.

**B08 Fas härleds från händelser.** Tidslinje och fas kan inte säga emot varandra, och ingen historik går förlorad.

**B09 Ursprung och säkerhet på varje uppgift.** Användare > regel > AI. Rättade uppgifter låses. Osäkra AI-uppgifter publiceras med markering i stället för att vänta på godkännande.

**B10 Faser per projekttyp i konfiguration.** Olika projekttyper har olika processer. Bortvalt: fast lista i koden.

**B11 Försvunnen sida betyder inte laga kraft.** Det finns ingen avdelning för antagna planer. Laga kraft kräver en matchad kungörelse.

**B12 Avvisade platser raderas inte.** Annars ger en ny fråga samma felaktiga svar.

**B13 Precision på varje koordinat.** Geokodning kan ge en ungefärlig träff som ser exakt ut.

**B14 Kommun som fält från början.** Billigt nu, dyrt senare.

**B15 Egna UUID som nycklar.** Externa ID:n kan ändras eller kollidera mellan kommuner.

**B16 SQLite med Alembic, GeoJSON för geometri.** Enkelt att börja med, flyttbart till PostgreSQL/PostGIS.

**B17 Knappar startar jobb i en kö.** Långa körningar avbryts inte, och schemaläggning blir senare bara ett annat sätt att lägga jobb.

**B18 Webbsidan läser bara via internt API.** Möjliggör andra klienter senare. TypeScript-typer genereras från API:et.

**B19 Python, FastAPI, Pydantic; React, TypeScript, Leaflet.** Bäst stöd för insamling, PDF, AI och geodata. Ett schema styr AI-validering, API och frontendtyper.

**B20 Docker Compose med separat worker.** Samma miljö på utvecklingsdator och server, enkel flytt, minsta behörighet.

**B21 Tailscale för åtkomst, inloggning från första versionen.** Servern nås från andra enheter i nätverket. Bortvalt: öppen port i routern.

**B22 GitHub för kod, servern hämtar själv.** Ingen inkommande anslutning behövs. Deploy-nyckel med bara läsrätt.

**B23 Separat utvecklingsmiljö med fixtures.** Skonar kommunens server och förhindrar att tester påverkar riktig data.

**B24 Tysta fel förbjudna, sanity checks efter varje körning.** Ett trasigt svar får aldrig se ut som "inga ändringar".

**B25 Tre körtyper: hämta nya, fullständig genomsökning, tolka om.** Ersätter tidigare idé om "Hämta allt" som skulle skicka allt till AI.

**B26 Startsida med karta och nyheter sida vid sida, panel och projektsida.** Tre nivåer: överblick, snabb titt, fördjupning. Listvy med sökning som komplement.

**B27 Första importen är baslinje.** Annars fylls nyhetsflödet med hundratals händelser.

**B28 Endast detaljplaner och större projekt i första versionen.** Bygglov läggs till senare som ny insamlare.
