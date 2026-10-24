# User flow

## Kupujący: od odkrycia do zakupu

```
Onboarding (rola: Kupujący, zainteresowania, lokalizacja)
  → Start: feed + chipy
  → [podgląd wideo po najechaniu / tapnięciu] → Karta oferty
  → decyzja:
       Kup teraz → Koszyk (lub od razu Dostawa) → Płatność (BLIK, karta, przelew) → Potwierdzenie
       Napisz → Czat → odpowiedź sprzedawcy → Kup teraz
       Zadzwoń → połączenie
       Obserwuj → kanał → powrót w „Obserwowane”
  → Po zakupie: status zamówienia → odbiór → Opinia (gwiazdki + komentarz + opcjonalne wideo)
```

Wyszukiwanie: Start → pole wyszukiwania → podpowiedzi → wyniki → Filtry (arkusz na mobile, panel po lewej na web) → karta oferty. Filtry zapisane jako chipy nad wynikami, każdy do zdjęcia jednym tapem.

## Kupujący: rezerwacja usługi

```
Karta usługi → film prezentujący → cennik → „Wybierz termin” (kalendarz + sloty)
  → adres i uwagi → Zaliczka lub płatność po wykonaniu → Potwierdzenie + wydarzenie w kalendarzu
  → Przypomnienie dzień wcześniej → Wykonanie → Opinia
```

## Usługodawca

```
Onboarding (rola: Usługodawca) → weryfikacja (telefon, firma opcjonalnie)
  → Kanał: baner, awatar, opis
  → Dodaj ofertę (Usługa): film/zdjęcia → tytuł, kategoria, cennik, obszar → godziny dostępności
  → Publikacja → Panel: nowe rezerwacje (Zaakceptuj / Zaproponuj inny termin) → Czat → Wykonanie → Opinia
```

## Merchant

```
Onboarding (rola: Merchant, dane firmy, konto do wypłat)
  → Dodaj ofertę (Produkt): zdjęcia + film + Short → cena, stan, dostawa, magazyn
  → Publikacja → Panel: zamówienia (Do wysłania → W drodze → Zrealizowane), statystyki, pytania w komentarzach
```

## Punkty decyzyjne i stany pustych

- Brak wyników → komunikat „Nic nie znaleźliśmy. Zmień filtry lub zobacz podobne oferty” + 3 polecane karty.
- Brak sieci → ostatni feed z oznaczeniem „Tryb offline”.
- Nieopłacone zamówienie → przypomnienie w Wiadomościach po 30 minutach.
