# Aplikacje WWW, semestr 2026Z

## Lab 5 – Relacje `ManyToManyField`, zaawansowany `ModelSerializer`, pola obliczane i widoki funkcyjne (`@api_view`)
---

### 1. Przejście na `ModelSerializer` i rozbudowa relacji – zasada DRY w praktyce

Na poprzednich zajęciach zauważyliśmy, że ręczne definiowanie każdego pola oraz metod `create()` i `update()` w klasie `serializers.Serializer` jest czasochłonne i dubluje reguły zdefiniowane w modelach ORM.

Klasa **[`ModelSerializer`](https://www.django-rest-framework.org/api-guide/serializers/#modelserializer)** automatyzuje ten proces:
* Automatycznie generuje zestaw pól serializatora na podstawie pól wskazanego modelu Django.
* Automatycznie generuje walidatory z definicji modelu (np. weryfikację `unique=True`, maksymalnej długości `max_length`, dopuszczalności wartości `null`/`blank`).
* Dostarcza gotowe, przetestowane implementacje metod `create()` oraz `update()` (włącznie z obsługą relacji wiele-do-wielu!).

Jednocześnie rozbudujemy dziś domenę projektu **EventHub** o relację wiele-do-wielu ([`ManyToManyField`](https://docs.djangoproject.com/en/6.1/topics/db/examples/many_to_many/)) za pomocą modelu `Tag`, a także poznamy techniki rozdzielania reprezentacji danych do odczytu (zagnieżdżone obiekty, [`SerializerMethodField`](https://www.django-rest-framework.org/api-guide/fields/#serializermethodfield)) od reprezentacji do zapisu (klucze główne).

---

### 2. Poziomy walidacji danych w Django REST Framework

Przed zapisaniem danych do bazy framework weryfikuje ich poprawność w trzech uporządkowanych krokach:
1. **Walidacja typu i wbudowanych parametrów pola:** Sprawdzana automatycznie przez klasę pola (np. czy liczba jest całkowita, czy data ma poprawny format ISO 8601, czy klucz obcy istnieje w bazie).
2. **[Walidacja pojedynczego pola (`validate_<nazwa_pola>`)](https://www.django-rest-framework.org/api-guide/serializers/#field-level-validation):**
   Metoda wewnątrz serializatora, która przyjmuje wartość konkretnego pola i zwraca zwalidowaną wartość lub rzuca wyjątek `serializers.ValidationError`:
   ```python
   def validate_date(self, value):
       if value < timezone.now():
           raise serializers.ValidationError("Data wydarzenia nie może być z przeszłości!")
       return value
   ```
3. **[Walidacja na poziomie całego obiektu (`validate`)](https://www.django-rest-framework.org/api-guide/serializers/#object-level-validation):**
   Metoda `validate(self, attrs)` służy do walidacji krzyżowej (zależności pomiędzy kilkoma polami jednocześnie). Przyjmuje słownik zwalidowanych wartości pól i zwraca go lub rzuca `ValidationError`.
   > **Wskazówka architektoniczna:** Przy częściowej aktualizacji (`PATCH`) w słowniku `attrs` znajdują się tylko te pola, które klient przesłał w żądaniu. Jeśli walidacja krzyżowa zależy od pola, którego klient nie przesłał w `PATCH`, jego aktualną wartość należy pobrać z edytowanej instancji modelu (`self.instance`).

---

### 3. Instrukcje krok po kroku

#### Krok 1: Rozbudowa modeli o relację `ManyToManyField` (Model `Tag`)

W dotychczasowym schemacie każde wydarzenie należało do jednej kategorii (`ForeignKey`). W praktyce wydarzenie może być oznaczone wieloma tagami tematycznymi (np. `#python`, `#ai`, `#backend`, `#networking`), a jeden tag może być przypisany do wielu różnych wydarzeń.

Otwórz plik `events/models.py`, dodaj model `Tag` oraz pole `tags` w modelu `Event`:

```python
from django.db import models
from django.contrib.auth.models import User
from django.core.validators import MinValueValidator


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


class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True, verbose_name="Nazwa tagu")
    slug = models.SlugField(max_length=60, unique=True, verbose_name="Slug tagu")

    class Meta:
        verbose_name = "Tag"
        verbose_name_plural = "Tagi"
        ordering = ['name']

    def __str__(self):
        return f"#{self.name}"


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
    tags = models.ManyToManyField(
        Tag,
        related_name="events",
        blank=True,
        verbose_name="Tagi tematyczne"
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

Wygeneruj i zastosuj migrację:
```bash
python manage.py makemigrations
python manage.py migrate
```

Zaktualizuj plik `events/admin.py`, rejestrując model `Tag` oraz dodając wygodny widżet `filter_horizontal` dla relacji wiele-do-wielu w `EventAdmin`:

```python
from django.contrib import admin
from .models import Category, Tag, Event


@admin.register(Tag)
class TagAdmin(admin.ModelAdmin):
    list_display = ('name', 'slug')
    search_fields = ('name',)
    prepopulated_fields = {'slug': ('name',)}
```
*(W klasie `EventAdmin` dopisz `filter_horizontal = ('tags',)` oraz dodaj pole `'tags'` do sekcji `"Podstawowe informacje"` w `fieldsets`)*.

---

#### Krok 2: Definicja `TagSerializer`, `CategorySerializer` i `EventModelSerializer`

Otwórz plik `events/serializers.py`. Zdefiniujemy w nim serializatory oparte na `ModelSerializer`, wykorzystując:
* [`ReadOnlyField`](https://www.django-rest-framework.org/api-guide/fields/#readonlyfield) – do spłaszczania pól z modeli powiązanych (np. `category.name`),
* **Zagnieżdżoną reprezentację relacji M2M do odczytu** (`TagSerializer(many=True, read_only=True)`) oraz listę kluczy głównych do zapisu (`PrimaryKeyRelatedField(many=True, write_only=True, source='tags')`),
* [`SerializerMethodField`](https://www.django-rest-framework.org/api-guide/fields/#serializermethodfield) – do wyliczania dynamicznych właściwości obiektu w locie (`is_free`, `days_until_event`).

```python
from rest_framework import serializers
from django.utils import timezone
from .models import Category, Tag, Event


class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name', 'slug']
        read_only_fields = ['id']


class CategorySerializer(serializers.ModelSerializer):
    events_count = serializers.IntegerField(source='events.count', read_only=True)

    class Meta:
        model = Category
        fields = ['id', 'name', 'slug', 'description', 'events_count', 'created_at']
        read_only_fields = ['id', 'created_at', 'events_count']


class EventModelSerializer(serializers.ModelSerializer):
    # 1. Pola pomocnicze tylko do odczytu (zamiast samych kluczy obcych ID)
    category_name = serializers.ReadOnlyField(source='category.name')
    organizer_username = serializers.ReadOnlyField(source='organizer.username')

    # 2. Obsługa relacji ManyToMany (Tagi):
    # - w odpowiedziach GET zwracamy pełne obiekty tagów (id, name, slug)
    tags_details = TagSerializer(source='tags', many=True, read_only=True)
    # - w żądaniach POST/PUT/PATCH przyjmujemy tablicę identyfikatorów, np. "tag_ids": [1, 2]
    tag_ids = serializers.PrimaryKeyRelatedField(
        many=True,
        queryset=Tag.objects.all(),
        source='tags',
        write_only=True,
        required=False
    )

    # 3. Pola dynamiczne wyliczane metodą (SerializerMethodField)
    is_free = serializers.SerializerMethodField()
    days_until_event = serializers.SerializerMethodField()

    class Meta:
        model = Event
        fields = [
            'id', 'title', 'slug', 'category', 'category_name',
            'organizer', 'organizer_username', 'tags_details', 'tag_ids',
            'description', 'location', 'date', 'capacity', 'price',
            'is_free', 'days_until_event', 'is_published',
            'created_at', 'updated_at'
        ]
        read_only_fields = ['id', 'created_at', 'updated_at']

    def get_is_free(self, obj) -> bool:
        """Zwraca True, jeśli wydarzenie jest bezpłatne."""
        return obj.price == 0

    def get_days_until_event(self, obj) -> int:
        """Oblicza liczbę pełnych dni pozostałych do rozpoczęcia wydarzenia (lub 0, jeśli już minęło)."""
        delta = obj.date - timezone.now()
        return max(0, delta.days)

    # Walidacja na poziomie pojedynczego pola: data wydarzenia
    def validate_date(self, value):
        if value <= timezone.now():
            raise serializers.ValidationError("Termin wydarzenia musi być datą w przyszłości!")
        return value

    # Walidacja na poziomie pojedynczego pola: minimalna pojemność sali
    def validate_capacity(self, value):
        if value < 5:
            raise serializers.ValidationError("Minimalna liczba uczestników wydarzenia wynosi 5 osób.")
        return value

    # Walidacja krzyżowa na poziomie całego obiektu (obsługuje również częściowy PATCH!)
    def validate(self, attrs):
        # Pobieramy wartości ze słownika attrs, a przy metodzie PATCH posiłkujemy się istniejącą instancją
        title = attrs.get('title', getattr(self.instance, 'title', ''))
        price = attrs.get('price', getattr(self.instance, 'price', 0))
        is_published = attrs.get('is_published', getattr(self.instance, 'is_published', True))
        description = attrs.get('description', getattr(self.instance, 'description', ''))

        # Reguła biznesowa 1: słowo "darmowy"/"bezpłatny"/"free" w tytule wymusza cenę 0.00 PLN
        if any(word in title.lower() for word in ['darmow', 'bezpłatn', 'free']) and price > 0:
            raise serializers.ValidationError({
                "price": "Wydarzenie oznaczone w tytule jako bezpłatne nie może mieć ceny wyższej niż 0.00 PLN."
            })

        # Reguła biznesowa 2: publikowane wydarzenie musi mieć opis o długości min. 20 znaków
        if is_published and len(description.strip()) < 20:
            raise serializers.ValidationError({
                "description": "Opublikowane wydarzenie musi posiadać szczegółowy opis (minimum 20 znaków)."
            })

        return attrs
```

---

#### Krok 3: Widoki funkcyjne DRF (`@api_view`) i optymalizacja zapytań ORM

W DRF nie używamy surowych obiektów `HttpRequest` i `HttpResponse`. Zamiast tego operujemy na:
* [`Request`](https://www.django-rest-framework.org/api-guide/requests/) – obiekt DRF rozszerzający żądanie HTTP o właściwość `request.data` (automatycznie parsującą JSON dla metod `POST`, `PUT`, `PATCH`) oraz `request.query_params` (odpowiednik `request.GET`).
* [`Response`](https://www.django-rest-framework.org/api-guide/responses/) – obiekt DRF przyjmujący słowniki lub listy Pythona, które negocjator treści renderuje do formatu JSON z jawnym [kodem statusu HTTP (`rest_framework.status`)](https://www.django-rest-framework.org/api-guide/status-codes/).
* [Dekoratorze `@api_view`](https://www.django-rest-framework.org/api-guide/views/#api_view) – opakowującym widoki funkcyjne i określającym dozwolone metody HTTP.

> **Optymalizacja zapytań SQL (Problem N+1):** Ponieważ `EventModelSerializer` odwołuje się do `event.category.name`, `event.organizer.username` oraz listy `event.tags.all()`, pobranie 20 wydarzeń bez optymalizacji wykonałoby aż $1 + 20 + 20 + 20 = 61$ zapytań do bazy danych! Zastosowanie `.select_related('category', 'organizer')` (dla relacji `ForeignKey` – złączenie `JOIN`) oraz `.prefetch_related('tags')` (dla relacji `ManyToManyField`) redukuje tę liczbę do zaledwie **2 zapytań SQL**.

Utwórz plik `events/api_views.py` i zaimplementuj poniższe widoki:

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
from django.shortcuts import get_object_or_404
from .models import Event, Category
from .serializers import EventModelSerializer, CategorySerializer


@api_view(['GET', 'POST'])
def event_list_create_api(request):
    """
    GET: Zwraca listę wydarzeń (opcjonalnie przefiltrowanych po kategorii).
    POST: Tworzy nowe wydarzenie po przejściu walidacji.
    """
    if request.method == 'GET':
        events = Event.objects.select_related('category', 'organizer').prefetch_related('tags').all()

        category_id = request.query_params.get('category')
        if category_id:
            events = events.filter(category_id=category_id)

        serializer = EventModelSerializer(events, many=True)
        return Response(serializer.data, status=status.HTTP_200_OK)

    elif request.method == 'POST':
        serializer = EventModelSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


@api_view(['GET', 'PUT', 'PATCH', 'DELETE'])
def event_detail_api(request, pk):
    """
    GET: Szczegóły pojedynczego wydarzenia.
    PUT: Pełna aktualizacja wydarzenia (wymaga wszystkich pól wymaganych).
    PATCH: Częściowa aktualizacja wybranych pól wydarzenia (partial=True).
    DELETE: Usunięcie wydarzenia.
    """
    queryset = Event.objects.select_related('category', 'organizer').prefetch_related('tags')
    event = get_object_or_404(queryset, pk=pk)

    if request.method == 'GET':
        serializer = EventModelSerializer(event)
        return Response(serializer.data, status=status.HTTP_200_OK)

    elif request.method == 'PUT':
        serializer = EventModelSerializer(event, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_200_OK)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'PATCH':
        serializer = EventModelSerializer(event, data=request.data, partial=True)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_200_OK)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        event.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)


@api_view(['GET'])
def category_list_api(request):
    """Zwraca listę wszystkich kategorii z liczbą powiązanych wydarzeń."""
    categories = Category.objects.all()
    serializer = CategorySerializer(categories, many=True)
    return Response(serializer.data, status=status.HTTP_200_OK)
```

---

#### Krok 4: Rejestracja endpointów REST API w routingu i testowanie

W pliku `events/urls.py` dodaj ścieżki dla utworzonych widoków API:

```python
from django.urls import path
from . import views, api_views

app_name = 'events'

urlpatterns = [
    # Widoki HTML (MVT)
    path('', views.event_list, name='event-list'),
    path('event/<slug:slug>/', views.event_detail, name='event-detail'),
    path('category/<slug:category_slug>/', views.events_by_category, name='events-by-category'),

    # Endpointy REST API (DRF - Widoki funkcyjne)
    path('api/events/', api_views.event_list_create_api, name='api-event-list-create'),
    path('api/events/<int:pk>/', api_views.event_detail_api, name='api-event-detail'),
    path('api/categories/', api_views.category_list_api, name='api-category-list'),
]
```

Uruchom serwer developerski (`python manage.py runserver`) i przetestuj endpointy:
1. Wejdź w przeglądarce pod adres [http://127.0.0.1:8000/events/api/events/](http://127.0.0.1:8000/events/api/events/) i zobacz **DRF Browsable API**.
2. Wyślij żądanie `POST` tworzące nowe wydarzenie z przypisanymi tagami (`"tag_ids": [1, 2]`) i sprawdź, że w odpowiedzi otrzymujesz tablicę obiektów `tags_details` oraz pola `is_free` i `days_until_event`.
3. Przetestuj częściową aktualizację `PATCH` na `/events/api/events/1/`, przesyłając wyłącznie `{"price": "99.00"}`.

---

### 4. Zadania do wykonania

1. **Pełny CRUD dla kategorii z ochroną przed usunięciem niepustej kategorii:**
   * Rozbuduj widok `category_list_api` w `events/api_views.py` o obsługę metody `POST` (tworzenie nowej kategorii ze statusem `201 Created` lub zwrot błędów walidacji ze statusem `400 Bad Request`).
   * Dopisz nowy widok funkcyjny `category_detail_api(request, pk)` obsługujący metody `['GET', 'PUT', 'PATCH', 'DELETE']` i podepnij go w `events/urls.py` pod ścieżką `'api/categories/<int:pk>/'`.
   * **Reguła biznesowa przy usuwaniu (`DELETE`):** Zanim wywołasz `category.delete()`, sprawdź, czy do tej kategorii są przypisane jakiekolwiek wydarzenia (`if category.events.exists():`). Jeśli tak, **zablokuj usunięcie** i zwróć odpowiedź ze statusem `status.HTTP_400_BAD_REQUEST` oraz treścią:
     `{"error": "Nie można usunąć kategorii, do której są przypisane wydarzenia."}`. Jeśli kategoria jest pusta, usuń ją i zwróć `204 No Content`.

2. **Endpointy dla modelu `Tag` oraz filtrowanie wydarzeń po tagu i cenie:**
   * W pliku `events/api_views.py` utwórz widok `tag_list_create_api(request)` obsługujący metody `GET` (lista wszystkich tagów posortowanych alfabetycznie) oraz `POST` (dodanie nowego tagu za pomocą `TagSerializer`). Zarejestruj go w `events/urls.py` pod adresem `'api/tags/'`.
   * Rozbuduj część `GET` w widoku `event_list_create_api` o obsługę dwóch nowych parametrów zapytania (`request.query_params`):
     * `?tag=<slug>` – zwraca tylko wydarzenia posiadające tag o podanym slugu (wskazówka: użyj `events.filter(tags__slug=tag_slug)`),
     * `?free=true` – jeśli parametr `free` ma wartość `'true'` (case-insensitive), zwraca wyłącznie darmowe wydarzenia (`price=0`).

3. **Rozdzielenie serializatora listy od serializatora szczegółów (`EventListSerializer`):**
   * Na liście wszystkich wydarzeń nie chcemy przesyłać długiego tekstu `description` ani szczegółowych dat aktualizacji, ale chcemy wyświetlić skrót opisu oraz liczbę przypisanych tagów.
   * W pliku `events/serializers.py` utwórz nową klasę `EventListSerializer(serializers.ModelSerializer)`, która zwraca wyłącznie pola:
     `['id', 'title', 'slug', 'category_name', 'organizer_username', 'date', 'location', 'price', 'is_free', 'short_description', 'tags_count']`.
   * Pole `short_description` zdefiniuj jako `SerializerMethodField`, które zwraca pierwsze 80 znaków opisu zakończone wielokropkiem `"..."` (jeśli opis jest dłuższy niż 80 znaków) lub pełny opis (jeśli jest krótszy).
   * Pole `tags_count` zdefiniuj jako `SerializerMethodField` (lub `IntegerField(source='tags.count', read_only=True)`) zwracające liczbę tagów przypisanych do wydarzenia.
   * Zmień widok `event_list_create_api` tak, aby dla metody `GET` używał `EventListSerializer`, natomiast dla metody `POST` (oraz w całym `event_detail_api`) nadal używał pełnego `EventModelSerializer`.

4. **Zaawansowana walidacja krzyżowa dla wydarzeń online i stacjonarnych:**
   * Rozbuduj metodę `validate(self, attrs)` w klasie `EventModelSerializer` o dwie nowe reguły biznesowe (pamiętając o obsłudze metody `PATCH` poprzez `getattr(self.instance, ...)`):
     * **Wydarzenia online:** Jeśli w polu `title` lub `description` występuje słowo `"online"` lub `"webinar"` (bez względu na wielkość liter), to wartość pola `location` **musi** zaczynać się od prefiksu `'http://'` lub `'https://'`. W przeciwnym razie rzuć `serializers.ValidationError({"location": "Dla wydarzeń online pole lokalizacji musi zawierać poprawny adres URL (http:// lub https://)."})`.
     * **Limit miejsc na wydarzeniach stacjonarnych:** Jeśli lokalizacja (`location`) **nie** zaczyna się od `'http://'` ani `'https://'` (czyli wydarzenie jest stacjonarne), maksymalna pojemność (`capacity`) nie może przekraczać `1000` osób. W przypadku przekroczenia rzuć błąd walidacji przypisany do pola `"capacity"`.

5. **Projekt własny (Kamień milowy 1):**
   * Rozbuduj schemat bazy danych swojego projektu zaliczeniowego do docelowych **4–5 powiązanych ze sobą modeli** (wprowadzając obok `ForeignKey` również relację `ManyToManyField` lub `OneToOneField`). Wykonaj migracje i skonfiguruj panel `admin.py`.
   * Dla wszystkich modeli zaimplementuj klasy `ModelSerializer`, wykorzystując `ReadOnlyField`, zagnieżdżone reprezentacje relacji oraz minimum dwa pola wyliczane przez `SerializerMethodField`.
   * Dodaj w głównym serializatorze minimum **3 nietrywialne reguły walidacji** (w tym walidację krzyżową w metodzie `validate()`).
   * Zaimplementuj widoki funkcyjne `@api_view` (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) z optymalizacją `select_related` / `prefetch_related` dla minimum dwóch zasobów swojego projektu.

6. **Git:** Zatwierdź zmiany commitem: `feat: Add Tag M2M relation, advanced ModelSerializers, custom validations, and FBV endpoints`.
