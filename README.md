# МЕТОДИЧЕСКИЕ УКАЗАНИЯ ДЛЯ СТУДЕНТОВ
**Тема:** Реализация базового CRUD для главной сущности проекта (MVC)
**Среда разработки:** VS Code, FileZilla, хостинг Beget (phpMyAdmin)

---

## 📌 Введение: Что мы делаем сегодня?
В каждом из ваших проектов есть **Главная сущность**. 
* Если у вас интернет-магазин — это `Товар`.
* Если таск-трекер — это `Задача`.
* Если система бронирования — это `Номер` или `Площадка`.

**Ваша цель на сегодня:** 
1. Создать таблицу для этой сущности в БД.
2. Написать **Модель** (для работы с БД).
3. Написать **Контроллер** (для управления логикой).
4. Написать **Представления (Views)** (для вывода данных на экран).

---

## ШАГ 1. Проектирование БД (phpMyAdmin)
Зайдите в панель Beget -> MySQL -> phpMyAdmin.
Создайте таблицу для вашей главной сущности. **Обязательное условие:** таблица должна быть связана с таблицей `users` (кто создал задачу, кто владелец заказа и т.д.).

*Пример SQL-запроса (адаптируйте под себя!):*
```sql
CREATE TABLE `items` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `user_id` INT(11) NOT NULL COMMENT 'Связь с пользователем',
  `title` VARCHAR(255) NOT NULL,
  `description` TEXT,
  `price` DECIMAL(10,2) DEFAULT '0.00',
  `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  -- Внешний ключ: если удалить юзера, удалятся и его записи (CASCADE)
  CONSTRAINT `fk_items_user` FOREIGN KEY (`user_id`) REFERENCES `users`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
*Наполните таблицу 3-4 тестовыми записями вручную через интерфейс phpMyAdmin, чтобы было что выводить на экран.*

---

## ШАГ 2. Создание Модели (Model)
Модель отвечает за все SQL-запросы. Контроллер не должен содержать SQL-кода!
Создайте файл `app/Models/Item.php` (назовите файл именем вашей сущности, например `Product.php` или `Task.php`).

```php
<?php
// app/Models/Item.php

class Item {
    private $pdo;

    public function __construct() {
        global $pdo; // Подключаем PDO из конфигурации
        $this->pdo = $pdo;
    }

    // Получить все записи
    public function getAll() {
        $stmt = $this->pdo->query("SELECT * FROM items ORDER BY created_at DESC");
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    // Получить одну запись по ID
    public function getById($id) {
        $stmt = $this->pdo->prepare("SELECT * FROM items WHERE id = ?");
        $stmt->execute([$id]);
        return $stmt->fetch(PDO::FETCH_ASSOC);
    }
}
```

---

## ШАГ 3. Создание Контроллера (Controller)
Создайте файл `app/Controllers/ItemController.php`.

```php
<?php
// app/Controllers/ItemController.php
require_once '../app/Models/Item.php';

class ItemController {
    private $model;

    public function __construct() {
        $this->model = new Item();
    }

    // Вывод списка (Каталог)
    public function index() {
        $items = $this->model->getAll();
        require_once '../views/items/index.php';
    }

    // Детальная страница
    public function show() {
        $id = $_GET['id'] ?? null;
        if (!$id) {
            die("Ошибка: ID не передан!");
        }
        
        $item = $this->model->getById($id);
        if (!$item) {
            die("Запись не найдена!");
        }
        
        require_once '../views/items/show.php';
    }
}
```
⚡ **Не забудьте!** Откройте файл `public_html/index.php` (ваш Роутер) и добавьте новые маршруты:
```php
$router->add('/catalog', 'ItemController', 'index');
$router->add('/item', 'ItemController', 'show');
```

---

## ШАГ 4. Создание Представлений (Views)
Создайте папку `views/items/` и в ней два файла.

**1. Файл `views/items/index.php` (Список):**
```php
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Каталог</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
<div class="container mt-5">
    <h2>Наш каталог</h2>
    <div class="row">
        <?php foreach ($items as $item): ?>
            <div class="col-md-4 mb-3">
                <div class="card shadow-sm">
                    <div class="card-body">
                        <!-- htmlspecialchars защищает от XSS атак! -->
                        <h5 class="card-title"><?= htmlspecialchars($item['title']) ?></h5>
                        <p class="card-text">Цена: <?= $item['price'] ?> руб.</p>
                        <a href="/item?id=<?= $item['id'] ?>" class="btn btn-primary">Подробнее</a>
                    </div>
                </div>
            </div>
        <?php endforeach; ?>
    </div>
</div>
</body>
</html>
```

**2. Файл `views/items/show.php` (Детальная страница):**
Создайте самостоятельно по аналогии, выводя полное описание (`$item['description']`).

---
---

# ПРОДВИНУТЫЙ УРОВЕНЬ (Уровень "А")
*Обязательно для студентов, претендующих на оценку "Отлично" и реализующих сложные проекты (интернет-магазины, маркетплейсы, сложные CRM).*

Простой вывод `SELECT *` не подходит для реальных проектов. Вам необходимо реализовать **Поиск, Безопасную сортировку и Пагинацию (постраничную навигацию)**.

### Инструкция А1: Умная Модель (Динамический SQL)
PDO не позволяет биндить (через `?`) имена столбцов для `ORDER BY`. Если передать `$_GET['sort']` напрямую в SQL — это приведет к SQL-инъекции. 
Замените метод `getAll()` в вашей Модели на следующий продвинутый метод `getFiltered()`:

```php
// app/Models/Item.php

public function getFiltered($search = '', $sortBy = 'created_at', $sortDir = 'DESC', $limit = 10, $offset = 0) {
    // 1. БЕЗОПАСНОСТЬ: Белый список разрешенных полей для сортировки
    $allowedSortColumns = ['price', 'created_at', 'title'];
    $allowedSortDirs = ['ASC', 'DESC'];

    // Если хакер попытается передать левое поле, сбрасываем на дефолт
    if (!in_array($sortBy, $allowedSortColumns)) {
        $sortBy = 'created_at';
    }
    if (!in_array(strtoupper($sortDir), $allowedSortDirs)) {
        $sortDir = 'DESC';
    }

    // 2. Формирование запроса
    $sql = "SELECT * FROM items";
    $params = [];

    // Если есть поиск
    if (!empty($search)) {
        $sql .= " WHERE title LIKE ?";
        $params[] = "%$search%"; // Ищем подстроку
    }

    // Добавляем сортировку и лимиты (здесь переменные безопасны благодаря проверке выше)
    $sql .= " ORDER BY $sortBy $sortDir LIMIT ? OFFSET ?";
    
    // 3. Подготовка и выполнение
    $stmt = $this->pdo->prepare($sql);
    
    // PDO по умолчанию биндит всё как строки. Для LIMIT нужны числа (INT).
    // Поэтому биндим параметры вручную:
    $paramIndex = 1;
    if (!empty($search)) {
        $stmt->bindValue($paramIndex++, "%$search%", PDO::PARAM_STR);
    }
    $stmt->bindValue($paramIndex++, (int)$limit, PDO::PARAM_INT);
    $stmt->bindValue($paramIndex, (int)$offset, PDO::PARAM_INT);
    
    $stmt->execute();
    return $stmt->fetchAll(PDO::FETCH_ASSOC);
}

// Метод для подсчета общего количества записей (нужно для пагинации)
public function getTotalCount($search = '') {
    $sql = "SELECT COUNT(*) FROM items";
    $params = [];
    if (!empty($search)) {
        $sql .= " WHERE title LIKE ?";
        $params[] = "%$search%";
    }
    $stmt = $this->pdo->prepare($sql);
    $stmt->execute($params);
    return $stmt->fetchColumn();
}
```

### Инструкция А2: Обновление Контроллера
Теперь Контроллер должен принимать параметры из URL (например: `/catalog?search=телефон&sort=price&dir=ASC&page=2`) и передавать их в Модель.

```php
// app/Controllers/ItemController.php -> метод index()

public function index() {
    // Сбор GET-параметров с дефолтными значениями
    $search = trim($_GET['search'] ?? '');
    $sortBy = $_GET['sort'] ?? 'created_at';
    $sortDir = $_GET['dir'] ?? 'DESC';
    
    // Пагинация
    $page = isset($_GET['page']) ? (int)$_GET['page'] : 1;
    if ($page < 1) $page = 1;
    $limit = 6; // Товаров на страницу
    $offset = ($page - 1) * $limit;

    // Получаем данные из умной модели
    $items = $this->model->getFiltered($search, $sortBy, $sortDir, $limit, $offset);
    
    // Считаем страницы
    $totalItems = $this->model->getTotalCount($search);
    $totalPages = ceil($totalItems / $limit);

    // Передаем ВСЕ переменные во View
    require_once '../views/items/index.php';
}
```

### Инструкция А3: Интеграция во View (HTML)
В файле `views/items/index.php` добавьте форму поиска и сортировки **ПЕРЕД** выводом карточек:

```html
<!-- Форма фильтрации -->
<form method="GET" action="/catalog" class="row g-3 mb-4">
    <div class="col-md-4">
        <input type="text" name="search" class="form-control" placeholder="Поиск..." value="<?= htmlspecialchars($search) ?>">
    </div>
    <div class="col-md-3">
        <select name="sort" class="form-select">
            <option value="created_at" <?= $sortBy == 'created_at' ? 'selected' : '' ?>>Новинки</option>
            <option value="price" <?= $sortBy == 'price' ? 'selected' : '' ?>>По цене</option>
            <option value="title" <?= $sortBy == 'title' ? 'selected' : '' ?>>По алфавиту</option>
        </select>
    </div>
    <div class="col-md-3">
        <select name="dir" class="form-select">
            <option value="ASC" <?= $sortDir == 'ASC' ? 'selected' : '' ?>>По возрастанию</option>
            <option value="DESC" <?= $sortDir == 'DESC' ? 'selected' : '' ?>>По убыванию</option>
        </select>
    </div>
    <div class="col-md-2">
        <button type="submit" class="btn btn-primary w-100">Применить</button>
    </div>
</form>
```

И добавьте **Пагинацию** ПОСЛЕ вывода карточек:
```html
<!-- Пагинация Bootstrap -->
<?php if ($totalPages > 1): ?>
<nav class="mt-4">
    <ul class="pagination justify-content-center">
        <?php for ($i = 1; $i <= $totalPages; $i++): ?>
            <!-- Сохраняем текущие параметры поиска при переключении страниц -->
            <li class="page-item <?= ($i == $page) ? 'active' : '' ?>">
                <a class="page-link" href="/catalog?search=<?= urlencode($search) ?>&sort=<?= $sortBy ?>&dir=<?= $sortDir ?>&page=<?= $i ?>">
                    <?= $i ?>
                </a>
            </li>
        <?php endfor; ?>
    </ul>
</nav>
<?php endif; ?>
```

**Критерий успешного выполнения Уровня "А":**
Вы можете искать записи, сортировать их по цене/дате, и при переходе на 2-ю страницу параметры поиска не сбрасываются. SQL-инъекции через параметр `sort` невозможны.
