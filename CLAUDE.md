# Byggplaner Trelleborg

Privat webbapplikation som samlar information om pågående och planerade detaljplaner och större byggprojekt i Trelleborgs kommun och presenterar den överskådligt med karta, nyhetsflöde och projektsidor. Informationen finns på trelleborg.se men är svår att överblicka.

Applikationen körs på en egen Linux-server i hemnätverket och nås bara av ägaren via Tailscale. Koden versionshanteras i ett privat GitHub-repo. Data och hemligheter finns aldrig i repot.

## Läs detta först

All design är bestämd i förväg och beskriven i `docs/`. Läs dokumentet för det område du arbetar med innan du skriver kod. Om något i specen verkar fel eller ofullständigt: fråga, ändra inte designen på egen hand. Om ett beslut ändras ska det föras in i `docs/10-beslutslogg.md`.

| Dokument | Innehåll |
|---|---|
| `docs/01-arkitektur.md` | Moduler, dataflöde, teknikval, katalogstruktur |
| `docs/02-datamodell.md` | Tabeller, faser, ursprung och osäkerhet |
| `docs/03-insamling.md` | Hämtning från trelleborg.se, ändringsdetektion, robusthet |
| `docs/04-tolkning-och-matchning.md` | AI-tolkning, regler, matchning, kostnadskontroll |
| `docs/05-geokodning.md` | Adress till koordinat, cache, avvisade platser |
| `docs/06-granssnitt.md` | Vyer, adresser, startsida, panel, projektsida, admin |
| `docs/07-sakerhet.md` | Hemligheter, inloggning, opålitlig indata |
| `docs/08-drift-och-server.md` | Serveruppsättning, deploy, säkerhetskopiering, utvecklingsmiljö |
| `docs/09-byggordning.md` | Byggsteg med acceptanskriterier |
| `docs/10-beslutslogg.md` | Varför saker är som de är |

## Arkitektur i korthet

En pipeline av moduler som bara kommunicerar via sparad data:

Insamlare (kod) → Råarkiv → Ändringsdetektor (kod) → Regeltolkare (kod) → AI-tolkare → Matchare → Projektregister → Internt API → Webbsida

Geokodaren arbetar mot registret och fyller i platser. Allt arbete körs som jobb i en kö som en separat worker bearbetar. Webbsidan läser bara via API:et.

Teknik: Python, FastAPI, Pydantic, SQLAlchemy och Alembic, SQLite (byts senare mot PostgreSQL/PostGIS), React med TypeScript och Vite, Leaflet, Docker Compose, uv.

## Regler som alltid gäller

1. **Inga hemligheter i kod eller repo.** Nycklar läses från miljövariabler (`.env`, som aldrig checkas in). `.env.example` innehåller bara platshållare. Nycklar loggas aldrig och skickas aldrig till webbläsaren.
2. **Data ligger utanför repot**, i den datakatalog som anges av `BYGGPLANER_DATA_DIR`. Skriv aldrig databas, arkiv, dokument eller loggar i projektmappen.
3. **AI-anrop sker bara från jobb som användaren startat med en knapp.** Inget i koden får anropa AI automatiskt eller schemalagt.
4. **Allt från kommunen och från AI är opålitlig indata.** Escapa vid visning, validera mot scheman, hämta bara från tillåtna domäner. Se `docs/07-sakerhet.md`.
5. **Var en hygglig besökare.** En förfrågan i taget, paus mellan förfrågningar, identifierande User-Agent, respektera robots.txt, backa vid 429/503. Hämta aldrig automatiskt från `/app/uploads/` på trelleborg.se.
6. **Tester körs mot sparade exempelsidor** i `backend/tests/fixtures/`, aldrig mot den riktiga sajten. Tester som kräver nätverk markeras och körs bara på begäran.
7. **Varje databasändring görs som en Alembic-migrering.** Ändra aldrig schemat för hand.
8. **Tysta fel är förbjudna.** Ett oväntat svar är ett fel, inte "inga resultat". Se `docs/03-insamling.md`.
9. **Använd stabila egna ID:n (UUID).** Externa ID:n som kommunens sid-ID eller plannummer sparas som uppgifter men används aldrig som nycklar.
10. **Uppgifter som användaren har rättat är låsta** och får aldrig skrivas över av regler eller AI.
11. **Konfiguration före kod.** Bevakningslista, faser, tidsgränser, kostnadstak och AI-modell ligger i konfiguration.
12. **Datum lagras i UTC** och visas i Europe/Stockholm.

## Arbetssätt

- Bygg i den ordning som anges i `docs/09-byggordning.md`. Ett steg i taget, och varje steg ska uppfylla sina acceptanskriterier innan nästa påbörjas.
- Små, tydliga commits. En gren per byggsteg.
- Kör `ruff`, typkontroll och tester innan commit. Hookarna med gitleaks får inte kringgås.
- Lägg till eller uppdatera exempelsidor i fixtures när insamlingen ändras.
