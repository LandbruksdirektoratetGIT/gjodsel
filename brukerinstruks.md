# Brukerinstruks

Denne veilederen er for organisasjoner som skal utvikle en integrasjon mot Landbruksdirektoratets Gjødselregister.
Det antas derfor at organisasjonen har sluttbrukere som er gjødselbrukere. Med det menes for eksempel:
- Landbruksforetak som bruker organisk gjødsel
- Parkanlegg
- Golfbaner

## I dette repoet finner du:
- **/dist:** Disse filene lar deg se den genererte dokumentasjonen for APIet til Gjødselregisterets kjernesystem.
Besøk [GitHub Pages](https://landbruksdirektoratetgit.github.io/gjodsel/) for å se det i aksjon
- **core.yml:** Kjernesystemet til Gjødselregisteret. Kjernesystemet lar deg blant annet sende inn meldinger og søknader om gjødselbruk.
- **event-api.yml:** Kjernesystemet eksponerer hendelser når endringer skjer som er relevante for gjødselbrukerene det omhandler.
Dette APIet inneholder API-spesifikasjonen på en websocket som gjør det mulig å abonnere på hendelser i Gjødselregisteret.
- **geojson.yml:** Kjernesystemet bruker stedfesting aktivt i APIet. Stedfesting av tiltak i meldinger og søknader i registeret er
i henhold til RFC-7946 og skilt ut i denne filen.


## Hvordan komme i gang

### Autentisering
Prosjektet er i en tidlig fase og autentisering mot APIene er derfor svært forenklet.
I første omgang kan APIet i testmiljøet benyttes med selvgenererte JWT Bearer Tokens.

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
Signeringsnøkkel for testmiljøet oppgis ved forespørsel til simen.slotta@landbruksdirektoratet.no

### Første kall

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

## Typisk bruksmønster

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

## Bruk av testdata
I dette prosjektet benyttes kun syntetisk testdata. Reelle eller personidentifiserbare data skal ikke sendes inn til testmiljøet av personvernhensyn. Syntetisk testdata kan genereres med verktøy som Tenor.
