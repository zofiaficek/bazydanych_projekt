# Baza Danych Hurtowni Sprzętu Outdoorowego

## Opis Projektu
Niniejsze repozytorium zawiera projekt relacyjnej bazy danych zaimplementowanej w systemie **PostgreSQL**. Projekt ma charakter edukacyjny i symuluje system obsługi hurtowni zajmującej się dystrybucją specjalistycznego sprzętu wspinaczkowego i turystycznego.

Celem projektu było zaprojektowanie struktury danych odpornej na błędy logiczne oraz implementacja automatyzacji procesów magazynowych przy użyciu języka proceduralnego PL/pgSQL.

---

## 📜 Table of Contents

- [Schemat Bazy Danych](#schemat-bazy-danych)
- [Warunki Integralnościowe](#warunki-integralnościowe)
---


## Schemat Bazy Danych
Poniższy schemat prezentuje relacje między tabelami oraz strukturę przepływu danych w systemie. Zastosowano model, w którym produkty są powiązane z producentami wyłącznie poprzez historię dostaw, co zapobiega powstawaniu pętli logicznych.

![](./schemat.png)

### Opis Struktury danych:
#### 1. Katalog Produktów i Partnerzy
*   **`produkty`** – Centralna tabela systemu. Zawiera definicje asortymentu (nazwa, cena bazowa) oraz powiązanie z kategorią.
*   **`kategorie`** – Tabela słownikowa służąca do grupowania produktów.
*   **`producenci`** – Baza dostawców zewnętrznych. Zawiera dane kontaktowe oraz NIP.
*   **`klienci`** – Baza odbiorców hurtowych, którzy składają zamówienia. Zawiera dane kontaktowe oraz NIP.

#### 2. Zarządzanie Magazynem
*   **`magazyn`** – Tabela przechowująca aktualny stan ilościowy dla każdego produktu. Jest aktualizowana automatycznie przez triggery (opisane poniżej) po zatwierdzeniu dostawy lub zamówienia. Relacja 1:1 z tabelą `produkty`.

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

### 5. Ograniczenia NOT NULL
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





## ⚙️ Kluczowe Funkcjonalności i Logika Biznesowa

### 2. Automatyzacja Magazynu (Triggery)

Rdzeniem logiki biznesowej są triggery (wyzwalacze) odpowiedzialne za automatyczne zarządzanie stanem magazynowym.

1.  **Trigger Zwiększający Stan Magazynu**
    * **Zdarzenie:** `AFTER INSERT` na tabeli `dane_dostawy`.
    * **Cel:** Automatycznie dodaje dostarczoną liczbę produktów do tabeli `magazyn` po zarejestrowaniu nowej dostawy.

2.  **Trigger Zmniejszający Stan Magazynu**
    * **Zdarzenie:** `AFTER INSERT` na tabeli `dane_zamowienia`.
    * **Cel:** Automatycznie odejmuje zamówioną liczbę produktów ze stanu w tabeli `magazyn` po złożeniu zamówienia.

3.  **Trigger Sprawdzający Dostępność Towaru**
    * **Zdarzenie:** `BEFORE INSERT` lub `BEFORE UPDATE` na tabeli `dane_zamowienia`.
    * **Cel:** Blokuje próbę dodania do zamówienia większej liczby produktów, niż jest fizycznie dostępne w magazynie.

4.  **Trigger Korygujący Stan Magazynu**
    * **Zdarzenie:** `AFTER UPDATE` lub `AFTER DELETE` na tabeli `dane_zamowienia`.
    * **Cel:** W przypadku anulowania pozycji lub zmniejszenia jej ilości w zamówieniu, trigger automatycznie przywraca odpowiednią liczbę produktów z powrotem do magazynu.

---

## 📦 Obiekty Bazy Danych

Dostęp do danych oraz operacje biznesowe są ułatwione przez dedykowane widoki, funkcje i procedury.

### Widoki (Views)

* **`AktualnyStanMagazynu`**: Dynamiczny raport pokazujący aktualną ilość na stanie dla każdego produktu, wraz z jego pełną nazwą, ceną i kategorią.
* **`BrakujaceProdukty`**: Widok filtrujący produkty, których stan magazynowy jest niski (poniżej 20 sztuk), i łączący je z danymi kontaktowymi dostawcy w celu ułatwienia procesu zamawiania.

### Funkcje (Functions)

* **`CenaZamowienia(zamowienie_id)`**: Oblicza i zwraca całkowitą wartość (sumę) dla konkretnego zamówienia na podstawie pozycji w tabeli `dane_zamowienia` (`ilosc` * `cena_zakupu`).

### Procedury (Procedures)

* **`AnulujZamowinie`**: Procedura biznesowa do usuwania pozycji z `dane_zamowienia`, która automatycznie uruchamia trigger korygujący (nr 3) w celu zwrócenia towaru na stan.
* **`RejestrujDostawe`**: Procedura biznesowa do dodawania pozycji do `dostawy` i `dane_dostawy`, która automatycznie uruchamia trigger (nr 1) w celu zwiększenia stanu magazynowego.

---

## 🔒 Bezpieczeństwo i Role Użytkowników

System definiuje trzy główne role użytkowników, aby zarządzać dostępem do danych.

| Rola | Uprawnienia (Podsumowanie) | Uzasadnienie |
| :--- | :--- | :--- |
| **Administrator** | Pełny dostęp (DDL/DML) | Zarządzanie strukturą i pełna kontrola nad bazą danych. |
| **Sprzedawca** | `SELECT`, `INSERT`, `UPDATE` (Klienci, Zamówienia) <br> `INSERT`, `DELETE`, `UPDATE` (Dane Zamówienia) | Tworzenie nowych klientów i obsługa pełnego cyklu zamówień. |
| **Magazynier** | `SELECT`, `INSERT`, `UPDATE` (Dostawy, Dane Dostawy) <br> `SELECT` (Magazyn) | Rejestrowanie nowych dostaw. Odczyt stanu magazynowego (modyfikacja odbywa się tylko przez triggery). |
| **Wszyscy** | `SELECT` (Produkty, Kategorie) | Wszystkie role potrzebują dostępu do informacji o produktach i cenach. |
