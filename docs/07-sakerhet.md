# Säkerhet

## Hemligheter

- Alla hemligheter (AI-nyckel, sessionshemlighet m.m.) läses från miljövariabler via `.env`.
- `.env` listas i `.gitignore`. `.env.example` innehåller bara namn och platshållare.
- På servern ägs `.env` av tjänsteanvändaren med rättigheterna `600`.
- Utveckling och drift har **olika** AI-nycklar. Utvecklingsnyckeln har ett lågt utgiftstak hos leverantören.
- Nyckeln används bara i backend. Den skickas aldrig till webbläsaren och visas aldrig i gränssnittet.
- Loggning har ett filter som maskerar allt som liknar nycklar, och felmeddelanden innehåller aldrig miljövariabler.
- gitleaks körs som pre-commit-hook.
- Utgiftstak sätts hos AI-leverantören utöver applikationens egna tak.

## Åtkomst

- Servern nås bara via Tailscale. Ingen port öppnas i routern.
- HTTPS via Tailscale (t.ex. `tailscale serve`) så att inloggningen aldrig skickas i klartext.
- Brandväggen (ufw) släpper bara in SSH och webben från Tailscale-gränssnittet och det lokala nätet.

## Inloggning

- En användare i första versionen. Lösenord hashas med argon2.
- Sessionscookie med `HttpOnly`, `Secure` och `SameSite=Strict`.
- CSRF-token krävs för alla anrop som ändrar något eller startar jobb.
- Servern godtar bara förfrågningar där `Host` matchar konfigurerade värdnamn (skydd mot DNS rebinding och mot att andra sidor anropar applikationen).
- Begränsning av inloggningsförsök.
- Lösenordet sätts och återställs med ett kommando på servern: `docker compose run --rm api byggplaner satt-losenord`.
- Datamodellen är förberedd för fler användare.

## Opålitlig indata

Allt från kommunen, från PDF:er och från AI är opålitligt.

- **Visning:** all extern text visas som text. Om HTML någon gång behöver visas saneras den med en vitlista.
- **AI:** inga verktyg som utför handlingar. Texten avgränsas tydligt som data. Svaret valideras mot schemat innan något sparas. Värsta fallet är en felaktig uppgift, som då är märkt som AI-genererad.
- **Nedladdningar:** bara från tillåtna domäner (`www.trelleborg.se`). Följ omdirigeringar bara inom tillåtna domäner. Kontrollera innehållstyp och storlek.
- **Filer:** sparas under sin hash, aldrig under namnet från webben. Inga sökvägar byggs av extern indata.
- **PDF-tolkning** körs i workern med tidsgräns och minnesgräns.

## Beroenden

- Lockfiler för Python (uv) och npm. Exakta versioner.
- Dependabot aktiveras i GitHub-repot.
- Docker-avbilder byggs från officiella basavbilder med fast version.

## Containrar

- Applikationen körs som icke-root-användare i containrarna.
- Bara datakatalogen monteras in. Inga andra mappar på servern är åtkomliga.
