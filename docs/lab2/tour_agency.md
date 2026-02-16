# Веб-приложение "Туристическое агентство"

## Описание проекта

Веб-приложение "Туристическое агентство" представляет собой полнофункциональную систему для управления турами, бронированиями и отзывами. Приложение разработано с использованием Django и предоставляет удобный интерфейс как для клиентов, так и для администраторов.

## Модели данных

### Country (Страна)
```python
class Country(models.Model):
    name = models.CharField(max_length=100, verbose_name="Название страны")
    description = models.TextField(verbose_name="Описание")
    
    def __str__(self):
        return self.name
    
    class Meta:
        verbose_name = "Страна"
        verbose_name_plural = "Страны"
```

### TourAgency (Туристическое агентство)
```python
class TourAgency(models.Model):
    name = models.CharField(max_length=100, verbose_name="Название агентства")
    address = models.CharField(max_length=200, verbose_name="Адрес")
    contact_info = models.CharField(max_length=100, verbose_name="Контактная информация")
    
    def __str__(self):
        return self.name
    
    class Meta:
        verbose_name = "Туристическое агентство"
        verbose_name_plural = "Туристические агентства"
```

### Tour (Тур)
```python
class Tour(models.Model):
    name = models.CharField(max_length=100, verbose_name="Название тура")
    description = models.TextField(verbose_name="Описание")
    country = models.ForeignKey(Country, on_delete=models.CASCADE, verbose_name="Страна")
    agency = models.ForeignKey(TourAgency, on_delete=models.CASCADE, verbose_name="Агентство")
    start_date = models.DateField(verbose_name="Дата начала")
    end_date = models.DateField(verbose_name="Дата окончания")
    price = models.DecimalField(max_digits=10, decimal_places=2, verbose_name="Цена")
    max_participants = models.PositiveIntegerField(verbose_name="Максимальное количество участников")
    
    def __str__(self):
        return self.name
    
    class Meta:
        verbose_name = "Тур"
        verbose_name_plural = "Туры"
```

### Reservation (Бронирование)
```python
class Reservation(models.Model):
    STATUS_CHOICES = [
        ('pending', 'Ожидание'),
        ('confirmed', 'Подтверждено'),
        ('cancelled', 'Отменено'),
    ]
    
    user = models.ForeignKey(User, on_delete=models.CASCADE, verbose_name="Пользователь")
    tour = models.ForeignKey(Tour, on_delete=models.CASCADE, verbose_name="Тур")
    reservation_date = models.DateTimeField(auto_now_add=True, verbose_name="Дата бронирования")
    num_people = models.PositiveIntegerField(verbose_name="Количество человек")
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending', verbose_name="Статус")
    
    def __str__(self):
        return f"{self.user.username} - {self.tour.name}"
    
    class Meta:
        verbose_name = "Бронирование"
        verbose_name_plural = "Бронирования"
```

### Review (Отзыв)
```python
class Review(models.Model):
    RATING_CHOICES = [
        (1, '1 - Ужасно'),
        (2, '2 - Плохо'),
        (3, '3 - Средне'),
        (4, '4 - Хорошо'),
        (5, '5 - Отлично'),
    ]
    
    user = models.ForeignKey(User, on_delete=models.CASCADE, verbose_name="Пользователь")
    tour = models.ForeignKey(Tour, on_delete=models.CASCADE, verbose_name="Тур")
    rating = models.IntegerField(choices=RATING_CHOICES, verbose_name="Рейтинг")
    comment = models.TextField(verbose_name="Комментарий")
    created_at = models.DateTimeField(auto_now_add=True, verbose_name="Дата создания")
    
    def __str__(self):
        return f"{self.user.username} - {self.tour.name} - {self.rating}"
    
    class Meta:
        verbose_name = "Отзыв"
        verbose_name_plural = "Отзывы"
```

## Представления (Views)

### Список туров
```python
def tour_list(request):
    tours = Tour.objects.all()
    return render(request, 'tours/tour_list.html', {'tours': tours})
```

### Детали тура
```python
def tour_detail(request, tour_id):
    tour = get_object_or_404(Tour, pk=tour_id)
    reviews = Review.objects.filter(tour=tour)
    return render(request, 'tours/tour_detail.html', {'tour': tour, 'reviews': reviews})
```

### Бронирование тура
```python
@login_required
def reserve_tour(request, tour_id):
    tour = get_object_or_404(Tour, pk=tour_id)
    
    if request.method == 'POST':
        form = ReservationForm(request.POST)
        if form.is_valid():
            reservation = form.save(commit=False)
            reservation.user = request.user
            reservation.tour = tour
            reservation.save()
            return redirect('my_reservations')
    else:
        form = ReservationForm()
    
    return render(request, 'tours/reserve_tour.html', {'form': form, 'tour': tour})
```

### Мои бронирования
```python
@login_required
def my_reservations(request):
    reservations = Reservation.objects.filter(user=request.user)
    return render(request, 'tours/my_reservations.html', {'reservations': reservations})
```

### Статистика продаж по странам
```python
@login_required
def sales_by_country(request):
    countries = Country.objects.all()
    country_data = []
    
    for country in countries:
        tours = Tour.objects.filter(country=country)
        reservations_count = Reservation.objects.filter(tour__in=tours, status='confirmed').count()
        total_sales = Reservation.objects.filter(tour__in=tours, status='confirmed').aggregate(
            total=Sum(F('tour__price') * F('num_people'))
        )['total'] or 0
        
        country_data.append({
            'country': country,
            'reservations_count': reservations_count,
            'total_sales': total_sales
        })
    
    return render(request, 'tours/sales_by_country.html', {'country_data': country_data})
```

## Формы

### Форма бронирования
```python
class ReservationForm(forms.ModelForm):
    class Meta:
        model = Reservation
        fields = ['num_people']
        widgets = {
            'num_people': forms.NumberInput(attrs={'min': 1}),
        }
```

### Форма отзыва
```python
class ReviewForm(forms.ModelForm):
    class Meta:
        model = Review
        fields = ['rating', 'comment']
        widgets = {
            'comment': forms.Textarea(attrs={'rows': 4}),
        }
```

## Шаблоны

### Базовый шаблон (base.html)
```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Туристическое агентство{% endblock %}</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.2.3/dist/css/bootstrap.min.css">
    <link rel="stylesheet" href="{% static 'css/style.css' %}">
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
        <div class="container">
            <a class="navbar-brand" href="{% url 'tour_list' %}">Туристическое агентство</a>
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav me-auto">
                    <li class="nav-item">
                        <a class="nav-link" href="{% url 'tour_list' %}">Туры</a>
                    </li>
                    {% if user.is_authenticated %}
                    <li class="nav-item">
                        <a class="nav-link" href="{% url 'my_reservations' %}">Мои бронирования</a>
                    </li>
                    {% endif %}
                </ul>
                <ul class="navbar-nav">
                    {% if user.is_authenticated %}
                    <li class="nav-item">
                        <span class="nav-link">Привет, {{ user.username }}!</span>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{% url 'logout' %}">Выйти</a>
                    </li>
                    {% else %}
                    <li class="nav-item">
                        <a class="nav-link" href="{% url 'login' %}">Войти</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{% url 'register' %}">Регистрация</a>
                    </li>
                    {% endif %}
                </ul>
            </div>
        </div>
    </nav>

    <div class="container mt-4">
        {% block content %}{% endblock %}
    </div>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.2.3/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

## URL маршруты

```python
urlpatterns = [
    path('', views.tour_list, name='tour_list'),
    path('tour/<int:tour_id>/', views.tour_detail, name='tour_detail'),
    path('reserve/<int:tour_id>/', views.reserve_tour, name='reserve_tour'),
    path('my-reservations/', views.my_reservations, name='my_reservations'),
    path('edit-reservation/<int:reservation_id>/', views.edit_reservation, name='edit_reservation'),
    path('delete-reservation/<int:reservation_id>/', views.delete_reservation, name='delete_reservation'),
    path('add-review/<int:tour_id>/', views.add_review, name='add_review'),
    path('sales-by-country/', views.sales_by_country, name='sales_by_country'),
    path('login/', auth_views.LoginView.as_view(template_name='tours/login.html'), name='login'),
    path('logout/', auth_views.LogoutView.as_view(next_page='tour_list'), name='logout'),
    path('register/', views.register, name='register'),
]
```

## Запуск проекта

### Локальный запуск:

1. Установите зависимости:
   ```bash
   pip install -r requirements.txt
   ```

2. Выполните миграции:
   ```bash
   python manage.py migrate
   ```

3. Загрузите тестовые данные:
   ```bash
   python manage.py loaddata tours/fixtures/test_data.json
   ```

4. Создайте суперпользователя:
   ```bash
   python manage.py createsuperuser
   ```

5. Запустите сервер:
   ```bash
   python manage.py runserver
   ```

### Запуск через Docker:

1. Запустите контейнеры:
   ```bash
   docker-compose up --build
   ```

2. Приложение будет доступно по адресу: http://localhost:8000