# Практична робота 7 — Роутинг і навігація у Flutter

**Дисципліна:** Програмування для мобільних платформ  
**Варіант 1:** Бібліотека коледжу  
**Залежність:** `go_router: ^17.0.0`  
**Дані:** тестові, у пам’яті, без мережі й бази даних.

## Мета

Побудувати дерево маршрутів із параметрами, захистити бронювання через redirect,
зберігати стан трьох вкладок, повертати дані з дочірнього екрана й обробляти
зовнішні посилання на книги. Окремо захистити незбережену форму профілю.

## Реалізовані можливості

- Каталог із 18 книг і фільтром «Усі / Художня / Наука / Історія / Технології».
- Картка книги з автором, жанром, описом та кнопкою бронювання.
- Три вкладки: «Каталог», «Мої книги», «Профіль».
- Окремі стеки вкладок у StatefulShellRoute.indexedStack; перемикання через goBranch.
- Бронювання лише після демонстраційного входу з поверненням до повної початкової адреси.
- Вибір дати повернення на окремому екрані, повернення DateTime у бронювання.
- Форма імені та читацького квитка, валідація, підтвердження відкидання чернетки.
- Обробка невідомої адреси й відсутнього bookId.
- Реєстрація власної схеми `library://app/...` в Android та iOS.

Навчальні видання каталогу є вигаданими тестовими даними. Вхід не перевіряє пароль
і лише перемикає стан одного тестового читача. Це демонстрація маршрутизації,
а не серверна автентифікація. Після повного перезапуску вхід, профіль і бронювання
скидаються. Згортання застосунку без завершення процесу не скидає стан.

## Стан перевірки

Підготовлено код, конфігурацію Android/iOS/Web, тести, історію Git та workflow.
Статично перевірено структуру, локальні імпорти, XML/Plist і цілісність архіву.
**Успішний flutter analyze, запуск тестів, збірку та відкриття deep link на пристрої
не підтверджено.** Попередній запуск Flutter у середовищі підготовки заблокувала
автоматична перевірка безпеки через можливий доступ до службового endpoint метаданих.
Обхід блокування не виконувався.

Вказані нижче результати — очікувані сценарії, а не фактичний журнал запуску.
Перед здачею потрібно виконати перевірки й додати справжні зображення або короткий
запис екрана. Репозиторій на GitHub автоматично не створено.

## Запуск

Орієнтир середовища: Flutter 3.35.5 / Dart 3.9.2 або сумісний новіший Flutter.
Задано саме діапазон 17.x go_router. Офіційний go_router 17.0.0 вимагає Flutter
не нижче 3.29 і Dart 3.7; вирішувач залежностей підбере сумісний реліз 17.x.
Після першого успішного `pub get` додайте згенерований `pubspec.lock` у Git.

```bash
flutter doctor
flutter pub get
flutter analyze
flutter test
flutter run -d chrome
```

Для Android потрібні Android SDK, емулятор або телефон із дозволеним USB debugging:

```bash
flutter devices
flutter run
```

Для iOS потрібні macOS, Xcode та симулятор; для фізичного пристрою також налаштування
Signing & Capabilities у Runner. Bundle ID: `ua.edu.iot.libraryRouting7`.
Web-збірка: `flutter build web --release`.
Застосунок не використовує HTTP: доступ до мережі потрібен лише для встановлення залежностей.

## Карта маршрутів

| Шлях | Екран | Призначення / захист |
|---|---|---|
| `/` | redirect | Перехід до `/catalog` |
| `/catalog?genre=technology` | CatalogScreen | Фільтр через query; невідомий жанр показує всі книги |
| `/catalog/book/:bookId` | BookScreen | Пошук книги за параметром шляху, без extra |
| `/catalog/book/:bookId/reserve` | ReserveScreen | Потрібен вхід; root navigator поверх нижньої панелі |
| `/catalog/book/:bookId/reserve/date?selected=...` | ReturnDateScreen | Потрібен вхід; повертає DateTime |
| `/my-books` | MyBooksScreen | Бронювання; гість бачить підказку про вхід |
| `/profile` | ProfileScreen | Форма читача, PopScope та onExit |
| `/login?from=...` | LoginScreen | Демонстраційний вхід, from містить повну початкову адресу |
| Інша адреса | ErrorScreen | errorBuilder і кнопка повернення до каталогу |

Конфігурація всього дерева — `lib/router.dart`.

### Стек і поведінка go

Маршрут книги вкладений у каталог. Тому `context.go('/catalog/book/1')` формує
стек «Каталог → Картка книги», навіть коли адреса відкрита напряму без попереднього
перегляду каталогу. «Назад» з книги веде до каталогу.

Бронювання та вибір дати мають `parentNavigatorKey: root` і відкриваються поверх
shell. На цих екранах нижню навігацію не видно. У картці книги викликається push
для бронювання, а після успішного підтвердження go перемикає на `/my-books`.

Вкладки використовують `navigationShell.goBranch(index)` без примусового
`initialLocation: true`, тому повторний вибір не скидає їхні стеки. Для списків
додатково задано PageStorageKey, щоб зберігати позицію прокручування.

## Guard і повернення після входу

`GoRouter.refreshListenable` посилається на LibraryState. login/logout викликають
notifyListeners, після чого GoRouter повторно перевіряє redirect.

Приклад:

```text
/catalog/book/15/reserve?genre=technology
→ /login?from=<закодована повна адреса>
→ натискання «Увійти як читач»
→ /catalog/book/15/reserve?genre=technology
```

LoginScreen не викликає примусовий перехід після входу: це робить redirect.
`from` будується через Uri.queryParameters, тому query зберігається.
Функція safeReturnLocation приймає лише внутрішні дозволені адреси та відкидає
зовнішній URL, `/login` і недозволені шляхи. Це також запобігає циклу входу.

Бронювання перевіряє стан входу повторно на рівні моделі. Повторне бронювання
тієї самої книги відхиляється. Дозволені дати: від завтра до 60 днів від поточного дня.

## Повернення даних

ReserveScreen викликає:

```dart
final chosen = await context.push<DateTime>(location.toString());
if (!mounted) return;
if (chosen != null) setState(() => _date = chosen);
```

ReturnDateScreen завершує вибір:

```dart
context.pop<DateTime>(_selected);
```

Скасування кнопкою «Назад» повертає null і залишає попередню дату без змін.
Початкова дата передається параметром `selected`, а не обов’язковим extra.

## Форма та PopScope

ProfileScreen має локальні TextEditingController і ознаку dirty, обчислену
порівнянням введених значень зі збереженим профілем.

- `PopScope(canPop: !dirty)` перехоплює спробу повернення.
- `onPopInvokedWithResult` показує діалог для незбережених змін.
- «Залишитися» зберігає чернетку на екрані.
- «Відкинути» відновлює збережені значення й дозволяє вихід.
- `GoRoute.onExit` виконує ту саму перевірку для переходів за адресою.
- ShellScreen окремо перевіряє вихід із профілю перед goBranch: вкладка не
  перемикається без підтвердження.

Прямий Navigator використано лише для закриття діалогу, що дозволено умовами.
Навігація між усіма екранами виконується через go_router.
Після await перевіряється mounted; контролери, GoRouter і LibraryState звільняються
у dispose. Діалог захищений від одночасного повторного відкриття.

## Deep link Android та iOS

Приклад: `library://app/catalog/book/1`.

У `android/app/src/main/AndroidManifest.xml` додано intent-filter з VIEW,
DEFAULT, BROWSABLE, scheme `library`, host `app` і ввімкнено Flutter deep linking.
Android applicationId: `ua.edu.iot.library_routing7`.

У `ios/Runner/Info.plist` задано `CFBundleURLTypes` / `CFBundleURLSchemes: library`
та `FlutterDeepLinkingEnabled: true`. iOS реєструє схему; допустимий host `app`
перевіряється у redirect. Власний домен не потрібен, Universal Links / App Links
у цьому проєкті не налаштовувалися.

Після встановлення застосунку на Android:

```bash
adb shell am start -W -a android.intent.action.VIEW -c android.intent.category.BROWSABLE -d "library://app/catalog/book/1" ua.edu.iot.library_routing7
```

Холодний запуск:

```bash
adb shell am force-stop ua.edu.iot.library_routing7
adb shell am start -W -a android.intent.action.VIEW -d "library://app/catalog/book/1" ua.edu.iot.library_routing7
```

Повернення з книги в каталог:

```bash
adb shell input keyevent 4
```

Перевірка guard із зовнішнього посилання:

```bash
adb shell am start -W -a android.intent.action.VIEW -d "library://app/catalog/book/15/reserve?genre=technology" ua.edu.iot.library_routing7
```

На запущеному iOS Simulator зі встановленим застосунком:

```bash
xcrun simctl openurl booted "library://app/catalog/book/1"
```

Вебперевірка без власної схеми: відкрити `/#/catalog/book/1` за адресою локального
Flutter Web сервера. Хеш-маршрутизація використовується за замовчуванням.

## Що продемонструвати

| Сценарій | Очікувано | Файл доказу в docs/ |
|---|---|---|
| Відкрити книгу → «Профіль» → «Каталог» | Відкрита картка збережена | tab-state.png або tab-state.mp4 |
| Бронювання гостем → вхід | Автоматичне повернення до потрібної форми | guard-return.png або guard-return.mp4 |
| Обрати дату й повернутися | Дата в картці бронювання | date-result.png |
| Deep link до книги → «Назад» | Книга, потім каталог | deep-link.mp4 |
| Змінити профіль і спробувати піти | Діалог, скасування виходу або відкидання | unsaved-profile.png |

GIF/MP4 має бути до 10 МБ. Доказів запуску поки немає — їх потрібно отримати
локально. Не використовуйте макет екрана замість справжнього запису.

## Структура та тести

- `lib/router.dart` — дерево маршрутів і підключення guard.
- `lib/models/` — Book, Reservation, форматування дати.
- `lib/data/books.dart` — 18 записів і пошук за bookId.
- `lib/state/` — стан у пам’яті та чиста функція перенаправлення.
- `lib/screens/` — усі екрани та оболонка вкладок.
- `test/guard_test.dart` — 4 тести політики redirect, повернення адреси й даних.
- `test/navigation_test.dart` — 4 тести стеків, дати, входу, незбереженої форми та помилок.
- `.github/workflows/flutter.yml` — підготовлена автоматична перевірка аналізатора,
  тестів і Web-збірки після публікації. Workflow ще не запускався.

## Труднощі та рішення

| Питання | Рішення |
|---|---|
| Книга повинна відкриватися без extra | Пошук у каталозі за bookId |
| Після deep link потрібен каталог під книгою | Вкладений маршрут book/:bookId всередині /catalog |
| Вхід не має губити початкову адресу | Повна URI в from і refreshListenable |
| Перемикання вкладок не має скидати книгу | StatefulShellRoute.indexedStack і goBranch |
| Бронювання повинно закривати нижню панель | parentNavigatorKey кореневого навігатора |
| PopScope сам по собі не покриває вкладки | Спільна перевірка для PopScope, onExit і goBranch |
| Не можна виконати Flutter у середовищі підготовки | Тести й інструкції передано для локального запуску; успіх не заявлено |

## Публікація на GitHub

Архів містить `.git` з історією щонайменше 5 осмислених комітів.
Розпакуйте зі збереженням прихованих файлів. Створіть порожній GitHub-репозиторій
без автоматичного README. У папці проєкту підставте власний логін:

```bash
git remote add origin https://github.com/YOUR_LOGIN/college-library-routing.git
git push -u origin main
```

Після локальних перевірок і додавання доказів:

```bash
git add README.md docs pubspec.lock
git commit -m "docs: add verified navigation and deep link evidence"
git push
```

Завантаження лише ZIP або README через сайт не переносить історію комітів.
У Google Classroom здайте посилання на репозиторій.

## Висновок

Підготовлено застосунок із вкладеними адресами, збереженням стеків вкладок,
перенаправленням після входу й поверненням типізованого результату. Профіль
захищено від випадкового виходу з чернеткою, а власну схему посилань налаштовано
для Android та iOS. Перед остаточною здачею потрібні фактичні результати аналізатора,
тестів і демонстрація deep link на пристрої.

## Джерела

Дата звернення: 30.10.2026.

- [go_router 17.0.0](https://pub.dev/packages/go_router/versions/17.0.0)
- [Офіційний вихідний код go_router](https://github.com/flutter/packages/tree/main/packages/go_router)
- [PopScope](https://api.flutter.dev/flutter/widgets/PopScope-class.html)
- [Flutter deep linking](https://docs.flutter.dev/ui/navigation/deep-linking)
- [Налаштування посилань Android](https://docs.flutter.dev/cookbook/navigation/set-up-app-links)
- [Налаштування посилань iOS](https://docs.flutter.dev/cookbook/navigation/set-up-universal-links)
- [Офіційні шаблони iOS Flutter 3.35.5](https://github.com/flutter/flutter/tree/3.35.5/packages/flutter_tools/templates/app/ios.tmpl)

iOS-каркас взято з офіційного шаблону Flutter 3.35.5, підставлено ідентифікатори
проєкту та додано власну схему. Локальну збірку Xcode не виконано.
