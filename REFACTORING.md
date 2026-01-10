# GTFS Fusion - Exploratory Refactoring Documentation

This document identifies repeating patterns, extracts common abstractions, and prepares the ground for further code organization.

**Goal**: More questions than answers - exploratory analysis of the codebase architecture.

---

## Data Processing Pipeline Overview

The GTFS data flows through several processing stages:

### 1. PARSE (CSV → Raw Objects)
- ZIP extraction & CSV parsing
- Worker-based parallel parsing for performance
- Produces raw JavaScript objects

### 2. NORMALIZE (Raw → Standardized)
- `normalizeKey`: CSV headers → GTFS standard field names
- `normalizeRecord`: Clean values (trim, remove quotes, handle nulls)
- Output: Clean GTFS-compliant records

### 3. INDEX (Records → Lookup Tables)
- Build indexes: `stopTimesIndex`, `stopsIndex`, `shapesIndex`, etc.
- Group related data for fast lookup
- Output: Queryable data structures

### 4. ENRICH (Add Computed Fields)
- `enrichTripsWithDirectionId`: Generate missing direction_id
- Shape simplification: Reduce coordinate density
- Route pattern analysis: Build canonical trip patterns
- Output: Enhanced domain model

### 5. CACHE (Persist Processed Data)
- `cacheShapes`: Store simplified shapes in localStorage
- Compression: Delta encoding + base36
- TTL: 30 days
- Output: Faster subsequent loads

### 6. RENDER (Display in UI)
- Map visualization (Leaflet)
- Timetable generation
- `escapeHtml`: Sanitize for safe HTML rendering
- Output: Interactive web interface

---

## 1. Normalizacja danych wejściowych (keys / values)

### Obecna struktura:
- `normalizeKey` - Normalizacja kluczy CSV do standardowych nazw GTFS
- `normalizeRecord` - Normalizacja całego rekordu (wartości)
- `escapeHtml` - Escapowanie HTML dla bezpieczeństwa UI
- `KEY_ALIAS` - Mapa aliasów pól GTFS
- Rozproszona logika trimowania, usuwania cudzysłowów

### Pytania i odpowiedzi:

**Q: Czy normalizacja kluczy i wartości to ten sam etap pipeline'u?**

A: Obecnie tak, ale służą różnym celom:
- `normalizeKey`: CSV → nazwy pól GTFS (deterministyczne, cache'owalne)
- `normalizeRecord`: Czyszczenie wartości CSV (cudzysłowy, whitespace)
- `escapeHtml`: Sanityzacja UI (osobna warstwa prezentacji)

**Q: Czy chcemy rozróżniać normalizację CSV, GTFS i UI?**

A: Tak - obecnie są rozdzielone:
- Normalizacja CSV: `normalizeRecord` (usuwa BOM, cudzysłowy, trim)
- Normalizacja GTFS: `normalizeKey` + `KEY_ALIAS` (standaryzacja nazw pól)
- Normalizacja UI: `escapeHtml` (zapobieganie XSS, bezpieczeństwo renderowania)

**Q: Czy KEY_ALIAS powinien być wstrzykiwalny (np. per feed)?**

A: Obecnie jest globalną stałą, ale design pozwala na rozszerzenie przez config.
Przyszłość: Można by scalić z wzorcem `state.customRouteTypeMap`.

**Q: Czy normalizeKey jest deterministyczne i cache'owalne globalnie?**

A: Tak - `keyCache` zapewnia memoizację. Funkcja pure, stabilne mapowanie.

**Q: Czy normalizeRecord powinno przyjmować „profil" (strict / loose)?**

A: Obecnie jeden tryb. Przyszłe ulepszenie: konfigurowalne poziomy walidacji.

**Q: Czy w przyszłości chcemy walidować typy już na tym etapie?**

A: Nie obecnie. Walidacja typów mogłaby być dodana jako osobny etap po normalizacji.

### Hipoteza:
Warto traktować to jako ETL step, a nie zbiór helperów. Moduły są dobrze zdefiniowane (sekcja 3.1).

---

## 2. Czas i daty GTFS

### Obecna struktura:
- `timeToMinutes` - Konwersja czasu GTFS na minuty
- `minutesToTime` - Konwersja minut na format GTFS
- `formatTime` - Formatowanie czasu dla UI
- `parseGTFSDate` - Parsowanie daty GTFS
- `formatDateToGTFS` - Formatowanie daty do GTFS

### Pytania i odpowiedzi:

**Q: Czy „czas" wewnętrznie powinien zawsze być w minutach?**

A: Tak - `timeToMinutes` jest kanonicznym formatem wewnętrznym.
Pozwala na prawidłową obsługę >24h (wymóg specyfikacji GTFS dla nocnych połączeń).

**Q: Czy format HH:MM:SS > 24h to logika domenowa czy prezentacyjna?**

A: Logika domenowa. Specyfikacja GTFS jawnie to dopuszcza (patrz moduł 3.2).
`formatTime` zapewnia warstwę prezentacji z zawijaniem dla wyświetlania.

**Q: Czy formatTime powinno znać reguły dnia (wrap 24h)?**

A: Tak - zawija ≥24h do zakresu 0-23 dla przyjaznego użytkownikowi wyświetlania.

**Q: Czy data GTFS zawsze jest w lokalnej strefie czasu?**

A: Tak - specyfikacja GTFS: YYYYMMDD bez informacji o strefie czasowej (domyślnie czas lokalny).

**Q: Czy planujemy obsługę feedów z różnymi TZ?**

A: Nie obecnie. Wymagałoby to znacznego refaktoringu logiki kalendarza.

**Q: Czy daty są tylko filtrem, czy częścią cache key?**

A: Oba. Używane w: filtrowaniu kalendarza ORAZ generowaniu klucza `columnOrderCache`.

### Hipoteza:
Jeden moduł Time/Date uprości myślenie o logice rozkładów. Obecnie dobrze zorganizowany w sekcji 3.2.

---

## 3. Geometria i geografia

### Obecna struktura:
- `haversineDistance` - Odległość między punktami (WGS84)
- `calculateBearing` - Azymut między punktami
- `simplifyDouglasPeucker` - Upraszczanie linii
- `simplifyShapes` - Upraszczanie wszystkich kształtów
- `calculateShapeCoverage` - Pokrycie przystanków przez kształty
- `fillShapeGaps` - Wypełnianie luk w kształtach

### Pytania i odpowiedzi:

**Q: Czy operujemy na WGS84 wszędzie, czy to założenie niejawne?**

A: Tak - niejawne założenie (standard GTFS). Zobacz moduł 3.3.

**Q: Czy tolerancje (np. 100m) są stałe czy powinny być konfigurowalne?**

A: Obecnie stałe (`MAX_DISTANCE_METERS=100m` w `calculateShapeCoverage`).
Tolerancja `simplifyDouglasPeucker` jest parametryzowalna (domyślnie 0.0001°).

**Q: Czy coverage shape↔stops to heurystyka czy twarda reguła?**

A: Heurystyka do oceny jakości, nie walidacja (patrz `calculateShapeCoverage`).

**Q: Czy simplifyShapes jest elementem renderingu czy parsowania?**

A: Parsowania/cache'owania. Wykonywane raz podczas ładowania danych, cache w localStorage.

**Q: Czy te funkcje mają sens poza GTFS (potencjał reuse)?**

A: Tak - funkcje geo są generyczne (`haversineDistance`, `calculateBearing`, itp.).
Można wyodrębnić do osobnej biblioteki narzędzi geo.

**Q: Czy uproszczone shapes powinny zależeć od zoom level?**

A: Nie - pojedynczy poziom uproszczenia. Leaflet obsługuje renderowanie zoom.

### Hipoteza:
To jest jeden „Geo layer", obecnie dobrze zorganizowany w module 3.3.

---

## 4. Cache i serializacja shapes

### Obecna struktura:
- `compressCoordinates` - Kompresja współrzędnych (delta + base36)
- `decompressCoordinates` - Dekompresja współrzędnych
- `hashShapesData` - Generowanie klucza cache
- `cacheShapes` - Zapis do localStorage
- `loadCachedShapes` - Odczyt z localStorage

### Pytania i odpowiedzi:

**Q: Czy localStorage to docelowy backend cache?**

A: Tak - odpowiedni dla SPA po stronie klienta. Brak cache po stronie serwera.

**Q: Czy hashShapesData jest wystarczająco odporny na kolizje?**

A: Wystarczający dla kluczy cache (nie kryptograficzny). Prosty hash na ID kształtów.

**Q: Czy TTL 30 dni jest arbitralny?**

A: Tak - rozsądna równowaga między starzeniem się danych a użytecznością cache.

**Q: Czy wersjonowanie cache (version: 2) powinno być jawniejsze?**

A: Już jest jawne (pole `version: 2`). Pozwala na migrację formatu.

**Q: Czy cache dotyczy tylko shapes, czy też np. routeProfiles?**

A: Obecnie tylko shapes. Design jest generyczny - można rozszerzyć na `routeProfiles`.

**Q: Czy cleanup cache powinien być deterministyczny?**

A: Obecnie best-effort przy `QuotaExceededError`. Brak systemu LRU/priorytetów.

**Q: Czy kompresja delta+base36 jest wystarczająco stabilna?**

A: Stabilna, stratna do 6 miejsc po przecinku (≈0.11m precyzji, akceptowalna dla transportu).

### Hipoteza:
To jest generyczny cache adapter, obecnie używany tylko dla shapes. Moduł 3.4 dobrze dokumentuje.

---

## 5. Analiza kierunku tras (direction_id)

### Obecna struktura:
- `getTripStopSequence` - Pobieranie sekwencji przystanków dla kursu
- `tripSequenceScore` - Scoring dopasowania sekwencji
- `isCircularRoute` - Detekcja tras okrężnych
- `enrichTripsWithDirectionId` - Główna funkcja wzbogacania
- `calculateBearing` - Użyte jako tiebreaker

### Pytania i odpowiedzi:

**Q: Czy direction_id to właściwie klasyfikacja binarna?**

A: Tak - specyfikacja GTFS pozwala 0/1. Przypadki brzegowe (okrężne, rozgałęzienia) obsługiwane specjalnie.

**Q: Co w przypadku tras >2 kierunków (pętle, odnogi)?**

A: Używa heurystyk: najdłuższy kurs jako wzorzec, dopasowanie podciągu.
Rozgałęzienia obsługiwane przez podciąg (ignoruje przystanki specyficzne dla odgałęzienia).

**Q: Czy „najdłuższy trip" to zawsze dobry wzorzec?**

A: Nie - heurystyka. Działa w większości przypadków, ale może zawieść dla złożonych sieci.

**Q: Czy subsequence matching ignoruje zbyt dużo informacji?**

A: Tak - celowo ignoruje przystanki na odnogach. Trade-off dla prostoty.

**Q: Czy bearing jako tiebreaker jest zawsze sensowny?**

A: Zwykle tak. Fallback na score-only gdy brak danych geo.

**Q: Co jeśli stopsIndex jest niekompletny?**

A: Graceful degradation: kursy bez przystanków dostają `direction_id=0`.

**Q: Czy chcemy móc debugować decyzję direction_id per trip?**

A: Nie jest obecnie logowane. Przyszłość: Dodać tryb debug ze śladem decyzji.

**Q: Czy direction_id powinien być deterministyczny między feedami?**

A: Nie - zależy od wzorców specyficznych dla feeda. Deterministyczny w ramach feeda.

### Hipoteza:
To jest pipeline decyzyjny, nie jedna funkcja. Dobrze udokumentowany w module 3.6 z opisem wszystkich kroków.

---

## 6. Heurystyki jakości danych

### Obecna struktura:
- `mostCommonString` - Znajdowanie najczęstszego stringa (konsensus)
- `looksLikeGarbageLabel` - Detekcja nieinformatywnych etykiet

### Pytania i odpowiedzi:

**Q: Czy te heurystyki są GTFS-specyficzne?**

A: Tak - polegają na konwencjach nazewnictwa GTFS (`short_name`, `long_name`).

**Q: Czy „garbage label" to pojęcie domenowe czy UI?**

A: Logika domenowa - określa użyteczność danych, nie tylko wyświetlanie.

**Q: Czy short_name vs long_name powinno mieć scoring?**

A: Niejawnie w `looksLikeGarbageLabel` - odrzuca `long_name` tylko-numeryczne.

**Q: Czy chcemy fallbacki zależne od agencji?**

A: Nie obecnie. Można rozszerzyć o reguły specyficzne dla agencji w przyszłości.

**Q: Czy takie heurystyki powinny być konfigurowalne?**

A: Obecnie hardcoded. Przyszłość: Reguły jakości oparte na konfiguracji.

### Hipoteza:
To są reguły jakości danych, warto trzymać razem. Moduł 3.5 je grupuje.

---

## 7. Kolejność wywołań i przepływ danych

### Pytania i odpowiedzi:

**Q: Które funkcje są „pure", a które mutują state?**

A: Oznaczone tagami `@pure` / `@impure`. Zobacz przegląd pipeline w sekcji 3.

**Q: Gdzie kończy się parsing, a zaczyna logika domenowa?**

A: Parsing → normalize → index → enrich (patrz DATA PROCESSING PIPELINE).

**Q: Czy worker parsing i main-thread parsing powinny dzielić utils?**

A: Tak - `normalizeKey`/`Record` używane w obu kontekstach (pure, shareable).

**Q: Czy kolejność parse → normalize → enrich → cache → render jest gdzieś jawnie opisana?**

A: Tak - zobacz nagłówek sekcji 3 dla kompletnej dokumentacji pipeline.

**Q: Czy state jest jedynym źródłem prawdy?**

A: Tak - obiekt `state` przechowuje wszystkie dane aplikacji. Brak ukrytego globalnego stanu.

**Q: Czy możliwe jest testowanie tych etapów osobno?**

A: Tak - funkcje pure są testowalne. Funkcje impure trudniejsze (DOM/localStorage).

---

## Adnotacje funkcji

W kodzie używamy następujących adnotacji:

- `@pure` = Brak efektów ubocznych, ten sam input → ten sam output (można memoizować)
- `@impure` = Ma efekty uboczne (aktualizacje DOM, localStorage, mutacje)
- `@cached` = Wyniki są memoizowane wewnętrznie
- `@presentation` = Logika specyficzna dla UI (nie logika domenowa)

---

## Modularność kodu

Funkcje są zorganizowane według zagadnień:

### 3.1: Normalizacja danych (Czyszczenie CSV/GTFS)
- `escapeHtml`, `normalizeKey`, `normalizeRecord`

### 3.2: Czas/Data (Logika temporalna GTFS)
- `timeToMinutes`, `minutesToTime`, `formatTime`, `parseGTFSDate`, `formatDateToGTFS`

### 3.3: Geometria/Geografia (Kalkulacje WGS84)
- `haversineDistance`, `calculateBearing`, `simplifyDouglasPeucker`, `calculateShapeCoverage`, `fillShapeGaps`

### 3.4: Cache/Serializacja (Persystencja localStorage)
- `compressCoordinates`, `decompressCoordinates`, `hashShapesData`, `cacheShapes`, `loadCachedShapes`

### 3.5: Jakość danych (Heurystyki dla jakości feedów)
- `mostCommonString`, `looksLikeGarbageLabel`

### 3.6: Wzbogacanie kierunku (Generowanie direction_id)
- `getTripStopSequence`, `tripSequenceScore`, `isCircularRoute`, `enrichTripsWithDirectionId`

---

## Wnioski i następne kroki

### Co działa dobrze:
✅ Jasny pipeline przetwarzania danych (6 etapów)
✅ Funkcje pogrupowane tematycznie w moduły
✅ Dobra separacja warstw (normalizacja, domena, prezentacja)
✅ Adnotacje czystości funkcji (`@pure`, `@impure`)

### Co można ulepszyć:
🔸 Konfigurowalne heurystyki (tolerancje, reguły jakości)
🔸 Wstrzykiwalne aliasy kluczy per-feed
🔸 Tryb debug dla decyzji `direction_id`
🔸 Testowanie jednostkowe (szczególnie funkcji pure)
🔸 Rozszerzenie cache na inne dane (routeProfiles)

### Potencjalne kolejne PR-y:
1. Ekstrakcja modułów geo do osobnej biblioteki (reuse)
2. System konfiguracji dla heurystyk i tolerancji
3. Testy jednostkowe dla funkcji pure
4. Rozszerzenie systemu cache
5. Walidacja typów w etapie normalizacji
