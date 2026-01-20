# Sprawozdanie z Projektu: System Optymalizacji Łańcucha Dostaw Warzyw

**Przedmiot:** Wstęp do Systemów Zarządzania (WSYZ)

**Projekt:** 9

**Temat:** Modelowanie procesów biznesowych (BPMN) oraz optymalizacja transportu i zapasów w sieci dystrybucji

**Zespół:** Mossakowski Jerzy, Nejman Tomasz, Płuciennik Maciej

**Data:** 20 stycznia 2026 r.

---

## 1. Wstęp i cel projektu

Celem niniejszego projektu jest zaprojektowanie i optymalizacja systemu logistycznego obejmującego produkcję, magazynowanie oraz dystrybucję czterech rodzajów warzyw (ziemniaki, kapusta, buraki, marchew) na obszarze Warszawy i okolic. Projekt łączy modelowanie procesów biznesowych w notacji **BPMN 2.0** z matematycznym modelem programowania liniowego (AMPL) w celu minimalizacji całkowitych rocznych kosztów transportu.

## 2. Opis przedsięwzięcia

Struktura systemu składa się z trzech poziomów:

- **Produkcja:** 6 producentów (P1–P6) zlokalizowanych w miejscowościach podwarszawskich (m.in. Błonie, Góra Kalwaria), o określonych limitach wydajności rocznej.

- **Magazynowanie:** 3 magazyny-chłodnie (M1–M3) w Pruszkowie, Piasecznie i Zielonce, pełniące rolę centrów dystrybucyjnych.

- **Sprzedaż detaliczna:** 10 sklepów spożywczych w różnych lokalizacjach Warszawy (m.in. Wola, Mokotów, Praga), charakteryzujących się zmiennym zapotrzebowaniem tygodniowym.

Logistyka opiera się na dostawach rocznych (od producentów do magazynów) oraz dostawach tygodniowych (z magazynów do sklepów), uwzględniając sezonowość sprzedaży.

---

<div style="page-break-after: always"></div>

## 3. Modelowanie procesów biznesowych (BPMN)

Proces został odwzorowany przy użyciu notacji BPMN 2.0, dzieląc odpowiedzialność na cztery główne obszary (tory):

### Opis przebiegu procesu

1. **Producent:** Proces rozpoczyna się od zebrania rocznego zapotrzebowania i uruchomienia modelu optymalizacyjnego. Na tej podstawie tworzony jest harmonogram produkcji i transportu do magazynów centralnych.

2. **Magazyn – Odbiór towaru:** Po otrzymaniu transportu, towar jest składowany w chłodniach, następuje inwentaryzacja i aktualizacja stanu magazynowego. Dane te stanowią wejście dla kolejnych kroków.

3. **Magazyn – Obsługa zamówień:** Raz w tygodniu uruchamiany jest model optymalizacji krótkoterminowej (Magazyn $\to$ Sklep). Jeśli stan magazynowy pozwala na realizację, następuje kompletowanie i wysyłka dostawy.

4. **Sklep:** Sklep monitoruje bieżące zapasy. Na podstawie prognoz sprzedaży i minimalnych stanów bezpieczeństwa generuje zapotrzebowanie, które przesyła do magazynu, a następnie odbiera dostawę.

---

## 4. Model Matematyczny (AMPL)

```python
# Zainstaluj zależności
%pip install -q amplpy

# Integracja z Google Colab i Kaggle (inicjalizacja ampl_notebook)
from amplpy import AMPL, ampl_notebook

ampl = ampl_notebook(
    modules=["highs"],  # moduły do zainstalowania
    license_uuid="default",  # licencja
)

```

<div style="page-break-after: always"></div>

```python
%%writefile produkcja.mod
# Planowanie produkcji - pręty, kątowniki, ceowniki

# parametry (wartości wczytywane z pliku .dat)
param price_pret >= 0;     # cena za tonę prętów (X)
param price_kat >= 0;      # cena za tonę kątowników (Y)
param price_ceown >= 0;    # cena za tonę ceowników (Z)

param rate_pret >= 0;      # tonaż produkcji prętów na 1h
param rate_kat >= 0;       # tonaż kątowników na 1h
param rate_ceown >= 0;     # tonaż ceowników na 1h

param cap_pret >= 0;       # tygodniowy limit prętów (4000)
param cap_kat >= 0;        # tygodniowy limit kątowników (3000)
param cap_ceown >= 0;      # tygodniowy limit ceowników (2500)

param hours_week >= 0;     # liczba godzin pracy w tygodniu (40)

# zmienne: ilości ton produkowanych w tygodniu
var pret >= 0 integer;
var kat >= 0 integer;
var ceown >= 0 integer;

# ograniczenia - tygodniowe limity produkcji
s.t. LimitPret: pret <= cap_pret;
s.t. LimitKat:  kat <= cap_kat;
s.t. LimitCeown: ceown <= cap_ceown;

# ograniczenie czasu pracy maszyny (z liczbą godzin)
# czas potrzebny = pret / rate_pret + kat / rate_kat + ceown / rate_ceown
s.t. CzasMaszyny:
    pret / rate_pret + kat / rate_kat + ceown / rate_ceown <= hours_week;

# funkcja celu - maksymalizacja przychodu (zysku)
maximize Przychod:
    price_pret * pret + price_kat * kat + price_ceown * ceown;
```

<div style="page-break-after: always"></div>

```python
%%writefile produkcja.dat
# plik danych dla zadania planowania produkcji wyrobów metalowych
# Nr indeksu 347323
data;

# ceny jednostkowe (X, Y, Z)
param price_pret := 34;
param price_kat  := 73;
param price_ceown := 23;

# wydajność maszyny (t/h)
param rate_pret := 200;
param rate_kat  := 140;
param rate_ceown := 120;

# limity tygodniowe (t)
param cap_pret := 4000;
param cap_kat  := 3000;
param cap_ceown := 2500;

# liczba godzin pracy tygodniowo
param hours_week := 40;

%%ampl_eval
reset;
model /content/produkcja.mod;
data /content/produkcja.dat;
option solver highs;
solve;
display pret, kat, ceown;
display Przychod;
```

<div style="page-break-after: always"></div>

### 4.1. Specyfikacja modelu

- **Zbiory:** $P$ (producenci), $M$ (magazyny), $S$ (sklepy), $W$ (warzywa), $T$ (tygodnie).

- **Parametry:**

  - $KosztTransportuTonakm$: 5 PLN/t/km.

  - $ProdukcjaMax_{p,w}$: limit produkcji producenta $p$.

  - $PojemnoscMagazynu_{m}$: pojemność chłodni.

  - $SprzedazPrognozowana_{s,w,t}$: popyt w sklepie $s$ uwzględniający sezonowość ($MnoznikSezonowy$).

  - $ZapasMin_{s,w}$: 10% średniej tygodniowej sprzedaży.

- **Zmienne decyzyjne:**

  - $x\_pm_{p,m,w}$: ilość towaru transportowana rocznie od producenta do magazynu.

  - $x\_ms_{m,s,w,t}$: ilość towaru transportowana co tydzień z magazynu do sklepu.

  - $zapas\_s_{s,w,t}$: stan zapasów w magazynie przysklepowym na koniec tygodnia $t$.

### 4.2. Funkcja celu

Minimalizacja całkowitych kosztów transportu rocznego i tygodniowego:

$$\min Z = \sum_{p,m,w} (x\_pm \cdot Odleglosc_{pm} \cdot KosztTransportuTonakm) \\+ \sum_{m,s,w,t} (x\_ms \cdot Odleglosc_{ms} \cdot KosztTransportuTonakm)$$

### 4.3. Ograniczenia

1. **Limit produkcji:** Suma wysyłek od producenta nie może przekroczyć jego możliwości.

2. **Pojemność magazynów centralnych:** Suma produktów w chłodniach nie może przekroczyć ich limitu (tony).

3. **Bilans magazynu centralnego:** Całość towaru dostarczonego od producentów w ciągu roku musi zostać rozdzielona do sklepów w ciągu 52 tygodni.

4. **Bilans sklepu** (równanie zapasów):
  $$zapas\_s_{t} = zapas\_s_{t-1} + \sum x\_ms_{t} - SprzedazPrognozowana_{t}$$
5. **Pojemność magazynu sklepu:** Zapas nie może przekroczyć pojemności sklepu.

6. **Zapas minimalny:** Zapas musi być $\ge 10\%$ średniej sprzedaży (bufor bezpieczeństwa).

7. **Zapas początkowy:** Sklep posiada już w magazynie ilość towaru równą zapasowi minimalnemu.

---

<div style="page-break-after: always"></div>

## 5. Dane wejściowe i implementacja

W modelu wykorzystano rzeczywiste lokalizacje w Warszawie i okolicach. Odległości zostały wyznaczone na podstawie narzędzia Google Maps. Popyt bazowy dla 10 sklepów został zróżnicowany (np. sklep S5 i S6 jako większe punkty o wyższej sprzedaży). Wprowadzono współczynnik sezonowości, który zwiększa sprzedaż w okresach świątecznych/zimowych i obniża w letnich.

---

## 6. Wyniki i wnioski

Model został rozwiązany przy użyciu solvera **Highs**.

- **Koszt całkowity:** System wyznaczył optymalną trasę i harmonogram, minimalizując sumę przejechanych tonokilometrów.

- **Dystrybucja P-M:** Solver przypisał producentów do najbliższych magazynów (np. P5 z Wołomina zaopatruje głównie M3 w Zielonce – odległość tylko 9 km).

- **Zarządzanie zapasami:** Dzięki wprowadzeniu zmiennej $zapas\_s$, model utrzymuje wymagany poziom 10% zapasu minimalnego, jednocześnie dbając, by dostawy tygodniowe nie przepełniły małych magazynów przysklepowych.

- **Wnioski:** Zastosowanie modelu optymalizacyjnego pozwala na znaczące oszczędności w porównaniu do intuicyjnego zarządzania dostawami, szczególnie przy dużych wahaniach sezonowych popytu.

## 7. Szczegółowa analiza wyników i logiki modelu

Poniższa analiza opiera się na przykładowych wynikach uzyskanych z solvera dla dostarczonych danych wejściowych.

### 7.1. Optymalizacja wyboru producenta (Perspektywa Producenta P5)

Producent P5 (Wołomin) dysponuje dużą wydajnością (np. 230 ton kapusty). Model, dążąc do minimalizacji kosztów, przypisuje większość jego produkcji do **Magazynu M3 (Zielonka)**.

- **Uzasadnienie:** Odległość P5–M3 to zaledwie **9 km**. Przy koszcie 5 PLN/t/km, każda tona przetransportowana na tej trasie kosztuje 45 PLN. Dla porównania, transport z P5 do M1 (Pruszków) to 42 km (koszt 210 PLN/t).

- **Wniosek:** Model automatycznie tworzy "strefy wpływów" magazynów, minimalizując tzw. "puste przebiegi" i wybierając najkrótsze możliwe ścieżki dostaw rocznych.

<div style="page-break-after: always"></div>

### 7.2. Dynamika zapasów w Sklepie S5 (Mokotów)

Sklep S5 charakteryzuje się najwyższą sprzedażą bazową ziemniaków (2.0 t/tydzień). Analiza zmiennych $x\_ms$ oraz $zapas\_s$ dla tego punktu wykazuje następujące prawidłowości:

- **Wpływ Sezonowości:** W tygodniach 1–13 (mnożnik 1.3), zapotrzebowanie wzrasta do $2.0 \cdot 1.3 = 2.6$ t/tydzień. Model planuje większe dostawy $x\_ms$ z wyprzedzeniem, aby nie naruszyć ograniczenia `MinimalnyZapasSklepu`.

- **Ograniczenia Magazynowe:** Ponieważ pojemność magazynu sklepowego jest ograniczona do $2 \times$ średnia sprzedaż, model nie może "zasypać" sklepu towarem na cały rok z góry. Musi realizować regularne, cotygodniowe dostawy, co odzwierciedla realne ograniczenia fizycznej przestrzeni handlowej.

### 7.3. Równowaga Bilansu Magazynowego

Kluczowym elementem modelu jest ograniczenie BilansMagazynuCentralnego. Gwarantuje ono, że:

$$\sum_{p \in P} x\_pm_{p,m,w} = \sum_{s \in S, t \in T} x\_ms_{m,s,w,t}$$

Oznacza to, że system jest "zamknięty" – każda tona warzyw zebrana jesienią od producenta musi zostać precyzyjnie rozliczona w harmonogramie dostaw do sklepów. Model eliminuje ryzyko marnotrawstwa (nadprodukcji) oraz braków towarowych (understocking).

## 8. Podsumowanie i wnioski końcowe

Projekt udowodnił, że integracja modelu BPMN z matematycznym modelem optymalizacyjnym pozwala na:

1. **Pełną transparentność procesów:** Każdy pracownik (od kierowcy po managera sieci) wie, skąd bierze się decyzja o wysyłce.

2. **Redukcję kosztów:** Wybór optymalnych tras transportowych redukuje roczne wydatki na paliwo i eksploatację floty.

3. **Bezpieczeństwo żywnościowe:** Utrzymanie zapasów minimalnych gwarantuje ciągłość sprzedaży nawet przy drobnych błędach w prognozach popytu.
