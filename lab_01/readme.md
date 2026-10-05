# Aplikacje WWW, semestr 2026Z

## Lab 1 – Konfiguracja środowiska, architektura Django i pierwszy widok
---

### 1. Wprowadzenie

Framework Django to jeden z najpopularniejszych, dojrzałych i kompleksowych frameworków backendowych w języku Python, zaprojektowany w oparciu o zasadę *"batteries included"* (posiada wbudowane mechanizmy autentykacji, obsługi bazy danych, panel administracyjny, system migracji i szablony).

W trakcie zajęć laboratoryjnych będziemy rozwijać spójny, przyrostowy projekt backendowy o nazwie **EventHub** – platformę do zarządzania wydarzeniami (konferencjami, warsztatami, koncertami) oraz rezerwacji biletów.

Równolegle, od drugich zajęć, każdy student (lub zespół dwuosobowy) będzie rozwijał swój własny, unikalny projekt zaliczeniowy.

---

### 2. Architektura Django w pigułce

Tradycyjna architektura Django określana jest skrótem **MVT (Model – View – Template)**:
1. **Model:** Definiuje strukturę danych i reguły biznesowe. Django ORM (Object-Relational Mapping) automatycznie tłumaczy klasy Pythona na tabele w bazie danych ([Dokumentacja modeli](https://docs.djangoproject.com/en/6.1/topics/db/models/)).
2. **View (Widok):** Logika biznesowa przetwarzająca żądanie HTTP ([`HttpRequest`](https://docs.djangoproject.com/en/6.1/ref/request-response/#httprequest-objects)) i zwracająca odpowiedź HTTP ([`HttpResponse`](https://docs.djangoproject.com/en/6.1/ref/request-response/#httpresponse-objects)). W świecie REST API widoki zwracają dane w formacie JSON zamiast renderować kod HTML.
3. **Template (Szablon):** Warstwa prezentacji generująca stronę HTML (omówimy ją szczegółowo na Lab 3; zob. [Dokumentacja szablonów](https://docs.djangoproject.com/en/6.1/topics/templates/)).
4. **URLconf (Routing):** Mapowanie adresów URL na odpowiednie funkcje lub klasy widoków ([Dokumentacja dyspozytora URL](https://docs.djangoproject.com/en/6.1/topics/http/urls/)).

Różnica między **projektem** a **aplikacją** w Django:
* **Projekt (Project):** Całościowa konfiguracja serwisu (ustawienia bazy danych, zainstalowane pakiety, routing główny, pliki konfiguracyjne). Tworzony za pomocą [`django-admin startproject <nazwa>`](https://docs.djangoproject.com/en/6.1/intro/tutorial01/).
* **Aplikacja (App):** Samodzielny, modularny komponent realizujący konkretny wycinek funkcjonalności (np. moduł wydarzeń, moduł użytkowników, moduł płatności). Tworzona za pomocą [`python manage.py startapp <nazwa>`](https://docs.djangoproject.com/en/6.1/ref/django-admin/#startapp).

---

### 3. Instrukcje krok po kroku

#### Krok 1: Weryfikacja środowiska Python 3.13
Upewnij się, że masz zainstalowany interpreter języka Python w wersji 3.13 lub nowszej:
```bash
python --version
# Wynik powinien wskazywać: Python 3.13.x
```
Sprawdź również menedżer pakietów `pip`:
```bash
pip --version
```

#### Krok 2: Utworzenie katalogu projektu i wirtualnego środowiska (.venv)
Wirtualne środowisko izoluje pakiety zainstalowane dla tego projektu od reszty systemu operacyjnego:
```bash
# Utwórz folder roboczy projektu
mkdir eventhub_project
cd eventhub_project

# Stwórz wirtualne środowisko w podfolderze
python -m venv .venv
# lub poprzez kreatora w wybranym przez Ciebie IDE
```

Aktywacja wirtualnego środowiska:
* **System Windows (PowerShell):**
  ```powershell
  .venv\Scripts\Activate.ps1
  ```
  *(Jeżeli pojawi się błąd dotyczący polityki wykonywania skryptów w PowerShell, wykonaj jednorazowo: `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`)*
* **System Windows (Command Prompt - cmd):**
  ```cmd
  .venv\Scripts\activate.bat
  ```
* **System macOS / Linux:**
  ```bash
  source .venv/bin/activate
  ```
Po poprawnej aktywacji przed znakiem zachęty w terminalu pojawi się prefiks `(.venv)`.

#### Krok 3: Instalacja frameworka Django 6.1.*
W aktywnym środowisku wirtualnym zaktualizuj `pip` i zainstaluj Django:
```bash
pip install --upgrade pip
pip install "django>=6.1.0,<6.2.0"
```
Weryfikacja instalacji:
```bash
python -m django --version
```

Zapisz listę zainstalowanych zależności w pliku `requirements.txt`:
```bash
pip freeze > requirements.txt
```

#### Krok 4: Utworzenie projektu Django i pierwszej aplikacji
Wygeneruj strukturę projektu o nazwie `eventhub`:
```bash
# Kropka na końcu oznacza utworzenie projektu w bieżącym katalogu (bez zbędnego podwójnego zagnieżdżania folderów)
django-admin startproject eventhub .
```

Następnie utwórz aplikację domenową o nazwie `events`:
```bash
python manage.py startapp events
```

Struktura katalogów powinna wyglądać następująco:
```text
eventhub_project/
│
├── .venv/                      # Izolowane środowisko Pythona
├── eventhub/                   # Główny pakiet konfiguracyjny projektu
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py             # Globalna konfiguracja projektu
│   ├── urls.py                 # Główny routing adresów URL
│   └── wsgi.py
├── events/                     # Aplikacja obsługująca wydarzenia i rezerwacje
│   ├── migrations/
│   │   └── __init__.py
│   ├── __init__.py
│   ├── admin.py                # Konfiguracja panelu Django Admin
│   ├── apps.py
│   ├── models.py               # Definicje modeli bazodanowych
│   ├── tests.py                # Testy jednostkowe
│   └── views.py                # Widoki aplikacji
├── manage.py                   # Narzędzie CLI do zarządzania projektem
└── requirements.txt            # Zależności projektu
```

#### Krok 5: Rejestracja aplikacji w pliku konfiguracyjnym
Otwórz plik `eventhub/settings.py` i dodaj nowo utworzoną aplikację `events` do listy [`INSTALLED_APPS`](https://docs.djangoproject.com/en/6.1/ref/settings/#installed-apps):

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # Aplikacje lokalne projektu
    'events.apps.EventsConfig',
]
```

#### Krok 6: Pierwszy widok i routing adresów URL
1. W pliku `events/views.py` utwórz prosty widok funkcyjny zwracający odpowiedź tekstową:
```python
from django.http import HttpResponse

def welcome_view(request):
    return HttpResponse("Witaj w platformie EventHub – systemie obsługi wydarzeń i biletów!")
```

2. Utwórz plik `events/urls.py`, który będzie przechowywał trasy specyficzne dla aplikacji `events`:
```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.welcome_view, name='events-welcome'),
]
```

3. Dołącz routing aplikacji `events` do głównego pliku `eventhub/urls.py`:
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('events/', include('events.urls')),
]
```

#### Krok 7: Uruchomienie deweloperskiego serwera WWW
Uruchom wbudowany serwer developerski Django poleceniem [`python manage.py runserver`](https://docs.djangoproject.com/en/6.1/ref/django-admin/#runserver):
```bash
python manage.py runserver
```
Otwórz przeglądarkę internetową i wejdź pod adres:
* [http://127.0.0.1:8000/events/](http://127.0.0.1:8000/events/) – powinieneś zobaczyć komunikat powitalny z widoku.
* [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/) – powinien pojawić się ekran logowania panelu administracyjnego Django.

Zatrzymaj serwer w terminalu skrótem klawiszowym `Ctrl + C`.

#### Krok 8: Inicjalizacja repozytorium Git i plik .gitignore
Przygotuj plik `.gitignore` w głównym katalogu projektu, aby wykluczyć pliki tymczasowe, środowisko wirtualne oraz bazę danych:

```gitignore
# Środowisko wirtualne
.venv/
env/
venv/

# Pliki Pythona i cache
__pycache__/
*.py[cod]
*$py.class

# Baza danych deweloperska (nie commitujemy bazy SQLite!)
*.sqlite3
*.db

# Pliki środowiskowe z kluczami
.env

# Pliki konfiguracyjne IDE
.idea/
.vscode/
*.swp
```

Zainicjalizuj repozytorium i wykonaj pierwszy commit:
```bash
git init
git add .
git commit -m "feat: Initial commit with Django 6.1 project structure and events app"
```

---

### 4. Zadania do wykonania na zajęciach

1. **Konfiguracja środowiska:** Zainstaluj Pythona 3.13, skonfiguruj środowisko `.venv` i zainstaluj Django `6.1.*`.
2. **Inicjalizacja EventHub:** Stwórz projekt `eventhub` oraz aplikację `events`, zarejestruj aplikację w `INSTALLED_APPS`.
3. **Rozbudowa widoków:**
   * W pliku `events/views.py` zdefiniuj drugi widok funkcyjny `about_view(request)`, który zwraca krótki opis funkcjonalności platformy EventHub (możliwości tworzenia wydarzeń, zakupu biletów, wystawiania ocen).
   * Dodaj trasę do tego widoku w `events/urls.py` pod ścieżką `about/` (pełny adres: `http://127.0.0.1:8000/events/about/`).
4. **Git:** Utwórz repozytorium na GitHubie, skonfiguruj zdalny adres (`git remote add origin ...`) i wykonaj `git push -u origin main`.
5. **Projekt zaliczeniowy:**
   * Skonsultuj z prowadzącym temat własnego projektu zaliczeniowego.
   * Zastanów się nad zakresem swojego projektu
