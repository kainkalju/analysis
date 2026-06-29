# Manus

Suurepärane! Loen kõigepealt Supadata oskuse juhised läbi, seejärel tõmban video transkripti ja koostan eestikeelse kokkuvõtte.

Supadata oskus töötas suurepäraselt! Siin on kogu töövoo kokkuvõte ja eestikeelne sisukokkuvõte.

---

## Supadata demo — töövoog

**Video:** [YouTube H-SgQP3Hif0](https://www.youtube.com/watch?v=H-SgQP3Hif0)
**Transkripti pikkus:** ~15 300 tähemärki | **Keel:** `en` (native)
**Kasutatud režiim:** `mode=native` — tõmbab olemasoleva YouTube'i subtiitrifaili, ei kuluta AI-krediite

Supadata API tagastas JSON-vastuse kujul `{ lang, availableLangs, content }`, kust eraldati puhas tekst ja edastati LLM-ile kokkuvõtte tegemiseks.

---

## Eestikeelne kokkuvõte

### 1. Video teema ja kontekst

Nightmare Eclipse, anonüümne turvauurija, avaldas avalikult mitu Microsofti zero-day haavatavust, sh peamine neist — **"Yellow Key"**. Video selgitab, kuidas Yellow Key eksploiteerib Windowsi recovery environment'i käitumist, et mööda minna BitLocker'i kaitsest, ning arutleb selle avalikustamise ja uurija GitHubist/GitLabist eemaldamise tagamaid.

### 2. Peamised punktid

| # | Punkt |
|---|-------|
| 1 | **Nightmare Eclipse** avaldas mitmeid Microsofti zero-day'sid ja eemaldati GitHubist ning hiljem ka GitLabist, mis tekitas vaidlusi *responsible disclosure* üle. |
| 2 | **Yellow Key** kasutab ära transactional NTFS-i (FSTX) funktsionaalsust — USB-mälule paigutatud `System Volume Information` kaustas olevad failisüsteemi tehingud *replay*-takse WinRE keskkonnas. |
| 3 | Ründaja asetab USB-le FSTX-transaktsiooni, mis **kustutab recovery konfiguratsioonifaili** (nt `winpel.ini`), mille tagajärjel WinRE viskab kasutaja otse käsureale, mitte tavapärasesse recovery UI-sse. |
| 4 | Kuna WinRE on osa käivitusprotsessist, on TPM juba andnud OS-ile dekrüptimisvõtme — käesolev keskkond saab kasutada BitLocker'i lahtikrüpteerimiseks vajalikke võtmeid. |
| 5 | See **ei riku otseselt BitLocker'i krüptograafiat** — probleem on WinRE käitumises (auto-FSTX käivitamine), mitte TPM-i või BitLocker'i enese disainis. |
| 6 | **Windows 11 on haavatavam kui Windows 10**, sest Win10 WinRE ei käivitanud FSTX-protsessi automaatselt; Windows 11 lubab automaatse replay, mis teeb rünnaku võimalikuks. |
| 7 | **Leevendused:** (a) eemaldada WinRE registrivõti, mis käivitab FSTX; (b) lubada **TPM+PIN** — see nõuab füüsilise ligipääsu kõrval ka PIN-koodi. |
| 8 | **Caveat:** uurija väidab hiljem, et TPM+PIN ei pruugi kaitsta, viidates täiendavale haavatavusele, mis võimaldab TPM-võtme saada ilma PIN-ita — olukord jääb ebakindlaks. |

### 3. Järeldused ja olulisus

Yellow Key näitab, kuidas näiliselt "mitte-krüptograafiline" viga (WinRE failisüsteemi tehingute käitlemine) võib viia BitLocker'i kaitse täieliku möödavõtmiseni, eriti kui süsteem kasutab ainult TPM-i ilma PIN-ita. Organisatsioonid ja kasutajad peaksid kaaluma **TPM+PIN kasutuselevõttu**, WinRE FSTX-käivituse keelamist ning füüsilise ligipääsu piiranguid, kuni Microsoft teeb ametliku paranduse. Juhtum tõstatab ka laiema küsimuse vastutustundliku avalikustamise ning platvormide (GitHub/GitLab) modereerimispoliitikate kohta.

---

> **Supadata oskuse demo tulemus:** `mode=native` transkript tõmmati edukalt ühe API-päringuga, ilma AI-genereerimiseta, ning edasine töötlus (kokkuvõte, tõlge) toimus kohalikult LLM-iga.



