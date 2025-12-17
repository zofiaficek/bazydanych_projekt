# Baza Danych Hurtowni Sprzętu Outdoorowego

## Opis Projektu
Niniejsze repozytorium zawiera projekt relacyjnej bazy danych zaimplementowanej w systemie **PostgreSQL**. Projekt ma charakter edukacyjny i symuluje system obsługi hurtowni zajmującej się dystrybucją specjalistycznego sprzętu wspinaczkowego i turystycznego.

Celem projektu było zaprojektowanie struktury danych odpornej na błędy logiczne oraz implementacja automatyzacji procesów magazynowych przy użyciu języka proceduralnego PL/pgSQL.

---

## 📜 Table of Contents

- [Schemat Bazy Danych](#schemat-bazy-danych)
- [Warunki Integralnościowe](#warunki-integralnościowe)
- [Oprogramowanie warstwy dostępu do danych](#oprogramowanie-warstwy-dostępu-do-danych)
- [Definicja Uprawnień i Zasad Bezpieczeństwa](#definicja-uprawnień-i-zasad-bezpieczeństwa)
---


## Schemat Bazy Danych
Poniższy schemat prezentuje relacje między tabelami oraz strukturę przepływu danych w systemie. Zastosowano model, w którym produkty są powiązane z producentami wyłącznie poprzez historię dostaw, co zapobiega powstawaniu pętli logicznych.

![](./schemat.png)

### Opis Struktury danych:
#### 1. Katalog Produktów i Partnerzy
*   `produkty` – Centralna tabela systemu. Zawiera definicje asortymentu (nazwa, cena bazowa) oraz powiązanie z kategorią.
*   `kategorie` – Tabela słownikowa służąca do grupowania produktów.
*   `producenci` – Baza dostawców zewnętrznych. Zawiera dane kontaktowe oraz NIP.
*   `klienci` – Baza odbiorców hurtowych, którzy składają zamówienia. Zawiera dane kontaktowe oraz NIP.

#### 2. Zarządzanie Magazynem
*   `magazyn` – Tabela przechowująca aktualny stan ilościowy dla każdego produktu. Jest aktualizowana automatycznie przez triggery (opisane poniżej) po zatwierdzeniu dostawy lub zamówienia. Relacja 1:1 z tabelą `produkty`.

#### 3. Logistyka i Transakcje 
W systemie zastosowano wzorzec 'Nagłówek-Szczegóły' dla dokumentów:

*   **Dostawy:**
    *   `dostawy` (Nagłówek): Data, numer faktury dostawcy, relacja do producenta.
    *   `dane_dostawy` (Pozycje): Konkretne produkty, ilości i ceny zakupu w ramach danej dostawy.
      
*   **Zamówienia:**
    *   `zamowienia` (Nagłówek): Data, klient, numer faktury sprzedaży.
    *   `dane_zamowienia` (Pozycje): Produkty sprzedawane w ramach zamówienia. Przed dodaniem rekordu trigger weryfikuje dostępność towaru w tabeli `magazyn`.

---

## Warunki Integralnościowe
Poniżej zestawiono mechanizmy zastosowane w celu zapewnienia spójności, unikalności i poprawności danych w systemie.
### 1. Klucze Podstawowe (PK)
Gwarantują jednoznaczną identyfikację każdego rekordu.
*   **Klucze proste:** `id_kategorii`, `id_produktu`, `id_klienta`, `id_producenta`, `id_dostawy`, `id_zamowienia`
*   **Klucze złożone:** `(id_dostawy, id_produktu)`, `(id_zamowienia, id_produktu)`

### 2. Klucze Obce (FK)
Definiują relacje między tabelami i zachowanie przy usuwaniu danych.
*   **Relacje standardowe:** Powiązanie dokumentów ze słownikami m.in. `fk_klienci`, `fk_producenci`, `fk_magazyn_produkty`.
*   **`ON DELETE CASCADE`:** W tabelach `dane_dostawy` i `dane_zamowienia`. Usunięcie nagłówka dokumentu usuwa jego pozycje.
*   **`ON DELETE SET DEFAULT`:** W tabeli `produkty` (relacja do kategorii). Usunięcie kategorii przypisuje produktom kategorię 0 ("nieznane").

### 3. Ograniczenia Unikalności (UNIQUE)
Blokują możliwość wprowadzenia duplikatów w kluczowych kolumnach.
*   **Firmy:** Pary `nip` oraz `nazwa_firmy` muszą być unikalne w tabelach `klienci` i `producenci`.
*   **Dokumenty:** `nr_faktury` musi być unikalny w tabelach `dostawy` i `zamowienia`.
*   **Słowniki:** nazwa kategorii nie może się powtarzać

### 4. Ograniczenia Sprawdzające (CHECK)
Reguły walidujące poprawność wartości numerycznych (domena danych).
*   **Ceny:** `cena_zakupu` > 0 (tabela `produkty`, `dane_zamowienia`), `cena_dostawy` > 0 (`tabela dane_dostawy`).
*   **Ilości transakcyjne:** `ilosc` > 0 (tabela `dane_dostawy`, `dane_zamowienia`).
*   **Bezpiecznik magazynowy:** `ilosc_na_stanie` >= 0 (tabela `magazyn`) – baza odrzuci transakcję, która spowodowałaby ujemny stan.

### 5. Ograniczenia Wymagalności (NOT NULL)
Wymuszają zapełnienie kluczowych pól 
*    **Firmy:** `nazwa`, `email`, `czy_aktywny` w tabelach `producenci` i `klienci`.
*    **Słowniki:** `nazwa` w tabelii `kategorie`.
*    **Dokumenty:** m.in `id_producenta`, `data_dostawy`, `id_klienta`, `data_zamowienia` w tabelach `dostawy` i `zamowienia`.
*    **Magazyn:** pole `ilosc_na_stanie`.

### 6. Ochrona przed usuwaniem
Zabezpieczenie przed utratą spójności historycznej dostaw i zamówień. Nie można usunąć rekordów z tabel `producenci` i `klienci`
*   **Mechanizm:** Trigger `BEFORE DELETE` na tabelach `klienci` i `producenci`.
*   **Działanie:** Fizyczne usunięcie rekordu jest blokowane. Zamiast tego system wykonuje `UPDATE`, zmieniając flagę `czy_aktywny` na `FALSE`.

---


## Oprogramowanie warstwy dostępu do danych
### 1. Triggery
#### Automatyzacja Magazynu
Mechanizm zapewniający spójność między dokumentami a stanem faktycznym.
*   **`trg_dodaj_do_magazynu` (AFTER INSERT):** Po zatwierdzeniu nowej dostawy (`dane_dostawy`), system automatycznie zwiększa liczbę sztuk w tabeli `magazyn`.
*   **`trg_odejmij_z_magazynu` (AFTER INSERT / UPDATE):** Po dodaniu lub edycji pozycji w zamówieniu (dane_zamowienia), system automatycznie koryguje stan magazynowy (odejmuje sprzedany towar lub zwraca różnicę przy korekcie).

#### Walidacja Reguł Biznesowych
Zabezpieczenie przed sprzedażą towaru, którego fizycznie nie ma.
*   **`trg_weryfikacja_stanu` (BEFORE INSERT / UPDATE):** Przed zapisaniem pozycji zamówienia funkcja sprawdza dostępność w tabeli `magazyn`. Jeśli zamawiana ilość przekracza stan (z uwzględnieniem edytowanego rekordu), operacja jest przerywana błędem (`RAISE EXCEPTION`), a transakcja wycofywana.

#### Automatyzacja Finansowa
System samodzielnie pobiera aktualne ceny z katalogu, eliminując błędy ręcznego wprowadzania danych.
*   **`trg_aktualizuj_wartosc_zamowienia` (BEFORE INSERT / UPDATE):** Pobiera aktualną cenę bazową produktu, dolicza marżę (10%) i zapisuje ostateczną cenę sprzedaży w pozycji zamówienia (ilość * cena * 1,1).
*   **`trg_aktualizuj_wartosc_dostawy` (BEFORE INSERT / UPDATE):** Pobiera cenę zakupu z katalogu i wylicza wartość pozycji dostawy (ilość * cena).

####  Bezpieczeństwo Danych
Zabezpieczenie przed utratą spójności historycznej dostaw i zamówień.
*   **`trg_archiwizacja_klienta` oraz `trg_archiwizacja_producenta` (BEFORE DELETE):** Zamiast trwale usuwać kontrahenta (co uszkodziłoby historię faktur), zmieniają jego status na nieaktywny (`czy_aktywny = FALSE`).

### 2. Funkcje
Funkcje ułatwiające wyciąganie zagregowanych danych finansowych.
*   **`pobierz_wartosc_zamowienia(id_zamowienia)`:** Zwraca łączną kwotę dla całego zamówienia, sumując wartości wszystkich pozycji towarowych przypisanych do danego dokumentu.
*   **`pobierz_wartosc_dostawy(id_dostawy)`:** Zwraca łączną kwotę dla całej dostawy, sumując wartości wszystkich pozycji towarowych przypisanych do danego dokumentu.
*   **`bilans_miesieczny(rok, miesiac)`:** Funkcja finansowa, obliczająca różnicę między łączną wartością sprzedaży (zamówienia od klientów) a kosztami zaopatrzenia (faktury za dostawy) w wybranym miesiącu.
*   **`wskaz_bestseller_kategorii(nazwa_kategorii)`:** Analizuje historyczne dane sprzedażowe dla wybranej grupy asortymentowej i zwraca nazwę produktu, który sprzedał się w największej liczbie sztuk.

### 3. Procedury
System wykorzystuje procedury do obsługi złożonych operacji zmian danych, które wymagają zachowania odpowiedniej kolejności działań.
*   **`anuluj_zamowienie(p_id_zamowienia)`:** Obsługa rezygnacji z zamówienia. Procedura najpierw identyfikuje towary w zamówieniu i zwraca je na stan w tabeli `magazyn`, a następnie trwale usuwa nagłówek zamówienia. Dzięki zastosowaniu kluczy obcych z opcją `CASCADE`, usunięcie zamówienia automatycznie czyści wszystkie powiązane z nim pozycje towarowe (w tabeli `dane_zamowienia`).
*   **`powtorz_zamowienie(stare_id, nowy_nr_faktury)`:** Szybka obsługa stałych klientów ("To samo co zwykle"). Procedura duplikuje strukturę historycznego zamówienia, tworząc nowy wpis z bieżącą datą. Ściśle współpracuje z triggerami – nie kopiuje starych cen, lecz wymusza na systemie pobranie aktualnych stawek z katalogu oraz ponowną weryfikację dostępności towaru w magazynie.

### 4. Widoki
W celu uproszczenia dostępu do danych oraz ukrycia złożoności zapytań SQL (wielokrotne złączenia JOIN), w systemie zdefiniowano wirtualne tabele (widoki).
*   **`aktualny_stan_magazynu`**: Prezentuje kompletny obraz stanu posiadania hurtowni. Widok łączy surowe dane liczbowe z tabeli `magazyn` z czytelnymi dla człowieka informacjami z tabel `produkty` i `kategorie`.
*   **`brakujace_produkty`**: Widok filtrujący produkty, których stan magazynowy jest niski (poniżej 20 sztuk).

---
  

## Definicja Uprawnień i Zasad Bezpieczeństwa
W projekcie wdrożono model kontroli dostępu oparty na rolach (RBAC – Role-Based Access Control). Zgodnie z zasadą najmniejszych uprawnień (Principle of Least Privilege), użytkownicy systemu otrzymali dostęp wyłącznie do tych struktur danych i funkcji, które są niezbędne do realizacji ich zadań służbowych.

Takie podejście realizuje dwa kluczowe cele:
*    Separacja odpowiedzialności: Dział handlowy nie może ingerować w dostawy, a magazynierzy nie mają możliwości manipulowania danymi klientów.
*    Ochrona integralności: Blokada bezpośredniego zapisu w tabelach krytycznych (np. magazyn) wymusza korzystanie z zaimplementowanej logiki biznesowej (triggerów i procedur), co zapobiega powstawaniu błędów.

System definiuje trzy główne role użytkowników:

| Rola | Uprawnienia (Podsumowanie) | Uzasadnienie |
| :--- | :--- | :--- |
| **Administrator** | Pełny dostęp (DDL/DML) | Zarządzanie strukturą i pełna kontrola nad bazą danych. |
| **Sprzedawca** | `SELECT`, `INSERT`, `UPDATE` (Klienci, Zamówienia) <br> `SELECT`, `INSERT`, `DELETE`, `UPDATE` (Dane Zamówienia) | Tworzenie nowych klientów i obsługa pełnego cyklu zamówień. |
| **Magazynier** | `SELECT`, `INSERT`, `UPDATE` (Dostawy, Dane Dostawy) <br> `SELECT` (Magazyn) | Rejestrowanie nowych dostaw. Odczyt stanu magazynowego (modyfikacja odbywa się tylko przez triggery). |
| **User** | `SELECT` (Produkty, Kategorie) | Każdy ma dostęp do informacji o produktach i cenach. |

---



