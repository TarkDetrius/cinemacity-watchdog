# cinemacity-watchdog

Hlídá rozpis [Cinema City](https://www.cinemacity.cz) a když přibude nový termín
**Odyssei v IMAXu**, založí v tomhle repu issue a **přiřadí ho vlastníkovi repa**.
GitHub z něj pošle e-mail i push do mobilní appky.

Na přiřazení záleží: e-mail chodí ve výchozím nastavení jen u „Participating"
notifikací (přiřazení, zmínky, odpovědi). Pouhé sledování repa („Watching")
dává jen web/mobile notifikaci — e-mail je pro něj v Settings → Notifications
vypnutý, dokud si ho člověk nezapne.

Běží v GitHub Actions, takže funguje i když je Mac vypnutý.

## Jak to funguje

- Workflow [`.github/workflows/watch.yml`](.github/workflows/watch.yml) běží
  **každou půlhodinu** (v :13 a :43 — mimo špičky, kdy GitHub cron nejvíc
  zahazuje běhy). Repo je veřejné, takže minuty Actions jsou zdarma bez limitu.
- [`watch.py`](watch.py) stáhne rozpis z veřejného JSON API cinemacity.cz
  (`/cz/data-api-service/v1/quickbook/10101/…`) — bez klíče, bez přihlášení.
- Seznam už viděných představení drží v [`state/seen.json`](state/seen.json),
  který si workflow po každém běhu commitne zpátky. Hlásí se tedy jen přírůstky.
- Nová představení → issue s časem, sálem, příznaky (70mm / titulky / vyprodáno)
  a přímým odkazem na nákup vstupenky. Hlásí se i termíny, které z rozpisu
  **zmizely** (zrušené projekce).
- Issue se **hned po založení zavírá**. Slouží jen jako doručovací kanál pro
  e-mail, který GitHub pošle už při jeho vzniku — seznam otevřených issues tak
  zůstává prázdný a nic není potřeba uklízet ručně. Obsah zůstává čitelný mezi
  zavřenými.
- Časy se počítají v zóně kina (`Europe/Prague`), ne v UTC runneru. Bez toho
  by projekce, která právě doběhla, vypadala jako budoucí a při zmizení
  z rozpisu by se falešně nahlásila jako zrušená.

Jeden běh je ~45 HTTP dotazů a trvá ~20 sekund.

## Co přesně se hlídá

Představení, kde **název filmu** obsahuje `odyss` **a** **název sálu** obsahuje
`imax`. Aktuálně tomu odpovídá jediné kino v ČR — **Praha Flora**, sál
`IMAX VOLVO`, kde Odyssea běží v 70mm s titulky.

Aby se netahal celý rozpis všech třinácti kin, hledá se dvoufázově: nejdřív se
zjistí, která kina vůbec mají IMAX sál (jedna sonda na nejbližší hrací den plus
nápověda z API přes atribut `70-mm`), a do hloubky se projdou jen ta. Kdyby
IMAX přibyl v jiném kině, chytí se to samo.

Chování jde změnit proměnnými prostředí ve workflow:

| Proměnná | Výchozí | Význam |
| --- | --- | --- |
| `FILM_PATTERN` | `odyss` | podřetězec názvu filmu (case-insensitive) |
| `AUDITORIUM_PATTERN` | `imax` | podřetězec názvu sálu |
| `HORIZON_DAYS` | `180` | jak daleko dopředu se ptát |
| `HINT_ATTR` | `70-mm` | atribut pro levné dohledání kandidátských kin |
| `REQUEST_DELAY` | `0.25` | pauza mezi dotazy na API (s) |

Hlídat cokoli jiného (třeba `FILM_PATTERN=dune`, `AUDITORIUM_PATTERN=4dx`) tedy
znamená přepsat dvě proměnné a smazat `state/seen.json`.

## Chci to hlídat taky (fork)

Watchdog nepotřebuje žádné tokeny ani secrets — API Cinema City je veřejné
a na zakládání issues stačí vestavěný `GITHUB_TOKEN`. Rozjedeš ho takhle:

1. **Forkni** si tohle repo.
2. **Settings → General → Features → zaškrtni `Issues`.** Forky mají issues
   vypnuté a bez nich by watchdog neměl kudy hlásit.
3. **Actions → „I understand my workflows, go ahead and enable them".**
   GitHub v forcích naplánované workflows nespouští, dokud je nepovolíš.
4. Hotovo. Issues se zakládají a přiřazují tobě, protože workflow používá
   `${{ github.repository_owner }}` — nic přepisovat nemusíš.

Stav v `state/seen.json` se forkne s sebou, takže tě to nezasype aktuálním
rozpisem a ozve se až s prvním novým termínem. Chceš-li hned vidět, co se
hraje teď, spusť workflow ručně s `force_report`.

Hlídat jiný film než Odysseu: přepiš `FILM_PATTERN` (a případně
`AUDITORIUM_PATTERN`) ve workflow a smaž obsah `state/seen.json`.

## Ruční spuštění

**Actions → Cinema City watchdog → Run workflow**. Zaškrtnutí *force_report*
nahlásí všechny aktuální termíny, i ty už známé — hodí se na ověření, že to žije,
nebo jako „ukaž mi, co teď hrajou“.

```bash
gh workflow run watch.yml --repo TarkDetrius/cinemacity-watchdog -f force_report=true
```

## Lokální spuštění

Čisté Python 3, žádné závislosti:

```bash
python3 watch.py --state state/seen.json
```

Užitečné přepínače: `--seed` (jen zapíše stav, nic nehlásí — dobré po změně
filtru), `--force-report` (vypíše vše bez ohledu na stav).

## Údržba

- **Kvóta Actions:** repo je záměrně veřejné — u veřejných rep jsou minuty
  Actions zdarma bez limitu. Kdyby se překlopilo na privátní, běhy by se začaly
  počítat do free limitu 2 000 minut měsíčně a půlhodinová kadence by ho
  přečerpala; pak je potřeba zároveň zpomalit cron (např. `23 */2 * * *`).
- **60denní pauza:** GitHub automaticky vypne cron, pokud v repu 60 dní nic
  nepřibude. Tady to nehrozí — workflow si sám commituje stav.
- **Až Odyssea dohraje,** watchdog jen přestane cokoli hlásit. Buď ho vypni
  (Actions → *Disable workflow*), nebo přepiš `FILM_PATTERN` na další film.
- Kdyby Cinema City API změnilo, workflow spadne s chybou a GitHub o tom
  pošle e-mail.
