# ТЗ для backend-разработчика

## 1. Общие требования и стек
- Бэкенд разрабатывается под WordPress 6.x + PHP 8.1+. Тема Kornident должна оставаться совместимой с ядром WP и Child-theme сценариями.
- Обязательное подключение Carbon Fields (autoload в `inc/carbon-fields/carbon-fields.php`) и всех существующих инклюдов из `inc/`. Новые функции выводятся через хуки `init`, `after_setup_theme`, `carbon_fields_register_fields`.
- Сборка фронта опирается на структуру каталога `assets/` (JS/CSS/изображения). Любые новые скрипты/стили регистрируются в `inc/assets.php` с поддержкой версионирования и зависимостей.
- Код, сгенерированный с помощью ИИ, проходит ручной ревью и smoke-тестирование перед выкладкой.

## 2. Контентные сущности
- Реализовать/актуализировать кастомные типы записей (CPT) по `inc/custom-post-types.php`: `service`, `doctor`, `stock`, `portfolio`.
  - Настройки: `public`, `has_archive`, `show_in_rest`, поддержка миниатюр, excerpt (для doctor/stock/portfolio), кастомные `menu_icon`.
  - Для `service` задать rewrite `/services` без префиксов, архив включен.
- Таксономия `service_category` (иерархическая, REST, admin column). Требуется возможность вложенности категорий и строгие права (`manage_categories`).
- Пользовательские мета `service_category`: SEO title, SEO description, изображения и WYSIWYG контент (см. `inc/taxonomy-fields.php`). Собственный JS для медиазагрузчика подключается в админке.
- Фильтр `post_type_link` должен формировать URL услуги как `/services/{category-path}/{service-slug}/` с поддержкой нескольких уровней категорий. Дополнительные правила rewrite обеспечивают работу URL вида `/services/{category}/` и `/services/{category}/{service}/`.
- Число записей в архивах: `stock` — 9, `portfolio` — 12 (реализовано через `pre_get_posts`). Сохранить эту логику и адаптировать при добавлении новых архивов.

## 3. Глобальные настройки и метаполя (Carbon Fields)
- Главная (`templates/home.php`): табы для баннера, секции записи (офферы), услуг/категорий (association посты/термины), преимуществ, врачей, отзывов (вставка скрипта), акций, статей, блока «О нас».
- Профиль врача (`post_type = doctor`): поля job, education для карточек.
- Страница «Цены» (`templates/prices.php`): complex `price_sections` → `services` c макетами tabbed-horizontal / tabbed-vertical, полями названия, описания, цены.
- Страница «О нас» (`page-about.php`): hero (слоган, заголовок, изображение), intro (заголовок, rich text, списки, картинка), принципы (complex cards), команда (association врачей).
- Страница «Сертификаты» (`page-certificates.php`): media_gallery.
- Портфолио (`post_type = portfolio`): complex `portfolio_sections` с rich text, radio media_mode (gallery/single), gallery или одиночные изображения, заголовки.
- Theme Options: логотип (header), блок контактов (заголовок, метро, график, email, телефоны, карта iframe, копирайт), SEO (enable, режим auto/manual, title suffix, default description, default OG image). В режиме *auto* при наличии популярных SEO плагинов встроенный вывод отключается.

## 4. Формы и AJAX
- Две формы: запись на приём (`form_type = appointment`) и обратный звонок (`form_type = callback`).
- AJAX обработчики `wp_ajax_*` и `wp_ajax_nopriv_*` описаны в `inc/ajax-forms.php`. Требования:
  - Проверка nonce (`appointment_form`, `callback_form`).
  - Валидация имени, телефона (10–12 цифр), обязательное согласие на обработку данных.
  - Сбор опциональных полей: услуга, комментарий, удобное время звонка.
  - Отправка email на `admin_email` и `contacts_email` (если указан и отличается). Письма HTML, заголовки `Content-Type`, `From`, `Reply-To`.
  - Сообщения об ошибках/успехе отдавать через `wp_send_json_*`.
- Файл `inc/form-handler.php` служит fallback-эндпоинтом: подключение WP через `wp-load.php`, принудительный JSON ответ, подавление PHP предупреждений. Валидация и логика совпадают с AJAX вариантами.
- Опциональное сохранение заявок в БД реализовать функцией `save_form_submission()` с таблицей `wp_form_submissions` (см. заготовку). Обязательно создать таблицу через `dbDelta` при первой записи.

## 5. SEO, маршрутизация и вывод
- Breadcrumbs и пагинации должны поддерживать CPT/таксономии. Проверить шаблоны `archive-*.php`, `single-*.php`, `taxonomy-service_category.php`.
- Встроенный SEO модуль: вывод `<title>`, `<meta name="description">`, Open Graph. При `seo_mode = auto` проверять активность Yoast, RankMath, AIOSEO, SEOPress и отключать дублирование мета.
- После правок перепрошивать permalinks (Settings → Permalinks → Save) — внести пункт в чек-лист деплоя.

## 6. Админ UX и безопасность
- Регистрировать меню/организовывать локализацию через `inc/menus.php`. Новые элементы меню оформлять через wp_nav_menu() хуки.
- Любые новые админские скрипты/стили подключать точечно (conditional enqueue), чтобы избежать глобальной нагрузки.
- В формах подавлять PHP ошибки на фронте, но писать `error_log` при исключениях (`form-handler.php`).
- Все пользовательские данные очищать функциями `sanitize_text_field`, `sanitize_textarea_field`, `wp_kses_post` или собственными валидаторами.

## 7. Тестирование и приёмка
- Smoke-тесты: отправка обеих форм (авторизованный/гость), проверка писем на два ящика, корректность валидаций и сообщений.
- Переключение `seo_enable`/`seo_mode` с установленным и отсутствующим SEO плагином.
- Проверка REST API для CPT (`/wp-json/wp/v2/{post_type}`) и правильности slug-генерации услуг.
- Вручную подтвердить пагинацию архивов `stock` и `portfolio`, а также вывод портфолио блоков и разделов прайса из Carbon Fields.
- Подготовить экспорт JSON настроек Carbon Fields, инструкцию для контент-менеджера (добавление блоков портфолио, разделов прайса, новых врачей/услуг).
- В чек-лист деплоя включить: flush permalinks, ревью AI-кода, smoke-тест форм, проверку переводов и fallback-шаблонов.