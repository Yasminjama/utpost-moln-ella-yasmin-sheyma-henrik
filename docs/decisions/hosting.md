# Beslutsdokument: hostingval

*Ett av teamets sex beslutsdokument. Skrivs i M6, hålls levande. Samma mall som `api-stil.md`, `databas.md` och `testing.md`. Det här dokumentet examinerar Cloud 1 (redogöra för hostinglösningar som AWS, Azure, Google Cloud) – skriv så att en läsare utan Azure-konto förstår valet.*

**Datum:**
**Beslut:** *(en eller två meningar: var kör Utpost i staging, var ska den köra i produktion, och vad är skillnaden?)*

## Bakgrund
*Vad har vi när M5 är klar: två images i GHCR taggade med sha, en pipeline som bygger dem, compose lokalt. Vad saknas för att någon utanför teamet ska kunna öppna appen? Vilka krav ställer kursen (M6: automatisk deploy till staging vid merge, secrets ur koden, publik URL)?*

## Vad vi faktiskt hyr
| Del | Tjänst | Nivå | Vem sköter uppdateringar/skalning | Vad det kostar oss i staging |
|---|---|---|---|---|
| Klient + API | Azure Container Apps (Consumption), en app med fyra containrar (client, api, postgres, mongo) | PaaS/serverless – vi hyr en plattform för containrar, inte servrar | Microsoft sköter servrar, nätverk och skalning (0–1 replika, skalar till noll). Vi sköter imagerna och konfigurationen. | 0 kr rapporterat hittills. Kostnadsdata kommer med ca 8-24h fördröjning och appen skalar till 0. Förbrukning på 7 dagar: 38 anrop, replika 0 mellan passen och som mest 2 vid uppdatering - appen skalar om till 0. |
| Postgres | `postgres:16` som sidecar i samma app, ingen volym | Ingen tjänst – vår egen container | Vi. Ingen backup, och datan försvinner vid en ny revision (nyseedad databas). | Ingår i raden ovan |
| MongoDB | `mongo:8.2` som sidecar i samma app, ingen volym | Ingen tjänst – vår egen container | Vi. Ingen backup, samma som Postgres. | Ingår i raden ovan |
| Images | GHCR (GitHub Container Registry) | SaaS | GitHub sköter lagringen. Vi bygger och taggar images (sha) i pipelinen. | 0 kr – publikt paket (gratis att lagra och hämta). |
| Deploy | GitHub Actions → `az containerapp update`, inloggning med OIDC (inga lösenord) | SaaS | GitHub kör jobbet, vi skriver det. | 0 kr – gratis för publika repon. Deploy-jobbet tog 91 s. |


## Alternativ vi jämförde
*Minst tre riktiga alternativ, och minst två leverantörer. Förslag: Azure Container Apps (det vi gjorde) · Azure App Service for Containers · Google Cloud Run · AWS ECS på Fargate (App Runner tar inte längre nya kunder) · en VM (IaaS) med docker compose · Render/Railway (PaaS utanför de tre stora). För varje: vad hade ändrats i repot, vad hade kostat, vad hade vi fått/förlorat?*

| Alternativ | Vad ändras i repot | Kostnad (staging) | För | Emot |
|---|---|---|---|---|
| | | | | |

## Staging och produktion är inte samma sak
*I staging kör databaserna som sidecars utan volym och seedas vid varje kallstart. Vad krävs för produktion? Managed databas (vilken, vad kostar den), backup, en app som inte skalar till noll, egen domän och certifikat, loggar. Skriv vad ni skulle byta och vad som INTE behöver bytas (imagen? pipelinen?).*

## Inlåsning
Om Azure försvann i morgon ändras bara det som är specifikt för Azure. Koden och imagerna vet inte vilket moln de kör i.

**Bara för Azure (ändras vid flytt):**
- `infra/azure/staging.yaml` – beskrivningen av Container App med de fyra containrarna
- `infra/azure/skapa-staging.sh` – skapar miljön och appen med `az`
- `infra/azure/identitet.sh` – OIDC-identiteten och rolltilldelningen i Azure
- jobbet *Deploy → Azure staging* i `.github/workflows/ci.yml` – `az containerapp update` och inloggningen

**Likadant i alla moln (ändras inte):**
- `api/Dockerfile` och `client/Dockerfile` – imagerna byggs på samma sätt oavsett plattform
- `client/nginx.conf.template` – nginx-mallen, som får sin uppströmsadress av `API_UPSTREAM`
- `.env.example` – listan över konfiguration som utifrån
- `api/src/` – applikationskoden
- `compose.yaml` – den lokala miljön
- resten av pipelinen (Kvalitet, Bygg, Integration, Image)

**Värden som skiljer mellan lokalt och Azure:**

| Värde | Lokalt (compose) | Azure |
|---|---|---|
| `API_UPSTREAM` | `api:4000` | `localhost:4000` (sidecars delar nätverk) |
| `DATABASE_URL` | `…@postgres:5432` | `…@localhost:5432` |
| `MONGO_URL` | `…@mongo:27017` | `…@localhost:27017` |
| `JWT_SECRET` | fast utvecklingsvärde i filen | slumpad secret i plattformen, aldrig i git |
| `SEED_ON_START` | `1` (harmlöst lokalt) | `1` i staging, seedar en tom databas vid kallstart |

Det som låser är alltså fyra Azure-filer och några konfigurationsvärden. Vid en flytt till Cloud Run eller ECS byter vi dem och behåller imagerna, koden och resten av pipelinen.


## Konsekvenser
**Kallstarten.** Appen skalar till noll mellan passen för att inte kosta något när ingen använder den. Priset är att första anropet efter en paus väcker den från noll: fyra containrar ska starta och databaserna seedas om, så första svaret tar cirka en minut och kan ge `502` under tiden. Det är okej för staging, men inte för produktion, där minst en replika ska vara igång. Eftersom databaserna saknar volym är en kallstart också en nyseedad databas – data vi skapat i staging försvinner.


**Skrivet av:**
