# Arkitektur

## Moduler

Varje modul har en uppgift och kommunicerar bara via data som sparats av föregående steg. Ingen modul anropar en annan direkt.

| Modul | Typ | Uppgift | Läser | Skriver |
|---|---|---|---|---|
| Insamlare | Kod | Hämtar sidor och listor från källor | Bevakningslista | Råarkiv, källor |
| Ändringsdetektor | Kod | Markerar nytt, ändrat, oförändrat och försvunnet | Råarkiv | Källstatus |
| Regeltolkare | Kod | Läser ut dokument, fas, plannummer och diarienummer ur filnamn och sidstruktur | Råarkiv | Dokument, händelser, uppgifter |
| AI-tolkare | AI | Fyller i det fasta schemat från sidtext eller PDF | Råarkiv, dokument | Tolkningsresultat |
| Matchare | Regler + AI | Kopplar tolkningsresultat till befintligt eller nytt projekt | Tolkningsresultat, register | Projekt, händelser, uppgifter |
| Geokodare | Kod (+ AI-förslag) | Platsbeskrivning till koordinat | Projektuppgifter | Platser |
| Projektregister | Databas | Enda källan till sanning | | |
| Internt API | Kod | Läsning för webbsidan, start av jobb, rättningar | Register | Jobb, rättningar |
| Webbsida | Kod | Visar data, startar jobb | API | Inget direkt |

Ny källa i framtiden (t.ex. bygglov, Post- och Inrikes Tidningar, Trafikverket, annan kommun) = ny insamlare som lämnar data i samma format. Inget efter insamlaren ska behöva ändras.

## Gränssnitt mellan moduler

Varje insamlare implementerar samma protokoll:

```python
class Insamlare(Protocol):
    kalltyp: str  # t.ex. "wp_sida", "wp_anslag"
    def lista(self, sedan: datetime | None) -> Iterable[KallReferens]: ...
    def hamta(self, ref: KallReferens) -> RaVersion: ...
    def sanity_check(self, resultat: Sammanfattning) -> list[Varning]: ...
```

AI-tolkaren och geokodaren har motsvarande protokoll, så att implementationer kan bytas (annan AI-leverantör, Lantmäteriet som geokodare).

## Jobb

Knappar startar jobb, de utför inte arbetet. Ett jobb läggs i tabellen `jobb`, och en separat worker-process plockar jobb, uppdaterar förlopp och skriver resultat. Webbsidan visar förloppet genom att fråga API:et.

Jobbtyper:

- `hamta_nya`: ändringar sedan senast lyckade körning, med en dags överlapp.
- `fullstandig_genomsokning`: alla källor hämtas, men bara nytt och ändrat tolkas.
- `tolka_om`: AI på befintligt arkiv, filtrerat (t.ex. allt tolkat med äldre promptversion). Ingen hämtning.
- `hamta_dokument`: laddar ner ett enskilt dokument på användarens begäran.
- `tolka_dokument`: AI-tolkning av ett enskilt dokument.
- `geokoda`: geokodar projekt som saknar plats.

Jobb med AI har två faser: först insamling och kostnadsuppskattning, därefter väntar jobbet på användarens bekräftelse (`status = vantar_bekraftelse`) innan AI-steget körs. Jobb ska vara återupptagbara: varje delsteg markeras klart, så att ett avbrutet jobb kan fortsätta.

Kön ligger i databasen (inga extra tjänster). Den kan senare bytas mot en dedikerad kö bakom samma gränssnitt.

## Teknikval

| Del | Val |
|---|---|
| Backend | Python (senaste stabila), FastAPI |
| Scheman | Pydantic, samma modeller för AI-validering och API |
| Databas | SQLite i WAL-läge via SQLAlchemy 2, migreringar med Alembic |
| HTTP | httpx med explicita tidsgränser |
| HTML | selectolax eller BeautifulSoup |
| PDF | pdfplumber |
| AI | Anthropics Python-SDK bakom tolkarens protokoll; modell i konfiguration |
| Frontend | React, TypeScript, Vite; typer genereras från FastAPI:s OpenAPI med openapi-typescript |
| Karta | Leaflet med OpenStreetMap-kartor |
| Beroenden | uv (Python), npm; båda med lockfiler |
| Kvalitet | pytest, ruff, mypy eller pyright, ESLint, gitleaks som pre-commit |
| Körning | Docker Compose: `api` (serverar även byggd frontend) och `worker` |

## Katalogstruktur

```
/
├── CLAUDE.md
├── docs/
├── backend/
│   ├── app/
│   │   ├── api/            # FastAPI-routrar
│   │   ├── insamling/      # insamlare per källtyp
│   │   ├── tolkning/       # regeltolkare, AI-tolkare, matchare
│   │   ├── geokodning/
│   │   ├── jobb/           # kö och worker
│   │   ├── modeller/       # SQLAlchemy
│   │   ├── scheman/        # Pydantic
│   │   ├── sakerhet/       # inloggning, CSRF, sanering
│   │   └── konfig.py
│   ├── prompts/            # versionerade promptfiler
│   ├── migrations/         # Alembic
│   └── tests/
│       ├── fixtures/       # sparade sidor från trelleborg.se
│       └── utvardering/    # testsamling för AI-kvalitet
├── frontend/
├── config/
│   ├── bevakning.yaml
│   └── faser.yaml
├── deploy/
│   ├── docker-compose.yml
│   ├── server-setup.md
│   ├── deploy.sh
│   └── backup.sh
└── .env.example
```

## Konfiguration

Standardvärden ligger i `config/` i repot. Inställningar som ändras via inställningssidan sparas i databasen och har företräde. Hemligheter ligger bara i `.env`.
