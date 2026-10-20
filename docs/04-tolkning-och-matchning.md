# Tolkning och matchning

## Ordning

1. **Regeltolkning** (gratis): filnamn, sidtitlar, mönster som "Detaljplan 262", diarienummer `PL-åååå-nr`, fastighetsbeteckningar.
2. **AI-tolkning** (kostar): fyller i det fasta schemat från sidtext eller PDF-text.
3. **Matchning**: kopplar resultatet till rätt projekt.

Regelresultat har företräde framför AI-resultat för samma uppgift.

## AI-tolkaren

### Schema

Definieras en gång med Pydantic och används för AI-anropet, valideringen och API:et. Version anges i `schemaversion`.

```python
class TolkatProjekt(BaseModel):
    namn: str
    projekttyp: Literal["detaljplan", "vagplan", "stadsutveckling", "program", "infrastruktur", "okand"]
    plannummer: str | None
    diarienummer: str | None
    fas: str | None            # måste finnas i fastabellen för projekttypen
    fas_sakerhet: Literal["saker", "osaker"]
    fas_datum: date | None
    synpunkter_senast: date | None
    ort: str | None
    stadsdel: str | None
    fastigheter: list[str]
    platsbeskrivning: str | None   # den mest sökbara platsangivelsen: gatuadress, korsning, känd plats
    sammanfattning: str            # 1–3 meningar på svenska, i egna ord
    syfte: str | None
    antal_bostader: int | None
    overordnat_projekt: str | None
    osakra_falt: list[str]
```

### Anrop

- Leverantör och modell anges i konfiguration, aldrig i koden.
- Strukturerat svar via verktygsanrop med JSON-schemat från Pydantic.
- AI:n har **inga verktyg som utför handlingar**. Den kan bara returnera data.
- Kommunens text skickas inom tydliga avgränsare och instruktionen anger att texten är data och att instruktioner i den ska ignoreras.
- Prompter ligger som versionerade filer i `backend/prompts/` (t.ex. `tolka_sida.v1.md`). Ny version = ny fil.
- Svar som inte validerar mot schemat sparas med `giltig = false` och skrivs inte till registret. En fas som inte finns för projekttypen avvisas.
- Allt AI-genererat märks med `ursprung = ai`. Fält i `osakra_falt` och `fas_sakerhet = osaker` ger `sakerhet = osaker`.
- Osäkra uppgifter publiceras direkt med markering och listas på sidan "Att granska".

### Kostnadskontroll

1. **Förhandsbesked:** när insamlingen är klar visar jobbet antal dokument och uppskattad kostnad. AI-steget körs först efter bekräftelse.
2. **Hårt tak per körning:** om faktisk kostnad når taket pausas jobbet (`pausad_kostnadstak`) och kan fortsättas efter ny bekräftelse.
3. **Månadstak:** när månadens summa når taket kan inga nya AI-jobb startas utan att taket höjs i inställningarna.
4. **Utgiftstak hos leverantören** sätts dessutom manuellt som yttersta skydd.
5. Varje anrop loggas i `ai_anrop` med tokens och kostnad.

### Kvalitetsmätning

`backend/tests/utvardering/` innehåller 5–10 sparade detaljplanssidor och dokument med facit som kontrollerats för hand. Ett kommando kör tolkaren mot samlingen och rapporterar träffsäkerhet per fält. Det ska köras före varje byte av modell, prompt eller schema. Körningen kostar pengar och startas bara manuellt.

## Matcharen

Försöker i tur och ordning:

1. **Källkoppling:** källan är redan kopplad till ett projekt.
2. **Plannummer + kommun:** t.ex. "DP 262" i Trelleborg.
3. **Diarienummer:** t.ex. `PL-2024-73`.
4. **Sparade matchningsregler** från tidigare sammanslagningar och delningar.
5. **AI-bedömning** med en lista över kandidater (namn, plats, typ). Svaret är ett befintligt projekt-ID, "nytt projekt" eller "osäker".

Steg 1–4 ger `sakerhet = saker`. Steg 5 ger `saker` bara om AI:n är säker, annars kopplas resultatet till bästa kandidat med `sakerhet = osaker` och hamnar på granskningslistan.

Kungörelser från anslagstavlan matchas på samma sätt och kan skapa fasändringar (t.ex. antagande, laga kraft).

## Tolka om

Jobbet `tolka_om` kör AI på arkivet utan att hämta från kommunen. Filter: per projekt, per källa, per dokument, eller "allt tolkat med äldre version än X". Kräver samma kostnadsbekräftelse.
