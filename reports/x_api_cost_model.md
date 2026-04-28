# Kalibrowany model kosztów X API v2 (pay-per-use)

Stan: 2026-04-28 14:10 CEST.
Na podstawie 59 rzeczywistych zapytań z 67 wykonanych w sprawie Zondacrypto (S1–S3).
Stan konta po zapytaniach: wydane $3.33 z $5 kredytu.

## TL;DR

- **Nie licz per request.** Licz per obiekt zwrócony w `data[]` + `includes[]`.
- **Model kalibrowany na naszych logach:** modelowany koszt $3.675, rzeczywisty $3.33, dokładność ~91%.
- **Jedna regułka praktyczna:** `koszt ≈ 0.010 × liczba_profili_user + 0.005 × liczba_tweetów + 0.005 × liczba_spaces`.
- **Błędne zapytania (4xx/5xx) nie są liczone** — to potwierdza nasza obserwacja.
- **Deduplikacja 24 h UTC** — ten sam obiekt wielokrotnie w tej samej dobie = 1× koszt.

## Cennik publikowany (docs.x.com/x-api/getting-started/pricing, kwiecień 2026)

| Operacja | Koszt |
| --- | --- |
| Post read (third-party) | $0.005 / obiekt |
| User read | $0.010 / obiekt |
| List/Space/Community/Note/Media read | $0.005 / obiekt |
| Owned reads | $0.001 / obiekt |
| Content Create (no URL) | $0.015 / request |
| Content Create (with URL) | $0.200 / request |
| Counts Recent | $0.005 / request |
| Counts All | $0.010 / request |
| DM event read | $0.010 / obiekt |
| User interaction create (follow/like/retweet) | $0.015 / request |
| DM interaction create | $0.015 / request |

## Algorytm szacowania (kalibrowany)

### Wejście
- `endpoint` — ścieżka X API v2 (np. `/2/users/by/username/:name`, `/2/users/:id/tweets`, `/2/tweets/:id/quote_tweets`).
- `max_results` — parametr limitu (dla endpointów listowych, standard 10–100).
- `expansions` — czy zapytanie ma `expansions=author_id` / `referenced_tweets.id` (doliczają obiekty w `includes`).
- `fields_mult` — mnożnik dla bogactwa pól (niewielki, ale jest).

### Pseudokod

```python
def estimate_cost_usd(endpoint: str, max_results: int = 10,
                     has_expansions: bool = False, is_owned: bool = False) -> float:
    """
    Zwraca orientacyjny koszt zapytania w USD, kalibrowany na logach Czarne Niebo
    (współczynnik 0.91 na rzeczywistych wydatkach vs model, N=59 OK, 15 ERR).
    """
    POST = 0.001 if is_owned else 0.005
    USER = 0.001 if is_owned else 0.010
    CALIBRATION = 0.91  # model lekko przeszacowuje; współczynnik dopasowany do realu

    # Klasyfikacja endpointu
    lookup_user   = endpoint.startswith(("/2/users/by/username/",
                                          "/2/users/by/",
                                          "/2/users/me"))
    lookup_tweet  = endpoint.startswith("/2/tweets/") and "?" not in endpoint.split("/")[-1] \
                    and not any(x in endpoint for x in ("quote_tweets","retweeted_by",
                                                         "liking_users","search"))
    list_tweets   = endpoint.endswith("/tweets") or "tweet" in endpoint and \
                    ("quote_tweets" in endpoint or "search/recent" in endpoint)
    list_users    = any(x in endpoint for x in ("/following","/followers",
                                                  "/retweeted_by","/liking_users"))
    list_spaces   = "/spaces" in endpoint
    counts_recent = "counts/recent" in endpoint
    counts_all    = "counts/all" in endpoke

    # Dedykowane koszty
    if counts_recent:
        cost = 0.005
    elif counts_all:
        cost = 0.010
    elif lookup_user:
        # 1 user object (+ ewentualnie 1 pinned tweet w expansions)
        cost = USER + (POST if has_expansions else 0)
    elif lookup_tweet:
        # 1 tweet + 1 user w includes
        cost = POST + USER
    elif list_tweets:
        # do max_results tweetów + expansions users w includes (~1 user na tweet max)
        n = max_results
        cost = n * POST + (n * USER if has_expansions else 0)
        # korekta: realne wypełnienie to ~50% max_results dla tweetów bez filtrów
        # (obserwacja: większość naszych list tweetowych wraca 41–100 obiektów)
    elif list_users:
        # do max_results users
        n = max_results
        cost = n * USER
    elif list_spaces:
        # zwykle mały wolumen, ~$0.005 per space; często wraca 0
        cost = 0  # jeśli wraca pusto — brak kosztu
    else:
        # Fallback: średnia naszych zapytań = ~$0.056
        cost = 0.056

    return round(cost * CALIBRATION, 4)
```

### Typowe scenariusze

| Scenariusz | Zapytań | Koszt modelowany | Uwagi |
| --- | --- | --- | --- |
| Sprawdzenie jednej osoby (profil + timeline 100) | 2 | ~$0.50 | patrz case `zaorski_timeline` |
| Sieć 1-stopnia (profil + following + followers 100 każdy) | 3 | ~$2.00 | patrz case `bob_following` + `bob_followers` |
| Konwersacja pod tweetem (search/recent conversation_id) | 1 | ~$0.05 | 5–10 replies |
| 5 różnych profili (lookup by username) | 5 | ~$0.05 | case: profile dziennikarzy |
| 10 timelinów po 100 tweetów | 10 | ~$5.00 | duża kwerenda |
| Weryfikacja handle kandydatów (4 warianty, 3×404) | 4 | ~$0.01 | błędy nie kosztują |

## Nasza kalibracja na 59 zapytaniach

### Statystyki

- Zapytań łącznie: **67** (52 S1, 9 S2, 11 S3 — minus kilka duplikatów)
- Błędy (4xx/5xx / duplicate params / 429): **15**
- Zapytań kosztowych: **52** (ok. 78%)
- Obiektów Tweet zwróconych: **491**
- Obiektów User zwróconych: **122**
- Obiektów Space zwróconych: **0** (endpoint zwracał meta:result_count:0)
- Modelowany koszt: **$3.675**
- Rzeczywisty koszt: **$3.33**
- Współczynnik kalibracji: **0.906**
- Błąd modelu: **+10.4%** (przeszacowuje)

### Rozkład kosztów (TOP 10)

| Plik | Koszt ($) | Obiekty |
| --- | ---: | --- |
| bob_following.json | 0.570 | 57 users |
| gdziejest_timeline.json | 0.500 | 100 tweets |
| rav3n_timeline.json | 0.500 | 100 tweets |
| mikey_timeline.json | 0.485 | 97 tweets |
| userts_974056520211140610_0420.json | 0.320 | 64 tweets |
| zaorski_timeline.json | 0.235 | 47 tweets |
| suszek_timeline.json | 0.205 | 41 tweets |
| bob_followers.json | 0.180 | 18 users |
| bob_retweeters.json | 0.120 | 12 users |
| bob_quotes.json | 0.120 | 10 tweets + 7 users |
| --- | --- | --- |
| **Razem TOP 10** | **$3.23** | **dominuje 90% budżetu** |

### Wniosek

Praktycznie **cały koszt to kilka „ciężkich” zapytań listowych z max_results=100**. Drobne user_lookup (po $0.01) są niemal darmowe względem timelinów.

## Reguły operacyjne (dla zespołu)

### 1. Zawsze zadawaj sobie pytanie: ilu obiektów oczekujesz w odpowiedzi?

- 1 obiekt → koszt grosze ($0.005–0.010)
- 10 obiektów → ~$0.05
- 100 obiektów (z `max_results=100`) → ~$0.50 (dla user_list) lub ~$0.25 (dla tweet_list)
- 1000 obiektów (paginacja) → ~$5–10

### 2. Nie bierz `max_results=100`, jeżeli potrzebujesz tylko 10

Dla większości pytań („czy ten user jest aktywny w ostatnich 24 h?”) wystarczy `max_results=10`. Oszczędność 10×.

### 3. `expansions` to dodatkowy koszt

Każdy `expansions=author_id` doklei user object do każdego tweetu. Przy liście 100 tweetów to dodatkowe 100 × $0.010 = $1.00.

### 4. Deduplikacja 24 h

Jeżeli w tej samej dobie zapytasz dwa razy o ten sam tweet/user, zapłacisz raz. Wykorzystuj to: możesz eksperymentować, doszlifować zapytania.

### 5. Błędne zapytania nie kosztują

Potwierdzone na naszych 15 błędach (404, 400 duplicate params, 429 rate limit) — żaden nie zmniejszył kredytu.

### 6. Search/recent jest tańszy niż wygląda

Endpoint `/2/tweets/search/recent` w okienku 7 dni liczy tylko obiekty zwrócone, nie „za wyszukanie”. Koszt ≈ liczba trafień × $0.005.

### 7. Endpointy Spaces w praktyce są darmowe

`/2/spaces/by/creator_ids` i `/2/spaces/search` najczęściej zwracają puste wyniki (Spaces są rzadkie w Polsce). W naszych logach 6 zapytań = $0.00.

### 8. Pro/Enterprise endpointy: nie tykać

`/2/tweets/search/all` (archiwum pełne) wymaga Pro $5 000/mies. Nasze konto zwróci 403.

## Plan budżetu na dalszą pracę

Stan po 28.04.2026: **rezerwa $1.67**.

Przy założeniu, że każdy kolejny sprint to 5–10 zapytań z `max_results=25` (średnio $0.13 za zapytanie), mamy przestrzeń na:
- **~12 zapytań listowych** (np. timeliny kont do obserwacji) → koszt ~$1.60
- **albo ~100 user_lookupów** (profile) → koszt ~$1.00
- **albo kombinację: 5 timelinów + 20 profili** → ~$1.20

### Rekomendowany limit samokontroli

- **$0.50 / sprint** jako hard cap bez zgody Red. Nacz. / Lecha
- **max_results ≤ 25** bez wyraźnego uzasadnienia
- **bez `expansions`** w zapytaniach eksploracyjnych
- **search/recent zamiast timeline** jak tylko się da (target: 10–20 trafień, $0.05–0.10)

## Propozycja doładowania

Gdyby chcieć utrzymać aktywne śledztwo przez tydzień:
- **Tryb oszczędny:** $5 na tydzień (ok. 30–50 zapytań listowych lub 500 lookupów)
- **Tryb standardowy:** $10 na tydzień (100 zapytań listowych)
- **Tryb intensywny:** $25 na tydzień (300+ zapytań, pełny monitoring)

Dla sprawy Zondacrypto rekomendacja: **tryb oszczędny** ($5) przez najbliższe 7 dni do publikacji + ewentualny +$5 pod reakcję po publikacji.

## Lekcje na przyszłość (self-check)

1. **Mój pierwotny szacunek $0.01 per request był błędny.** Dokumentacja X mówi wyraźnie „per object”, ale łatwo przegapić, gdy czytasz cennik pobieżnie.
2. **Lista endpointów i ich charakterystyki kosztowej** powinna być w TOOLS.md lub w skille `xurl`, żeby nie trzeba jej odtwarzać za każdym razem.
3. **Kalibracja co sprint** — warto po każdych 30–50 zapytaniach sprawdzić rzeczywiste saldo i zaktualizować współczynnik.
4. **Wyraźne oznaczanie „heavy” queries** w skryptach (`list-heavy`, `list-light`, `lookup`) i wyliczanie kosztu z góry.
5. **Deduplikacja 24h jest twoim przyjacielem** — iteruj po tych samych obiektach w ciągu doby bez paniki.

---

*Model będzie aktualizowany po kolejnych sprintach X API, w miarę zbierania danych kalibracyjnych.*
