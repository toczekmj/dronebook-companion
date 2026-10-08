# DroneBook Companion: plan

Status: **plan do akceptacji** (7.10.2026). Kod jeszcze nie powstał.

Companion to mała aplikacja desktopowa (Windows, macOS, Linux), która:
1. odczytuje czas lotów z symulatorów spoza Steama (na start VelociDrone) i dopisuje go do logbooka w DroneBooku,
2. pozwala zalogować się do DroneBooka i korzystać z niego w osobnym oknie, bez otwierania przeglądarki.

Ma być prosta w utrzymaniu: **jeden kod na trzy systemy**, a sam DroneBook nie jest przepisywany, tylko osadzony.

---

## 1. Rekomendacja w skrócie

| Decyzja | Wybór | Dlaczego |
|---|---|---|
| Technologia | **Tauri 2** (Rust + React/TypeScript) | Jeden kod na 3 systemy, instalator ~5–15 MB zamiast ~100 MB w Electronie, pełny dostęp do plików z Rusta, wbudowany updater z podpisem. |
| UI companiona | React 19 + Vite + Tailwind v4 + Paraglide (EN/PL) | Ten sam stos co DroneBook, więc te same tokeny kolorów, ikony i część tłumaczeń. |
| Wspólne komponenty | **Monorepo: companion jako `apps/companion` w repo `dronebook`, wspólny pakiet `packages/ui`** | Jedna kopia tokenów i komponentów; zmiana przycisku w DroneBooku od razu dotyczy companiona. Szczegóły w §4.4. |
| Logowanie do DroneBooka | **Osadzone okno ze stroną dronebook.vercel.app** | Zero przepisywania: to ta sama aplikacja webowa, zawsze w aktualnej wersji. |
| Synchronizacja | **Parowanie urządzenia (device code, wzorowane na RFC 8628) → token companiona z wąskim zakresem** | Pilot loguje się raz, klika „Zezwól” i gotowe. Token umie tylko dopisywać sesje z symulatorów, da się go odwołać w Ustawieniach. |
| Przechowywanie tokenu | Systemowy pęk kluczy (Windows Credential Manager, macOS Keychain, Secret Service na Linuksie) | Token nigdy nie leży w pliku jawnym tekstem. |
| Dystrybucja | GitHub Releases + Tauri updater, linki na `/download` | Mechanizm już przygotowany w makiecie (`COMPANION_INSTALLERS`). |

---

## 2. Jak companion ma się synchronizować z DroneBookiem

### 2.1 Rozważone opcje

| Opcja | Jak to wygląda dla pilota | Plusy | Minusy |
|---|---|---|---|
| **A. Klucz API wklejany ręcznie** (mamy już `/api/keys`, `dbk_…`) | Ustawienia → API → „Utwórz klucz” → kopiuj → wklej w companionie | Gotowe w 90%, zero nowego kodu po stronie serwera | Kopiowanie sekretu przez schowek; obecne klucze służą do publicznego API katalogu dronów, nie do pisania w logbooku; łatwo wkleić klucz w złe miejsce |
| **B. Pełne logowanie e-mail + hasło w companionie i sesja Better Auth jako Bearer** | Formularz logowania w companionie | Znany wzorzec | Companion dostaje **pełny dostęp do konta** (sync wszystkiego, zmiana danych). Kradzież tokenu z dysku = przejęte konto. Trzeba włączyć plugin `bearer` dla całego API |
| **C. Plugin `deviceAuthorization` z Better Auth** | Companion pokazuje kod, pilot zatwierdza go na stronie | Gotowy, przetestowany przepływ (RFC 8628); jesteśmy na 1.7.7, czyli po poprawce CVE-2026-45337 (dotyczyła 1.6.0–1.6.10) | Na końcu wydaje **zwykły token sesji** (pełne konto, jak w B) i wymaga pluginu `bearer`; każde odpytanie `/device/token` trafia do Postgresa (budzi bazę) |
| **D. Własne parowanie urządzenia → token companiona o wąskim zakresie** ✅ | Companion otwiera okno DroneBooka z kodem, pilot (zalogowany) klika „Zezwól” | Jedno logowanie, nic nie trzeba kopiować; token umie **tylko** to, czego potrzebuje companion; odwołanie jednym kliknięciem; odpytywanie nie budzi bazy | ~200 linii własnego kodu na serwerze (ale na gotowych klockach: HMAC z `apikeys.ts`, magazyn `ops`, limity z `guard.ts`) |
| E. OAuth 2.1 / OIDC (DroneBook jako dostawca tożsamości) | Okno „Zaloguj przez DroneBook” | Standard „na wyrost” | Dużo konfiguracji (plugin OAuth Provider, rejestracja klientów, PKCE, redirect na `127.0.0.1` lub `dronebook://`) dla jednej aplikacji jednego wydawcy. Przerost formy |

### 2.2 Rekomendacja: D, parowanie urządzenia

Przepływ (jak przy logowaniu telewizora do Netflixa, tylko bez przepisywania kodu, bo okno otwiera się samo):

```
Companion                           DroneBook (serwer)                    Pilot
    │ POST /api/companion/pair            │                                   │
    │  {deviceName, os, appVersion}       │                                   │
    │ ──────────────────────────────────► │ zapis w ops: pair/<deviceCode>    │
    │ ◄── {deviceCode, userCode:"KQ7M-3XPA",                                  │
    │      verifyUrl, interval:5, expiresIn:600}                              │
    │                                     │                                   │
    │ otwiera okno DroneBooka:  /companion/link?code=KQ7M-3XPA                │
    │                                     │  (jeśli trzeba: zwykłe logowanie) │
    │                                     │ ◄──── „Połącz VelociDrone PC      │
    │                                     │        (Windows)? Kod KQ7M-3XPA”  │
    │                                     │        [Zezwól] [Odrzuć] ─────────│
    │                                     │ POST /api/companion/approve       │
    │                                     │ (ciasteczko sesji + userCode)     │
    │ POST /api/companion/token  (co 5 s) │                                   │
    │ ──────────────────────────────────► │                                   │
    │ ◄── {token:"dbc_…", deviceId, user:{name}}  (jednorazowo)               │
    │ zapis tokenu w pęku kluczy systemu  │                                   │
```

Szczegóły:
- **Kod pokazany w obu miejscach.** Strona zatwierdzenia wyświetla kod, nazwę urządzenia i system; companion pokazuje ten sam kod. Pilot widzi, że zatwierdza swój komputer (ochrona przed phishingiem z RFC 8628 §5.4).
- **Logowanie raz.** Okno z DroneBookiem w companionie jest jednocześnie miejscem parowania, więc pilot loguje się tylko raz, w znanym formularzu. Jest też przycisk „Otwórz w przeglądarce”, dla kogoś, kto woli swój menedżer haseł.
- **Kody:** `userCode` 8 znaków bez mylących się 0/O/1/I, ważny 10 minut, jednorazowy; `deviceCode` 32 bajty losowe. Odrzucenie lub wygaśnięcie kasuje wpis.
- **Odpytywanie nie budzi bazy:** wpisy parowania leżą w magazynie `ops` (Upstash Redis na Vercelu, Blobs na Netlify, dysk w Dockerze). Postgres jest dotykany tylko przy zatwierdzeniu i przy wysyłce sesji. To ważne przy darmowych planach.
- **Token:** format `dbc_<deviceId>_<hmac>`, ten sam schemat co obecne `dbk_` w `server/lib/apikeys.ts` (HMAC z `BETTER_AUTH_SECRET`, porównanie w stałym czasie). W rejestrze trzymamy tylko metadane urządzenia, nie sekret.
- **Zakres tokenu:** wyłącznie `/api/companion/*` (wysyłka sesji, status, konfiguracja). Nie działa na `/api/sync`, `/api/auth/*`, `/api/keys`, `/api/admin/*`. Nawet wykradziony token pozwala najwyżej dopisać czas symulatora, i to po walidacji.
- **Ważność:** bez daty wygaśnięcia, ale z „ostatnio widziany”. Urządzenie nieużywane 180 dni wygasa samo. Pilot może je odłączyć w Ustawieniach; odłączenie konta lub jego usunięcie unieważnia wszystkie tokeny (dopisać do istniejącego `databaseHooks.user.delete`).
- **Limit:** do 5 urządzeń na konto.

### 2.3 Okno z DroneBookiem a sesja companiona

To dwie osobne rzeczy i tak ma zostać:
- Okno DroneBooka ma **zwykłą sesję w ciasteczku**, dokładnie jak w przeglądarce (90 dni, odnawiana). Wylogowanie w tym oknie nie rozłącza companiona.
- Companion synchronizuje **swoim tokenem**, nawet gdy okno DroneBooka jest zamknięte, a pilot wylogowany.

Dzięki temu sesja przeglądarkowa nigdy nie trafia do kodu companiona, a token companiona nigdy nie trafia do strony.

---

## 3. Architektura po stronie DroneBooka

### 3.1 Nowe endpointy (`server/handlers/companion.ts`, trasa `/api/companion/*`)

| Metoda i ścieżka | Kto | Co robi |
|---|---|---|
| `POST /api/companion/pair` | anonimowo | Tworzy parę kodów. Limit np. 10/min/IP i 50/dzień/IP. |
| `GET  /api/companion/pair/:userCode` | sesja (ciasteczko) | Dane do strony zatwierdzenia: nazwa urządzenia, system, wersja, czas. |
| `POST /api/companion/approve` · `/deny` | sesja | Zatwierdza lub odrzuca kod. Kod „zajmuje” pierwsza sesja, która go otworzy (lekcja z CVE w Better Auth). |
| `POST /api/companion/token` | `deviceCode` | Odpowiedzi jak w RFC 8628: `authorization_pending`, `slow_down`, `expired_token`, `access_denied`, w końcu token (wydany tylko raz). |
| `GET  /api/companion/me` | token `dbc_` | Kto jest zalogowany, ustawienia dla companiona (lista symulatorów, minimalna wersja aplikacji, limity). |
| `POST /api/companion/sessions` | token `dbc_` | Wysyłka sesji z symulatorów (opis niżej). |
| `GET  /api/companion/devices` · `PATCH`/`DELETE /:id` | sesja | Lista połączonych komputerów, zmiana nazwy, odłączenie. |

Obsługa we wszystkich adapterach (Netlify, Vercel, Node) przychodzi za darmo, bo trasy rejestruje się w `server/routes.ts`.

### 3.2 Model danych

Bez nowej tabeli w Postgresie na start:
- Urządzenia w magazynie `ops`: `companion/devices/<deviceId>` → `{ id, userId, name, os, appVersion, createdAt, lastSeenAt, revokedAt? }` (jak `keys/<id>`).
- Parowanie: `companion/pair/<deviceCodeHash>` i indeks `companion/code/<userCode>`, z TTL 10 min.
- Loty: **zwykłe rekordy `flights`** w tabeli `records`, tak jak robi to Steam. W schemacie (`src/shared/schema.ts`) zmiana `source: z.literal('steam')` na `z.enum(['steam', 'companion'])`. Edycja wpisu w aplikacji usuwa `source`, jak dziś.

### 3.3 Wysyłka sesji

```jsonc
POST /api/companion/sessions
{
  "sessions": [
    {
      "sim": "VelociDrone",            // musi być na liście symulatorów z app-config
      "externalId": "vd-2026-10-07-…", // stabilny identyfikator z companiona → idempotencja
      "date": "2026-10-07",            // lokalna data pilota
      "startTime": "19:40",            // opcjonalnie
      "durationMin": 42.5,
      "quad": "Apex 5\"",              // opcjonalnie → simQuad
      "track": "Bando 3",              // opcjonalnie → notes
      "laps": 37                       // opcjonalnie → notes
    }
  ]
}
```

Serwer:
- **Nie ufa companionowi** (ta sama zasada co w całym DroneBooku): ścisła walidacja zod, maks. 100 sesji na żądanie, `durationMin` > 0 i ≤ 24 h na dzień i symulator po zsumowaniu, data w zakresie „dziś w dowolnej strefie” albo do 2 lat wstecz (dla importu historii), żadnych nieznanych pól.
- Zamienia sesje na wpisy `flights` o identyfikatorze wyliczonym z `externalId` (`comp-<sha256 skrót>`), więc ponowna wysyłka niczego nie dubluje.
- **Domyślnie scala sesje w jeden wpis na dzień i symulator** (tak wyglądają dziś wpisy ze Steama, logbook nie puchnie). W ustawieniach companiona przełącznik „Osobny wpis dla każdej sesji”.
- Ten sam pierwszy import co przy Steamie: przy pierwszym połączeniu companion pokazuje łączny czas sprzed połączenia i pyta, czy go zaimportować jako jeden wpis.
- Liczy się do dziennego limitu z `guard.ts` (osobny kubełek na urządzenie, np. 2000 żądań/dzień).

### 3.4 Zmiany w aplikacji webowej

- **Nowa strona `/companion/link`** (publiczna ścieżka, wymaga logowania): kod, nazwa urządzenia, „Zezwól” / „Odrzuć”, potem „Gotowe, możesz wrócić do companiona”. EN/PL w Paraglide.
- **`CompanionCard` w Ustawieniach** dostaje prawdziwy stan: „Połączono · 2 komputery · ostatnia synchronizacja 5 min temu”, lista urządzeń z „Odłącz”.
- Wpisy z companiona w logbooku dostają małą plakietkę źródła (jak Steam).
- `/download` bez zmian w kodzie: wystarczy uzupełnić `url` w `COMPANION_INSTALLERS`, gdy pojawią się wydania.
- CSP (`frame-ancestors 'none'`) nie przeszkadza: companion otwiera DroneBooka jako zwykłe okno, nie jako ramkę.
- CORS niepotrzebny: companion wysyła żądania z warstwy Rusta, nie z JavaScriptu w oknie.

---

## 4. Architektura companiona

### 4.1 Warstwy

```
┌──────────────────────── Companion (Tauri 2) ────────────────────────┐
│  Okno „main” (lokalny React)      Okno „dronebook” (zdalna strona)   │
│  status, symulatory, ustawienia   https://dronebook.vercel.app/app   │
│  ma dostęp do komend Rusta        BEZ dostępu do komend (brak IPC)   │
│            │ invoke()                                                │
│  ┌─────────▼───────────────────────────────────────────────────┐    │
│  │ Rdzeń w Ruście                                              │    │
│  │  • SimSource (trait): detect() · scan(cursor) → Vec<Session>│    │
│  │      ├─ VelociDrone (pliki/baza lokalna, opcjonalnie WS)    │    │
│  │      └─ później: Liftoff UDP, inne                          │    │
│  │  • Watcher plików (crate notify) + skan co N minut          │    │
│  │  • Kolejka wysyłki w SQLite (działa offline, ponawia)       │    │
│  │  • Klient API (reqwest, tylko HTTPS) + backoff              │    │
│  │  • Token w pęku kluczy (crate keyring)                      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│  Pluginy Tauri: tray, autostart, single-instance, updater,           │
│  deep-link (dronebook://), log, opener, window-state                 │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Najważniejsze decyzje

- **Pliki czyta Rust, nie JavaScript.** Na desktopie (poza Mac App Store) aplikacja ma zwykły dostęp do plików użytkownika. Rust czyta katalogi symulatorów i zwraca do UI gotowe dane. Plugin `fs` w JS nie jest potrzebny, więc nie trzeba otwierać szerokich uprawnień dla okna.
- **Dwa okna, dwa poziomy zaufania.** W Tauri 2 okno, które nie pasuje do żadnej „capability”, nie ma w ogóle dostępu do IPC. Okno `dronebook` ładuje zdalną stronę i celowo nie ma żadnej capability (brak wpisu `remote`). Nawet gdyby ktoś wstrzyknął skrypt do strony, nie dobierze się do plików ani tokenu.
- **Adres serwera do ustawienia.** Domyślnie `https://dronebook.vercel.app`, ale backend jest przenośny (Vercel, Netlify, Docker na Twoim serwerze przez Tailscale), więc w ustawieniach zaawansowanych jest pole „Adres serwera DroneBook”. Token jest przypisany do adresu, zmiana adresu wymaga ponownego parowania.
- **Kolejka offline.** Każda wykryta sesja trafia najpierw do lokalnej bazy SQLite (`pending` → `sent`). Brak internetu niczego nie gubi; wysyłka ponawia się z rosnącym odstępem.
- **Kursor per symulator.** Companion zapamiętuje, co już wysłał (np. łączny czas i znacznik ostatniej sesji), więc „Reset game” w VelociDrone nie cofnie ani nie zdubluje nalotu: spadek licznika jest traktowany jako nowy punkt startowy.
- **Działa w tle.** Ikona w zasobniku, opcjonalny autostart z systemem, jedna instancja. Zamknięcie okna chowa je do zasobnika.
- **Mały UI.** Ekrany: Start (połączono jako…, ostatnia synchronizacja, lista symulatorów: wykryty / nie znaleziono / wyłączony), Ustawienia (autostart, scalanie per dzień, adres serwera, język), Okno DroneBooka, O programie (wersja, aktualizacje, logi).

### 4.3 Symulatory

**VelociDrone (pierwszy):**
- Nie ma go na Steamie. Ma lokalną „bazę” z historią gry i łącznym czasem lotu; „Reset game” ją czyści. Format i położenie nie są udokumentowane.
- Znane miejsca do sprawdzenia: macOS `~/Library/Application Support/VelociDrone/VelociDrone/` (według strony pomocy wydawcy, sprzed ok. 2 lat), Windows: folder, w którym rozpakowano launcher (PatchKit), oraz rejestr `HKCU\Software\VelociDrone\VelociDrone` (ustawienia Unity) i `%USERPROFILE%\AppData\LocalLow\…`. Linux: do ustalenia.
- **Trop z Reddita (niepotwierdzony):** łączny czas lotu siedzi w bazie SQLite. macOS: `~/Library/Application Support/com.velocidrone.velocidrone/user11.db` (folder `Library` jest ukryty), tabela `sim_states`, kolumna `logged_time`, wartość w sekundach jako liczba z ułamkiem (np. `147473.414` ≈ 41 h). To łączny licznik, nie lista sesji, więc liczymy przyrosty jak przy Steamie.
  - Czytamy wyłącznie do odczytu: najlepiej kopia pliku (razem z ewentualnymi `-wal`/`-shm`) i odczyt z kopii, ewentualnie `file:…?mode=ro&immutable=1`. Companion nigdy nie zapisuje do tej bazy. Autor wątku edytował ją w TextEdit i zepsuł instalację, musiał przeinstalować grę.
  - Do sprawdzenia w Etapie 0: czy `user11` to stała nazwa, czy zależy od konta/wersji (wtedy szukamy `user*.db` i wybieramy właściwy plik), ile wierszy ma `sim_states` i który jest bieżący, oraz położenie pliku na Windowsie i Linuksie (prawdopodobnie odpowiednik `com.velocidrone.velocidrone` w AppData/`~/.local/share`, niezweryfikowane).
- **Etap 0 planu to ustalenie formatu na Twoich plikach** (patrz niżej). Od tego zależy, czy dostaniemy pojedyncze sesje, czy tylko łączny licznik (wtedy liczymy przyrosty dzienne jak przy Steamie).
- Dodatkowo, opcjonalnie: WebSocket `ws://<IP w LAN>:60003/velocidrone` (włączany w ustawieniach gry, wymaga roli race managera i pustej wiadomości co 10 s). Daje tylko zdarzenia wyścigów (start/stop, okrążenia, czasy), więc nadaje się na „sesje wyścigowe z czasami okrążeń”, a nie jako główne źródło nalotu.

**Później:** Liftoff przez telemetrię UDP (`TelemetryConfiguration.json`) dałby prawdziwy czas w powietrzu zamiast czasu z menu, który liczy Steam. Wymaga ustalenia formatu pakietów na żywym strumieniu. Inne symulatory spoza Steama dodaje się jako kolejne implementacje `SimSource`.

### 4.4 UI i wspólne komponenty

Najpierw to, co rozwiązuje się samo: **okno DroneBooka w companionie to ta sama strona**, więc wygląda identycznie z definicji. Wspólny wygląd trzeba zapewnić tylko dla kilku własnych ekranów companiona (Start, Ustawienia, O programie, parowanie).

Do tych ekranów potrzebne są elementy, które DroneBook już ma:
- **tokeny** z `src/styles/styles.css` (`@theme`: kolory jasne/ciemne, rozmiary czcionek, promienie),
- **receptury klas** z `src/styles/tw.ts` (`btn`, `link`, `cx`…),
- **komponenty bez zależności od danych:** `components/ui` (Badge, Empty, Field, Header, Icon, Loading, Segmented, Sheet, Stat, Stats), `components/layout` (Card, FoldCard, List, ListRow, ListItem, ListHeading…), `components/form` (FieldRow, CheckboxRow, FormError, ToggleChips, ChoiceChips), `Logo` i `CompanionLogo`.

Zostają w aplikacji webowej: wszystko, co sięga do `@/data` (pliki, sesja, SiteNav), `ConfigSelect` (lista z app-config) i ekrany funkcji.

**Rozważone sposoby współdzielenia:**

| Sposób | Plusy | Minusy |
|---|---|---|
| Kopiowanie plików do companiona | Najprościej na start | Wygląd zaczyna się rozjeżdżać przy pierwszej zmianie w DroneBooku |
| Submoduł git | Jedna kopia | Niewygodne aktualizacje, łatwo zapomnieć o podbiciu wersji |
| Pakiet `@dronebook/ui` w GitHub Packages, osobne repo companiona | Czysta separacja, wersjonowanie | Każda zmiana to publikacja nowej wersji i podbicie zależności; prywatny rejestr wymaga tokenu w CI i lokalnie |
| **Monorepo (npm workspaces)** ✅ | Jedna kopia, zmiana widoczna od razu w obu aplikacjach, jeden lint (z istniejącymi regułami), jeden PR na zmianę backendu i companiona | Repo `dronebook` rośnie; CI musi budować companiona tylko wtedy, gdy zmieniły się jego pliki |

**Rekomendacja: monorepo.** Układ:

```
dronebook/
  src/ server/ adapters/ …    aplikacja webowa bez zmian w ścieżkach
  packages/
    ui/                       tokeny (theme.css), tw.ts, komponenty ui/layout/form, Logo
    shared/                   obecne src/shared (schema, appConfig, velocidrone) – też dla serwera
  apps/
    companion/                Tauri 2: src-tauri/ (Rust) + src/ (React)
```

- Przeniesienie `src/components/{ui,layout,form}` do `packages/ui` to mechaniczna zmiana importów (`@/components/ui/Sheet` → `@dronebook/ui/Sheet`), sprawdzana tak jak przy restrukturyzacji: zbudowany CSS identyczny bajt w bajt i zrzuty ekranu bez różnic.
- **Tailwind v4:** w CSS każdej aplikacji `@import '@dronebook/ui/theme.css'` oraz `@source` wskazujące na `packages/ui`, żeby klasy z komponentów trafiły do zbudowanego CSS.
- **Teksty w komponentach** (np. „Zamknij” w `Sheet`): wspólny plik komunikatów `packages/ui/messages/{en,pl}.json`, dołączany do Paraglide obu aplikacji. Komponenty nadal importują `m` z własnego Paraglide aplikacji, więc przełącznik języka działa w obu.
- **Granice pilnuje lint:** do `dronebook/feature-boundaries` dochodzi zasada, że `packages/ui` nie importuje niczego z `src/` ani `apps/`, a `apps/companion` korzysta tylko z `packages/*`.
- **Vercel buduje tylko aplikację webową**, jak dziś. Workflow companiona w GitHub Actions uruchamia się tylko przy zmianach w `apps/companion/**` lub `packages/**`.
- **Repo `dronebook-companion`** staje się wtedy publicznym repo **tylko na wydania** (README, instalatory, `latest.json` dla updatera). Rozwiązuje to przy okazji problem z §6: kod zostaje prywatny, a aktualizacje są pobierane z publicznego repo. CI w `dronebook` publikuje tam wydanie przy tagu `companion-v1.2.3`.

Gdybyś jednak wolał osobne repo z kodem companiona, drugi wybór to pakiet `@dronebook/ui` w GitHub Packages publikowany z repo `dronebook` przy każdej zmianie w `packages/ui`.

---

## 5. Dlaczego Tauri 2 (i co z alternatywami)

| | **Tauri 2** ✅ | Electron | Wails (Go) | Flutter / Avalonia / MAUI |
|---|---|---|---|---|
| Jeden kod na 3 systemy | tak | tak | tak | tak |
| UI w React jak DroneBook | tak | tak | tak | nie, osobny UI do napisania |
| Rozmiar instalatora | ~5–15 MB | ~80–120 MB | ~10 MB | 20–60 MB |
| Pamięć w tle | mała | duża (cały Chromium) | mała | średnia |
| Dostęp do plików | Rust, bez ograniczeń | Node, bez ograniczeń | Go | tak |
| Osadzone okno ze zdalną stroną | tak, z izolacją uprawnień | tak (BrowserView), bezpieczeństwo trzeba pilnować ręcznie | tak | słabiej |
| Auto-update | oficjalny plugin, podpis obowiązkowy | electron-updater | brak oficjalnego | różnie |
| Język „backendu” | Rust | TypeScript | Go | Dart / C# |

Uczciwie o minusach Tauri:
- **Rust.** Rdzeń companiona to kilka modułów (odczyt plików, kolejka, klient HTTP), więc to mała ilość kodu, ale nowy język. Jeśli wolisz zostać przy TypeScripcie wszędzie, Electron jest jedyną sensowną alternatywą, za cenę ~100 MB i większego zużycia pamięci.
- **Różne silniki WWW:** WebView2 (Chromium) na Windowsie, WebKit na macOS i WebKitGTK na Linuksie. Lokalny UI jest mały, więc ryzyko niewielkie; DroneBook w oknie trzeba raz przetestować na każdym systemie (mapa przestrzeni, PDF, service worker PWA).
- Na Linuksie wymagane `libwebkit2gtk-4.1` (jest w aktualnych Ubuntu/Fedorze; AppImage to obsługuje).

---

## 6. Wydania, aktualizacje, podpisywanie

- **Budowanie:** GitHub Actions z oficjalną akcją `tauri-action`, macierz Windows / macOS (Apple Silicon + Intel) / Linux. Tag `v1.2.3` → szkic wydania z instalatorami: `.msi`/`.exe` (NSIS), `.dmg`, `.AppImage`, `.deb`, `.rpm`.
- **Updater Tauri:** podpis aktualizacji jest obowiązkowy (para kluczy; prywatny w sekretach GitHuba, publiczny w `tauri.conf.json`). Companion sprawdza `latest.json` przy starcie i co 24 h.
- **Gdzie leżą pliki:** wydania na GitHubie pobiera się bez logowania tylko z **publicznego** repo. Do wyboru:
  1. repo `dronebook-companion` publiczne (kod companiona nie zawiera sekretów, więc to bezpieczne), albo
  2. repo prywatne, a instalatory i `latest.json` kopiowane przez CI do magazynu plików DroneBooka i serwowane z `/api/companion/releases/*`.
  Rekomendacja: przy monorepo (§4.4) kod companiona zostaje w prywatnym `dronebook`, a publiczne `dronebook-companion` trzyma tylko wydania. Przy osobnym repo z kodem: **publiczne repo**, bo to najprostsza dystrybucja i darmowe CDN.
- **Podpisywanie systemowe:**
  - *Windows:* Azure Artifact Signing (ok. 9,99 USD/mies.) jest dla osób prywatnych tylko z USA i Kanady, więc z Polski odpada. Opcje: certyfikat OV od zwykłego CA (ok. 150–300 USD/rok, np. polski Certum, cena do sprawdzenia; Certum ma też tańszy certyfikat dla projektów open source), albo na start bez podpisu (SmartScreen pokaże „Więcej informacji → Uruchom mimo to”). Nawet z podpisem OV SmartScreen ostrzega, dopóki aplikacja nie zbierze reputacji.
  - *macOS:* Apple Developer Program, 99 USD/rok, daje podpis Developer ID i notaryzację. Bez tego Gatekeeper blokuje aplikację i trzeba ją odblokować w Ustawieniach systemowych → Prywatność i ochrona.
  - *Linux:* podpis niepotrzebny.
  - Rekomendacja: **wersje testowe bez podpisu** (dla Ciebie i kilku osób to wystarczy), decyzja o certyfikatach dopiero przed publicznym wydaniem.

---

## 7. Bezpieczeństwo (podsumowanie)

- Token companiona ma wąski zakres, jest odwoływalny, trzymany w pęku kluczy systemu, wysyłany tylko po HTTPS, nigdy nie trafia do logów ani do okna ze stroną.
- Serwer waliduje wszystko, co przysyła companion (zakresy, sumy dzienne, idempotencja, znane symulatory), a limity z `guard.ts` obejmują każde urządzenie osobno.
- Parowanie: krótkie kody, 10 minut, jednorazowe, „zajmowane” przez pierwszą sesję, z kodem i nazwą urządzenia widocznymi na stronie zatwierdzenia; limity na IP dla `/pair` i `/token`.
- Okno ze zdalnym DroneBookiem bez dostępu do IPC; nawigacja w nim ograniczona do domeny DroneBooka, inne linki otwierają się w przeglądarce systemowej.
- Lokalny UI z restrykcyjnym CSP, uprawnienia Tauri tylko do własnych komend.
- Aktualizacje podpisane kluczem updatera; companion odmawia instalacji niepodpisanej paczki.
- Usunięcie konta lub odłączenie urządzenia natychmiast unieważnia tokeny; zdarzenia `companion-paired` / `companion-revoked` trafiają do dziennika zdarzeń w `/admin`.
- Gdy w przyszłości pojawi się logowanie przez Google: Google blokuje logowanie w osadzonych oknach, więc wtedy przycisk „Otwórz w przeglądarce” staje się drogą domyślną dla tych kont (przepływ parowania działa tak samo).

---

## 8. Etapy

| # | Etap | Gdzie | Wynik |
|---|---|---|---|
| 0 | **Rozpoznanie danych VelociDrone** | Twój komputer | Paczka z folderu danych VelociDrone (i eksport klucza rejestru na Windowsie). Ustalamy format, czy są pojedyncze sesje, czy tylko licznik. |
| 0b | Wydzielenie `packages/ui` i `packages/shared` | `dronebook` (1 PR) | npm workspaces, przeniesione komponenty i tokeny, ten sam wygląd (CSS bajt w bajt, zrzuty bez różnic). |
| 1 | Backend parowania i sesji | `dronebook` (1 PR) | `/api/companion/*`, token `dbc_`, `source: 'companion'`, strona `/companion/link`, prawdziwy stan `CompanionCard`, EN/PL, testy dymne w `npm run smoke`. |
| 2 | Szkielet companiona | `dronebook/apps/companion` | Tauri 2 + React, zasobnik, dwa okna, parowanie, pęk kluczy, adres serwera, CI budujące 3 systemy (bez podpisu). |
| 3 | Czytnik VelociDrone | `dronebook/apps/companion` | `SimSource` dla VelociDrone, kolejka offline, pierwszy import historii, wpisy w logbooku. |
| 4 | Aktualizacje i wydanie | oba repo | Updater, `latest.json`, linki na `/download`, decyzja o podpisach. |
| 5 | Później | | WebSocket VelociDrone (wyścigi, okrążenia), Liftoff UDP, kolejne symulatory. |

Etapy 1 i 2 można robić równolegle, gdy etap 0 trwa.

---

## 9. Ryzyka

| Ryzyko | Wpływ | Co z tym robimy |
|---|---|---|
| Format danych VelociDrone nieczytelny lub zaszyfrowany | Companion nie ma czego czytać | Etap 0 przed kodem; plan B: WebSocket wyścigowy + ręczny „licznik startowy”. |
| VelociDrone zmieni format w aktualizacji | Synchronizacja przestaje działać | Czytnik wykrywa nieznany format i pokazuje komunikat zamiast zgadywać; wersjonowane parsery. |
| „Reset game” zeruje licznik | Utracone lub zdublowane godziny | Kursor po stronie companiona, spadek licznika = nowa baza. |
| Różnice WebKitGTK / WebKit | Coś w oknie DroneBooka wygląda inaczej | Jednorazowy test każdej zakładki na 3 systemach przed etapem 4. |
| Koszt podpisów (99 USD + ~150–300 USD rocznie) | Ostrzeżenia systemu przy instalacji | Wersje testowe bez podpisu; decyzja przed publicznym wydaniem. |
| Nowy język (Rust) | Wolniejszy start | Rdzeń jest mały, a większość logiki (walidacja, zamiana na wpisy) siedzi na serwerze w TypeScripcie. |
| Darmowe plany hostingu | Zużycie limitów | Odpytywanie parowania poza Postgresem, wysyłka paczkami co kilka minut, nie po każdym okrążeniu. |

---

## 10. Decyzje dla Ciebie

1. **Model synchronizacji:** parowanie z tokenem o wąskim zakresie (rekomendacja) czy prostsze wklejanie klucza API?
2. **Tauri 2 (Rust)** czy wolisz Electron, żeby zostać przy TypeScripcie?
3. ~~Gdzie kod companiona~~ **Zdecydowane 7.10.2026: monorepo** (`dronebook/apps/companion` + `packages/ui`), `dronebook-companion` jako publiczne repo tylko na wydania.
4. **Wpisy w logbooku:** jeden na dzień i symulator (rekomendacja, jak Steam) czy każda sesja osobno?
5. **Etap 0:** możesz spakować folder danych VelociDrone ze swojego komputera i wrzucić go tutaj?

## Źródła

- Better Auth, Device Authorization: https://better-auth.com/docs/plugins/device-authorization
- CVE-2026-45337 (Better Auth 1.6.0–1.6.10): https://corgea.com/advisories/vulnerabilities/CVE-2026-45337
- RFC 8628, OAuth 2.0 Device Authorization Grant: https://www.rfc-editor.org/rfc/rfc8628
- Tauri 2, capabilities i pole `remote`: https://v2.tauri.app/reference/acl/capability
- Tauri 2, updater: https://v2.tauri.app/plugin/updater/
- Microsoft, opcje podpisywania kodu: https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/code-signing-options
- Azure Artifact Signing, cennik: https://azure.microsoft.com/pricing/details/artifact-signing/
- Bat Cave Games (VelociDrone), odinstalowanie na Macu: https://batcavegames.freshdesk.com/support/solutions/articles/16000095364-how-to-uninstall-on-a-mac
- r/Velocidrone, „68 hours in sim because I forgot to exit session” (komentarz u/garza-0 o `user11.db` / `sim_states.logged_time`), zrzuty ekranu od Michała w projekcie, 2026-10-08.
- VelociDrone WebSocket, przykład klienta: https://github.com/eedok/VelocidroneWebSocketConsumer
