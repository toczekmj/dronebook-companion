# DroneBook Companion

Aplikacja desktopowa (Windows, macOS, Linux) dla [DroneBook](https://dronebook.vercel.app):

- odczytuje czas lotów z symulatorów spoza Steama (na start VelociDrone) i dopisuje go do logbooka,
- pozwala zalogować się do DroneBooka i korzystać z niego w osobnym oknie.

Planowany stos: Tauri 2 (rdzeń w Ruście) + React 19, Vite, Tailwind v4, Paraglide (EN/PL). Rekomendacja planu: kod w monorepo `dronebook` (`apps/companion` + wspólne `packages/ui`), a to repo tylko na wydania.

Status: **plan**, kod jeszcze nie powstał. Pełny plan: [PLAN.md](PLAN.md).
