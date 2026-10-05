# Aplikacje WWW, semestr 2026Z

**Sylabus:** [17S1-APLWWW_2026Z_1702-SI-IO_KRK.pdf](17S1-APLWWW_2026Z_1702-SI-IO_KRK.pdf)

## 1. Cel i zakres przedmiotu

Celem przedmiotu jest kompleksowe przygotowanie studenta do projektowania i wytwarzania nowoczesnych aplikacji internetowych ze szczególnym naciskiem na architekturę **backendową** z wykorzystaniem języka **Python** oraz frameworków **Django** i **Django REST Framework (DRF)**.

Kurs zorganizowany jest w modelu przyrostowym – od podstaw tradycyjnej architektury monolitycznej (MVT – Model-View-Template), poprzez mechanizmy serializacji i budowy interfejsów API, aż po profesjonalne, bezpieczne REST API oparte na tokenach JWT, uprawnieniach obiektowych, zautomatyzowanej dokumentacji OpenAPI (Swagger) i testach jednostkowych.

W trakcie 10 spotkań laboratoryjnych student pozna m.in.:
* Konfigurację nowoczesnego środowiska deweloperskiego (Python 3.13, venv, Git).
* Projektowanie relacyjnych schematów danych i zarządzanie nimi za pomocą **Django ORM** oraz migracji.
* Możliwości i personalizację panelu administracyjnego (**Django Admin**).
* Cykl żądanie–odpowiedź w architekturze **MVT** z silnikiem szablonów HTML.
* Architekturę **REST API** i format JSON.
* Mechanizmy serializacji i deserializacji danych: od niskopoziomowego `serializers.Serializer` do zoptymalizowanego `ModelSerializer`.
* Wielopoziomową walidację danych (na poziomie pola, obiektu i reguł biznesowych).
* Ewolucję widoków API: widoki funkcyjne (`@api_view`), klasowe (`APIView`), generyczne (`generics.*`) oraz `ModelViewSet` z `DefaultRouter`.
* Bezpieczeństwo i uwierzytelnianie bezstanowe z wykorzystaniem standardu **JWT (JSON Web Tokens)** za pośrednictwem biblioteki `djangorestframework-simplejwt`.
* Kontrolę dostępu i autoryzację: role, grupy użytkowników, wbudowane uprawnienia oraz własne klasy uprawnień obiektowych (`BasePermission`).
* Zaawansowaną logikę biznesową wykraczającą poza standardowy CRUD (transakcyjność, ograniczenia stanów, agregacje danych).
* Filtrowanie, wyszukiwanie i paginację danych w API (`django-filter`).
* Automatyczną dokumentację interaktywną w standardzie OpenAPI 3.0 (**Swagger UI / Redoc** za pomocą `drf-spectacular`).
* Testy automatyczne komponentów API z wykorzystaniem `APITestCase` i `APIClient`.

---

## 2. Wymagania techniczne i oprogramowanie

Do realizacji ćwiczeń oraz projektu zaliczeniowego wymagane jest:
* **Python** w wersji **3.13** lub nowszej ([python.org](https://www.python.org/downloads/)).
* **Framework Django** w wersji **6.1.*** ([Dokumentacja Django 6.1](https://docs.djangoproject.com/en/6.1/) | [Release Notes 6.1](https://docs.djangoproject.com/en/6.1/releases/6.1/)).
* **Django REST Framework** (`djangorestframework`) w najnowszej stabilnej wersji ([Dokumentacja DRF](https://www.django-rest-framework.org/)).
* **Biblioteka SimpleJWT** (`djangorestframework-simplejwt`) do obsługi tokenów JWT ([Dokumentacja SimpleJWT](https://django-rest-framework-simplejwt.readthedocs.io/en/latest/)).
* **Dokumentacja OpenAPI 3.0 / Swagger UI:** [`drf-spectacular`](https://drf-spectacular.readthedocs.io/en/latest/).
* **Środowisko programistyczne (IDE):** **PyCharm Professional** (bezpłatna licencja akademicka) lub **Visual Studio Code** (z rozszerzeniami Python i Pylance).
* **System kontroli wersji Git** oraz konto w serwisie **GitHub**.
* **Klient do testowania API:** Postman, Insomnia lub wbudowane narzędzia IDE (np. pliki `.http`), alternatywnie wiersz poleceń `curl`.
* **Baza danych:** domyślnie silnik **SQLite 3** (wbudowany w Pythona, rekomendowany do celów dydaktycznych i bezproblemowej prezentacji projektu).

---

## 3. Warunki zaliczenia przedmiotu

1. Efektem finalnym pracy na laboratoriach jest **autorski projekt REST API** zrealizowany w technologii Django REST Framework.
2. Studenci mogą realizować projekt w zespołach **2-osobowych** (po wcześniejszym uzgodnieniu z prowadzącym) lub **indywidualnie**.
3. Projekt musi być prowadzony w **publicznym repozytorium GitHub** od początku trwania semestru:
   * W przypadku pracy w zespole obaj członkowie muszą aktywnie commitować kod ze swoich kont GitHub (równomierny wkład).
   * Repozytorium musi posiadać czytelny plik `README.md` z instrukcją uruchomienia projektu w środowisku wirtualnym.
4. Kryteria oceny projektu:
   * **Systematyczność pracy:** regularne, opisowe commity odzwierciedlające postępy z kolejnych laboratoriów.
   * **Jakość i czystość kodu:** podział na moduły, przestrzeganie konwencji PEP 8, brak zbędnego zduplikowanego kodu (DRY).
   * **Poprawność architektury:** właściwe wykorzystanie modeli, relacji, serializerów, widoków i routerów.
   * **Bezpieczeństwo:** poprawna implementacja uwierzytelniania JWT oraz kontrola uprawnień do zasobów.
   * **Dokumentacja API:** wdrożenie interaktywnego Swagger UI.
   * **Testy automatyczne:** obecność zestawu testów jednostkowych/integracyjnych weryfikujących kluczowe endpointy.
5. **Harmonogram projektu:**
   * **Lab 2:** Zgłoszenie i akceptacja tematu oraz wstępnego schematu modeli przez prowadzącego.
   * **Lab 5:** Weryfikacja kamienia milowego 1: działające modele, migracje, panel admina oraz bazowe serializery.
   * **Lab 8:** Weryfikacja kamienia milowego 2: endpointy REST, autentykacja JWT i role użytkowników.
   * **Lab 10:** Prezentacja gotowej aplikacji, demonstracja działania endpointów, obrona projektu i wystawienie ocen.

---

## 4. Wymagania techniczne projektu zaliczeniowego

Każdy projekt musi spełniać poniższe kryteria minimalne:

1. **Schemat bazy danych (Modele):**
   * Minimum **4–5 powiązanych ze sobą modeli** adekwatnych do wybranej domeny biznesowej.
   * Zastosowanie relacji: klucz obcy (jeden-do-wielu `ForeignKey`), relacja wiele-do-wielu (`ManyToManyField`) lub jeden-do-jeden (`OneToOneField`).
   * Zdefiniowane opcje pól (walidacje, wartości domyślne, `related_name`, `on_delete`).
   * Zdefiniowana metoda `__str__()` oraz klasa wewnętrzna `Meta` dla każdego modelu.
2. **Uwierzytelnianie i autoryzacja (JWT & RBAC):**
   * Uwierzytelnianie bezstanowe za pomocą **JWT** (`djangorestframework-simplejwt`).
   * Endpointy: generowanie tokenów (`/api/token/`) oraz odświeżanie tokenów (`/api/token/refresh/`).
   * Co najmniej **2 różne role użytkowników** o odmiennym zakresie uprawnień (np. Klient/Uczestnik vs Pracownik/Organizator vs Administrator).
   * Minimum jedna własna klasa uprawnień (`BasePermission`) sprawdzająca własność obiektu (np. edycja możliwa tylko przez autora lub admina).
3. **Interfejs API (Endpointy):**
   * Opcjonalna rejestracja nowego użytkownika (`/api/register/`) jeżeli temat aplikacji odpowiada takiemu podejściu.
   * Kompletne operacje **CRUD** dla głównych encji systemu z wykorzystaniem `ModelViewSet` lub klasowych widoków generycznych.
   * **Minimum 2 dedykowane endpointy wykraczające poza prosty schemat CRUD** (np. realizacja rezerwacji/zakupu z walidacją stanów, generowanie raportu/statystyk, anulowanie z regułą czasową, podsumowanie zasobów bieżącego użytkownika).
4. **Dokumentacja i jakość:**
   * Zautomatyzowana dokumentacja **OpenAPI/Swagger** (np. `drf-spectacular`) dostępna w przeglądarce pod adresem `/api/docs/`.
   * Paginacja oraz filtrowanie/wyszukiwanie na listach zasobów.
   * Minimum **3 testy automatyczne** z użyciem `APITestCase` weryfikujące statusy HTTP i logikę uprawnień.

---

## 5. Przegląd programu laboratoriów (10 spotkań)

Projekt referencyjny budowany wspólnie krok po kroku na zajęciach: **EventHub** (Platforma Wydarzeń i Biletów).

| Nr | Temat laboratorium | Kluczowe zagadnienia |
| :---: | :--- | :--- |
| **01** | Środowisko i fundamenty Django | Python 3.13, venv, Git, `startproject`, `startapp`, routing URL, pierwszy widok |
| **02** | Modele i Django Admin | Django ORM, relacje `ForeignKey`, migracje, superuser, konfiguracja `ModelAdmin`, QuerySet |
| **03** | Silnik szablonów Django (MVT) | Szablony HTML, dziedziczenie szablonów (`base.html`), filtry, pliki statyczne, widoki MVT |
| **04** | Wprowadzenie do DRF i serializacja | Filozofia REST, format JSON, bazowy `serializers.Serializer`, `create()` i `update()`, shell |
| **05** | `ModelSerializer` i endpointy FBV | `ModelSerializer`, walidacja wielopoziomowa, widoki funkcyjne `@api_view`, kody HTTP |
| **06** | Nowoczesna architektura widoków DRF | `APIView` -> `generics.*` -> `ModelViewSet` i `DefaultRouter`, własne akcje `@action` |
| **07** | Uwierzytelnianie JWT i relacja z Userem | SimpleJWT (`access`/`refresh`), nagłówek `Bearer`, `perform_create`, model `Ticket`, `/api/my-tickets/` |
| **08** | Autoryzacja i uprawnienia (Permissions) | `IsAuthenticated`, role w Django Admin, własne uprawnienia obiektowe `BasePermission` |
| **09** | Logika biznesowa, Swagger i testy | Rezerwacje, statystyki, filtrowanie (`django-filter`), Swagger UI (`drf-spectacular`), `APITestCase` |
| **10** | Prezentacja projektów i ocena | Obrona projektów studenckich, demonstracja API, przegląd kodu i wystawienie ocen |
| **Extra 01** | Konteneryzacja i produkcja | Docker, Linux, Nginx (Reverse Proxy), Gunicorn (WSGI), PostgreSQL, 12-Factor App |
| **Extra 02** | Gotowy Frontend SPA i Codespaces | Integracja SPA z REST API, Same-Origin vs CORS, GitHub Codespaces (GitHub for Education) |
| **Extra 03** | Spec-Driven Development (SDD) & AI | Architektura specyfikacji (.spec/), sterowanie agentem AI, Living Documentation, spójność kodu |


