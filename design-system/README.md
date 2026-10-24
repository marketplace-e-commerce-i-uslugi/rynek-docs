Rynek to wideo-pierwszy marketplace produktów i usług na rynek polski. Interfejs jest po polsku, w tonie ciepłym i rzeczowym: zwracamy się do użytkownika na „Ty”, polecenia w trybie rozkazującym („Dodaj ofertę”, „Zarezerwuj termin”), bez wykrzykników i bez emoji. Nazwa robocza „Rynek” jest zapisywana małymi literami z kropką w kolorze `brand` (`rynek.`); prawdziwego logo jeszcze nie ma, więc używamy wyłącznie tego znaku słownego.

## Treść i język

- Ceny zapisuj w formacie polskim: `389,00 zł`, tysiące oddzielone twardą spacją (`1 150,00 zł`), dla usług `od 35 zł/m²` lub `od 45 zł/godz.`.
- Daty skracaj po polsku: `pt., 23 paź.`, godziny w formacie 24-godzinnym (`14:30`). Liczby obserwujących i wyświetleń skracaj do `12,4 tys.`.
- Oceny pokazuj jako gwiazdka + liczba + liczność opinii: `4,9 (128)`. Gwiazdka bez liczby jest niedozwolona.
- Przyciski mają czasownik na początku: „Kup teraz”, „Dodaj do koszyka”, „Napisz”, „Zadzwoń”, „Zarezerwuj termin”, „Obserwuj”, „Opublikuj”.
- Stan oferty to jedno ze słów: `Nowy`, `Używany`, `Poleasingowy`, `Uszkodzony`; typ oferty: `Produkt` lub `Usługa`.

## Kolor

- Kolor główny `brand` (sosnowa zieleń) ma przyciski podstawowe, aktywną zakładkę, linki i ikony aktywne; tekst na nim to `on-brand`.
- Kolor akcentowy `accent` (miód) jest zarezerwowany dla jednej czynności zakupowej na ekran: „Kup teraz” lub „Zarezerwuj”. Tekst na nim to zawsze `on-accent`. Nigdy nie używaj `accent` jako koloru tekstu ani tła dużych powierzchni.
- Powierzchnie: `surface` (tło), `surface-raised` (karty, arkusze, pola), `surface-muted` (chipy, wyszukiwarka, dymki). Tekst: `ink` i `ink-muted`.
- Stany: wzrost lub potwierdzenie to `positive` (niebieski), spadek lub błąd to `danger`; zawsze razem ze słowem lub strzałką, bo para nie może opierać się wyłącznie na barwie.
- Aktywny chip jest odwrócony: tło `ink`, tekst `surface` (wzór z YouTube). Wybrana rola lub opcja to `brand-tint` z obrysem `brand` 2 px.
- Tryb jasny i ciemny to dwa motywy tych samych tokenów (`light`, `dark`). W ciemnym `brand` jest jaśniejszy, a `on-brand` ciemny — nie wpisuj białego tekstu na `brand` na sztywno.

## Typografia

Krój `Manrope` (pełne polskie znaki, ładowany z Google Fonts), awaryjnie `system-ui`. Tytuły ekranów: `h1`; tytuły sekcji i półek: `h2`; tytuł karty w feedzie: `body-strong` (2 linie, obcięcie wielokropkiem); metadane: `caption` w `ink-muted`; ceny: `price`, w boksie zakupu `price-lg`. Nie schodź poniżej 12 px (`overline`), nie używaj więcej niż trzech grubości na ekranie.

## Układ, promienie, cienie

- Margines ekranu mobilnego to `space-4`; odstęp między kartami `space-3`; między sekcjami feedu `space-5`.
- Karty i miniatury mają `radius-lg`, pola formularzy `radius-md`, przyciski, chipy i wyszukiwarka `radius-pill`, arkusze i modale `radius-xl`.
- Feed jest płaski: karty ofert nie mają ramki ani cienia, odróżnia je miniatura. `shadow-card` tylko dla hover na web, arkuszy i ramek urządzeń.
- Siatka web: 3 kolumny feedu przy szerokości treści ≥ 900 px, 4 na ekranach ≥ 1280 px, 2 na tablecie; mobile: 1 kolumna dużych kart + półki poziome.

## Wideo i obrazy

- Każda oferta ma miniaturę 16:9 `radius-lg`; wideo oznacza znacznik czasu w prawym dolnym rogu (`scrim` + biały tekst) i ikona odtwarzania.
- Na web, po najechaniu, miniatura odtwarza wyciszony podgląd; pasek postępu w kolorze `accent` u dołu. Przy `prefers-reduced-motion` podgląd się nie uruchamia.
- Shorts mają proporcję 9:16 i podpis na `scrim`. Każde wideo wymaga napisów (pole „Napisy” przy dodawaniu) i tekstu alternatywnego miniatury.

## Ikony

Ikony liniowe 24 px, obrys 2 px, zaokrąglone końce; w makietach rysują je klasy `.ic.<nazwa>` w `bundle.css` (home, play, users, chat, user, search, filter, cart, heart, star, plus, chart, phone, calendar, check, bell, box, video, image, pin, send, chevron, back, shield, menu, close, truck, card, store, tool, like, comment, share, more, attach). Ikona nigdy nie występuje sama w kontrolce bez dostępnej nazwy.

## Dostępność

- Tekst ma kontrast co najmniej 4,5:1 w obu motywach; obrysy kontrolek (`line-strong`), ikony i pierścień fokusu co najmniej 3:1.
- Elementy dotykowe mają minimum `touch-target` (48 px); chipy 44 px z odstępem 8 px. Wersja `sm` przycisku (44 px) dozwolona tylko na web przy myszy.
- Fokus: pierścień `focus` 3 px, odstęp 2 px, na każdym elemencie interaktywnym. Kolejność tabulatora zgodna z układem.
- Stany (sukces, błąd, status zamówienia) zawsze mają tekst. Statusy zamówień: `Do wysłania`, `W drodze`, `Zrealizowane`.

## Skąd co wzięliśmy

Pełna mapa elementów YouTube i Allegro/OLX jest w sekcji „Źródła inspiracji”. Na ekranach każdą adnotację oznacza znacznik: `YouTube`, `Allegro/OLX` lub `Własne`.

## Komponenty i ekrany

Grupy: Akcje (Button, Chip), Karty (OfferCard, ServiceCard, ShortTile, StatTile), Nawigacja (BottomNav, SideNav), Komunikacja (ChatBubble), Rezerwacje (BookingCalendar), Filtry (FilterPanel), Kanał (ChannelHeader) oraz grupa Ekrany z dziewięcioma makietami (mobile + web).
