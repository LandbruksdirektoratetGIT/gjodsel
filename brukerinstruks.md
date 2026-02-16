# Brukerinstruks

Denne veilederen er for organisasjoner som skal utvikle en integrasjon mot Landbruksdirektoratets Gjødselregister.
Det antas derfor at organisasjonen har sluttbrukere som er gjødselbrukere. Med det menes for eksempel:
- Landbruksforetak
- Parkanlegg
- Golfbaner

# I dette repoet finner du:
- **/dist:** Disse filene lar deg se den genererte dokumentasjonen for APIet til Gjødselregisterets kjernesystem.
Besøk [GitHub Pages](https://landbruksdirektoratetgit.github.io/gjodsel/) for å se det i aksjon
- **core.yml:** Kjernesystemet til Gjødselregisteret. Kjernesystemet lar deg blant annet sende inn meldinger og søknader om gjødselbruk.
- **event-api.yml:** Kjernesystemet eksponerer hendelser når endringer skjer som er relevante for gjødselbrukerene det omhandler.
Dette APIet inneholder API-spesifikasjonen på en websocket som gjør det mulig å abonnere på hendelser i Gjødselregisteret.
- **geojson.yml:** Kjernesystemet bruker stedfesting aktivt i APIet. Stedfesting av tiltak i meldinger og søknader i registeret er
i henhold til RFC-7946 og skilt ut i denne filen.


# Hvordan komme i gang

## Autentisering via Maskinporten
Tjenesten benytter Maskinportens Systembruker-flyt for autentisering av organisasjonen og autorisering fra gjødselsbruker.

### Oppsett av Maskinporten-klient
Din organisasjon må signere dokumenter i [DigDirs samarbeidsportal](https://minside-samarbeid.digdir.no/my-organisation/integrations/admin) før klienten kan opprettes.

Om du ikke har tilgang til samarbeidsportalen, fyll ut [dette skjemaet](https://forms.office.com/Pages/ResponsePage.aspx?id=D1aOAK8I7EygVrNUR1A5kZbWwz0nwnRGrfJqFQYggctURTRCTFpCVklHODJYQ0NEMVNWSjNaWDVJSiQlQCN0PWcu).

Deretter må integrasjonen registreres som en klient i Maskinporten. Dette kan gjøres via [selvbetjeningsportalen i test](https://sjolvbetjening.test.samarbeid.digdir.no)
Autentiseringsmetode bør være private_key_jwt slik at fremtidige tokenforespørsler ikke sender over hemmeligheter i klartekst.

Gi klienten scopes:
- **ldir:gjodsel**: dette vil gi deg en advarsel i grensesnittet helt frem til LDIR godkjenner integrasjonen din.
- **altinn:authentication/systemregister.write**
- **altinn:authentication/systemuser.request.read**
- **altinn:authentication/systemuser.request.write**

De siste tre scopene er kun synlige og valgbare etter å ha fylt ut [dette skjemaet](https://forms.office.com/Pages/ResponsePage.aspx?id=D1aOAK8I7EygVrNUR1A5kcdP2Xp78HZOttvolvmHfSJUOFFBMThaOTI1UlVEVU9VM0FaTVZLMzg0Vi4u).
Det er kun nødvendig å krysse av på "systembruker". Behandlingstid er typisk samme eller neste dag.

Virksomhetens organisasjonsnummer må nå oversendes LDIR for godkjenning.
Denne godkjenningen gis til alle foretak med hensiktsmessige behov for å bruke gjødselregisteret i testmiljøet.

For å få tilgang til produksjon krever at blant annet følgende fremlegges: 
- Hvordan gjødselbrukere autentiseres i løsningen.
- Hvordan brukersesjoner håndteres.
- Hvordan gjødselbrukerens persondata håndteres og hvilket omfang av persondata som håndteres.

Resultatet av denne godkjenningsprosessen er tilgang til registeret og en sertifisering fra LDIR.

### Oppsett i Altinns systemregister
Maskinporten-klienten må kobles sammen med en oppføring i Altinns Systemregister.
Følg [guiden, fra og med steg 3](https://docs.altinn.studio/nb/authorization/getting-started/systemuser/).

Systemregisteret har ikke et grafisk grensesnitt for å opprette oppføringer.
Det må gjøres HTTPS-kall med client-credentials-grant tokens fra Maskinporten-klienten
som ble opprettet. Token som brukes mot systemregister-APIet må må ha scope **altinn:authentication/systemregister.write**.

Eksempel-JSON som skal oversendes systemregisteret:
```json
{
  "id": "<mitt-orgnr>_<applikasjonsnavn>",
  "vendor": {
    "authority": "iso6523-actorid-upis",
    "ID": "0192:<mitt-orgnr>"
  },
  "name": {
    "nb": "Mitt Skifteplanverktøy",
    "en": "My Parcel Planning Tool",
    "nn": "Mitt Skifteplanverktøy"
  },
  "description": {
    "nb": "En beskrivelse",
    "en": "An application description",
    "nn": "Ei beskriving"
  },
  "rights": [
    {
      "resource": [
        {
          "id": "urn:altinn:resource",
          "value": "scope-gjodsel"
        }
      ]
    }
  ],
  "accessPackages": [
  ],
  "clientId": [
    "<min-maskinporten-clientId-UUID>"
  ],
  "allowedredirecturls": [
    "https://minvirksomhet/consent/redirect"
  ],
  "isVisible": false
}
```

### Innhente samtykker fra gjødselbrukere
Samtykker fra gjødselbruker innhentes ved å opprette en systembrukerforespørsel mot Altinn.

[Beskrivelse av endepunkt, Altinn API](https://docs.altinn.studio/nb/authorization/guides/system-vendor/system-user/systemuserrequest/#1-opprette-systembruker-for-eget-system)

I test kan også syntetiske organisasjoner benyttes. Du kan se hvilke som er tilgjengelige via [Tenor testdatasøk](https://www.skatteetaten.no/testdata/)

Maskinporten-token som benyttes mot dette endepunktet må ha scope **altinn:authentication/systemuser.request.read** og
**altinn:authentication/systemuser.request.write**

Eksempelforespørsel mot Altinn:
```json
{
  "systemId": "<mitt-orgnr>_<applikasjonsnavn>",
  "partyOrgNo": "<gjødselbrukerens-orgnr>",
  "rights": [
    {
      "resource": [
        {
          "id": "urn:altinn:resource",
          "value": "scope-gjodsel"
        }
      ]
    }
  ],
  "accessPackages": [
    {
      "urn": "urn:altinn:accesspackage:jordbruk"
    }
  ],
  "redirectUrl": "https://minvirksomhet/consent/redirect"
}
```
APIet returnerer et JSON-dokument med feltet "confirmUrl". Gjødselbrukeren må navigeres til denne URLen i en nettleser.

Du kan følge med på status på samtykket via [et endepunkt](https://docs.altinn.studio/nb/api/authentication/systemuserapi/systemuserrequest/external/#hent-en-systembruker-foresporsel-med-eksterne-referanse).
Eventuelt kan du bruke brukerens ankomst på redirectURIen din som trigger på at samtykke er gitt eller avslått.

Samtykke må innhentes for samtlige virksomheter integrasjonen din skal handle på vegne av.

### Hente Maskinporten-token som representerer gjødselbruker
Når samtykke er gitt, kan integrasjonen hente access tokens fra Maskinporten som kan utføre operasjoner på vegne av
gjødselbrukeren.

Token-request følger formatet beskrevet [her](https://docs.digdir.no/docs/Maskinporten/maskinporten_func_systembruker.html#foresp%C3%B8rsel).
I denne sammenhengen er integrasjonen "leverandør" og gjødselbrukeren "systembruker". Husk å be om scope **ldir:gjodsel**.

Ved positiv respons fra Maskinporten har du nå et access token som kan hentes og brukes mot Gjødselregisteret 
uavhengig av gjødselbrukerens aktive innvirkning frem til gjødselbrukeren trekker sitt samtykke.

## Autentisering (LEGACY)
APIet kan inntil videre aksesseres med selvgenererte JWT Bearer token for testformål.
Denne støtten vil opphøre i løpet av første halvdel av 2026.

Tokens ser slik ut i sin mest grunnleggende form:

Header:
```json
{
  "alg": "HS256"
}
```
Claims:
```json
{
  "sub": "0192:<din org-ID>",
  "iss": "https://example.com",
  "iat": 0,
  "exp": 9999999999,
  "consumer": {
    "ID": "0192:<gjødselbrukerens org-ID"
  }
}
```
JSON-dokumentene base64-enkodes og punktum-konkateneres i henhold til [RFC-7519](https://datatracker.ietf.org/doc/html/rfc7519)
Deretter må JWTen signeres i henhold til [RFC-7515](https://datatracker.ietf.org/doc/html/rfc7515).
Det anbefales at et verktøy eller bibliotek benyttes til dette.
Signeringsnøkkel for testmiljøet oppgis ved forespørsel til gjodsel@landbruksdirektoratet.no

## Første kall

Du kan bruke API-spesifikasjonene til å generere en API-klient ved hjelp av et
[egnet bibliotek](https://github.com/OpenAPITools/openapi-generator).

Merk imidlertid at event-api.yml er en asyncapi-spec og trenger derfor en [annen generator](https://github.com/asyncapi/generator)

Samtlige forespørsler til APIet må inneholde en token-header:
```http request
Authorization: Bearer ey.....
```

Dette gjelder også event-APIet, som må motta token-header på det initielle ws-upgrade kallet.

Gjør gjerne en cURL for å sjekke om APIet godtar tokenet ditt:
```bash
curl https://gjoedsel-fagsystem-utv.non-prod.landbruksdirektoratet.no/api/gjoedseltyper -H 'Authorization: Bearer ey...'
```

# Typisk bruksmønster

Per 19-11-2025 er flyten for innsending av melding om bruk av renseanleggslam til gjødslingsformål den funksjonaliteten
som er mest stabil i APIet. Om du velger å sette i gang med en integrasjon anbefales det at du begynner her.

Anbefalt sekvens for å sende inn en slik melding er som følger:
- Opprett en websocket til event-APIet. På denne måten får du fortløpende hendelser tilbake når endringer skjer på
innsendte meldinger og søknader.
- **GET /gjoedseltyper:** De fleste meldinger og søknader krever at det indikeres hvilken type gjødsel det meldes/søkes
  om å bruke. Dette endepunktet inneholder listen over kjente verdier.
- **POST /stedfester:** De fleste meldinger og søknader om tiltak krever at tiltaket stedfestes. Send derfor inn en 
eller flere geografiske polygoner på forhånd. Disse kan (og bør) gjenbrukes til flere meldinger og søknader i fremtiden.
Endepunktet returnerer IDer på de nyopprettede stedfestene du trenger senere.
- **POST /meldinger:** Nå kan melding om slam sendes inn. Husk å inkludere IDen til stedfestingen du sendte inn tidligere.
Eksempelverdier ligger i spesifikasjonen (MeldingForespoerselDto og SlamMeldingDto)
- Du skal nå motta en melding fra Event-APIet av typen MeldingEndretHendelse med status "INNSENDT" og endring "OPPRETTET".
Event-APIet forventer at IDen på denne meldingen sendes tilbake på socketen med en melding av typen Erkjennelse. 
Om dette ikke gjøres vil hendelsen periodevis bli sendt på nytt.

# Bruk av testdata
I dette prosjektet benyttes kun syntetisk testdata. Reelle eller personidentifiserbare data skal ikke sendes inn til testmiljøet av personvernhensyn. Syntetisk testdata kan genereres med verktøy som Tenor.
