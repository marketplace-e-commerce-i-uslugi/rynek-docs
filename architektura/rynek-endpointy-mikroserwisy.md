# Rynek — endpointy podzielone na mikroserwisy (Java + TikTok API)

Dokument opisuje, jakie endpointy są potrzebne do ekranów z design systemu „Rynek” i do którego mikroserwisu każdy należy. To propozycja startowa (MVP + rozsądny zapas), nie gotowa specyfikacja OpenAPI.

Założenia: Java 21, Spring Boot 3, osobna baza PostgreSQL na serwis, Kafka do zdarzeń, Redis do cache i limitów, Elasticsearch/OpenSearch do wyszukiwania, wideo przez TikTok API.

---

## 0. Najważniejsze o TikToku — przeczytaj przed projektowaniem

Z dokumentacji TikTok for Developers (sprawdzone w tej sesji) wynika kilka ograniczeń, które wpływają na produkt:

1. **Display API zwraca tylko filmy zalogowanego (autoryzującego) użytkownika.** `POST /v2/video/list/` oddaje publiczne filmy właściciela tokenu, a `POST /v2/video/query/` potwierdza, że filmy należą do tego użytkownika. Nie znalazłem w dokumentacji opcji pobierania filmów dowolnych innych kont. Wniosek: **sprzedawca musi połączyć własne konto TikTok z Rynkiem**, a Rynek pokazuje jego filmy.
2. **Linki do okładek wygasają.** `cover_image_url` ma parametr `x-expires`; dokumentacja każe odświeżać go przez `video/query`. Czas życia nie jest podany — zakładaj, że krótki, i odświeżaj cyklicznie. TikTok zaleca też synchronizację filmów użytkownika co 12 godzin.
3. **Odtwarzanie to osadzenie (`embed_link` / `embed_html`)**, nie surowy plik wideo. Podgląd wyciszony „po najechaniu” z design systemu nie jest więc gwarantowany — bezpieczniej: okładka w feedzie, odtwarzanie po tapnięciu. Napisy do filmów (wymóg z README) kontroluje TikTok, nie Rynek.
4. **Publikowanie z Rynku na TikTok (Content Posting API)** wymaga zatwierdzenia scope `video.publish`. Dopóki aplikacja nie przejdzie audytu, **wszystkie posty są prywatne**. Dla `PULL_FROM_URL` domena musi być zweryfikowana.
5. **Tokeny:** access token ważny 24 h, refresh token 365 dni. Serwis musi odświeżać je w tle; `client_secret` i tokeny tylko po stronie backendu, szyfrowane w bazie.
6. **Ścieżka awaryjna bez OAuth:** publiczny `oEmbed` (`GET https://www.tiktok.com/oembed?url=...`) pozwala osadzić film po samym linku — sprzedawca wkleja link. Dokumentacja nie podaje tu limitów.
7. **Nie potwierdziłem:** limitów zapytań (rate limit), czasu wygaśnięcia okładek i `embed_link`, dziennego limitu postów, limitów rozmiaru plików. Sprawdź w konsoli deweloperskiej TikTok. Przejrzyj też regulamin TikTok pod kątem użycia w marketplace — to decyzja produktowo-prawna, nie techniczna.

**Zalecenie architektoniczne:** schowaj TikToka za interfejsem `VideoProvider` w `tiktok-integration-service`, żeby w razie zmiany zasad dało się dodać własny hosting wideo bez zmian w pozostałych serwisach.

**Wpływ na design:** krok „Zdjęcia i film” w kreatorze oferty ma trzy ścieżki: (A) wybierz film z połączonego konta TikTok, (B) wklej link do filmu, (C) nagraj w Rynku i opublikuj na TikTok (wymaga audytu).

---

## 1. Konwencje wspólne

| Temat | Zasada |
|---|---|
| Prefiks | `/api/v1/...`, gateway kieruje po prefiksie ścieżki |
| Auth | JWT Bearer wystawiany przez `identity-service`, walidowany w gateway i w serwisach (Spring Security Resource Server) |
| Role | `BUYER`, `SERVICE_PROVIDER`, `MERCHANT`, `ADMIN`; jeden użytkownik może mieć kilka |
| Paginacja | kursorowa: `?cursor=...&limit=20` → `{items, nextCursor}` |
| Kwoty | liczba całkowita w groszach + waluta: `{"amount": 38900, "currency": "PLN"}` |
| Błędy | RFC 7807 (`application/problem+json`) |
| Idempotencja | nagłówek `Idempotency-Key` na `POST /orders`, `POST /payments`, `POST /bookings` |
| Język | `Accept-Language: pl-PL`, treści błędów po polsku |
| Ścieżki wewnętrzne | `/internal/...` — tylko między serwisami, nieudostępnione przez gateway |
| Zdarzenia | Kafka, nazwy `domena.zdarzenie` (sekcja 17) |

Legenda kolumny **Dostęp**: `pub` = bez logowania, `user` = zalogowany, `seller` = właściciel kanału (usługodawca lub merchant), `admin`, `int` = wewnętrzny.

---

## 2. `api-gateway` (Spring Cloud Gateway)

Brak logiki biznesowej. Odpowiada za routing, walidację JWT, limity zapytań, CORS i agregację nagłówków.

| Prefiks ścieżki | Serwis docelowy |
|---|---|
| `/api/v1/auth`, `/api/v1/users` | identity-service |
| `/api/v1/channels` | channel-service |
| `/api/v1/offers`, `/api/v1/categories` | catalog-service |
| `/api/v1/media` | media-service |
| `/api/v1/tiktok`, `/api/v1/videos` | tiktok-integration-service |
| `/api/v1/feed` | feed-service |
| `/api/v1/search` | search-service |
| `/api/v1/questions`, `/api/v1/reviews`, `/api/v1/reports` | engagement-service |
| `/api/v1/chats`, `/ws/chat` | chat-service |
| `/api/v1/cart` | cart-service |
| `/api/v1/orders`, `/api/v1/seller/orders` | order-service |
| `/api/v1/payments`, `/api/v1/seller/payouts` | payment-service |
| `/api/v1/services`, `/api/v1/bookings`, `/api/v1/seller/bookings` | booking-service |
| `/api/v1/shipping` | shipping-service |
| `/api/v1/notifications` | notification-service |
| `/api/v1/analytics`, `/api/v1/seller/analytics` | analytics-service |

Zdrowie: `GET /actuator/health` na każdym serwisie.

---

## 3. `identity-service` — konta, logowanie, role, profil

Ekrany: 1 Onboarding, Profil, każdy ekran wymagający logowania.

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| POST | `/api/v1/auth/register` | Rejestracja (e-mail/telefon + hasło) | pub |
| POST | `/api/v1/auth/login` | Logowanie, zwraca access + refresh token | pub |
| POST | `/api/v1/auth/oauth/google` | Logowanie przez Google | pub |
| POST | `/api/v1/auth/refresh` | Odświeżenie tokenu | pub |
| POST | `/api/v1/auth/logout` | Unieważnienie refresh tokenu | user |
| POST | `/api/v1/auth/password/forgot` | Prośba o reset hasła | pub |
| POST | `/api/v1/auth/password/reset` | Ustawienie nowego hasła | pub |
| POST | `/api/v1/auth/phone/verify/request` | Wysłanie kodu SMS | user |
| POST | `/api/v1/auth/phone/verify/confirm` | Potwierdzenie kodu | user |
| GET | `/api/v1/users/me` | Profil zalogowanego | user |
| PATCH | `/api/v1/users/me` | Zmiana danych profilu | user |
| DELETE | `/api/v1/users/me` | Usunięcie konta (RODO) | user |
| PUT | `/api/v1/users/me/roles` | Wybór ról z onboardingu (kupujący / usługodawca / merchant) | user |
| PUT | `/api/v1/users/me/preferences` | Zainteresowania, lokalizacja, motyw jasny/ciemny | user |
| GET | `/api/v1/users/me/addresses` | Lista adresów | user |
| POST | `/api/v1/users/me/addresses` | Dodanie adresu | user |
| PATCH | `/api/v1/users/me/addresses/{addressId}` | Edycja adresu | user |
| DELETE | `/api/v1/users/me/addresses/{addressId}` | Usunięcie adresu | user |
| GET | `/api/v1/users/me/saved` | Zapisane oferty | user |
| PUT | `/api/v1/users/me/saved/{offerId}` | Zapisz ofertę | user |
| DELETE | `/api/v1/users/me/saved/{offerId}` | Usuń z zapisanych | user |
| GET | `/internal/users/{userId}` | Dane użytkownika dla innych serwisów | int |

---

## 4. `channel-service` — kanał sprzedawcy i obserwowanie

Ekrany: 5 Profil sprzedawcy jako kanał. Zakładki kanału korzystają z innych serwisów: **Oferty** → `catalog-service` (`GET /offers?channelId=`), **Filmy** → `tiktok-integration-service`, **Opinie** → `engagement-service`.

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| POST | `/api/v1/channels` | Utworzenie kanału (po wyborze roli usługodawca/merchant) | user |
| GET | `/api/v1/channels/{channelId}` | Dane kanału: nazwa, handle, opis, liczba obserwujących, liczba ofert, ocena, status weryfikacji | pub |
| GET | `/api/v1/channels/by-handle/{handle}` | Kanał po handle (`@elektromarket`) | pub |
| PATCH | `/api/v1/channels/{channelId}` | Edycja nazwy, opisu, danych kontaktowych | seller |
| PUT | `/api/v1/channels/{channelId}/avatar` | Ustawienie awatara (`mediaId` z media-service) | seller |
| PUT | `/api/v1/channels/{channelId}/banner` | Ustawienie banera (`mediaId`) | seller |
| GET | `/api/v1/channels/{channelId}/about` | Zakładka „O nas”: opis, adres, godziny, dane firmy | pub |
| PUT | `/api/v1/channels/{channelId}/business-details` | NIP, nazwa firmy, adres rejestrowy | seller |
| POST | `/api/v1/channels/{channelId}/verification` | Zgłoszenie weryfikacji (dokumenty) | seller |
| GET | `/api/v1/channels/{channelId}/verification` | Status weryfikacji | seller |
| POST | `/api/v1/channels/{channelId}/follow` | Obserwuj | user |
| DELETE | `/api/v1/channels/{channelId}/follow` | Przestań obserwować | user |
| PATCH | `/api/v1/channels/{channelId}/follow` | Włącz/wyłącz powiadomienia (dzwonek) | user |
| GET | `/api/v1/users/me/following` | Lista obserwowanych kanałów (boczne menu, „Obserwowane”) | user |
| GET | `/internal/channels/{channelId}` | Dane kanału dla innych serwisów | int |
| GET | `/internal/channels?ids=` | Wsadowo (karty ofert) | int |

---

## 5. `catalog-service` — oferty (produkty i usługi), kategorie

Ekrany: 4 Karta oferty, 6 Dodawanie oferty, 9 Panel (lista ofert). Serwis jest źródłem prawdy o ofercie; po zmianie publikuje zdarzenia dla wyszukiwarki i feedu.

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| GET | `/api/v1/categories` | Drzewo kategorii (chipy, filtry) | pub |
| GET | `/api/v1/categories/{categoryId}/attributes` | Atrybuty kategorii (parametry w formularzu) | pub |
| POST | `/api/v1/offers` | Utworzenie szkicu oferty (`type`: PRODUCT / SERVICE) | seller |
| GET | `/api/v1/offers/{offerId}` | Karta oferty (cena, opis, parametry, sprzedawca, media, dostawa) | pub |
| PATCH | `/api/v1/offers/{offerId}` | Zapis zmian / autozapis szkicu | seller |
| PUT | `/api/v1/offers/{offerId}/media` | Kolejność mediów: zdjęcia (`mediaId`) + film (`videoId`) | seller |
| PUT | `/api/v1/offers/{offerId}/service-details` | Dla usług: cennik („od…”, jednostka), obszar działania, czas trwania, zaliczka | seller |
| PATCH | `/api/v1/offers/{offerId}/stock` | Stan magazynowy (produkty) | seller |
| GET | `/api/v1/offers/{offerId}/quality-score` | Wskaźnik jakości oferty (podpowiedzi z kreatora) | seller |
| POST | `/api/v1/offers/{offerId}/publish` | Publikacja (walidacja kompletności) | seller |
| POST | `/api/v1/offers/{offerId}/unpublish` | Wstrzymanie oferty | seller |
| POST | `/api/v1/offers/{offerId}/duplicate` | Duplikuj ofertę | seller |
| DELETE | `/api/v1/offers/{offerId}` | Usunięcie | seller |
| GET | `/api/v1/offers/mine?status=` | Moje oferty (panel sprzedawcy), filtr statusu | seller |
| GET | `/api/v1/offers?channelId=&sort=` | Oferty kanału (zakładka „Oferty”) | pub |
| GET | `/api/v1/offers/batch?ids=` | Wsadowe pobranie kart (feed, koszyk, czat) | pub |
| GET | `/internal/offers/{offerId}/price-and-stock` | Aktualna cena i dostępność dla koszyka/zamówienia | int |
| POST | `/internal/offers/{offerId}/stock/reserve` | Rezerwacja zapasu na czas płatności | int |
| POST | `/internal/offers/{offerId}/stock/release` | Zwolnienie rezerwacji | int |

---

## 6. `media-service` — zdjęcia i inne pliki

Wideo obsługuje TikTok, ale zdjęcia ofert, awatary i banery trzeba przechowywać u siebie (S3-kompatybilne, URL-e podpisane).

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| POST | `/api/v1/media/uploads` | Wystawia podpisany URL do wysłania pliku (`purpose`: OFFER_PHOTO, AVATAR, BANNER, KYC_DOCUMENT) | user |
| POST | `/api/v1/media/uploads/{uploadId}/complete` | Potwierdzenie wysłania, start przetwarzania (miniatury, WebP) | user |
| GET | `/api/v1/media/{mediaId}` | Metadane i adresy wariantów | pub |
| PATCH | `/api/v1/media/{mediaId}` | Tekst alternatywny (`altText`) | user |
| DELETE | `/api/v1/media/{mediaId}` | Usunięcie | user |

---

## 7. `tiktok-integration-service` — TikTok (konto, filmy, publikowanie)

Jedyny serwis, który zna `client_key`, `client_secret` i tokeny TikToka. Pozostałe serwisy pytają go tylko o gotowe dane filmu (`videoId`, okładka, `embedUrl`).

### 7.1 Endpointy Rynku

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| GET | `/api/v1/tiktok/connect/url` | Zwraca adres autoryzacji TikTok z parametrem `state` (CSRF) | seller |
| GET | `/api/v1/tiktok/callback` | `redirect_uri` zarejestrowany w TikTok: wymiana `code` na tokeny, zapis zaszyfrowany | pub (z `state`) |
| GET | `/api/v1/tiktok/connection` | Status połączenia (połączone, wygasa, zakres uprawnień) | seller |
| DELETE | `/api/v1/tiktok/connection` | Rozłączenie konta + unieważnienie tokenu | seller |
| POST | `/api/v1/tiktok/sync` | Ręczna synchronizacja filmów | seller |
| GET | `/api/v1/tiktok/me/videos?cursor=` | Filmy z połączonego konta do wyboru w kreatorze oferty | seller |
| POST | `/api/v1/tiktok/me/videos/{externalVideoId}/attach` | Przypisanie filmu do oferty (`offerId`) | seller |
| POST | `/api/v1/videos/by-link` | Dodanie filmu po linku (oEmbed), bez połączenia konta | seller |
| GET | `/api/v1/videos/{videoId}` | Dane odtwarzania: `embedUrl`, `coverUrl` (świeża), czas trwania | pub |
| GET | `/api/v1/offers/{offerId}/video` | Film przypisany do oferty | pub |
| GET | `/api/v1/channels/{channelId}/videos?cursor=` | Zakładka „Filmy” kanału | pub |
| GET | `/api/v1/videos/shorts?cursor=` | Pionowe filmy do strumienia Shorts (źródło dla feedu) | pub |
| POST | `/api/v1/tiktok/posts` | Publikacja filmu z Rynku na TikTok: init posta, zwraca `publishId` i dane wysyłki | seller |
| GET | `/api/v1/tiktok/posts/{publishId}/status` | Status publikacji | seller |
| POST | `/api/v1/tiktok/webhooks` | Odbiornik zdarzeń TikTok (publikacja zakończona/nieudana, film publiczny) | pub (podpis) |
| GET | `/internal/videos?offerIds=` | Wsadowo: filmy i okładki dla kart ofert | int |

Zadania w tle (scheduler): odświeżanie access tokenów przed wygaśnięciem, odświeżanie okładek przez `video/query`, synchronizacja filmów co ok. 12 h, wykrywanie usuniętych/prywatnych filmów i oznaczanie ofert.

### 7.2 Wywołania do TikTok API (wychodzące)

Host: `https://open.tiktokapis.com`, chyba że podano inaczej.

| Cel | Metoda i adres | Scope |
|---|---|---|
| Autoryzacja (przeglądarka) | `GET https://www.tiktok.com/v2/auth/authorize/` (`client_key`, `scope`, `response_type=code`, `redirect_uri`, `state`) | — |
| Wymiana kodu na tokeny | `POST /v2/oauth/token/` (`grant_type=authorization_code`) | — |
| Odświeżenie tokenu | `POST /v2/oauth/token/` (`grant_type=refresh_token`) | — |
| Unieważnienie | `POST /v2/oauth/revoke/` | — |
| Dane konta | `GET /v2/user/info/?fields=open_id,union_id,avatar_url,display_name` | `user.info.basic` |
| Lista filmów | `POST /v2/video/list/` (`max_count` do 20, `cursor`) | `video.list` |
| Szczegóły / odświeżenie okładek | `POST /v2/video/query/` (`filters.video_ids`, do 20) | `video.list` |
| Informacje o twórcy przed postem | `POST /v2/post/publish/creator_info/query/` | — |
| Publikacja bezpośrednia | `POST /v2/post/publish/video/init/` | `video.publish` |
| Wysłanie do skrzynki użytkownika | `POST /v2/post/publish/inbox/video/init/` | `video.upload` |
| Wysłanie pliku | `PUT <upload_url>` (ważny godzinę) | — |
| Status publikacji | `POST /v2/post/publish/status/fetch/` | `video.upload` / `video.publish` |
| Osadzenie po linku | `GET https://www.tiktok.com/oembed?url=` | — |

Pola filmu przydatne dla Rynku: `id`, `title`, `video_description`, `duration`, `width`, `height`, `cover_image_url`, `embed_link`, `embed_html`, `share_url`, `view_count`, `like_count`, `comment_count`.

Limity z dokumentacji: init publikacji 6 żądań/min na token, status 30/min. Bez audytu posty tylko prywatne.

---

## 8. `feed-service` — strona główna, Shorts, „Następne”

Ekrany: 2 Strona główna, Shorts, Obserwowane, karta oferty („Następne”). Serwis składa feed z gotowych danych (zdarzenia z catalog, tiktok, engagement), nie wywołuje ich synchronicznie dla każdego kafla.

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| GET | `/api/v1/feed/home?chip=&cursor=` | Feed rekomendacji; `chip` = kategoria z paska chipów | pub (personalizacja po zalogowaniu) |
| GET | `/api/v1/feed/chips` | Chipy kategorii dla użytkownika (Wszystko, Dla Ciebie, kategorie, Usługi, Okazje) | pub |
| GET | `/api/v1/feed/shorts?cursor=` | Strumień pionowych filmów z powiązanymi ofertami | pub |
| GET | `/api/v1/feed/following?cursor=` | Nowe oferty i filmy z obserwowanych kanałów | user |
| GET | `/api/v1/feed/related?offerId=&cursor=` | „Następne” obok karty oferty | pub |
| POST | `/api/v1/feed/events` | Zdarzenia dla rekomendacji: wyświetlenie, tapnięcie, czas oglądania, „nie interesuje mnie” | pub |

---

## 9. `search-service` — wyszukiwarka i filtry

Ekrany: 3 Wyniki wyszukiwania. Indeks Elasticsearch/OpenSearch budowany ze zdarzeń `offer.*` i `channel.*`.

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| GET | `/api/v1/search/offers` | Wyniki; parametry: `q`, `category`, `type`, `priceMin`, `priceMax`, `city`, `radiusKm`, `condition`, `minRating`, `hasVideo`, `sort`, `cursor`. Odpowiedź zawiera liczniki filtrów (`facets`) i `total` | pub |
| GET | `/api/v1/search/suggest?q=` | Podpowiedzi zapytań | pub |
| GET | `/api/v1/search/channels?q=` | Wyszukiwanie kanałów sprzedawców | pub |
| GET | `/api/v1/search/locations?q=` | Podpowiedzi miejscowości (filtr lokalizacji) | pub |
| GET | `/api/v1/search/history` | Historia wyszukiwań | user |
| DELETE | `/api/v1/search/history` | Wyczyść historię | user |
| GET | `/api/v1/search/saved` | Zapisane wyszukiwania | user |
| POST | `/api/v1/search/saved` | Zapisz wyszukiwanie z powiadomieniami o nowych ofertach | user |
| DELETE | `/api/v1/search/saved/{savedSearchId}` | Usuń zapisane wyszukiwanie | user |
| POST | `/internal/search/reindex` | Pełna reindeksacja | int/admin |

---

## 10. `engagement-service` — pytania (komentarze), opinie, polubienia, zgłoszenia

Ekrany: 4 Karta oferty (pytania), 5 zakładka Opinie, Shorts (polubienia).

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| GET | `/api/v1/offers/{offerId}/questions?sort=&cursor=` | Pytania i odpowiedzi pod ofertą | pub |
| POST | `/api/v1/offers/{offerId}/questions` | Zadanie pytania | user |
| POST | `/api/v1/questions/{questionId}/answers` | Odpowiedź sprzedawcy (wyróżniona) | seller |
| PATCH | `/api/v1/questions/{questionId}` | Edycja własnego pytania | user |
| DELETE | `/api/v1/questions/{questionId}` | Usunięcie własnego pytania | user |
| PUT | `/api/v1/questions/{questionId}/like` | Polub pytanie („przydatne”) | user |
| GET | `/api/v1/channels/{channelId}/reviews?cursor=` | Opinie o sprzedawcy | pub |
| GET | `/api/v1/channels/{channelId}/rating-summary` | Średnia i rozkład ocen | pub |
| POST | `/api/v1/orders/{orderId}/review` | Opinia po zakupie (opinia zweryfikowanego zakupu) | user |
| POST | `/api/v1/bookings/{bookingId}/review` | Opinia po wykonanej usłudze | user |
| POST | `/api/v1/reviews/{reviewId}/reply` | Odpowiedź sprzedawcy na opinię | seller |
| PUT | `/api/v1/offers/{offerId}/like` | Polubienie oferty / Shorta | user |
| DELETE | `/api/v1/offers/{offerId}/like` | Cofnięcie polubienia | user |
| POST | `/api/v1/reports` | Zgłoszenie treści (oferta, pytanie, opinia, czat) | user |
| GET | `/internal/ratings?channelIds=` | Wsadowo: oceny dla kart ofert | int |

---

## 11. `chat-service` — wiadomości

Ekrany: 7 Czat. Wiadomości w czasie rzeczywistym przez WebSocket/STOMP, historia przez REST.

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| GET | `/api/v1/chats?role=buying\|selling&cursor=` | Lista rozmów (Wszystkie / Kupuję / Sprzedaję) | user |
| POST | `/api/v1/chats` | Rozpoczęcie rozmowy z kontekstem oferty (`offerId`) | user |
| GET | `/api/v1/chats/{chatId}` | Rozmowa wraz z przypiętą ofertą | user |
| GET | `/api/v1/chats/{chatId}/messages?cursor=` | Historia wiadomości | user |
| POST | `/api/v1/chats/{chatId}/messages` | Wysłanie wiadomości (tekst, załącznik `mediaId`) | user |
| POST | `/api/v1/chats/{chatId}/read` | Oznaczenie jako przeczytane | user |
| POST | `/api/v1/chats/{chatId}/price-offers` | „Zaproponuj cenę” | user |
| POST | `/api/v1/chats/{chatId}/price-offers/{priceOfferId}/accept` | Akceptacja propozycji ceny | seller |
| POST | `/api/v1/chats/{chatId}/price-offers/{priceOfferId}/decline` | Odrzucenie | seller |
| POST | `/api/v1/chats/{chatId}/block` | Zablokowanie rozmówcy | user |
| GET | `/api/v1/chats/quick-replies?context=` | Szybkie odpowiedzi („Czy nadal aktualne?”…) | user |
| GET | `/api/v1/chats/unread-count` | Licznik nieprzeczytanych (ikona w pasku nawigacji) | user |
| WS | `/ws/chat` | Kanał na żywo: nowa wiadomość, przeczytanie, pisze… | user |

---

## 12. `cart-service` — koszyk

Ekrany: 8 Koszyk.

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| GET | `/api/v1/cart` | Koszyk pogrupowany wg sprzedawcy, z aktualnymi cenami | user |
| POST | `/api/v1/cart/items` | Dodanie pozycji (`offerId`, `quantity`) | user |
| PATCH | `/api/v1/cart/items/{itemId}` | Zmiana ilości | user |
| DELETE | `/api/v1/cart/items/{itemId}` | Usunięcie pozycji | user |
| PUT | `/api/v1/cart/sellers/{channelId}/shipping` | Wybór dostawy dla sprzedawcy (metoda, punkt odbioru) | user |
| POST | `/api/v1/cart/checkout/preview` | Przeliczenie podsumowania: produkty, dostawa, razem | user |
| DELETE | `/api/v1/cart` | Opróżnienie koszyka | user |

---

## 13. `order-service` — zamówienia

Ekrany: 8 Płatność, Profil → Moje zamówienia, 9 Panel → Zamówienia. Statusy zamówień: `Do wysłania`, `W drodze`, `Zrealizowane` (oraz `Oczekuje na płatność`, `Anulowane`).

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| POST | `/api/v1/orders` | Utworzenie zamówienia z koszyka lub „Kup teraz” (`Idempotency-Key`); rezerwuje zapas | user |
| GET | `/api/v1/orders/mine?status=&cursor=` | Moje zamówienia | user |
| GET | `/api/v1/orders/{orderId}` | Szczegóły zamówienia | user / seller |
| GET | `/api/v1/orders/{orderId}/timeline` | Oś statusów | user / seller |
| POST | `/api/v1/orders/{orderId}/cancel` | Anulowanie (przed wysyłką) | user |
| POST | `/api/v1/orders/{orderId}/returns` | Zgłoszenie zwrotu | user |
| GET | `/api/v1/seller/orders?status=&cursor=` | Zamówienia sprzedawcy | seller |
| POST | `/api/v1/seller/orders/{orderId}/ship` | Oznaczenie jako wysłane (numer przesyłki) | seller |
| POST | `/api/v1/seller/orders/{orderId}/status` | Zmiana statusu | seller |
| POST | `/internal/orders/{orderId}/mark-paid` | Oznaczenie jako opłacone (po zdarzeniu z płatności) | int |

---

## 14. `payment-service` — płatności i wypłaty

Ekrany: 8 Płatność, 9 Panel (przychód). Zalecenie: jeden dostawca płatności (np. Przelewy24, PayU albo Stripe — decyzja biznesowa) za interfejsem `PaymentProvider`; BLIK, karta i przelew online. Dane kart nigdy nie przechodzą przez Rynek.

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| GET | `/api/v1/payments/methods` | Dostępne metody płatności | user |
| POST | `/api/v1/payments` | Rozpoczęcie płatności za zamówienie lub zaliczkę rezerwacji (`referenceType`: ORDER / BOOKING, `Idempotency-Key`) | user |
| POST | `/api/v1/payments/{paymentId}/blik` | Potwierdzenie kodem BLIK | user |
| GET | `/api/v1/payments/{paymentId}` | Status płatności | user |
| POST | `/api/v1/payments/webhooks/{provider}` | Webhook dostawcy płatności (weryfikacja podpisu) | pub (podpis) |
| POST | `/api/v1/payments/{paymentId}/refund` | Zwrot środków | seller / admin |
| GET | `/api/v1/seller/payouts` | Historia wypłat | seller |
| GET | `/api/v1/seller/balance` | Saldo i środki oczekujące | seller |
| PUT | `/api/v1/seller/payout-account` | Rachunek do wypłat | seller |

---

## 15. `booking-service` — rezerwacje usług

Ekrany: 4 Karta usługi (kalendarz), 8 Rezerwacja usługi.

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| GET | `/api/v1/services/{offerId}/availability` | Reguły dostępności usługodawcy | seller |
| PUT | `/api/v1/services/{offerId}/availability` | Godziny pracy, wyjątki, urlopy, czas trwania slotu | seller |
| GET | `/api/v1/services/{offerId}/slots?from=&to=` | Wolne sloty (kalendarz i godziny) | pub |
| GET | `/api/v1/services/{offerId}/next-slot` | Najbliższy termin (karta „Najbliższy termin”) | pub |
| POST | `/api/v1/bookings` | Rezerwacja slotu z krótkim przetrzymaniem (np. 10 min), adres i uwagi (`Idempotency-Key`) | user |
| GET | `/api/v1/bookings/mine?status=` | Moje rezerwacje | user |
| GET | `/api/v1/bookings/{bookingId}` | Szczegóły | user / seller |
| POST | `/api/v1/bookings/{bookingId}/cancel` | Anulowanie | user / seller |
| GET | `/api/v1/bookings/{bookingId}/calendar.ics` | Wydarzenie do kalendarza | user |
| GET | `/api/v1/seller/bookings?status=&from=` | Rezerwacje usługodawcy | seller |
| POST | `/api/v1/seller/bookings/{bookingId}/accept` | Akceptacja | seller |
| POST | `/api/v1/seller/bookings/{bookingId}/decline` | Odrzucenie | seller |
| POST | `/api/v1/seller/bookings/{bookingId}/propose-time` | Propozycja innego terminu | seller |
| POST | `/api/v1/seller/bookings/{bookingId}/complete` | Oznaczenie jako wykonane (uruchamia prośbę o opinię) | seller |

---

## 16. `shipping-service`, `notification-service`, `analytics-service`

### 16.1 `shipping-service` — dostawa

Ekrany: 4 Karta oferty (koszt dostawy), 8 Koszyk. Integracje z przewoźnikami (np. InPost, kurierzy) za interfejsem `CarrierProvider`.

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| GET | `/api/v1/shipping/methods?offerIds=&postalCode=` | Metody i ceny dostawy dla pozycji | pub |
| GET | `/api/v1/shipping/pickup-points?lat=&lng=&city=` | Punkty odbioru (paczkomaty) | pub |
| POST | `/api/v1/shipping/shipments` | Utworzenie przesyłki i etykiety | seller |
| GET | `/api/v1/shipping/shipments/{shipmentId}/label` | Etykieta do wydruku | seller |
| GET | `/api/v1/shipping/shipments/{shipmentId}/tracking` | Śledzenie przesyłki | user / seller |
| POST | `/api/v1/shipping/webhooks/{carrier}` | Webhook przewoźnika (zmiana statusu) | pub (podpis) |

### 16.2 `notification-service` — powiadomienia

Wysyłka reaguje na zdarzenia Kafka (nowa wiadomość, zamówienie, rezerwacja, nowy film obserwowanego kanału). Endpointy tylko dla aplikacji klienckich.

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| POST | `/api/v1/notifications/devices` | Rejestracja tokenu push (FCM/APNs) | user |
| DELETE | `/api/v1/notifications/devices/{deviceId}` | Wyrejestrowanie urządzenia | user |
| GET | `/api/v1/notifications?cursor=` | Lista powiadomień (dzwonek) | user |
| POST | `/api/v1/notifications/{notificationId}/read` | Oznaczenie jako przeczytane | user |
| POST | `/api/v1/notifications/read-all` | Oznacz wszystkie | user |
| GET | `/api/v1/notifications/settings` | Ustawienia powiadomień | user |
| PUT | `/api/v1/notifications/settings` | Zmiana ustawień (push, e-mail, kategorie) | user |

### 16.3 `analytics-service` — statystyki sprzedawcy

Ekrany: 9 Panel sprzedawcy. Statystyki filmów (wyświetlenia na TikToku) pobiera z `tiktok-integration-service`.

| Metoda | Ścieżka | Opis | Dostęp |
|---|---|---|---|
| POST | `/api/v1/analytics/events` | Wsadowy odbiór zdarzeń klienta (wyświetlenie oferty, odtworzenie filmu, klik „Kup teraz”) | pub |
| GET | `/api/v1/seller/analytics/overview?range=` | Kafle KPI: wyświetlenia, obejrzenia filmów, zamówienia, przychód + zmiana do poprzedniego okresu | seller |
| GET | `/api/v1/seller/analytics/timeseries?metric=&range=` | Dane do wykresu | seller |
| GET | `/api/v1/seller/analytics/top-offers?range=` | Najlepsze oferty | seller |
| GET | `/api/v1/seller/analytics/offers/{offerId}` | Statystyki jednej oferty (lejek: wyświetlenia → karta → koszyk → zakup) | seller |

---

## 17. Zdarzenia Kafka (kto publikuje, kto słucha)

| Zdarzenie | Publikuje | Słuchają |
|---|---|---|
| `user.registered`, `user.roles-changed` | identity | channel, notification |
| `channel.created`, `channel.updated` | channel | search, feed |
| `channel.followed`, `channel.unfollowed` | channel | feed, notification, analytics |
| `offer.published`, `offer.updated`, `offer.unpublished` | catalog | search, feed, cart, analytics |
| `video.synced`, `video.attached`, `video.removed` | tiktok-integration | catalog, feed, search |
| `question.created`, `question.answered` | engagement | notification |
| `review.created` | engagement | channel (ocena), search, notification |
| `chat.message-sent` | chat | notification |
| `order.created`, `order.paid`, `order.shipped`, `order.completed`, `order.cancelled` | order | catalog (zapas), notification, analytics, engagement |
| `payment.succeeded`, `payment.failed`, `payment.refunded` | payment | order, booking, notification |
| `booking.created`, `booking.confirmed`, `booking.cancelled`, `booking.completed` | booking | notification, analytics, engagement |
| `shipment.status-changed` | shipping | order, notification |

---

## 18. Ekran → wywołania (ściąga dla frontendu w Angularze)

| Ekran | Główne wywołania |
|---|---|
| 1 Onboarding | `PUT /users/me/roles`, `PUT /users/me/preferences`, `GET /categories`, `GET /search/locations` |
| 2 Strona główna | `GET /feed/chips`, `GET /feed/home`, `GET /feed/shorts`, `POST /feed/events`, `GET /videos/{id}` |
| 3 Wyniki | `GET /search/offers`, `GET /search/suggest`, `GET /categories`, `POST /search/saved` |
| 4 Karta oferty (produkt) | `GET /offers/{id}`, `GET /offers/{id}/video`, `GET /offers/{id}/questions`, `GET /feed/related`, `POST /cart/items`, `POST /chats`, `PUT /users/me/saved/{id}` |
| 4 Karta usługi | `GET /offers/{id}`, `GET /services/{id}/slots`, `POST /bookings` |
| 5 Kanał sprzedawcy | `GET /channels/by-handle/{h}`, `POST /channels/{id}/follow`, `GET /offers?channelId=`, `GET /channels/{id}/videos`, `GET /channels/{id}/reviews` |
| 6 Dodawanie oferty | `POST /offers`, `PATCH /offers/{id}`, `GET /categories/{id}/attributes`, `POST /media/uploads`, `GET /tiktok/me/videos`, `POST /videos/by-link`, `PUT /offers/{id}/media`, `GET /offers/{id}/quality-score`, `POST /offers/{id}/publish` |
| 7 Czat | `GET /chats`, `GET /chats/{id}/messages`, `POST /chats/{id}/messages`, `POST /chats/{id}/price-offers`, WS `/ws/chat` |
| 8 Koszyk i płatność | `GET /cart`, `PUT /cart/sellers/{id}/shipping`, `GET /shipping/pickup-points`, `POST /orders`, `POST /payments`, `POST /bookings`, `GET /payments/{id}` |
| 9 Panel sprzedawcy | `GET /seller/analytics/overview`, `GET /seller/analytics/timeseries`, `GET /seller/orders`, `GET /offers/mine`, `GET /seller/bookings`, `GET /seller/balance` |

---

## 19. Sugerowana kolejność budowy

1. `api-gateway`, `identity-service`, `channel-service` — konta i kanały.
2. `media-service`, `catalog-service` — oferty ze zdjęciami (bez wideo).
3. `search-service`, `feed-service` — odkrywanie ofert.
4. `tiktok-integration-service` — połączenie konta, lista filmów, osadzanie; **najpierw ścieżka z linkiem (oEmbed) i z `video.list`**, publikowanie na TikTok dopiero po decyzji o audycie.
5. `cart-service`, `order-service`, `payment-service`, `shipping-service` — sprzedaż produktów.
6. `booking-service` — usługi.
7. `chat-service`, `engagement-service`, `notification-service`.
8. `analytics-service` — panel sprzedawcy.

Jeśli chcesz mniej serwisów na start: `identity` + `channel` można scalić, `cart` + `order` też, a `notification` i `analytics` odłożyć. Granice w tym dokumencie są jednak takie, żeby późniejszy podział nie wymagał zmiany adresów.

---

Źródła TikTok (dokumentacja używana przy tym dokumencie):
- https://developers.tiktok.com/doc/display-api-get-started/
- https://developers.tiktok.com/docs/en/tiktok-api-v2-video-list
- https://developers.tiktok.com/docs/en/tiktok-api-v2-video-query
- https://developers.tiktok.com/doc/content-posting-api-get-started
- https://developers.tiktok.com/docs/en/content-posting-api-reference-direct-post
- https://developers.tiktok.com/docs/en/content-posting-api-reference-upload-video
- https://developers.tiktok.com/docs/en/content-posting-api-reference-get-video-status
- https://developers.tiktok.com/doc/login-kit-web
- https://developers.tiktok.com/doc/login-kit-manage-user-access-tokens
- https://developers.tiktok.com/docs/en/embed-videos
