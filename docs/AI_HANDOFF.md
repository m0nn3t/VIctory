# Handoff dla kolejnego AI: VIctory, translacja shaderow

Data przekazania: 2026-11-25

## Cel tej notatki

Ten dokument przekazuje kontekst projektu, problem renderowania GTA VI oraz stan pracy nad obsluga root UAV. Kolejny agent ma najpierw zabezpieczyc i zrozumiec aktualne lokalne zmiany, a dopiero potem kontynuowac implementacje. Nie zakladaj, ze aktualny branch jest czysty ani ze ostatni build/test nadal przechodzi.

## Projekt

VIctory to natywna warstwa translacyjna dla uruchamiania builda Xbox Series X|S GTA VI na Windows. Kod tytulu x86-64 jest uruchamiany w procesie hosta; VIctory implementuje brakujace uslugi konsoli i mapuje GPU D3D12.X na hostowe D3D12. To nie jest emulator calego systemu ani renderer gry napisany od zera.

Shader pipeline jest w `crates/victory-shader`:

1. Odczytuje kontenery shaderow Xbox i natywny kod GFX10.
2. `ir::build` dekoduje GFX10 do SSA.
3. `structure` zamienia semantyke EXEC/wave lane masks na per-lane warunki i flow control.
4. `resource` sledzi adresy i deskryptory do root parameters, tabel i heapow.
5. `emit` obniża SSA do DXIL.
6. `victory-gpu` tworzy hostowe pipeline'y D3D12, root signature i bindowania.
7. Wyniki translacji/cache sa uzywane przy tworzeniu pipeline'u w runtime. `victory translate --cache` jest warm-upem tego samego cache, a nie odrebnym formatem translacji.

Szczegolowy opis architektury znajduje sie w `docs/architecture.md`.

## Sprzet i objawy

Sprzet podany przez uzytkownika: Windows 11, Intel i5-13600K, 32 GB DDR5-6400, AMD RX 9060 XT 16 GB, NVMe. README opisuje wczesniejszy test na RX 6700 XT; nie utozsamiaj tych dwoch maszyn.

Gra uruchamia sie, menu dzialaja, tytul laduje do swiata pierwszej misji. Wczesniejszy raport z runtime mowil o ok. 2.4 FPS / 425 ms na klatke i brakujacych elementach sceny. Nowsze wyniki `victory translate --validate` pokazuja, ze duza czesc shaderow nie dochodzi do emisji DXIL albo walidacji. Nieztranslowane shadery maja pipeline/draw/dispatch odrzucany przez runtime, wiec brak geometrii i compute moze byc bezposrednim skutkiem.

Wymagania repozytorium dot. assetow sa scisle: zadne pliki gry, zrzuty shaderow, disassembly, logi zawierajace dane gry ani pliki pochodne nie moga byc dodawane do repo. `shader_errors.log` zawieral podsumowanie z runtime/translate i nie powinien byc commitowany. Testy syntetyczne sa dozwolone.

## Najnowszy raport translacji przekazany przez uzytkownika

Komenda uzyta przez uzytkownika:

```powershell
E:\gry\GTA6.Xbox.v1.0.2.1-CYBERLEEK\VIctory\target\release\victory.exe translate E:\gry\GTA6.Xbox.v1.0.2.1-CYBERLEEK\shaders.rpf --validate
```

Wyniki ogolne:

- 26,503 kontenery, 614 duplikatow pominietych.
- 26,084 z 26,084 programow zbudowalo IR.
- 26,084 z 26,084 przeszlo structurisation.
- Resources: 25,724 z 26,084 tracked.
- DXIL vertex/pixel: 11,468 emitted z 14,027; 11,405 validated.
- DXIL compute: 1,357 emitted z 2,582; 1,333 validated.
- DXIL mesh: 4,888 emitted z 8,713; 4,830 validated.
- `validated` to liczba shaderow poprawnie zweryfikowanych przez DXIL validator; nie oznacza jeszcze dzialajacego pipeline na docelowym GPU.

Najczestsze odmowy z tego raportu, wedlug podanych liczebnosci:

1. 5,803 `a root UAV or typed root view`.
2. 553 `a 64-bit image atomic`.
3. 281 `a select on an untracked mask`.
4. 260 `a lane mask holding lanes elsewhere read as data`.
5. 207 `an interpolation with computed barycentrics`.
6. 203 `a lane mask read as data`.
7. 195 `an address in an SGPR that is no root parameter`.
8. 136 `a packed input half no element holds`.

Jest wiele dalszych mniejszych kategorii. Raport grupuje odmowy po opisie; duplikaty shaderow byly pomijane. Te wyniki to stan z konkretnego uruchomienia, nie gwarancja dla obecnego zrodla.

## Postep: etap 1 wykonany (2026-11-25)

Zaimplementowano end-to-end root UAV / root typed buffer support:

- `victory-shader` tworzy props-aware root UAV handles przez DXIL `createHandleFromBinding` i `annotateHandle`; emituje metadata dla raw, structured i typed buffer kinds.
- `Lowered` i `HostShader` przenosza root UAV slots razem z memory/bindless flags.
- `victory-gpu` dodaje root UAV parameters do host root signature i bindowanie GPU virtual address przez `SetGraphicsRootUnorderedAccessView` / `SetComputeRootUnorderedAccessView`.
- Root UAV shadow ranges sa oznaczane jako GPU-written, aby zapisy mogly wrocic do pamieci tytulu.
- Shader cache zapisuje root UAV slots i memory/bindless flags; format cache podniesiono z `VSH1` do `VSH2`.
- Test `root_uav_raw_and_structured_lower` obejmuje raw, structured i typed buffer store. Przechodzi; DXIL validator jest uruchamiany w tescie, jesli jest dostepny.
- Test shader-cache round-trip przechodzi.

Po `cargo build --release -p victory` uruchomiono ponownie lokalny pack:

```powershell
target\release\victory.exe translate ..\shaders.rpf --validate
```

Wyniki: wszystkie 26,503 kontenery i 26,084 programy przetworzone; resources tracked 25,886/26,084. DXIL: vertex/pixel 14,205 emitted / 14,205 validated z 14,247; compute 2,621 emitted / 1,886 validated z 2,679; mesh 8,765 emitted / 8,694 validated z 8,765. Odmowa `a root UAV or typed root view` nie wystapila juz w raporcie.

Nowe najczestsze poz zostale problemy z tego uruchomienia: 735 shaderow z bledem validatora dotyczacym niekompatybilnej operacji entry stage; 195 adresow w SGPR poza root parameters; 71 shaderow mesh z output size > 32768; 40 rejestrow laczacych lane mask i data, odczytanych jako data; 36 descriptor reads only as data; 8 lane mask reads as data; 6 untracked lane-mask joins. Stage 2 powinien teraz skupic sie na maskach/selectach, ale zaczac od odtworzenia konkretnych shaderow/hashy, nie od szerokiej zmiany heurystyk.

Weryfikacja kodu: `cargo test -p victory-shader` przechodzi (212 passed, 2 ignored); `cargo test -p victory-gpu` przechodzi (78 passed, 1 ignored); `cargo test --workspace --exclude xtask` przechodzi. `cargo build --release -p victory` przechodzi. Pelne `cargo test --workspace` nadal konczy sie porazka tylko w `xtask::policy::tests::repository_passes_policy`, bo skan repo wykrywa istniejace uzytkownika logi/pliki tymczasowe niedozwolone przez asset policy. Nie kasuj tych plikow bez zgody.

## Uzgodniony plan: trzy osobne etapy

Nie implementowac wszystkich trzech naraz. Kazdy etap ma konczyc sie testami workspace, a po zmianie translatora warto uruchomic ponownie `victory translate --validate`, by porownac liczebnosci.

### Etap 1 (teraz): root UAV / typed root view

Powod priorytetu: ta kategoria jest najwieksza (5,803). Obecna architektura ma juz przeplyw root CBV i root SRV; root UAV musi zostac przeprowadzony end-to-end przez shader metadata, DXIL resource metadata, D3D12 root signature, bindowanie adresu i cache.

Oczekiwany przeplyw do zbadania i zaimplementowania:

- Rozpoznac, kiedy `Source::Built` wskazuje na `RootType::Descriptor` i odpowiadajacy mu `UsedResource.kind == Uav`.
- W `victory-shader::emit` utworzyc handle root UAV o poprawnych DXIL resource props i wpisac UAV do wlasciwego metadata resource group. Uzgodnic jedna reprezentacje props; jesli jeden slot wymaga niezgodnych props, jawnie zwrocic blad zamiast ponownie uzyc niepoprawnego handle.
- Przeniesc liste slotow (i jesli potrzebne props) z `Shader` przez `Lowered` i `HostShader`.
- W GPU pipeline dodac deduplikowane root UAV parameters z typem `D3D12_ROOT_PARAMETER_TYPE_UAV` i poprawnym shader register / space.
- W draw/dispatch bindingu pobrac tytulowy root GPU virtual address i wywolac odpowiednie `SetGraphicsRootUnorderedAccessView` lub `SetComputeRootUnorderedAccessView`.
- Zaktualizowac serializacje/deserializacje shader cache. Jesli format cache zmienia sie niekompatybilnie, zwiekszyc wersje/zmienic klucz wedlug lokalnej konwencji.
- Sprawdzic memory state/coherency hostowego shadow buffer i wymagania dla root UAV; sama zmiana typu root signature nie wystarcza, jesli zasoby nie przechodza do stanu unordered access.
- Odroznic raw/structured UAV od prawdziwego typed root view. D3D12 root descriptors maja ograniczenia; nie zakladaj, ze kazdy typed resource da sie przedstawic jako root UAV VA. Nie deklaruj typed support bez zgodnej semantyki/metadata i host binding. Gdy API nie moze go wyrazic, wesprzyj bezpieczny podzbior i zostaw precyzyjny blad dla reszty.

Testy etapu 1:

- Syntetyczny shader root UAV raw store/load, ktory sprawdza liste `root_uavs`, DXIL validation i poprawne metadata.
- Syntetyczne przypadki dla UAV load/store/atomic o wspieranych props.
- GPU/root-signature test analogiczny do istniejacych CBV/SRV root-view testow.
- Cache round-trip z root UAV slotem.
- Test odmowy dla typed root view, jesli ten format nadal nie jest poprawnie reprezentowalny.

### Etap 2: maski lane i select

Naprawic wspolna przyczyne dwoch kategorii:

- `a select on an untracked mask` (281).
- `a lane mask holding lanes elsewhere read as data` (260).

Pracowac lokalnie w `structure.rs`, `emit/lower.rs` i testach. Ustalic, czy konkretna maska jest lane-derived, uniform, constant, phi/join, opaque czy trzyma lanes spoza aktywnego bloku. Nie zamieniac kazdej nieznanej maski na `true`, `false` ani surowa wartosc SGPR bez semantycznego dowodu. Testowac wave32 i wave64 oraz block/phi/join i uzycia jako danych.

Wazna wskazowka ze stanu roboczego: poprzednia sesja probowala rozszerzyc `Structured::mask_bits()` o inne maski i zmienic lookup `v_cndmask_b32`, ale pozniejszy raport nadal pokazywal 281 select failures. Aktualne `lower.rs` nadal zawiera bezposredni lookup przez `st.masks` dla `v_cndmask_b32` i osobny `mask_bits()` dla carry-in. Nie uznawaj tego problemu za rozwiazany; odtworz jeden konkretny przypadek i dowiedz sie, czy jest to ten sam root cause.

### Etap 3: 64-bit image atomic

Obsluga atomicow 64-bitowych na image/UAV (553). Najpierw spisac rzeczywiste GFX10 opcode'y, format/typ obrazu, zwracane wartosci i dostepne funkcje DXIL/D3D12 na wymaganym shader modelu. Nie emulowac jako dwa niezalezne 32-bitowe atomics: nie zachowuje to atomowej semantyki 64-bitowej. Jesli host nie oferuje wymaganej operacji, odmowa ma pozostac jawna; poprawna emulacja moze wymagać lock/loop protocol i nie jest malym fixem.

## Bardzo wazne: stan worktree

W trakcie pracy zostaly scalone i oznaczone jako resolved piec plikow, ktore byly `UU`: `victory-gpu/src/draw.rs`, `victory-gpu/src/pipeline/host.rs`, `victory-gpu/src/shader_cache.rs`, `victory-shader/src/emit/lower.rs` i `victory-shader/src/emit/valu_tests.rs`. `git ls-files -u` bylo puste po oznaczeniu. Nie utworzono commita.

Repo nadal ma rozlegle staged zmiany, kilka plikow `MM` (w tym `victory-shader/src/emit/host.rs`, `mod.rs`, `shader.rs`, `waterfall_tests.rs`) i untracked backupy/pliki tymczasowe. Test polityki wykrywa `shader_errors.log`, `victory_debug.log` i pliki typu `script.bat`/tymczasowe jako niedozwolone. To dane/stany uzytkownika: nie usuwaj, nie dodawaj ani nie zmieniaj ich bez zgody. Nie uzywaj `git reset --hard`, `git checkout --` ani `git clean`.

Zawsze przed dalszymi zmianami sprawdz aktualny `git status --short`, poniewaz uzytkownik moze juz zmienic stan. Pliki z kodem opisane wyzej przechodza testy; pelny test asset-policy jest oddzielnie zablokowany przez wymienione pliki.

## Weryfikacja: co jest, a czego nie ma

- W czystszym, wczesniejszym stanie sesji `cargo test --workspace` oraz `cargo build --release -p victory` przechodzily, ale to bylo przed obecnym duzym zestawem zmian/konfliktow.
- Obecne konflikty powodujace duplikaty lub markery oznaczaja, ze nie mozna zakladac, ze source kompiluje.
- Nie bylo zweryfikowanego runtime ponownego tlumaczenia z aktualnym binarnym buildem po root-UAV zmianach.
- Nie mow, ze liczba root UAV errors spadla, dopoki nie zbudujesz aktualnego `victory.exe` i nie uruchomisz `translate --validate` na plikach uzytkownika.
- Zanim wyciagniesz wnioski z testow, rozwiaz/uzgodnij aktywne konflikty zgodnie z polityka pracy na zmianach uzytkownika. Jesli zakres konfliktow jest niejednoznaczny, najpierw opisz konkretny konflikt i popros o decyzje, zamiast wyrzucac jedna strone.

Kolejnosc testow po bezpiecznym uzgodnieniu zmian:

1. `cargo test -p victory-shader`.
2. `cargo test --workspace`.
3. `cargo build --release -p victory`.
4. Gdy dostepne sa lokalne pliki gry poza repo: `target\release\victory.exe translate <game_dir>\shaders.rpf --validate`; nie kopiowac raportow/assetow do repo.

## Styl pracy w repo

- Trzymaj zmiany minimalne, podazaj za istniejacymi abstrakcjami CBV/SRV/root views.
- Dodawaj syntetyczne testy, nie rzeczywiste shadery ani wyciagniete instrukcje gry.
- Po kazdej niezaleznej poprawce uruchamiaj test, ktory moze ja falsyfikowac; potem workspace.
- Nie zmieniaj niezaleznie unrelated staged/unstaged plikow.
- Nie commituj, chyba ze uzytkownik wyraznie o to poprosi.
- Przed finalnym raportem rozdziel: co naprawiono, co przeszlo testy, czego nie udalo sie zweryfikowac i jakie kategorie ostrzezen zostaly.

## Kolejne zadanie

Etap 1 jest wykonany i ma pozytywne testy oraz translate report. Nie pracuj dalej nad root UAV bez nowego odtworzenia bledu. Kontynuuj etap 2: lane-mask selects i masks read as data, zaczynajac od jednego konkretnego hash/offsetu i syntetycznego repro. Pilnuj, by nie poluzowac semantyki EXEC ani nie zamieniac nieznanej maski na dowolna wartosc. Etap 3 pozostaje: image atomics 64-bit, ktorych nie wolno emulowac jako dwoch niezaleznych 32-bit atomics.
