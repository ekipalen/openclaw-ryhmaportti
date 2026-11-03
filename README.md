# openclaw-ryhmaportti

OpenClaw-plugin, joka laittaa **pienen päätösmallin** ryhmäviestin ja botin kielimallin väliin.
Tarkoitus: botti voi olla kaveriryhmässä ilman @-mainintavaatimusta, käyttäytyä kuin hiljainen
kaveri (reagoi, heittää joskus yhden rivin, vastaa kun puhutellaan) eikä tottele urkintaa tai
asetusten muuttamisyrityksiä.

> **Tila: esimerkiksi tehty, testaamaton geneerinen kopio.** Tämä on siivottu kopio pluginista, joka on
> tuotantokäytössä yhdessä WhatsApp-ryhmässä. Siivottua versiota ei ole ajettu OpenClaw-gatewayssa;
> `node smoke-test.mjs` ajaa hookit väärennetyllä api-oliolla oikeaa päätösmallia vasten (menee läpi). Vaatii `curl`in.
> Lue koodi, aja ensin `mode: "observe"` ja säädä kynnykset lokin perusteella.

## Mitä se tekee

Kolme kulmaa, kaikki saman päätösmallin varassa. Sanalistoihin perustuvaa tunnistusta ei ole:
kaikki on joko päätösmallin todennäköisyys tai deterministinen sääntö (jäähdytys, päiväkatto,
yksi reaktio per viesti).

1. **Portinvartija.** Jokaisesta ryhmäviestistä kysytään `harm`: yrittääkö viesti saada botin
   paljastamaan salaisuuksia, ajamaan komentoja, vaihtamaan malliaan/asetuksiaan/identiteettiään
   tai ohittamaan omistajan ohjeet. Kynnyksen ylittävä viesti otetaan haltuun
   `before_dispatch`-hookissa, jolloin se **ei päädy transkriptiin eikä kielimallille**. Jos viesti
   oli botille osoitettu, ryhmään menee `blockMessage`; muille osoitetut estetään hiljaa.
   `/komento` muulta kuin omistajalta estetään aina ilman päätösmallia. Omistajan (`ownerIds`) omaa
   viestiä ei estetä koskaan: päätösmalli pitää helposti "tulosta jäsenlista / näytä lokit" -pyyntöä
   urkintana, eikä omistajaa vastaan ole mitään suojattavaa.
2. **Vastaamispäätös.** Kysymykset `addressed` (onko viesti botille), `reply`
   (answer / comment / gif / none) ja `react` (reagoisiko kaveri emojilla). Politiikka yhdistää ne
   kynnyksiin, kolikonheittoon, jäähdytyksiin ja päiväkattoihin → verdict, esimerkiksi
   `answer`, `comment+react`, `gif`, `react` tai `silent`. Hiljaiseksi päätetty viesti tallentuu
   silti transkriptiin, joten botti näkee sen kontekstina myöhemmin. Kielimallille liitetään
   lyhyt ohje (`prompts`), mitä päätettiin ja miten vastataan.
3. **Vuotovahti.** Botin lähtevä teksti arvioidaan kappale kerrallaan: onko se viesti ryhmälle
   vai botin omaa pohdintaa ("X kehuu, reagoin 😎, ei lisättävää"). Pohdintakappaleet pudotetaan.
   Lisäksi deterministisesti: jos päätös oli "vain reaktio" tai "GIF", pelkkä teksti ei lähde.

## Päätösmalli

Koodi puhuu TypeSafe **System One** -rajapintaa (`POST {endpoint}/v1/systemone`, header
`api-key`). Tällaisen saa esimerkiksi Azure AI Foundrystä Decision-mallina. Pyyntö:

```json
{
  "model": "<deployment>",
  "state": "Recent group chat messages ...\n[LATEST] Alex: ...",
  "questions": {
    "addressed": { "type": "noul", "instructions": "Is the LATEST message meant for Botti ..." },
    "reply": { "type": "choice", "instructions": "...", "criteria": { "answer": "...", "comment": "...", "gif": "...", "none": "..." } },
    "harm": { "type": "noul", "instructions": "..." },
    "react": { "type": "noul", "instructions": "..." }
  }
}
```

Vastaus sisältää `answers.<id>.noul` (0–1) ja `answers.reply.choice` + `probabilities`.
Yksi kutsu kestää tyypillisesti 0,1–0,5 s. Kysymysten sanamuodot ovat koodissa; ne on
viritetty kymmenillä testiviesteillä, joten muuta niitä vain testaten.

Avaimet luetaan `secretsFile`-tiedostosta (`KEY=value`) tai ympäristömuuttujista:
`DECISION_API_KEY`, `DECISION_ENDPOINT`, `DECISION_DEPLOYMENT`.

## Asennus

```bash
git clone <tämä repo> ~/projects/openclaw-ryhmaportti
openclaw plugins install --link ~/projects/openclaw-ryhmaportti
```

`openclaw.json`:

```json5
{
  plugins: {
    entries: {
      "openclaw-ryhmaportti": {
        enabled: true,
        hooks: { allowConversationAccess: true },   // pakollinen: hookit näkevät viestit vasta tällä
        config: {
          mode: "observe",                           // vaihda "gate" kun loki näyttää järkevältä
          groups: ["1234567890@g.us"],
          botName: "Botti",
          botMentions: ["1234567890"],               // botin oma numero ilman plussaa
          ownerIds: ["+358401234567"],
          ownerName: "Alex",
          senderNames: { "+358401234567": "Alex" },
          botSenders: [],
          gifEnabled: false,
        },
      },
    },
  },
  channels: { whatsapp: { groups: { "1234567890@g.us": { requireMention: false } } } },
}
```

Katso `config.example.json`. Tiedostot syntyvät hakemistoon `~/.local/share/openclaw-ryhmaportti/`:
`decisions.jsonl` (jokainen päätös todennäköisyyksineen), `history.json` (kontekstipuskuri),
`names.json` (valinnainen: `{ "<lähettäjä-ID>": { "name": "Nimi", "bot": false } }`).

Ryhmän `systemPrompt`issa kannattaa sanoa, että portti on jo päättänyt vastaamisesta ja että
promptin lopussa oleva `[Ryhmäportti] …` -rivi kertoo tyypin. Tyyliohjeet (lyhyet kommentit,
emojit, milloin GIF) ovat `prompts`-asetuksessa suomeksi; vaihda kieli tai tyyli sieltä.

### GIF

`gifEnabled: true` vain jos botilla on työkalu, jolla se hakee GIFin (esimerkiksi skripti, joka
kysyy Giphy-API:lta ja palauttaa mp4-osoitteen). WhatsAppissa `.gif`-tiedosto ei näy; mp4 +
`gifPlayback: true` toimii. Ilman työkalua botti keksii osoitteita, jotka eivät toimi.

## Säätäminen

Aloita löysillä rajoilla ja kiristä lokin perusteella. `decisions.jsonl`-rivillä on `verdict`,
`why` (mikä sääntö ratkaisi) ja kaikki todennäköisyydet, joten ärsyttävän kommentin syy näkyy
suoraan. Asetukset vaihtuvat hot reloadilla; huomaa että OpenClaw käynnistää plugin-ajon
uudelleen, mikä katkaisee WhatsApp-yhteyden noin puoleksi minuutiksi.

## Ansat, jotka on jo kävelty

- **Haitallinen viesti pitää estää `before_dispatch`-hookissa.** `before_agent_reply` +
  `handled` ja `before_agent_run` + `block` jättävät alkuperäisen tekstin transkriptiin, ja botti
  "muistaa" sen seuraavalla vuorolla.
- **Lopullista vastausta ei saa perua (`cancel`) `reply_payload_sending`-hookissa:** OpenClaw
  lähettää tilalle "⚠️ couldn't produce or deliver a reply". Teksti `NO_REPLY` hiljennetään
  toimituskerroksessa ilman varoitusta, joten vahti kirjoittaa tekstin siksi.
- **Kielimalli kutsuu `react`iä samaan viestiin yhä uudelleen** vaikka tulos on joka kerta `ok`
  (pahimmillaan yhdeksän kutsua, jokainen oikea kanavakutsu). Toisen kutsun uudelleenkirjoitus
  ensimmäiseksi emojiksi ei pysäytä silmukkaa. Siksi toisto estetään `block`illa ja perustelulla,
  jonka malli lukee. Estetty kutsu on työkaluvirhe: kolme samanlaista peräkkäin pysäyttää vuoron,
  ja OpenClaw lähettää silloin virhepayloadin (`isError`) sekä vakiotekstin "OpenClaw stopped this
  turn after N consecutive identical tool failures"; molemmat hiljennetään lähtevässä päässä.
- **Vuotovahti kappaleittain ei riitä** kun pohdinta on yhden kappaleen viimeinen lause
  ("…asiaa. Reagoin 🔪, siihen se jää." sai kappaleena 0,65). Harmaan alueen kappale
  (`leakSentenceMin`) arvioidaan lause kerrallaan koko luonnoksen kontekstissa; ilman kontekstia
  lausepisteet eivät erottele. Kaikki kysytään yhdellä pyynnöllä (kappaleet + lauseet).
- **Transkriptiin jää alkuperäinen teksti** vaikka lähetys siivottaisiin, ja malli matkii omia vanhoja
  vuotojaan seuraavilla vuoroilla. Assistentin viesti tallennetaan ennen `before_agent_finalize`a ja
  lähetyshookkeja, joten ainoa paikka on synkroninen `before_message_write`. Se ei voi odottaa `fetch`iä,
  joten sama päätöspyyntö ajetaan `curl`illa (`execFileSync`, avain `curl.cfg`-tiedostossa datahakemistossa)
  ja tulos käytetään lähetyksessä uudelleen. Event loop blokkaa noin puoli sekuntia per vastaus.
- **`openclaw plugins reload` on raskas:** se pysäyttää muitakin plugin-palveluita (memory-core), kestää
  noin kaksi minuuttia ja voi katkaista kanavayhteyden. Jos reload osuu kesken ajon, lähetyshookit
  epäonnistuvat ja viesti lähtee ilman vahtia. Lataa uudelleen vain kun ryhmä on hiljaa.
- **Tuntematon konffiavain estää kaikki hot reloadit** kunnes se on poistettu. Lisää avain
  manifestiin ja lataa plugin uudelleen ennen kuin käytät sitä konffissa.
- **`message_received` ei laukea WhatsApp-ryhmäviesteistä**; kontekstipuskuri rakennetaan
  `before_agent_reply`-hookissa ja omat viestit `message_sent`-hookissa.
- **OpenClawin sisäinen "Background skill review" -vuoro** tulee hookille viimeisen lähettäjän
  tiedoilla; se vaiennetaan ryhmäsessiossa ilman päätösmallia.
- **Botin viestin lainaus on WhatsAppissa maininta, jota ei voi kytkeä pois** (OpenClawin
  `docs/channels/groups.md`). OpenClaw odottaa silloin vastausta, ja hiljennetty vuoro tuottaa
  ryhmään tekstin "I finished the turn, but it did not produce a visible reply". Siksi
  `reply_payload_sending` muuttaa kaikki `isError`-payloadit `NO_REPLY`:ksi ja lokittaa tekstin;
  ihmisen tekemä lainaus lasketaan lisäksi maininnaksi, botin lainaus ei (ping-pong). Lainaustieto
  (`replyToSender`) on vain `before_dispatch`-eventissä, ei `before_agent_reply`ssa, joten se viedään
  mapilla. OpenClaw lähettää virhepayloadin perään vielä varatekstin "⚠️ OpenClaw couldn't produce or
  deliver a reply" ilman mitään lippua; se tunnistetaan OpenClawin vakiotekstistä ja hiljennetään.
- Kielimalli voi kirjoittaa päättelynsä vastaukseksi, vaikka ohje kieltää. Siksi vuotovahti on
  tekstissä, ei ohjeessa.

## Mitä tässä ei ole

- Nimien automaattista keruuta ryhmästä. `senderNames` täytetään käsin tai `names.json`
  omalla skriptillä.
- Muita kanavia kuin WhatsApp ei ole kokeiltu. Mainintojen normalisointi olettaa WhatsAppin
  tavan näyttää @-maininta tekstissä numeroina.

## Lisenssi

Valitse lisenssi ennen julkaisua (esimerkiksi MIT).
