# Aplikacje WWW, semestr 2026Z

## Lab 4 – Wprowadzenie do Django REST Framework (DRF) i anatomia serializacji
---

### 1. Wprowadzenie: Czym jest REST API i po co nam DRF?

Na poprzednich zajęciach serwer generował gotowy kod HTML (MVT). Współczesne systemy wymagają jednak komunikacji z różnorodnymi klientami:
* Aplikacje internetowe Single Page Application (SPA) w technologiach React, Vue, Angular,
* Natywne aplikacje mobilne (iOS, Android, Flutter),
* Systemy zewnętrzne i mikroserwisy (komunikacja machine-to-machine).

Takie aplikacje nie potrzebują kodu HTML (nagłówków, znaczników `<div>` czy stylów CSS), lecz **czystych danych** przesyłanych w zunifikowanym formacie – najczęściej **JSON (JavaScript Object Notation)**.

**[Django REST Framework (DRF)](https://www.django-rest-framework.org/)** to biblioteka rozszerzająca możliwości Django o narzędzia do szybkiego, bezpiecznego i standaryzowanego budowania interfejsów REST API.

---

### 2. Cykl życia danych: Serializacja i Deserializacja

Dwa podstawowe procesy, na których opiera się każde API ([Dokumentacja serializatorów DRF](https://www.django-rest-framework.org/api-guide/serializers/)):
1. **Serializacja:** Przekształcenie złożonych obiektów Pythona (np. instancji modelu `Event` lub kolekcji `QuerySet` pobranych z bazy) na natywne typy danych Pythona (słowniki, listy, liczby), które następnie renderer zamienia na strumień bajtów w formacie **JSON**:
   $$\text{Instancja Modelu} \xrightarrow{\text{Serializer}} \text{Słownik Pythona} \xrightarrow{\text{JSONRenderer}} \text{JSON (strumień tekstu/bajtów)}$$

2. **Deserializacja:** Proces odwrotny – odebranie surowych danych JSON z żądania HTTP (np. z metody `POST` lub `PUT`), sparsowanie ich do słownika Pythona, przeprowadzenie walidacji reguł biznesowych (`.is_valid()`), a po jej sukcesie utrwalenie danych w bazie (`.save()`):
   $$\text{JSON od klienta} \xrightarrow{\text{JSONParser}} \text{Słownik danych} \xrightarrow{\text{Serializer (.is\_valid())}} \text{Instancja Modelu (.save())}$$

---

### 3. Instrukcje krok po kroku

#### Krok 1: Instalacja Django REST Framework
W aktywnym środowisku wirtualnym `.venv` zainstaluj pakiet `djangorestframework`:
```bash
pip install djangorestframework
pip freeze > requirements.txt
```

Otwórz plik `eventhub/settings.py` i dodaj wpis `'rest_framework'` do listy `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Biblioteki zewnętrzne
    'rest_framework',

    # Aplikacje lokalne
    'events.apps.EventsConfig',
]
```

---

#### Krok 2: Tworzenie bazowego serializatora `serializers.Serializer`

Aby dokładnie zrozumieć mechanikę serializacji, zaczniemy od ręcznej [deklaracji pól serializatora](https://www.django-rest-framework.org/api-guide/serializers/#declaring-serializers) za pomocą bazowej klasy `serializers.Serializer` (odpowiednika `forms.Form` w klasycznym Django). Do odwzorowania kluczy obcych posłuży nam pole relacyjne [`PrimaryKeyRelatedField`](https://www.django-rest-framework.org/api-guide/relations/#primarykeyrelatedfield).

Utwórz plik `events/serializers.py` i zaimplementuj klasę `EventSerializer`:

```python
from rest_framework import serializers
from .models import Event, Category
from django.contrib.auth.models import User


class EventSerializer(serializers.Serializer):
    """
    Niskopoziomowy serializator modelu Event dziedziczący po serializers.Serializer.
    Wszystkie pola oraz operacje zapisu definiujemy jawnie.
    """
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=True, max_length=200)
    slug = serializers.SlugField(required=True, max_length=220)
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(),
        source='category',
        write_only=False
    )
    organizer_id = serializers.PrimaryKeyRelatedField(
        queryset=User.objects.all(),
        source='organizer'
    )
    description = serializers.CharField(style={'base_template': 'textarea.html'})
    location = serializers.CharField(max_length=255)
    date = serializers.DateTimeField()
    capacity = serializers.IntegerField(min_value=1)
    price = serializers.DecimalField(max_digits=8, decimal_places=2, default=0.00)
    is_published = serializers.BooleanField(default=True)
    created_at = serializers.DateTimeField(read_only=True)

    # Implementacja operacji utrwalania danych (saving instances):
    # Zob. https://www.django-rest-framework.org/api-guide/serializers/#saving-instances
    def create(self, validated_data):
        """
        Tworzy i zwraca nową instancję modelu Event na podstawie zwalidowanych danych.
        """
        return Event.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        Aktualizuje istniejącą instancję modelu Event i zwraca ją.
        """
        instance.title = validated_data.get('title', instance.title)
        instance.slug = validated_data.get('slug', instance.slug)
        instance.category = validated_data.get('category', instance.category)
        instance.organizer = validated_data.get('organizer', instance.organizer)
        instance.description = validated_data.get('description', instance.description)
        instance.location = validated_data.get('location', instance.location)
        instance.date = validated_data.get('date', instance.date)
        instance.capacity = validated_data.get('capacity', instance.capacity)
        instance.price = validated_data.get('price', instance.price)
        instance.is_published = validated_data.get('is_published', instance.is_published)
        instance.save()
        return instance
```

---

#### Krok 3: Eksperymenty w powłoce Pythona (`manage.py shell`)

Uruchom interaktywną konsolę Django:
```bash
python manage.py shell
```

##### 1. Testowanie procesu serializacji (Model -> Python dict -> JSON):
Serializator przekształca model na słownik Pythona, a [`JSONRenderer`](https://www.django-rest-framework.org/api-guide/renderers/#jsonrenderer) formatuje go do postaci JSON:
```python
import io
from events.models import Event
from events.serializers import EventSerializer
from rest_framework.renderers import JSONRenderer

# Pobieramy pierwsze wydarzenie z bazy danych
event = Event.objects.first()

# Inicjalizujemy serializator obiektem modelu
serializer = EventSerializer(event)

# 1. serializer.data zwraca natywny słownik Pythona:
print(serializer.data)
# {'id': 1, 'title': 'Django Conference 2026', 'slug': 'django-conference-2026', ...}

# 2. JSONRenderer renderuje słownik do surowego ciągu bajtów JSON:
json_bytes = JSONRenderer().render(serializer.data)
print(json_bytes)
# b'{"id":1,"title":"Django Conference 2026", ...}'
```

##### 2. Testowanie procesu deserializacji i walidacji (JSON -> dict -> Model):
Strumień JSON jest dekodowany przez [`JSONParser`](https://www.django-rest-framework.org/api-guide/parsers/#jsonparser), a następnie weryfikowany przez [mechanizm walidacji DRF](https://www.django-rest-framework.org/api-guide/serializers/#validation) (`.is_valid()`):
```python
from rest_framework.parsers import JSONParser
from django.contrib.auth.models import User
from events.models import Category

# Symulujemy odebranie strumienia JSON z żądania HTTP
stream = io.BytesIO(json_bytes)
data = JSONParser().parse(stream)

# Zmodyfikujmy dane wejściowe, aby stworzyć nowe wydarzenie
data['title'] = "Nowe Warsztaty Pythona"
data['slug'] = "nowe-warsztaty-pythona"

# Inicjalizujemy deserializator przekazując parametr data=...
deserializer = EventSerializer(data=data)

# Wywołujemy walidację
if deserializer.is_valid():
    print("Dane są poprawne!")
    print(deserializer.validated_data)
    # Zapis nowego rekordu do bazy danych
    new_event = deserializer.save()
    print(f"Utworzono wydarzenie o id={new_event.id}")
else:
    print("Błędy walidacji:", deserializer.errors)
```

##### 3. Symulacja błędnych danych i obsługa błędów:
```python
# Przekażmy błędne dane: ujemny limit miejsc i brak wymaganych pól
invalid_data = {
    'title': '',
    'capacity': -5
}
bad_serializer = EventSerializer(data=invalid_data)
print("Czy poprawne?", bad_serializer.is_valid())  # False
print("Błędy:", bad_serializer.errors)
# Zwraca słownik zawierający kody i komunikaty błędów dla poszczególnych pól!
```

Wyjdź z powłoki (`exit()`).

---

### 4. Refleksja: Dlaczego `serializers.Serializer` bywa uciążliwy?

Zauważ, że w klasie `EventSerializer`:
1. Musieliśmy ręcznie powtórzyć definicje wszystkich pól, które zdefiniowaliśmy już wcześniej w `models.Event`.
2. Musieliśmy ręcznie napisać metodę `create()` oraz `update()`.

W kolejnym laboratorium (Lab 5) poznamy klasę `ModelSerializer`, która automatycznie generuje pola na podstawie modelu, drastycznie skracając kod i eliminując duplikację reguł (zasada DRY – *Don't Repeat Yourself*).

---

### 5. Zadania do wykonania

1. **Instalacja i konfiguracja:** Zainstaluj pakiet `djangorestframework` i dodaj go do `INSTALLED_APPS` w projekcie `eventhub`.
2. **Implementacja EventSerializer:** Zaimplementuj klasę `EventSerializer` w pliku `events/serializers.py` dziedzicząc po `serializers.Serializer`.
3. **Dokumentacja testu serializacji:** W folderze `events/` stwórz podkatalog `docs/` i w nim plik `serializer_shell_test.md`. Skopiuj do niego polecenia i rzeczywiste wyniki z powłoki `python manage.py shell` prezentujące:
   * Pomyślną serializację obiektu `Event` do formatu JSON.
   * Pomyślną deserializację z poprawnym utworzeniem nowego obiektu w bazie danych.
   * Próbę deserializacji z niepoprawnymi danymi (np. brak kategorii, niepoprawny format daty) i zrzut słownika `.errors`.
4. **Projekt zaliczeniowy:**
   * W repozytorium własnego projektu zainstaluj DRF.
   * Utwórz plik `serializers.py` i zaimplementuj jeden ręczny serializator (`serializers.Serializer`) dla wybranego modelu ze swojego projektu.
5. **Git:** Zatwierdź zmiany commitem: `feat: Add DRF and low-level EventSerializer with shell test docs`.
