# Dodawanie światła

Konfiguracja urządzeń jest w `S85.Core`. `FB_Home.aLights` to statyczna tablica rekordów `ST_LightInfo`: `sLocationId`, lokalne `sId`, `sType` (`light`) i snapshot `bOn`. Tożsamość światła to para lokalizacja/id, nie indeks. Przed pierwszym wywołaniem `FB_Home` zawartość jest domyślna i nie stanowi gotowego katalogu. Po `fbHome(...)` czytaj katalog w tej samej task, a snapshot odpowiada logicznemu wyjściu z tego skanu. Atomowość między taskami nie jest zapewniona.

## Identyfikatory i katalog

Lokalizacja ma format `Area.Code.Floor.Code-Location.Number`, z kodem piętra i numerem lokalizacji zapisanymi na dwóch cyfrach. Potwierdzone wartości to Workshop `H.01-09`, Entryway `H.01-01` oraz Hall `H.01-02`. Hall odpowiada House H, Floor 1 i LocationNumber 2. Zobacz [dokumentację domu](https://app.notion.com/p/c23e05ddd8c84e09a17cc938f241f1bd), [rekord Entryway](https://app.notion.com/p/b1520dc6bc7b429da33e74b833b5d88d) oraz [rekord Hall](https://app.notion.com/p/4710f5a1fcd64f0abc083e8c376b1760). Lokalne id Workshop to `main`, Entryway to `main` i `temporary`, a Hall to `main`, `outer_spots` i `middle_spot`.

Pola są `STRING(16)`. Zachowaj wartość mieszczącą się w 16 znakach i używaj krótkich semantycznych segmentów, np. małych liter, cyfr i podkreśleń (`main`, `temporary`, `wall_west`), bez lokalizacji ani pozycji w katalogu. W Core nie ma lookup po id: błędny id niczego nie wybiera i nie steruje światłem. Testy konfiguracji mają sprawdzać niepuste i unikalne pary, ale błąd katalogu nie blokuje lokalnego sterowania.

Do bezpiecznej enumeracji użyj liczby elementów wyprowadzonej z rozmiaru tablicy, nie wpisuj jej osobno w kliencie:

```iecst
fbHome(stInputs := stInputs, stOutputs => stOutputs);
nFirst := 1;
nCount := SIZEOF(fbHome.aLights) / SIZEOF(fbHome.aLights[1]);
nLast := nFirst + nCount - 1;
FOR nIndex := nFirst TO nLast DO
    sLocation := fbHome.aLights[nIndex].sLocationId;
    sId := fbHome.aLights[nIndex].sId;
    bOn := fbHome.aLights[nIndex].bOn;
END_FOR
```

## Checklist Core

Wspólne zachowanie przełączania przyciskiem i obsługi poleceń światła znajduje się w `S85.Core/Domains/Lighting/POUs/FB_ButtonLight.TcPOU`. Wykorzystaj je w adapterze lokalizacji, a `FB_Location_Workshop` pozostaw adapterem kontraktu Workshop.

Liczba wymaganych plików zależy od istniejących struktur i adapterów. Sprawdź aktualny kod, pliki projektu oraz testy przed określeniem zakresu zmiany. Dla nowego sygnału sprzętowego wymagane jest zatwierdzone mapowanie w `S85.PROD/S85.PROD.tsproj`. Nie zmieniaj sprzętu ani mapowań bez autoryzacji.

1. Dodaj wejście i wyjście w `ST_EntrywayInputs`/`ST_EntrywayOutputs` tylko gdy są potrzebne. Zachowaj logiczny kontrakt niezależny od sprzętu.
2. W `FB_Location_Entryway` dodaj instancję `FB_ButtonLight` i stałą nowego id, np. `cWallWestLightId := 'wall_west'`. Wykorzystaj istniejące `cLocationId`, nie definiuj lokalizacji ponownie. Wywołaj FB z wejściem przycisku i przypisz `bLightRelay` do nowego wyjścia. Następnie opublikuj `ST_LightInfo` z tymi id i stanem wyjścia.
3. Powiększ statyczną tablicę lokalizacji i `FB_Home.aLights`; przepisz snapshoty po wywołaniu lokalizacji. Metadane lokalizacji przechowuj w Core, nie duplikuj konfiguracji w adapterach.
4. Dodaj testy konfiguracji katalogu: niepuste pola, unikalne pary, zachowanie stałych id po przełączeniach oraz zgodność `bOn` z logicznym wyjściem.

## LOCAL i PROD

Logika jest w Core, symulacja i sprzęt są osobnymi adapterami. W LOCAL sprawdź konkretne pliki `S85.LOCAL/S85.LOCAL/Infrastructure/IO/Simulation/POUs/FB_SimulationInputs.TcPOU`, `FB_SimulationOutputs.TcPOU`, `S85.LOCAL/S85.LOCAL/Infrastructure/IO/Simulation/GVLs/GVL_EntrywaySimulation.TcGVL` oraz analogiczny `GVL_WorkshopSimulation.TcGVL`. Dodaj sygnały do GVL i adapterów symulacji, a TcUnit testy w `S85.LOCAL/S85.LOCAL/Tests`. Zachowaj istniejące testy.

PROD używa `S85.PROD/S85/Infrastructure/IO/Beckhoff/POUs/FB_HardwareInputs.TcPOU`, `FB_HardwareOutputs.TcPOU`, `S85.PROD/S85/Infrastructure/IO/Beckhoff/GVLs/GVL_EntrywayIO.TcGVL` i `GVL_WorkshopIO.TcGVL`. Aktualizuj adaptery tylko dla zaakceptowanych sygnałów. Fizyczne mapowanie EtherCAT w `S85.PROD/S85.PROD.tsproj` wymaga osobnej zgody; nie jest częścią zmiany logiki Core. Nie zmieniaj sprzętu, mapowań, tasków ani targetów bez autoryzacji.

## Weryfikacja

Uruchom symulację i TcUnit wyłącznie w LOCAL; nie traktuj nieuruchomionych testów jako zaliczonych. Buduj tylko zmieniony projekt systemowy zgodnie z [instrukcją XAE](twincat-xae.md). Nie wykonuj online ani aktywacji bez odrębnej zgody. Weryfikację fizycznego mapowania prowadź oddzielnie po jego zatwierdzeniu.
