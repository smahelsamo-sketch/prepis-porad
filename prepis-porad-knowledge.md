# Prepis Porád — Project Knowledge Base

## Prehľad projektu
Webová PWA aplikácia na automatický prepis hovorov a porád s rozoznávaním rečníkov. Deployed na GitHub Pages.

**URL:** `https://smahelsamo-sketch.github.io/prepis-porad/`
**GitHub:** `https://github.com/smahelsamo-sketch/prepis-porad`
**Používateľ:** `smahel.samo@gmail.com`

---

## Architektúra
- Jeden HTML súbor (`index.html`) + `manifest.json` + `sw.js` + ikony
- Žiadny backend — všetko beží v prehliadači
- AssemblyAI API na prepis + diarizáciu
- Google Drive API na ukladanie prepisov
- PWA — dá sa nainštalovať na plochu (Android Chrome, iOS Safari)

---

## API a služby

### AssemblyAI
- Endpoint upload: `https://api.assemblyai.com/v2/upload`
- Endpoint transcript: `https://api.assemblyai.com/v2/transcript`
- **KRITICKÉ:** Parameter musí byť `speech_models: ['universal-2']` (pole, nie string)
  - `speech_model` (string) — deprecated, vracia chybu
  - `"speech_models" must be a non-empty list` — táto chyba znamená zlý formát
- Polling každé 4 sekundy, max 15 minút
- Diarizácia: `speaker_labels: true`, voliteľne `speakers_expected: N`
- Jazyk: `language_code: 'sk'`

### Google Drive OAuth
- Typ: OAuth 2.0, response_type=token (implicit flow)
- Client ID uložený v `localStorage` kľúč `prepis_gclient`
- **Authorized JavaScript origins:** `https://smahelsamo-sketch.github.io`
- **Authorized redirect URIs:** `https://smahelsamo-sketch.github.io/prepis-porad/`
- Token expiruje po ~1 hodine — treba sa znova prihlásiť
- Aplikácia je v produkcii (nie test mode) — nie je potrebné pridávať test users
- **Chyba `invalid_client`** — najčastejšia príčina: WhatsApp automaticky pridáva `https://` pred Client ID pri kopírovaní
- **Chyba `access_denied`** — aplikácia bola v test mode, opravené presunom do produkcie

### Google Drive API
- Upload endpoint: `https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&convert=true`
- Ukladá ako Google Docs dokument (konvertuje z plain text)
- **Problém s priečinkami:** `mkPath()` funkcia niekedy nevie nájsť existujúci priečinok `workflow` a vytvorí nový. Príčina: Drive API vyhľadávanie je case-sensitive a môže vrátiť viac výsledkov. Zatiaľ akceptované ako known bug.
- Cieľová štruktúra: `workflow/Prepisy/YYYY-MM Mesiac/`

---

## Ukladanie dát (localStorage)
| Kľúč | Obsah |
|------|-------|
| `prepis_aai_key` | AssemblyAI API kľúč |
| `prepis_gclient` | Google OAuth Client ID |
| `prepis_folder` | Cesta k zložke na Drive |
| `install_dismissed` | Timestamp dismissnutia PWA bannera |
| `ios_hint_shown` | Flag pre iOS inštalačný hint |

---

## Nahrávanie zvuku
- Preferovaný formát: `audio/mp4` → M4A (lepšia kompatibilita, dá sa pretáčať)
- Fallback: `audio/webm;codecs=opus` → WEBM
- WEBM má problém s pretáčaním v niektorých prehrávačoch
- **KRITICKÉ:** Nahrávka sa musí stiahnuť IHNEĎ po zastavení (`mediaRec.onstop`) — ak sa stránka obnoví, `currentBlob` sa stratí z pamäte a nahrávka je nenávratne preč
- Auto-download názov: `Nahrávka YYYY-MM-DD HH-MM.m4a`

---

## Známe problémy a riešenia

### Cache problém po aktualizácii
- Po nahratí nového `index.html` na GitHub Pages prehliadač servíruje starú verziu z cache
- **Riešenie:** Pridať `?v=N` na koniec URL (napr. `?v=4`) — prinúti prehliadač načítať novú verziu
- Service worker cachuje súbory — treba inkrementovať verziu v `sw.js` pri väčších zmenách

### `allow pasting` v DevTools konzole
- Chrome na Macu niekedy blokuje vkladanie kódu do konzoly
- Treba napísať `allow pasting` manuálne (nie vložiť) a stlačiť Enter
- Ak to nefunguje ani tak — použiť `location.href=URL.createObjectURL(currentBlob)` na záchranné stiahnutie nahrávky

### Google OAuth token expiruje
- Token platí ~1 hodinu
- Pri dlhých nahrávkach (100+ minút) token vyprší počas prepisu
- **TODO:** Implementovať refresh token alebo upozorniť používateľa pred expiráciou

### Kvalita prepisu slovenčiny
- AssemblyAI má horšiu kvalitu pre slovenčinu ako pre angličtinu
- Zlepšuje to: lepší mikrofón, tichšia miestnosť, menšia vzdialenosť od mikrofónu
- Odporúčaný mikrofón: Blue Yeti (omnidirectional režim) alebo Jabra Speak 510

---

## Štruktúra súborov na GitHub
```
prepis-porad/
  index.html      ← hlavná appka
  manifest.json   ← PWA manifest
  sw.js           ← service worker
  icon-192.png    ← ikona appky
  icon-512.png    ← ikona appky
```

---

## Štruktúra Google Drive
```
workflow/
  Prepisy/
    2026-03 Marec/
      Prepis 31.3.2026
    2026-04 Apríl/
      Prepis 1.4.2026
```
Mesiac v slovenčine: Január, Február, Marec, Apríl, Máj, Jún, Júl, August, September, Október, November, December

---

## TODO / Plánované vylepšenia
- [ ] Automatické prihlásenie Google keď token expiruje
- [ ] Oprava vyhľadávania existujúceho priečinka `workflow` na Drive
- [ ] Zlepšenie kvality prepisu (možno iný model alebo preprocessing zvuku)
- [ ] Ukladanie nahrávok aj na Google Drive (voliteľné)
- [ ] Pomenovanie rečníkov sa neuloží — pri ďalšom prepise treba znova

---

## Deployment
1. Upraviť `index.html`
2. Nahrať na GitHub: `Add file → Upload files → Commit changes`
3. GitHub Pages sa aktualizuje do ~1 minúty
4. Ak prehliadač servíruje starú verziu: pridať `?v=N` na koniec URL
