# Drift och server

## Översikt

| Miljö | Maskin | Data | AI-nyckel | Hämtar från kommunen |
|---|---|---|---|---|
| Utveckling | Ägarens dator med Claude Code | Testdatabas och fixtures | Utvecklingsnyckel med lågt tak | Nej, bara fixtures (nätverkstester på begäran) |
| Drift | Separat Linux-server | Riktig datakatalog | Produktionsnyckel | Ja |

Flöde: dator → push till privat GitHub-repo → servern hämtar med deploy-skriptet.

## GitHub

- Privat repo. Tvåstegsverifiering på kontot.
- Dependabot aktiverat.
- Servern har en **deploy-nyckel** med bara läsrätt till detta repo.
- I repot: kod, migreringar, konfigurationsstandarder, `.env.example`, dokumentation.
- Aldrig i repot: `.env`, databas, råarkiv, dokument, loggar, säkerhetskopior.

## Serveruppsättning (checklista)

`deploy/server-setup.md` ska innehålla stegen i detalj, och så mycket som möjligt automatiseras i ett skript.

1. Installera Ubuntu Server LTS eller Debian stable, minimal installation utan skrivbord. SSD rekommenderas.
2. Skapa administratörsanvändare. SSH med nyckel; lösenordsinloggning och root-inloggning via SSH avstängda.
3. Aktivera automatiska säkerhetsuppdateringar (`unattended-upgrades`).
4. Brandvägg (ufw): neka allt inkommande, tillåt SSH och webb från Tailscale och lokalt nät.
5. Installera Tailscale och anslut servern. Konfigurera HTTPS.
6. Installera Docker och Docker Compose-pluginet.
7. Skapa tjänsteanvändaren `byggplaner` utan administratörsrättigheter.
8. Skapa datakatalogen `/srv/byggplaner/data` med underkataloger `db`, `raarkiv`, `dokument`, `loggar` och ägs av tjänsteanvändaren.
9. Lägg in deploy-nyckeln och klona repot till `/srv/byggplaner/app`.
10. Skapa `/srv/byggplaner/app/.env` från `.env.example` med rättigheterna `600`.
11. Starta tjänsterna och sätt lösenord.
12. Schemalägg säkerhetskopiering och kör en provåterställning.

## Containrar

`deploy/docker-compose.yml`:

- `api`: FastAPI som även serverar den byggda frontend. Lyssnar bara lokalt; Tailscale står framför.
- `worker`: bearbetar jobbkön.
- Båda monterar datakatalogen och läser `.env`.
- `restart: unless-stopped` så att tjänsterna startar efter omstart och krascher.

## Deploy

`deploy/deploy.sh`:

1. Säkerhetskopiera databasen.
2. `git pull` från main.
3. Bygg avbilderna.
4. Kör Alembic-migreringar.
5. Starta om tjänsterna.
6. Kontrollera att hälsokontrollen svarar. Annars: visa hur man går tillbaka till föregående version.

## Säkerhetskopiering

`deploy/backup.sh`, schemalagd dagligen:

- Databasen kopieras med SQLites backup-funktion (inte genom att kopiera filen medan den används).
- Råarkiv och dokument kopieras inkrementellt.
- Kopian ska lämna maskinen (extern disk, ägarens dator eller annan plats). Verktyg som restic rekommenderas, med kryptering.
- Råarkivet är det enda som inte går att återskapa, eftersom kommunen tar bort sidor.
- **Provåterställning** minst var tredje månad och direkt efter uppsättningen. Datum för senaste provåterställning visas på systemstatussidan.

## Loggar

- Strukturerade loggar i datakatalogen, roterade efter storlek och ålder.
- Nycklar maskeras.

## Utvecklingsmiljö

- Samma Docker Compose-upplägg med en egen datakatalog.
- Fixtures: sparade API-svar och HTML-sidor från trelleborg.se i `backend/tests/fixtures/`. Ett hjälpkommando sparar nya fixtures från den riktiga sajten på begäran, med samma artighetsregler.
- Tester mot den riktiga sajten markeras och körs bara manuellt.
