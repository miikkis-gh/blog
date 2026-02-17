---
title: Äänenkloonausputken rakentaminen Mac Minillä
description: Omia kokemuksia ja askeleita puhesynteesiputken rakentamisesta Chatterbox TTS:llä paikallisesti Apple Siliconilla.
publishDate: '2025-02-10'
published: true
lang: fi
tags:
  - voice-cloning
  - tts
  - apple-silicon
  - chatterbox
  - python
slug: aanenkloonausputki-mac-mini
translationKey: voice-cloning-pipeline
---

# Äänenkloonausputken rakentaminen Mac Minillä

**Omia kokemuksia ja askeleita puhesynteesiputken rakentamisesta Chatterbox TTS:llä paikallisesti Apple Siliconilla.**

---

Minulla oli referenssinauhoitus ja käsikirjoitus, jonka halusin generoida samalla äänellä. Työkaluksi valitsin Chatterbox TTS:n, avoimen lähdekoodin äänenkloonausmallin, joka ottaa lyhyen ääninäytteen ja tuottaa uutta puhetta samalla äänellä. Periaate on suoraviivainen: syötä WAV-tiedosto jossa joku puhuu, anna teksti, ja malli puhuu tekstin kyseisen henkilön äänellä. Käytännössä matkalta löytyi useampi vaihe kuin odotin.

## Ensimmäinen yritys

Aloitin yksinkertaisella Python-skriptillä. Lataa malli, syötä koko teksti ja referenssiäänileike, saa ääntä:

```python
from chatterbox.tts import ChatterboxTTS

model = ChatterboxTTS.from_pretrained(device="mps")
wav = model.generate(
    "Tämä on hyvin pitkä käsikirjoitus monella kappaleella...",
    audio_prompt_path="reference.wav",
)
torchaudio.save("output.wav", wav, model.sr)
```

Tulos ei ollut käyttökelpoinen. Pitkiä taukoja oudoissa paikoissa, sanoja jotka kuulostivat epäselviltä, ja malli jäi ajoittain jumiin toistamaan samaa fraasia. Itse mallissa ei ollut vikaa — Chatterbox on toimiva teknologia — mutta käytin sitä tavalla jota se ei ole suunniteltu. TTS-mallit toimivat parhaiten lyhyillä tekstisegmenteillä ja huolellisesti valmistellulla syötteellä, ei pitkillä tekstimassoilla kerralla.

## Siirtyminen palasiin perustuvaan putkeen

Seuraava askel oli käsitellä äänentuottaminen putkena. Yhden generointikutsun sijaan skripti pilkkoo tekstin pieniin palasiin, tuottaa äänen jokaiselle palalle erikseen ja liittää ne yhteen hallituilla tauoilla ja ristihäivytyksillä.

Pilkkomislogiikka jakaa tekstin lauserajoilla noudattaen enimmäismerkkimäärää per pala:

```python
def chunk_text(text: str, max_chars: int = 200) -> list[dict]:
    paragraphs = [p.strip() for p in text.strip().split("\n") if p.strip()]
    chunks = []
    for para_idx, paragraph in enumerate(paragraphs):
        sentences = re.split(r'(?<=[.!?])\s+', paragraph.strip())
        current = ""
        for sentence in sentences:
            if len(sentence) > max_chars:
                if current:
                    chunks.append(_make_chunk(current, False))
                    current = ""
                chunks.append(_make_chunk(sentence, False))
            elif len(current) + len(sentence) + 1 <= max_chars:
                current = f"{current} {sentence}".strip()
            else:
                chunks.append(_make_chunk(current, False))
                current = sentence
        if current.strip():
            is_para_break = para_idx < len(paragraphs) - 1
            chunks.append(_make_chunk(current, is_para_break))
    return chunks
```

Jokainen pala merkitään sillä, päättääkö se kappaleen, jotta kokoaja voi lisätä oikean tauon pituuden — nolla pilkku kuusi sekuntia lauseiden väliin, yksi pilkku kaksi sekuntia kappaleiden väliin:

```python
sent_gap = torch.zeros(1, int(sr * 0.6))
para_gap = torch.zeros(1, int(sr * 1.2))

parts = []
for i, (wav, info) in enumerate(results):
    parts.append(wav)
    if i < len(results) - 1:
        parts.append(para_gap if info["paragraph_break"] else sent_gap)
final = torch.cat(parts, dim=1)
```

Tämä paransi tulosta huomattavasti. Lyhyet segmentit pitävät mallin johdonmukaisempana, ja jos yksittäinen pala menee huonosti, sen voi generoida uudelleen erikseen. Koko putki pyörii M4 Pro Mac Minillä kahdellakymmenelläneljällä gigatavulla yhtenäistä muistia Applen Metal Performance Shaders -tekniikalla — GPU-palvelinta ei tarvita.

## Tekstin valmistelu

Yksi asia johon en osannut varautua oli se, kuinka paljon tekstin muotoilu vaikuttaa tulokseen. Kirjoitettu teksti ja puhuttu teksti noudattavat eri sääntöjä, ja TTS-malli kompastuu moniin kirjoituskielen konventioihin.

Tekstin esikäsittelijä hoitaa yleisimmät ongelmat automaattisesti:

```python
ABBREVIATIONS = {
    "DNS": "D.N.S.", "HTTP": "H.T.T.P.", "API": "A.P.I.",
    "SQL": "sequel", "CEO": "C.E.O.", "AI": "A.I.",
    "GPU": "G.P.U.", "CPU": "C.P.U.", "URL": "U.R.L.",
    "vs": "versus", "etc.": "etcetera",
    "e.g.": "for example", "i.e.": "that is",
    # ... yli 30 merkintää
}

def preprocess_text(text: str) -> str:
    # Normalisoi unicode-välimerkit
    text = text.replace("\u2014", " — ").replace("\u2013", " — ")
    text = text.replace("\u201c", '"').replace("\u201d", '"')

    # Laajenna tunnetut lyhenteet
    for abbr, expansion in ABBREVIATIONS.items():
        text = re.sub(r'\b' + re.escape(abbr) + r'\b', expansion, text)

    # Pienennä ISOT KIRJAIMET — Chatterbox tavaa ne kirjain kirjaimelta
    text = re.sub(r'\b[A-Z]{2,}\b',
                  lambda m: m.group(0).lower() if len(m.group(0)) > 1 else m.group(0),
                  text)

    # Muunna numerot sanoiksi: "721" → "seven hundred twenty one"
    text = re.sub(r'(?<![a-zA-Z\d_])\d[\d,.]*(?![a-zA-Z\d_])', _expand_number, text)

    # Siivoa välimerkkivirheet
    text = re.sub(r' +', ' ', text)
    text = re.sub(r'([!?.]){2,}', r'\1', text)
    return text.strip()
```

Yksi konkreettinen esimerkki: Chatterbox käsittelee kokonaan isoilla kirjaimilla kirjoitetut sanat lyhenteenä ja tavaa ne kirjain kirjaimelta. "BOOM" muuttuu muotoon "B-O-O-M". Ratkaisu on suoraviivainen — pienennä kaikki ja anna lauserakenteen hoitaa painotus.

Numerot saavat oman muuntimen joka käsittelee kokonaisluvut miljardeihin asti ja desimaaliluvut:

```python
def _int_to_words(n: int) -> str:
    if n == 0: return "zero"
    ones = ["", "one", "two", "three", "four", "five", ...]
    tens = ["", "", "twenty", "thirty", "forty", ...]
    parts = []
    for val, name in [(1_000_000_000, "billion"), (1_000_000, "million"),
                      (1_000, "thousand")]:
        if n >= val:
            parts.append(f"{_int_to_words(n // val)} {name}")
            n %= val
    # ... käsittelee loppuosan
    return " ".join(parts)
```

## Kappalerakenne ja rytmitys

Skripti käyttää kappaleiden välisiä tyhjiä rivejä pidempien taukojen lisäämiseen ja käsittelee kaiken kappaleen sisällä jatkuvana puheena. Käytännössä tämä tarkoittaa, että kappalerakenne ohjaa suoraan lopullisen äänen rytmiä.

Näin syöteteksti vastaa tuotettua ääntä:

```
Tämä on ensimmäinen ajatus. Se jatkuu luonnollisesti.    ← 0.6s tauko lauseiden välissä
Nämä virtaavat yhteen yhtenä puhuttuna kokonaisuutena.

                                                          ← 1.2s tauko (kappaleen vaihto)
Tämä on uusi aihe. Kuuntelija kuulee tauon tässä.
```

Lyhyet yhden tai kahden lauseen kappaleet korostavat yksittäistä ajatusta taukojen avulla. Pidemmät kappaleet muodostavat yhtenäisen jakson. Näiden vuorottelu tuo vaihtelua lopputulokseen.

## Sävyntunnistus ja referenssivastaavuus

Lisäsin putkeen sävynluokittelun, joka yrittää yhdistää jokaisen palan sopivaan referenssileikkeeseen:

```python
def _detect_tone(text: str) -> str:
    text_lower = text.lower()
    questions = text.count("?")
    exclaims = text.count("!")
    word_count = len(text.split())

    if questions >= 2 or (questions == 1 and word_count < 15):
        return "questioning"
    if exclaims >= 1:
        return "emphatic"
    if any(w in text_lower for w in [
        "incredible", "fantastic", "tremendous", "amazing",
        "best", "greatest", "nobody", "everyone", "absolutely",
    ]):
        return "emphatic"
    if any(w in text_lower for w in [
        "disaster", "terrible", "worst", "mess", "broken",
        "ridiculous", "sad", "never", "nothing",
    ]):
        return "dramatic"
    if any(w in text_lower for w in [
        "look,", "so,", "here's the deal", "you know",
        "basically", "honestly", "frankly", "think about",
    ]):
        return "conversational"
    return "neutral"
```

Referenssileikkeet luokitellaan energiatason mukaan RMS:n ja nollaläpikäyntien avulla:

```python
def classify_clip_energy(path: str) -> str:
    wav, sr = torchaudio.load(path)
    wav = wav.mean(dim=0)
    rms = wav.pow(2).mean().sqrt().item()
    zcr = ((wav[:-1] * wav[1:]) < 0).float().mean().item()
    if rms > 0.06 and zcr > 0.15:
        return "emphatic"
    elif rms > 0.04:
        return "conversational"
    return "neutral"
```

Ajatuksena on, että painokas teksti saa korkean energian referenssileikkeen ja rauhallinen selitys hiljaisemman. Käytännössä tämä tuo jonkin verran vaihtelua lopputulokseen verrattuna siihen, että käyttäisi samaa referenssiä kaikille palasille.

## Useiden ottojen pisteytys

TTS-generoinnissa on aina vaihtelua — sama syöte tuottaa hieman erilaisen tuloksen joka kerta. Tämän vuoksi putki generoi jokaisen palan useaan kertaan ja valitsee parhaan moniulotteisen pisteytyksen perusteella:

```python
def score_generation(wav: torch.Tensor, sr: int, text: str) -> dict:
    mono = wav.squeeze()
    duration = mono.shape[0] / sr

    # Keston uskottavuus: puhe ≈ 12-18 merkkiä/s
    expected = len(text) / 14.0
    ratio = duration / max(expected, 0.1)
    dur_score = (1.0 - abs(1.0 - ratio) * 0.5) if 0.5 <= ratio <= 2.0 else 0.1

    # Hiljaisuussuhto — merkitse vain todellinen hiljaisuus, ei hengitystauot
    thresh = 10 ** (-55.0 / 20)
    sil_ratio = (mono.abs() < thresh).float().mean().item()
    sil_score = max(0.0, 1.0 - max(0.0, sil_ratio - 0.15) * 4.0)

    # Leikkautumisen tunnistus
    clip_ratio = (mono.abs() > 0.98).float().mean().item()
    clip_score = max(0.0, 1.0 - clip_ratio * 100.0)

    # Energian johdonmukaisuus 100ms kehyksissä
    frame_sz = int(sr * 0.1)
    n_frames = mono.shape[0] // frame_sz
    if n_frames > 4:
        energies = torch.tensor([
            mono[f * frame_sz:(f + 1) * frame_sz].pow(2).mean().sqrt().item()
            for f in range(n_frames)
        ])
        cv = (energies.std() / max(energies.mean(), 1e-8)).item()
        cons_score = max(0.0, 1.0 - max(0.0, cv - 0.8) * 0.8)
    else:
        cons_score = 0.7

    # Energiatasapaino: ensimmäinen vs toinen puolisko (toiston tunnistus)
    half = mono.shape[0] // 2
    balance = (mono[half:].pow(2).mean().sqrt() /
               max(mono[:half].pow(2).mean().sqrt(), 1e-8)).item()
    rep_score = max(0.0, 1.0 - abs(1.0 - balance) * 0.8)

    composite = (dur_score * 0.30 + sil_score * 0.20 + clip_score * 0.15 +
                 cons_score * 0.20 + rep_score * 0.15)

    return {"composite": composite, "duration": dur_score, "silence": sil_score,
            "clipping": clip_score, "consistency": cons_score, "repetition": rep_score,
            "audio_duration": duration}
```

Generointisilmukka ajaa N ottoa per pala ja valitsee korkeimman kokonaispisteen:

```python
candidates = []
for t in range(takes):
    wav = model.generate(txt, audio_prompt_path=ref,
                         exaggeration=0.7, cfg_weight=0.2)
    wav = trim_silence(wav, sr)
    wav = apply_fade(wav, sr)
    sc = score_generation(wav, sr, txt)
    candidates.append((wav, sc))

candidates.sort(key=lambda x: x[1]["composite"], reverse=True)
best_wav, best_sc = candidates[0]
```

Tämä oli yksittäisistä parannuksista vaikuttavin. Se ei poista huonoja generointeja, mutta antaa mahdollisuuden valita paremman vaihtoehdon automaattisesti.

## Ristihäivytyskokoaminen

Korkeimmassa laatutilassa palat yhdistetään pehmeillä ristihäivytysliitoksilla kovien leikkausten sijaan:

```python
def crossfade_concat(a: torch.Tensor, b: torch.Tensor, sr: int,
                     ms: int = 80) -> torch.Tensor:
    n = min(int(sr * ms / 1000), a.shape[-1], b.shape[-1])
    if n < 10:
        return torch.cat([a, b], dim=-1)
    fade_out = torch.linspace(1, 0, n, device=a.device)
    fade_in = torch.linspace(0, 1, n, device=b.device)
    overlap = a[..., -n:] * fade_out + b[..., :n] * fade_in
    return torch.cat([a[..., :-n], overlap, b[..., n:]], dim=-1)
```

Kuudenkymmenen kahdeksankymmenen millisekunnin limitys poistaa napsahdukset palojen väliltä, jotka kuuluvat pelkällä peräkkäinliittämisellä.

## Referenssiäänen poiminta

Pitkien referenssitiedostojen kohdalla skripti analysoi energiatasot koko nauhoituksen läpi ja poimii segmentit joissa on eniten puhe-energiaa, riittävin välein äänellisen vaihtelun varmistamiseksi:

```python
def find_best_clips(path: str, num_clips: int = 5,
                    clip_dur: float = 10.0, min_gap: float = 5.0):
    energies = analyze_audio_energy(path, window_sec=0.5)
    total_dur = torchaudio.info(path).num_frames / torchaudio.info(path).sample_rate

    # Pisteytä jokainen mahdollinen leikkeen paikka kokonaisenergian mukaan
    win = int(clip_dur / 0.5)
    scored = []
    for i in range(len(energies) - win):
        t = energies[i][0]
        e = sum(x[1] for x in energies[i:i + win])
        # Rankaise leikkeitä joissa on täysi hiljaisuus
        if min(x[1] for x in energies[i:i + win]) < 0.001:
            e *= 0.5
        scored.append((t, e))

    # Valitse parhaat leikkeet minimivälillä toisistaan
    scored.sort(key=lambda x: x[1], reverse=True)
    selected = []
    for t, e in scored:
        if len(selected) >= num_clips:
            break
        if all(abs(t - s) >= min_gap for s in selected):
            selected.append(t)
    return sorted(selected)
```

Jokainen poimittu leike äänenvoimakkuusnormalisoidaan miinus kuuteentoista LUFS:iin ennen kuin malli näkee sen:

```python
def normalize_reference(wav: torch.Tensor, sr: int,
                        target_lufs: float = -16.0) -> torch.Tensor:
    rms = wav.pow(2).mean().sqrt()
    gain_db = target_lufs - (20 * torch.log10(rms)).item()
    gain_db = max(min(gain_db, 30.0), -20.0)  # Rajoita kohinan vahvistuksen estämiseksi
    wav = wav * 10 ** (gain_db / 20)
    peak = wav.abs().max()
    if peak > 0.99:
        wav = wav * (0.99 / peak)  # Kovaleikkaussuoja
    return wav
```

## Analyysiraportti ja pisteytyksen kalibrointi

Jokainen ajo tuottaa JSON-raportin palakohtaisella analyysillä. Tämä osoittautui käytännössä hyödyllisimmäksi osaksi koko putkea, koska sen avulla näkee mitkä palat ovat heikkoja ja miksi:

```python
chunk_report = {
    "chunk": i + 1,
    "text": txt,
    "chars": len(txt),
    "tone": chunk["tone"],
    "reference_clip": Path(ref).name,
    "takes": [],        # ottokohtaiset pisteet ja ajoitus
    "best_score": None,  # voittaneen oton täysi pisteerittely
    "issues": [],        # automaattisesti havaitut ongelmat
    "generation_time_s": 0,
}
```

Ongelmien tunnistus merkitsee havaitut ongelmat kynnysarvoilla:

```python
if best_sc["composite"] < 0.55:
    chunk_report["issues"].append("low_quality_score")
if best_sc["silence"] < 0.3:
    chunk_report["issues"].append("excessive_silence")
if best_sc["repetition"] < 0.4:
    chunk_report["issues"].append("possible_repetition")
if best_sc["consistency"] < 0.3:
    chunk_report["issues"].append("energy_inconsistency")
if best_sc["audio_duration"] > len(txt) / 6.0:
    chunk_report["issues"].append("unusually_long_audio")
```

Kynnysarvojen säätäminen vaati muutaman iteraation. Ensimmäisessä versiossa hiljaisuuskynnys oli miinus neljässäkymmenessä desibelissä, mikä havaitsi luonnolliset hengitystauot "hiljaisuutena" ja merkitsi joka ikisen palan. Nostin kynnyksen miinus viiteenkymmeneen viiteen desibeliin ja sallin viidentoista prosentin hiljaisuuden ennen rankaisua. Tämän jälkeen pisteytys alkoi heijastaa todellista laatua paremmin.

Raportti auttoi myös löytämään todellisia sisältöongelmia. Eräs pala sai alhaiset pisteet koska teksti sisälsi lainatun fraasin sisäkkäisten lainausmerkkien sisällä — juuri sellaista josta TTS-malli menee sekaisin. Sen yksinkertaistaminen korjasi ongelman seuraavalla ajolla.

## Käyttö

Pipeline tukee kolmea laatuasetusta:

```bash
# Nopea esikatselu — yksi otto per pala, ei ristihäivytystä
python clone_voice.py -i script.txt --quality draft

# Tasapainoinen — kaksi ottoa, valitse paras (oletus)
python clone_voice.py -i script.txt --quality good --resume -v

# Maksimilaatu — kolme ottoa, ristihäivytyskokoaminen
python clone_voice.py -i script.txt --quality best --resume -v

# Automaattinen leikkeiden poiminta pitkästä referenssitiedostosta
python clone_voice.py -i script.txt -r interview.wav --num-clips 7 --clip-duration 10

# Esikatsele palojen jakautumista ennen generointia
python clone_voice.py -i script.txt -v --quality draft
```

## Lopputulokset

Useiden iteraatioiden jälkeen — tekstin siivoamista, pisteytyksen kalibrointia ja referenssiäänen optimointia — putki tuottaa tulosta jossa suurin osa palasista on laadultaan hyvää tai erinomaista.

| Mittari | Ensimmäinen ajo | Viimeinen ajo |
|---|---|---|
| Keskimääräinen laatu | 0.685 | 0.945 |
| Minimilaatu | 0.635 | 0.742 |
| Merkityt palat | 30/30 | 0/30 |
| Erinomainen (>0.80) | 0 | 29 |

Kaikki pyörii Mac Minillä paikallisesti. Neljän ja puolen minuutin käsikirjoituksen generointi kestää noin kaksikymmentäkaksi minuuttia oletuslaatutilassa.

## Täydellinen toteutus

Kaikki yllä kuvatut komponentit on yhdistetty yhdeksi komentorivityökaluksi. Skripti tarjoaa kolme laatuesiasetusta, automaattisen referenssileikkeiden poimimisen, välimuistin keskeytyneille ajoille ja JSON-raportoinnin.

**Lataa**: [clone_voice.py](clone_voice.py)

### Käyttö

```bash
# Peruskäyttö oletusasetuksilla
python clone_voice.py

# Käsittele tekstitiedosto maksimilaadulla
python clone_voice.py -i script.txt --quality best

# Poimii automaattisesti referenssileikkeet pitkästä äänitiedostosta
python clone_voice.py -r interview.wav --num-clips 7

# Käytä useita referenssileikkeitä
python clone_voice.py -r clip1.wav clip2.wav clip3.wav

# Jatka keskeytettyä generointia
python clone_voice.py -i script.txt --resume --quality best

# Esikatsele tekstin pilkkomista ja analyysiä
python clone_voice.py -i script.txt -v --quality draft
```

### Laatuesiasutukset

| Esiasetus | Ottoja/pala | Ristihäivytys | Käyttötarkoitus |
|-----------|-------------|---------------|-----------------|
| draft     | 1           | Ei            | Nopea esikatselu |
| good      | 2           | Ei            | Tasapainoinen (oletus) |
| best      | 3           | Kyllä         | Maksimilaatu |

### Keskeiset ominaisuudet

**Automaattinen referenssikäsittely**: Pitkät referenssitiedostot analysoidaan energiajakauman osalta ja pilkotaan kymmenen sekunnin leikkeiksi. Järjestelmä poimii segmentit joissa on korkea puhe-energia välttäen hiljaisia osia.

**Sävyyn perustuva valinta**: Jokainen tekstipala analysoidaan tunnepitoisuuden osalta (neutraali, keskusteleva, painokas, dramaattinen, kysyvä) ja yhdistetään sopiviin referenssileikkeisiin energiaominaisuuksien perusteella.

**Paras N:stä -generointi**: Jokaisesta palasta tuotetaan useita generointeja ja korkein pistemäärä valitaan automaattisesti. Pisteet ottavat huomioon keston uskottavuuden, hiljaisuussuhteen, leikkautumisen, energian johdonmukaisuuden ja toiston tunnistuksen.

**Jatkamisominaisuus**: Palat tallennetaan välimuistiin sisältöhajautuksen perusteella, mikä mahdollistaa keskeytettyjen ajojen jatkamisen ilman valmiiden segmenttien uudelleengenerointia.

**Raportointi**: JSON-raportit sisältävät palakohtaiset pisteet, ajoitustiedot, havaitut ongelmat ja suositukset laadun parantamiseksi.

### Tuotoksen analyysi

Työkalu generoi JSON-raportin ääniulostuloon rinnalle:

```json
{
  "summary": {
    "total_chunks": 30,
    "successful_chunks": 30,
    "quality_scores": {
      "avg": 0.945,
      "min": 0.742,
      "distribution": {
        "excellent_above_0.80": 29,
        "good_0.70_to_0.79": 1
      }
    }
  },
  "flagged_summary": [],
  "recommendations": [
    "No issues detected. Output quality looks good."
  ]
}
```

### Parametrien säätö

**Exaggeration** (0.0–1.0, oletus 0.7): Ohjaa ääniluonteen vastaavuutta. Korkeammat arvot kaappaavat enemmän persoonallisuutta referenssiäänestä mutta voivat tuoda artefakteja.

**CFG Weight** (0.0–1.0, oletus 0.2): Luokittelijattoman ohjauksen vahvuus. Korkeammat arvot lisäävät referenssiäänen ominaisuuksien noudattamista.

**Max Characters** (oletus 200): Maksimipalan koko. Lyhyemmät palat parantavat johdonmukaisuutta mutta lisäävät käsittelyaikaa ja voivat kuulostaa katkeilevalta.

**Speed** (oletus 1.0): Toistonopeuden säätö sovelletaan generoinnin jälkeen. Arvot välillä 0.9–1.1 kuulostavat luonnollisilta.

Täydellinen skripti on saatavilla itsenäisenä Python-tiedostona joka vaatii vain `torch`, `torchaudio` ja Chatterbox TTS -paketin.

## Yhteenveto

Tämä projekti opetti minulle muutaman asian. Malli itsessään on vain yksi osa kokonaisuutta — tekstin valmistelu ja referenssiäänen laatu vaikuttavat lopputulokseen vähintään yhtä paljon. Automaattinen pisteytys teki prosessista hallittavan, koska sen avulla näkee suoraan mitkä palat tarvitsevat huomiota. Ja palasiin perustuva lähestymistapa muutti ongelman kertaluonteisesta arvailusta toistettavaksi prosessiksi.

Ero ensimmäisen toimivan version ja lopullisen tuloksen välillä oli iso, mutta jokainen yksittäinen parannus oli pieni ja konkreettinen. Kynnysarvon muutos, tekstin muotoilusääntö, referenssileikkeen valintalogiikka. Mitään yksittäistä temppua ei ollut — vain useampi pieni askel oikeaan suuntaan.
