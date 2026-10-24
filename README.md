# rynek-docs

Wspólna dokumentacja platformy **Rynek**: wideo-pierwszego marketplace'u produktów i usług na rynek polski.

## Zawartość

- `architektura/rynek-endpointy-mikroserwisy.md`: przegląd całej architektury, czyli podział na mikroserwisy, konwencje API (§1), routing gateway (§2), zdarzenia Kafka (§17), ściąga ekran → API (§18), kolejność budowy (§19).
  Specyfikacja endpointów każdego serwisu żyje w jego repozytorium (`rynek-<serwis>/README.md`). Ten plik służy jako mapa całości i stan wyjściowy.
- `design-system/`: design system „Rynek”: zasady UI (`README.md`), koncepcja, architektura informacji, user flow, inspiracje (`01`–`04`), tokeny (`tokens.css`, `tokens.json`), komponenty i podglądy ekranów (`index.html` otwiera listę).

## Zasady

- Decyzje architektoniczne dotyczące wielu serwisów zapisuj tutaj, np. w przyszłym folderze `adr/`.
- Interfejsy między serwisami (OpenAPI, schematy zdarzeń) należą do `rynek-contracts`, nie tutaj.
