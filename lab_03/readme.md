# Aplikacje WWW, semestr 2026Z

## Lab 3 – Architektura MVT: Silnik szablonów Django (Templates) i dynamiczne widoki HTML
---

### 1. Dlaczego omawiamy szablony przed wejściem w świat REST API?

Współczesne aplikacje internetowe dzielą się na dwa główne podejścia architektoniczne:
1. **Server-Side Rendering (SSR / Klasyczny Monolit MVT):** Serwer odbiera żądanie HTTP, pobiera dane z bazy za pomocą ORM, wstrzykuje je do szablonu HTML i odsyła do przeglądarki gotowy dokument HTML wraz ze stylami CSS.
2. **Decoupled Architecture (Backend API + Frontend SPA/Mobile):** Serwer backendowy jest całkowicie odseparowany od warstwy wizualnej – udostępnia jedynie endpointy REST API zwracające czyste dane (zazwyczaj w formacie JSON). Interfejs użytkownika renderowany jest po stronie klienta (np. w React, Vue, Angular lub aplikacji mobilnej).

Aby w pełni docenić korzyści płynące z Django REST Framework (DRF) i zrozumieć, czym różni się zwracanie danych w JSON od renderowania widoku w przeglądarce, na tym laboratorium zbudujemy klasyczną warstwę prezentacji w architekturze **MVT (Model-View-Template)**.

---

### 2. Silnik szablonów Django (Django Template Language – DTL)

[Szablony Django (DTL)](https://docs.djangoproject.com/en/6.1/topics/templates/) to pliki HTML wzbogacone o specjalne [wbudowane znaczniki i filtry](https://docs.djangoproject.com/en/6.1/ref/templates/builtins/):
* `{{ zmienna }}` – wypisanie wartości zmiennej przekazanej ze słownika kontekstu (`context`).
* `{{ zmienna|filtr }}` – przekształcenie wartości za pomocą filtra (np. `{{ event.title|upper }}`, `{{ event.date|date:"d.m.Y H:i" }}`).
* `{% znacznik %}` – instrukcje sterujące logiką szablonu (pętle `{% for item in items %}`, warunki `{% if condition %}`, dziedziczenie `{% extends %}`, bloki `{% block %}`).
* `{% url 'nazwa_trasy' argument %}` – dynamiczne generowanie adresów URL bez wpisywania ich na sztywno ([Reverse resolution of URLs](https://docs.djangoproject.com/en/6.1/topics/http/urls/#reverse-resolution-of-urls)).

---

### 3. Instrukcje krok po kroku

#### Krok 1: Struktura katalogów dla szablonów i plików statycznych

Django domyślnie wyszukuje szablony w podfolderze `templates/<nazwa_aplikacji>/` wewnątrz każdej aplikacji, aby uniknąć kolizji nazw pomiędzy różnymi aplikacjami projektu. [Pliki statyczne](https://docs.djangoproject.com/en/6.1/howto/static-files/) (CSS, JS, grafiki) umieszczamy analogicznie w `static/<nazwa_aplikacji>/`.

W folderze `events/` stwórz następującą strukturę:
```text
events/
├── static/
│   └── events/
│       └── css/
│           └── style.css
└── templates/
    └── events/
        ├── base.html              # Szablon nadrzędny (layout bazowy)
        ├── event_list.html         # Lista wszystkich nadchodzących wydarzeń
        └── event_detail.html       # Strona szczegółów pojedynczego wydarzenia
```

---

#### Krok 2: Szablon bazowy `base.html` z dziedziczeniem bloków

Plik `events/templates/events/base.html` stanowi szkielet wszystkich podstron serwisu (nagłówek, menu nawigacyjne, stopka):

```html
{% load static %}
<!DOCTYPE html>
<html lang="pl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}EventHub – Platforma Wydarzeń{% endblock %}</title>
    <link rel="stylesheet" href="{% static 'events/css/style.css' %}">
</head>
<body>
    <header class="navbar">
        <div class="container">
            <h1 class="logo"><a href="{% url 'events:event-list' %}">Event<span>Hub</span></a></h1>
            <nav>
                <a href="{% url 'events:event-list' %}">Wydarzenia</a>
                <a href="/admin/" target="_blank">Panel Administratora</a>
            </nav>
        </div>
    </header>

    <main class="container">
        {% block content %}
        <!-- Tutaj zostanie wstrzyknięta treść poszczególnych widoków -->
        {% endblock %}
    </main>

    <footer class="footer">
        <div class="container">
            <p>&copy; 2026 EventHub. Aplikacje WWW – Wydział Matematyki i Informatyki UWM.</p>
        </div>
    </footer>
</body>
</html>
```

---

#### Krok 3: Style CSS w pliku `events/static/events/css/style.css`

Utwórz plik stylów nadający aplikacji estetyczny, nowoczesny wygląd:

```css
* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
}

body {
    background-color: #f4f6f9;
    color: #333;
    line-height: 1.6;
}

.container {
    max-width: 1100px;
    margin: auto;
    padding: 20px;
}

.navbar {
    background-color: #1e293b;
    color: #fff;
    padding: 15px 0;
}

.navbar .container {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 0 20px;
}

.navbar .logo a {
    color: #fff;
    text-decoration: none;
    font-size: 1.5rem;
    font-weight: bold;
}

.navbar .logo span {
    color: #38bdf8;
}

.navbar nav a {
    color: #cbd5e1;
    text-decoration: none;
    margin-left: 20px;
    transition: color 0.2s;
}

.navbar nav a:hover {
    color: #fff;
}

.event-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
    gap: 20px;
    margin-top: 20px;
}

.card {
    background: #fff;
    border-radius: 8px;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
    padding: 20px;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
}

.card h2 {
    font-size: 1.25rem;
    margin-bottom: 10px;
}

.card .badge {
    display: inline-block;
    background-color: #e0f2fe;
    color: #0369a1;
    padding: 4px 8px;
    border-radius: 4px;
    font-size: 0.85rem;
    margin-bottom: 10px;
}

.card .meta {
    font-size: 0.9rem;
    color: #64748b;
    margin-bottom: 15px;
}

.card .btn {
    display: inline-block;
    text-align: center;
    background-color: #0284c7;
    color: #fff;
    text-decoration: none;
    padding: 8px 16px;
    border-radius: 6px;
    font-weight: 500;
}

.card .btn:hover {
    background-color: #0369a1;
}

.detail-view {
    background: #fff;
    padding: 30px;
    border-radius: 8px;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
    margin-top: 20px;
}

.detail-view h1 {
    margin-bottom: 15px;
}

.footer {
    text-align: center;
    padding: 30px 0;
    margin-top: 40px;
    color: #94a3b8;
    font-size: 0.9rem;
}
```

---

#### Krok 4: Szablon listy wydarzeń `event_list.html`

W pliku `events/templates/events/event_list.html`:

```html
{% extends 'events/base.html' %}

{% block title %}Aktualne Wydarzenia – EventHub{% endblock %}

{% block content %}
    <h2>Nadchodzące wydarzenia</h2>
    <p>Przeglądaj najnowsze konferencje, warsztaty i spotkania branżowe:</p>

    {% if events %}
        <div class="event-grid">
            {% for event in events %}
                <div class="card">
                    <div>
                        <span class="badge">{{ event.category.name }}</span>
                        <h2>{{ event.title }}</h2>
                        <p class="meta">
                            📅 {{ event.date|date:"d M Y, H:i" }}<br>
                            📍 {{ event.location }}<br>
                            💰 {% if event.price == 0 %}<strong>Bezpłatne</strong>{% else %}{{ event.price }} PLN{% endif %}
                        </p>
                        <p>{{ event.description|truncatewords:20 }}</p>
                    </div>
                    <div style="margin-top: 15px;">
                        <a href="{% url 'events:event-detail' event.slug %}" class="btn">Zobacz szczegóły</a>
                    </div>
                </div>
            {% endfor %}
        </div>
    {% else %}
        <p style="margin-top: 20px;">Brak opublikowanych wydarzeń w tej chwili.</p>
    {% endif %}
{% endblock %}
```

---

#### Krok 5: Szablon szczegółów wydarzenia `event_detail.html`

W pliku `events/templates/events/event_detail.html`:

```html
{% extends 'events/base.html' %}

{% block title %}{{ event.title }} – EventHub{% endblock %}

{% block content %}
    <div class="detail-view">
        <a href="{% url 'events:event-list' %}" style="color: #0284c7; text-decoration: none;">&larr; Wróć do listy wydarzeń</a>
        <h1 style="margin-top: 15px;">{{ event.title }}</h1>
        <span class="badge">{{ event.category.name }}</span>

        <div class="meta" style="margin: 20px 0; font-size: 1rem;">
            <p><strong>Termin:</strong> {{ event.date|date:"l, d F Y, godz. H:i" }}</p>
            <p><strong>Lokalizacja:</strong> {{ event.location }}</p>
            <p><strong>Organizator:</strong> {{ event.organizer.get_full_name|default:event.organizer.username }}</p>
            <p><strong>Dostępne miejsca:</strong> {{ event.capacity }} osób</p>
            <p><strong>Koszt udziału:</strong> {% if event.price == 0 %}Bezpłatne{% else %}{{ event.price }} PLN{% endif %}</p>
        </div>

        <h3>Opis wydarzenia</h3>
        <p style="white-space: pre-line; margin-top: 10px;">{{ event.description }}</p>

        <div style="margin-top: 30px; padding: 20px; background-color: #f8fafc; border-radius: 6px;">
            <h4>Chcesz wziąć udział?</h4>
            <p>Rezerwacja biletów przez przeglądarkę będzie dostępna wkrótce za pośrednictwem naszego REST API (Lab 4-9)!</p>
        </div>
    </div>
{% endblock %}
```

---

#### Krok 6: Implementacja widoków w `events/views.py`

Zastąp zawartość `events/views.py` dynamicznymi widokami pobierającymi dane z bazy przy użyciu [funkcji skrótowych Django (`render`, `get_object_or_404`)](https://docs.djangoproject.com/en/6.1/topics/http/shortcuts/) oraz optymalizacji zapytań [`select_related()`](https://docs.djangoproject.com/en/6.1/ref/models/querysets/#select-related) (zapobiegającej problemowi N+1 zapytań SQL):

```python
from django.shortcuts import render, get_object_or_404
from .models import Event, Category


def event_list(request):
    """Widok listy opublikowanych wydarzeń."""
    events = Event.objects.filter(is_published=True).select_related('category', 'organizer')
    return render(request, 'events/event_list.html', {'events': events})


def event_detail(request, slug):
    """Widok pojedynczego wydarzenia na podstawie parametru slug."""
    event = get_object_or_404(
        Event.objects.select_related('category', 'organizer'),
        slug=slug,
        is_published=True
    )
    return render(request, 'events/event_detail.html', {'event': event})


def events_by_category(request, category_slug):
    """Wydarzenia przefiltrowane według kategorii."""
    category = get_object_or_404(Category, slug=category_slug)
    events = category.events.filter(is_published=True).select_related('organizer')
    return render(request, 'events/event_list.html', {
        'events': events,
        'current_category': category
    })
```

---

#### Krok 7: Konfiguracja routingu URL z przestrzenią nazw (`app_name`)

W pliku `events/urls.py` dodaj zmienną `app_name = 'events'`, która definiuje [przestrzeń nazw URL (URL namespaces)](https://docs.djangoproject.com/en/6.1/topics/http/urls/#url-namespaces) i pozwala na jednoznaczne używanie tagów `{% url 'events:event-detail' ... %}`:

```python
from django.urls import path
from . import views

app_name = 'events'

urlpatterns = [
    path('', views.event_list, name='event-list'),
    path('event/<slug:slug>/', views.event_detail, name='event-detail'),
    path('category/<slug:category_slug>/', views.events_by_category, name='events-by-category'),
]
```

Uruchom serwer developerski i przetestuj działanie strony głównej oraz podstron wydarzeń pod adresem [http://127.0.0.1:8000/events/](http://127.0.0.1:8000/events/).

---

### 4. Zadania do wykonania

1. **Implementacja szablonów:** Odtwórz strukturę szablonów i plików statycznych dla projektu EventHub zgodnie z instrukcją.
2. **Dodanie menu kategorii i wyróżnienie aktywnej pozycji:** W szablonie `event_list.html` dodaj pasek nawigacyjny z linkami do wszystkich dostępnych kategorii, aby użytkownik mógł kliknąć kategorię i przefiltrować wydarzenia (wykorzystaj trasę `events:events-by-category` oraz link powrotny „Wszystkie” prowadzący do `events:event-list`).
   *(Wskazówka: przekaż listę `categories = Category.objects.all()` w kontekście obu widoków: `event_list` oraz `events_by_category`, a w szablonie użyj warunku `{% if current_category == category %}`, aby wyróżnić aktualnie wybraną kategorię innym stylem CSS)*.
3. **Wyszukiwarka wydarzeń (obsługa parametrów `GET`):** W szablonie `event_list.html` dodaj prosty formularz wyszukiwania (`<form method="get" action="{% url 'events:event-list' %}">`) z polem tekstowym `<input type="text" name="q" placeholder="Szukaj po tytule...">` oraz przyciskiem „Szukaj”. W widoku `event_list` odczytaj parametr `q = request.GET.get('q')` i jeśli nie jest pusty, przefiltruj listę wydarzeń (`events = events.filter(title__icontains=q)`). Przekaż zmienną `q` z powrotem do kontekstu szablonu, aby zachować wpisaną frazę w polu formularza.
4. **Sekcja „Podobne wydarzenia” na stronie szczegółów:** W widoku `event_detail` pobierz z bazy maksymalnie 3 inne opublikowane wydarzenia należące do tej samej kategorii co przeglądane wydarzenie, wykluczając bieżący obiekt (użyj `.filter(category=event.category, is_published=True).exclude(id=event.id)[:3]`). Przekaż je w słowniku `context` jako `related_events` i wyświetl na dole szablonu `event_detail.html` z wykorzystaniem klauzuli `{% for rel_event in related_events %} ... {% empty %} ... {% endfor %}`.
5. **Komponenty wielokrotnego użytku (`{% include %}`) i filtry czasu:** Wydziel kod HTML pojedynczej karty wydarzenia (`<div class="card">...</div>`) do osobnego pliku podszablonu `events/templates/events/includes/event_card.html` i dołącz go w pętli w `event_list.html` za pomocą znacznika [`{% include %}`](https://docs.djangoproject.com/en/6.1/ref/templates/builtins/#include). Dodatkowo rozbuduj kartę o informację, za ile czasu odbędzie się wydarzenie, wykorzystując wbudowany filtr [`{{ event.date|timeuntil }}`](https://docs.djangoproject.com/en/6.1/ref/templates/builtins/#timeuntil).
6. **Własny projekt:**
   * W swoim projekcie zaliczeniowym przygotuj szablon bazowy `base.html` oraz co najmniej jeden widok dynamiczny renderujący listę głównych obiektów z bazy.
   * Sprawdź poprawność działania podpinając trasę w pliku `urls.py`.
7. **Git:** Zatwierdź zmiany commitem: `feat: Add MVT templates, static CSS, and dynamic event views`.
