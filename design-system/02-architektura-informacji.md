# Architektura informacji

## Nawigacja główna

Pięć miejsc, identycznych na mobile (dolny pasek) i web (boczne menu):

| Pozycja | Zawartość | Wzorzec |
|---|---|---|
| Start | Feed rekomendacji + chipy kategorii | YouTube Home |
| Shorts | Pionowy strumień krótkich wideo z przyciskiem „Kup teraz” | YouTube Shorts |
| Obserwowane | Nowe oferty i filmy z kanałów, które obserwujesz; lista kanałów | YouTube Subskrypcje |
| Wiadomości | Czaty kupujący–sprzedawca, pytania, rezerwacje | OLX / Allegro |
| Profil | Konto, zamówienia, zapisane, panel sprzedawcy, ustawienia | — |

Poza paskiem: wyszukiwarka (zawsze u góry), koszyk, powiadomienia, „Dodaj ofertę” (web: przycisk w nagłówku; mobile: akcja w Profilu i na kanale własnym).

## Mapa serwisu

```
Start
 ├─ Feed (chipy: Dla Ciebie · kategorie · Usługi · Okazje)
 ├─ Wyniki wyszukiwania → Filtry (kategoria, cena, lokalizacja, stan, ocena, typ, tylko z filmem)
 └─ Karta oferty
     ├─ Produkt: galeria+wideo · cena · Kup teraz · Napisz · Zadzwoń · pytania · następne
     └─ Usługa: galeria+wideo · cennik · kalendarz rezerwacji · Zarezerwuj · pytania
Shorts → pionowy odtwarzacz → oferta
Obserwowane → kanał sprzedawcy (Oferty · Filmy · Opinie · O nas)
Wiadomości → lista rozmów → wątek (karta oferty, szybkie odpowiedzi, propozycja ceny)
Koszyk → Dostawa → Płatność → Potwierdzenie   |   Rezerwacja → Płatność/zaliczka → Potwierdzenie
Profil
 ├─ Moje zamówienia i rezerwacje · Zapisane · Opinie
 ├─ Dodaj ofertę (zdjęcia + wideo → szczegóły → publikacja)
 └─ Panel sprzedawcy (Przegląd · Oferty · Zamówienia · Filmy · Statystyki · Opinie)
```

## Układ ekranów

Web: boczne menu 232 px, nagłówek z wyszukiwarką, treść w siatce 3–4 kolumn. Mobile: nagłówek z wyszukiwarką i powiadomieniami, chipy, kolumna dużych kart przeplatana półką Shorts, dolny pasek. Karta oferty na web ma układ jak strona odtwarzania YouTube: media po lewej, boks zakupu i „Następne” po prawej.
