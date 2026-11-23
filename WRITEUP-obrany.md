# SafeBox — proč rate-limit a IP-ban nestačí

Writeup k úkolu: SafeBox (KSL-2026-0417). Navazuje na původní zranitelnost
(předvídatelný access code `code = (T × 7919) mod 100000`) a analyzuje obranná
opatření, která byla na server dodatečně nasazena.

## 1. Co bylo nasazeno

Po prvních automatizovaných zápisech přibyly dvě obrany:

1. **Rate-limiting** — omezení počtu přijatých zápisů za jednotku času z jednoho
   zdroje. Projevuje se tím, že část pokusů dostane chybu místo `OK`
   (v měření ~15–20 % `ERR`).
2. **IP-ban** — po překročení prahu je zdrojová IP zablokována úplně.

Obě opatření doporučuje i závěr původního reportu (sekce 6: *"Add rate-limiting
to write operations"*). Jsou správná jako **vrstva navíc**, ale samy o sobě
zranitelnost neodstraňují. Níže je proč.

## 2. Proč to útok principiálně nezastaví

Klíčový fakt: **ani rate-limit, ani IP-ban nezmění nic na tom, že access code je
stále veřejně odvoditelný.** Tajemství tam pořád žádné není — kdokoli zná pravidlo
`(T × 7919) mod 100000` a svoje hodiny, spočítá platný kód. Obrany útok jen
**zpomalují**, neřeší jeho příčinu.

### Rate-limit = zpomalovací práh, ne zámek
Limit sníží *rychlost*, ale ne *možnost*. Útočník, kterému nejde o to být nejrychlejší,
se jednoduše drží pod prahem a zapisuje dál donekonečna. Pro soutěž „kdo víc" to mění
tempo, ne výsledek.

### IP-ban = blokování podle špatného znaku
Ban podle IP stojí na předpokladu, že **IP adresa = identita útočníka**. Tento
předpoklad je mylný:

- IP adresa není trvalá ani jedinečná pro osobu. Jedna osoba má běžně k dispozici
  více výstupních IP (jiná síť, mobilní data, atd.) a naopak za jednou veřejnou IP
  (NAT) sedí klidně celá budova.
- Proto ban podle IP buď **netrefí** (útočník se objeví z jiné adresy), nebo
  **přestřelí** a zablokuje i nevinné uživatele sdílející tutéž veřejnou IP.

To je obecný bezpečnostní princip: **blokovat podle IP znamená blokovat podle
síťové cesty, ne podle identity.** Dokud se autentizace opírá o něco, co útočník
nemá problém změnit, je obejitelná.

## 3. Co to tedy skutečně opravuje

Jediná obrana, která přežije i změnu IP a libovolné zpomalení, je **přesunout
tajemství tam, kde ho útočník nemá** — přesně jak říká sekce 6 reportu:

- **Nepředvídatelný tajný token** místo kódu odvozeného z času. Útočník ho nezná,
  ať mění IP nebo tempo jak chce.
- **Challenge–response** — server pošle při každém spojení čerstvou náhodnou výzvu,
  klient ji musí správně zodpovědět. Nelze předpočítat z hodin.
- **Autentizace zápisů** — zápis smí jen ověřený klient. IP u toho slouží nanejvýš
  k logování, ne jako jediná bariéra.

Rate-limit a IP-ban k tomu patří jako **defense in depth** (zdržení a detekce
zneužití), ale ne jako **náhrada** autentizace.

## 4. Závěr

Nasazené limity jsou správný krok, ale řeší symptom (rychlost zneužití), ne příčinu
(kód je odvoditelný bez tajemství). Dokud zůstane kód veřejně spočítatelný, bude útok
jen pomalejší, ne nemožný — proto se na rate-limit/IP-ban nedá spoléhat jako na
hlavní ochranu. Hlavní ochranou musí být nepředvídatelné tajemství ověřované
serverem.
