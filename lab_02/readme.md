# Aplikacje WWW, semestr 2026Z

## Lab 2 – Modele danych (Django ORM), relacje, migracje i panel Django Admin
---

### 1. Wprowadzenie do Django ORM

W architekturze Django nie piszemy ręcznie zapytań SQL typu `CREATE TABLE` czy `SELECT * FROM ...`. Zamiast tego wykorzystujemy **ORM (Object-Relational Mapping)** – mechanizm, który pozwala definiować schemat relacyjnej bazy danych za pomocą klas języka Python dziedziczących po [`django.db.models.Model`](https://docs.djangoproject.com/en/6.1/topics/db/models/).

Każda klasa modelu odpowiada jednej tabeli w bazie danych, atrybut klasy ([pole modelu](https://docs.djangoproject.com/en/6.1/ref/models/fields/#model-field-types)) odpowiada kolumnie w tabeli, a każda instancja klasy reprezentuje pojedynczy wiersz (rekord).

---

### 2. Projekt schematu bazy danych dla platformy EventHub

W aplikacji `events` zdefiniujemy dwa podstawowe, powiązane ze sobą modele:
1. `Category` – kategorie tematyczne wydarzeń (np. IT & Programowanie, Konferencje Biznesowe, Warsztaty Designu).
2. `Event` – szczegółowe informacje o wydarzeniu (tytuł, data, miejsce, limit miejsc, cena biletu, powiązanie relacją [`ForeignKey`](https://docs.djangoproject.com/en/6.1/ref/models/fields/#foreignkey) z kategorią oraz organizatorem).

Zauważ, że model `Event` korzysta z wbudowanego w Django modelu użytkownika [`django.contrib.auth.models.User`](https://docs.djangoproject.com/en/6.1/topics/auth/default/#user-objects), reprezentującego organizatora wydarzenia.

---

### 3. Instrukcje krok po kroku

#### Krok 1: Definicja modeli w pliku `events/models.py`

Otwórz plik `events/models.py` i wprowadź poniższy kod:

```python
from django.db import models
from django.contrib.auth.models import User
from django.core.validators import MinValueValidator, MaxValueValidator


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True, verbose_name="Nazwa kategorii")
    slug = models.SlugField(max_length=120, unique=True, verbose_name="Slug (URL)")
    description = models.TextField(blank=True, verbose_name="Opis kategorii")
    created_at = models.DateTimeField(auto_now_add=True, verbose_name="Data utworzenia")

    class Meta:
        verbose_name = "Kategoria"
        verbose_name_plural = "Kategorie"
        ordering = ['name']

    def __str__(self):
        return self.name


class Event(models.Model):
    title = models.CharField(max_length=200, verbose_name="Tytuł wydarzenia")
    slug = models.SlugField(max_length=220, unique=True, verbose_name="Slug (URL)")
    category = models.ForeignKey(
        Category,
        on_delete=models.CASCADE,
        related_name="events",
        verbose_name="Kategoria"
    )
    organizer = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name="organized_events",
        verbose_name="Organizator"
    )
    description = models.TextField(verbose_name="Szczegółowy opis")
    location = models.CharField(max_length=255, verbose_name="Miejsce / Adres / Link online")
    date = models.DateTimeField(verbose_name="Data i godzina rozpoczęcia")
    capacity = models.PositiveIntegerField(
        validators=[MinValueValidator(1)],
        verbose_name="Maksymalna liczba uczestników"
    )
    price = models.DecimalField(
        max_digits=8,
        decimal_places=2,
        default=0.00,
        validators=[MinValueValidator(0.00)],
        verbose_name="Cena biletu (PLN)"
    )
    is_published = models.BooleanField(default=True, verbose_name="Opublikowane")
    created_at = models.DateTimeField(auto_now_add=True, verbose_name="Data dodania")
    updated_at = models.DateTimeField(auto_now=True, verbose_name="Ostatnia aktualizacja")

    class Meta:
        verbose_name = "Wydarzenie"
        verbose_name_plural = "Wydarzenia"
        ordering = ['date']

    def __str__(self):
        return f"{self.title} ({self.date.strftime('%Y-%m-%d %H:%M')})"
```

**Kluczowe elementy powyższego kodu:**
* `related_name="events"` – pozwala na odpytanie relacji wstecznej z obiektu kategorii: `category.events.all()`.
* `on_delete=models.CASCADE` – w przypadku usunięcia kategorii, powiązane z nią wydarzenia również zostaną usunięte.
* `MinValueValidator` – [wbudowany walidator](https://docs.djangoproject.com/en/6.1/ref/validators/) sprawdzający dopuszczalną wartość minimalną pola.
* `auto_now_add=True` – wartość ustawiana automatycznie tylko raz, w momencie tworzenia rekordu.
* `auto_now=True` – wartość aktualizowana za każdym razem, gdy wywoływane jest `.save()`.
* `class Meta` – [opcje metadanych modelu](https://docs.djangoproject.com/en/6.1/ref/models/options/): domyślne sortowanie (`ordering`) oraz polskie nazwy w panelu admina.

---

#### Krok 2: Przygotowanie i nałożenie migracji

[Migracje w Django](https://docs.djangoproject.com/en/6.1/topics/migrations/) to pliki w języku Python rejestrujące historię zmian w modelach:
1. Przygotuj pliki migracji na podstawie zmian w `models.py`:
   ```bash
   python manage.py makemigrations
   ```
   W folderze `events/migrations/` pojawi się plik `0001_initial.py`.
2. Podejrzyj kod SQL, który zostanie wykonany na bazie ([`sqlmigrate`](https://docs.djangoproject.com/en/6.1/ref/django-admin/#sqlmigrate)):
   ```bash
   python manage.py sqlmigrate events 0001
   ```
3. Zastosuj migracje do bazy danych ([`migrate`](https://docs.djangoproject.com/en/6.1/ref/django-admin/#migrate)):
   ```bash
   python manage.py migrate
   ```
   Operacja ta utworzy tabele zarówno dla wbudowanych modułów Django (użytkownicy, sesje, uprawnienia), jak i dla aplikacji `events`.

---

#### Krok 3: Tworzenie konta administratora (Superuser)

Aby zalogować się do wbudowanego panelu administracyjnego, stwórz konto z uprawnieniami superużytkownika ([`createsuperuser`](https://docs.djangoproject.com/en/6.1/ref/django-admin/#createsuperuser)):
```bash
python manage.py createsuperuser
```
Podaj nazwę użytkownika (np. `admin`), adres e-mail oraz hasło.

---

#### Krok 4: Rejestracja i personalizacja modeli w Django Admin

W pliku `events/admin.py` zarejestrujemy modele oraz skonfigurujemy interfejs [panelu administracyjnego Django](https://docs.djangoproject.com/en/6.1/ref/contrib/admin/) za pomocą klas [`ModelAdmin`](https://docs.djangoproject.com/en/6.1/ref/contrib/admin/#modeladmin-objects):

```python
from django.contrib import admin
from .models import Category, Event


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ('name', 'slug', 'events_count', 'created_at')
    search_fields = ('name', 'description')
    prepopulated_fields = {'slug': ('name',)}

    # Dekorator @admin.display formatuje niestandardowe kolumny na liście rekordów
    # Zob. https://docs.djangoproject.com/en/6.1/ref/contrib/admin/#django.contrib.admin.display
    @admin.display(description="Liczba wydarzeń")
    def events_count(self, obj):
        return obj.events.count()


@admin.register(Event)
class EventAdmin(admin.ModelAdmin):
    list_display = ('title', 'category', 'organizer', 'date', 'price', 'capacity', 'is_published')
    list_filter = ('category', 'is_published', 'date')
    search_fields = ('title', 'description', 'location')
    prepopulated_fields = {'slug': ('title',)}
    list_editable = ('is_published', 'price')
    date_hierarchy = 'date'
    ordering = ('date',)

    fieldsets = (
        ("Podstawowe informacje", {
            'fields': ('title', 'slug', 'category', 'organizer', 'description')
        }),
        ("Szczegóły organizacyjne", {
            'fields': ('location', 'date', 'capacity', 'price', 'is_published')
        }),
    )
```

Uruchom serwer developerski:
```bash
python manage.py runserver
```
Wejdź pod adres [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/), zaloguj się i dodaj:
1. Minimum 3 kategorie (np. `Technologia`, `Biznes`, `Warsztaty`).
2. Minimum 4 różne wydarzenia przypisane do tych kategorii.

---

#### Krok 5: Operacje na bazie za pomocą powłoki Django (`manage.py shell`)

Django dostarcza interaktywną powłokę Pythona ([`manage.py shell`](https://docs.djangoproject.com/en/6.1/ref/django-admin/#shell)) ze wstępnie załadowanym środowiskiem projektu:
```bash
python manage.py shell
```

Wykonaj poniższe operacje, aby poznać [API klasy `QuerySet`](https://docs.djangoproject.com/en/6.1/ref/models/querysets/):

```python
from django.contrib.auth.models import User
from events.models import Category, Event
from decimal import Decimal

# 1. Pobranie wszystkich kategorii
Category.objects.all()

# 2. Pobranie pojedynczego obiektu
cat = Category.objects.get(id=1)
print(cat.name)

# 3. Dodanie nowej kategorii z poziomu kodu
new_cat = Category.objects.create(name="Design i UX", slug="design-i-ux", description="Warsztaty projektowe")

# 4. Filtrowanie wydarzeń (wydarzenia z ceną większą niż 0 zł)
paid_events = Event.objects.filter(price__gt=Decimal('0.00'))
for event in paid_events:
    print(f"{event.title} - {event.price} PLN")

# 5. Wyszukiwanie frazy w tytule (case-insensitive)
Event.objects.filter(title__icontains="warsztat")

# 6. Relacja wsteczna: pobranie wszystkich wydarzeń dla danej kategorii
cat.events.all()

# Wyjście z shella
exit()
```

---

### 4. Zadania do wykonania

1. **Implementacja w EventHub:** Wprowadź modele `Category` oraz `Event` w projekcie `eventhub`, wykonaj migracje i utwórz superużytkownika.
2. **Personalizacja Admina:** Skonfiguruj `CategoryAdmin` oraz `EventAdmin` zgodnie z kodem z listingu. Zwróć uwagę na działanie parametru `prepopulated_fields` (automatyczne uzupełnianie sluga podczas wpisywania tytułu).
3. **Plik z zapytaniami QuerySet:** W folderze `events/` utwórz plik `orm_queries.py` i umieść w nim kod wykonujący następujące zapytania:
   * Wyświetlenie wszystkich opublikowanych wydarzeń (`is_published=True`) posortowanych chronologicznie.
   * Wyświetlenie wydarzeń, których limit miejsc wynosi co najmniej 50 osób.
   * Wyświetlenie łącznej liczby darmowych wydarzeń (`price=0.00`).
   * Zmiana statusu `is_published` wybranego wydarzenia z `True` na `False` i zapisanie zmian w bazie (`.save()`).
4. **Projekt własny (Kamień milowy):**
   * Przedstaw prowadzącemu wybrany temat projektu zaliczeniowego.
   * W nowym repozytorium projektu zaliczeniowego zaimplementuj minimum 2 początkowe modele danych, wykonaj migracje i zarejestruj je w panelu administracyjnym.
