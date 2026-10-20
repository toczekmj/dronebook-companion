# DroneBook Companion

Aplikacja desktopowa (Windows, macOS, Linux) dla [DroneBook](https://dronebook.vercel.app):

- odczytuje czas lotów z symulatorów spoza Steama (na start VelociDrone) i dopisuje go do logbooka,
- pozwala zalogować się do DroneBooka i korzystać z niego w osobnym oknie.

Kod jest w monorepo [toczekmj/dronebook](https://github.com/toczekmj/dronebook) (`apps/companion`), a to repo
służy do wydań. Pliki do pobrania: [Releases](https://github.com/toczekmj/dronebook-companion/releases) albo
strona [/download](https://dronebook.vercel.app/download). Plan: [PLAN.md](PLAN.md).

## Pierwsze uruchomienie

To wersje testowe, niepodpisane płatnym certyfikatem, więc system za pierwszym razem prosi o potwierdzenie.

**Windows:** SmartScreen → Więcej informacji → Uruchom mimo to.

**macOS:** otwórz `.dmg` (Apple silicon dla M1 i nowszych, Intel dla starszych) i przeciągnij aplikację do
folderu Programy. Uruchom ją raz, a gdy macOS odmówi, wejdź w Ustawienia systemowe → Prywatność i ochrona
i kliknij **Otwórz mimo to**.

Jeśli macOS pisze, że aplikacja **„jest uszkodzona i nie można jej otworzyć”**, to nie jest uszkodzony plik, tylko
blokada pobranej, nienotaryzowanej aplikacji. Wpisz w Terminalu i otwórz ją ponownie:

```bash
xattr -cr "/Applications/DroneBook Companion.app"
```

**Linux:** AppImage trzeba oznaczyć jako wykonywalny (`chmod +x`); `.deb` i `.rpm` instaluje się normalnie.
