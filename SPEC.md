# Спецификация требований: Mandelbrot-Fractal

Восстановлена по коду коммита `7621fdd` (ветка `origin/dev`), 2026-11-18. Описывает
только реализованное поведение. Обзор проекта — [docs/README.md](docs/README.md).

## 1. Назначение и границы

Интерактивная визуализация множества Мандельброта в окне SFML с расчётом кадра
конвейером сендеров `stdexec` в пуле потоков. Происхождение — практическое
задание курса «Мидл разработчик С++», спринт 9 (`README.md:14`).

В границах: расчёт итераций и цвета, параллельный рендер по полосам, обработка
мыши и закрытия окна, вывод в окно, ограничение частоты кадров, тесты GoogleTest.

Вне границ (в коде нет): сохранение изображений, клавиатура, изменение размера
окна, настройка параметров без перекомпиляции, произвольная точность чисел.

## 2. Контекст

- **Акторы:** пользователь за окном приложения (мышь). Кто именно пользуется
  проектом, кроме проверки задания курса, в файлах не указано.
- **Внешние системы при работе:** нет. Только оконная система через SFML.
- **Внешние системы при сборке:** GitHub (CPM.cmake, NVIDIA/stdexec,
  `CMakeLists.txt:20-35`), Conan-репозиторий (`gtest/1.13.0`, `cmake/3.30.0`,
  `conanfile.py:12-13`), apt (`libsfml-dev`, `.devcontainer/Dockerfile:11`).
- **Среда выполнения:** dev-контейнер Ubuntu с GCC 15 (`.devcontainer/Dockerfile:1,8,21-22`,
  `.devcontainer/devcontainer.json:2`); нужен графический дисплей для окна SFML.

## 3. Функциональные требования

### Математика фрактала

- **FR-1.** Система должна для точки `c` итерировать `z = z² + c` с `z₀ = 0` и
  возвращать число шагов, выполненных до первого `|z|² > R²` (проверка перед
  каждым шагом), либо `max_iterations`, если выхода не было.
  Source: `include/mandelbrot_fractal_utils.hpp:32-45`. Verification:
  `MandelbrotSenderTest.CalculatesCorrectIterationsForKnownPoints` (косвенно).
- **FR-2.** Система должна переводить пиксель `(x, y)` в `x_min + x/W·width` +
  `i·(y_min + y/H·height)` текущей области. Source:
  `include/mandelbrot_fractal_utils.hpp:47-54`. Verification:
  `MandelbrotSenderTest.CalculatesCorrectIterationsForKnownPoints` (косвенно).
- **FR-3.** Система должна окрашивать точки с `iterations == max_iterations` в
  чёрный, остальные — в цвет HSV с оттенком `360·iterations/max_iterations`,
  `S = V = 1`, переведённый в RGB 0–255. Source:
  `include/mandelbrot_fractal_utils.hpp:56-104`. Verification:
  `MandelbrotRendererTest.RenderAsyncProducesCorrectSizeOutput` (только чёрный).

### Параллельный рендер

- **FR-4.** Система должна по `PixelRegion` вычислять матрицу итераций полосы, используя
  координаты полного кадра, и завершаться `set_value(PixelMatrix)` или
  `set_error(exception_ptr)`. Source: `include/mandelbrot_sender.hpp:17-37`.
  Verification: `MandelbrotSenderTest.CalculatesCorrectIterationsForKnownPoints`.
- **FR-5.** Система должна делить кадр на `N` горизонтальных полос по `height / N`
  строк, отдавая остаток последней полосе. Source:
  `include/mandelbrot_renderer.hpp:19-27`. Verification:
  `MandelbrotRendererTest.RenderAsyncProducesCorrectSizeOutput` (только размер; 16 / 4 без остатка).
- **FR-6.** Система должна запускать расчёт каждой полосы в `exec::static_thread_pool`
  (`schedule` → `let_value(MandelbrotSender)`) и переводить её итерации в цвета.
  Source: `include/mandelbrot_renderer.hpp:32-46`. Verification:
  `MandelbrotRendererTest.RenderAsyncProducesCorrectSizeOutput` (косвенно; работа в пуле не проверяется).
- **FR-7.** Система должна объединять полосы через `when_all` в матрицу
  `height × width` и возвращать `RenderResult` с областью, настройками и
  `render_time`. Source: `include/mandelbrot_renderer.hpp:48-73`. Verification:
  `MandelbrotRendererTest.RenderAsyncProducesCorrectSizeOutput`.
- **FR-8.** Система должна запускать рендер только при `need_rerender == true`,
  сбрасывая флаг, а иначе сразу возвращать пустой `RenderResult`. Source:
  `include/mandelbrot.hpp:18-33`. Verification:
  `PipelineIntegrationTest.FullRenderPipelineSkipsRenderWhenNotNeeded` (только ветка пропуска).
- **FR-9.** Система должна передавать результат рендера на планировщик из окружения
  получателя (`continues_on`). Source: `include/mandelbrot.hpp:21-30`.
  Verification: no test.

### Ввод

- **FR-10.** Система должна завершать обработчик событий `set_stopped`, если
  `should_exit` выставлен до или после обработки событий, иначе `set_value()`.
  Source: `include/sfml_events_handler.hpp:23-35`. Verification:
  `SfmlEventHandlerTest.SetsShouldExitOnWindowClose` (только предустановленный флаг).
- **FR-11.** Система должна по событию `Closed` выставлять `should_exit`, а по
  нажатию/отпусканию левой и правой кнопок мыши — соответствующие флаги. Source:
  `include/sfml_events_handler.hpp:42-61`. Verification: no test.
- **FR-12.** Система должна, пока нажата кнопка мыши и прошло не меньше 100 мс с
  прошлого шага, выполнять шаг зума, если курсор внутри кадра. Source:
  `include/sfml_events_handler.hpp:19,63-76`. Verification:
  `SfmlEventHandlerTest.SetsRerenderOnZoom`.
- **FR-13.** Система должна масштабировать область в 0.9 раза (левая кнопка) или
  в 1/0.9 (правая), сохраняя точку под курсором, и выставлять `need_rerender`.
  Source: `include/sfml_events_handler.hpp:78-93`. Verification:
  `SfmlEventHandlerTest.SetsRerenderOnZoom` (только приближение; `need_rerender` равен `true`
  уже по умолчанию, `include/types.hpp:38`, так что его проверка ничего не доказывает).

### Вывод

- **FR-14.** Система должна копировать непустой `color_data` попиксельно в `sf::Image`
  и не менять его при пустом. Source: `include/sfml_renderer.hpp:24-36`.
  Verification: `SFMLRenderTest.RendersDataWhenProvided`, `SFMLRenderTest.SkipsUpdateForEmptyData`.
- **FR-15.** Система должна в каждом кадре обновлять текстуру из `sf::Image` и
  перерисовывать окно (`clear`, `draw`, `display`). Source:
  `include/sfml_renderer.hpp:38-45`. Verification: no test.

### Приложение

- **FR-16.** Система должна открывать окно 800 × 600 «Mandelbrot Fractal» с
  отключённым повтором клавиш. Source: `src/main.cpp:47,57-63`. Verification: no test.
- **FR-17.** Система должна строить кадр в порядке: события → расчёт → вывод →
  ожидание кадра. Source: `src/main.cpp:69-75`. Verification: no test.
- **FR-18.** Система должна повторять конвейер до `should_exit` и ждать его в
  главном потоке (`sync_wait`). Source: `src/main.cpp:77-80`. Verification: no test.
- **FR-19.** Система должна досыпать до `1000 / fps` мс от начала кадра при `fps = 60`.
  Source: `src/main.cpp:29-43,75`. Verification: no test.
- **FR-20.** Система должна при исключении печатать `Error: <what>` и завершаться
  с кодом 1, иначе с кодом 0. Source: `src/main.cpp:84-93`. Verification: no test.

### Объявлено, но не реализовано

- `set_stopped_t()` объявлен в сигнатурах `MandelbrotSender`,
  `CalculateMandelbrotAsyncSender`, `SFMLRender`, но эти сендеры его не вызывают
  (`include/mandelbrot_sender.hpp:50`, `include/mandelbrot.hpp:50`,
  `include/sfml_renderer.hpp:57`).
- `RenderResult::render_time` заполняется, но не используется
  (`include/types.hpp:33`, `include/mandelbrot_renderer.hpp:67-72`).
- `SFMLRender::render_settings_` хранится, но в состояние операции не передаётся
  (`include/sfml_renderer.hpp:64,73-80`).
- `README.md:90` упоминает конфигурацию «Launch GeometryApp» — её нет в
  `.vscode/launch.json`; `CMakeLists.txt:50,76` ссылаются на `matplotplusplus`,
  который не подключается.

## 4. Интерфейсы

| Ввод пользователя | Действие | Источник |
|---|---|---|
| Удержание левой кнопки мыши | приближение 0.9 к курсору, не чаще раза в 100 мс | `include/sfml_events_handler.hpp:63-93` |
| Удержание правой кнопки | отдаление 1/0.9, не чаще раза в 100 мс | там же |
| Закрытие окна | выход, код 0 | `include/sfml_events_handler.hpp:46-47`, `src/main.cpp:92` |

| Сендер | Вход | Завершения | Источник |
|---|---|---|---|
| `SfmlEventHandler` | окно, настройки, `AppState&`, `sf::Clock&` | `set_value()`, `set_error`, `set_stopped` | `include/sfml_events_handler.hpp:98-112` |
| `CalculateMandelbrotAsyncSender` | `AppState&`, настройки, `MandelbrotRenderer&` | `set_value(RenderResult)`, `set_error` | `include/mandelbrot.hpp:42-57` |
| `MandelbrotRenderer::RenderAsync<N>` | `ViewPort`, `RenderSettings` | `RenderResult` | `include/mandelbrot_renderer.hpp:17-74` |
| `MandelbrotSender` | `ViewPort`, `RenderSettings`, `PixelRegion` | `set_value(PixelMatrix)`, `set_error` | `include/mandelbrot_sender.hpp:42-57` |
| `SFMLRender` | `RenderResult`, `sf::Image/Texture/Sprite/RenderWindow&` | `set_value()`, `set_error` | `include/sfml_renderer.hpp:54-81` |

| Сборочная цель | Тип | Источник |
|---|---|---|
| `MandelbrotFractal_imp` | статическая библиотека | `CMakeLists.txt:42` |
| `MandelbrotFractal` | приложение | `CMakeLists.txt:56-57` |
| `MandelbrotFractal_tests` | тесты GoogleTest, тест CTest | `CMakeLists.txt:65-80` |

CLI нет: `main()` без аргументов (`src/main.cpp:84`). В stdout пишется только строка
ошибки (`src/main.cpp:89`).

## 5. Данные

Все данные в памяти, файлов приложение не читает и не пишет.

| Сущность | Поля | Источник |
|---|---|---|
| `ViewPort` | `x_min=-2.5`, `x_max=1.5`, `y_min=-2.0`, `y_max=2.0` | `include/mandelbrot_fractal_utils.hpp:10-18` |
| `RenderSettings` | `width=800`, `height=600`, `max_iterations=100`, `escape_radius=2.0` | `include/types.hpp:15-20` |
| `PixelRegion` | `start_row`, `end_row`, `start_col`, `end_col` (конец не включается) | `include/types.hpp:22-27` |
| `RenderResult` | `color_data`, `viewport`, `settings`, `render_time` | `include/types.hpp:29-34` |
| `AppState` | `viewport`, `need_rerender=true`, флаги кнопок, `should_exit` | `include/types.hpp:36-42` |
| `PixelMatrix` / `ColorMatrix` | `vector<vector<uint32_t>>` / `vector<vector<RgbColor>>` | `include/types.hpp:12-13` |

## 6. Нефункциональные требования

- **NFR-1.** C++23 без расширений компилятора. Source: `CMakeLists.txt:6-8`.
- **NFR-2.** Расчёт параллелен: 8 потоков пула, 8 полос. Source: `include/types.hpp:10`,
  `src/main.cpp:59`, `include/mandelbrot.hpp:23`.
- **NFR-3.** Кадр не короче 16 мс (`1000 / 60` в целых). Source: `src/main.cpp:34,75`.
- **NFR-4.** Исключения внутри сендеров передаются как `set_error(exception_ptr)`.
  Source: `include/mandelbrot_sender.hpp:34-36`, `include/mandelbrot.hpp:34-36`,
  `include/sfml_events_handler.hpp:36-38`, `include/sfml_renderer.hpp:46-48`.
- **NFR-5.** Стиль кода: clang-format на базе LLVM, 120 колонок, отступ 4, формат при
  сохранении. Source: `.clang-format`, `.vscode/settings.json:3-4`.
- **NFR-6.** Компиляция GCC/Clang останавливается на первой ошибке. Source:
  `CMakeLists.txt:11-14`.

## 7. Ограничения и зависимости

Lock-файлов нет; версии взяты из конфигурации.

| Зависимость | Версия | Источник |
|---|---|---|
| CMake | ≥ 3.30; Conan tool `cmake/3.30.0` | `CMakeLists.txt:1`, `conanfile.py:13` |
| GoogleTest | `gtest/1.13.0` | `conanfile.py:12` |
| SFML (graphics, window, system) | не закреплена, `libsfml-dev` из apt; API 2.x (вывод: `sf::Event::type`, `Image::create`) | `CMakeLists.txt:18`, `.devcontainer/Dockerfile:11` |
| NVIDIA/stdexec | ветка `main`, не закреплена | `CMakeLists.txt:28-35` |
| CPM.cmake | `origin/master`, не закреплён | `CMakeLists.txt:21-26` |
| GCC | 15 | `.devcontainer/Dockerfile:8,21-22` |
| Conan | не закреплён, pipx | `.devcontainer/Dockerfile:59` |

## 8. Приёмка

1. В dev-контейнере собрать: `mkdir build ; cd build && conan build -b missing -s build_type=Debug ..`
   (`.vscode/tasks.json:8-10`).
2. В `build/` запустить `./MandelbrotFractal_tests` (`README.md:71-74`) на машине с
   дисплеем; ожидаются 7 пройденных тестов.
3. Запустить `./MandelbrotFractal` (`README.md:64-67`) и вручную проверить FR-11 –
   FR-16, FR-20: окно 800 × 600, зум левой и правой кнопкой к курсору, выход по
   закрытию с кодом 0.

Этот документ при написании такой проверкой не подтверждён: сборка и запуск не выполнялись.

## 9. Открытые вопросы

- Собирается ли этот коммит с текущей веткой `main` stdexec — не проверялось.
- Проявляется ли уничтожение `op` до завершения в `include/mandelbrot.hpp:29-30`
  (вывод из кода). Коммит `3156770` в `origin/main` заменяет именно эти строки
  (`connect` + `start` локального `op`) на блокирующий `sync_wait` (`git show 3156770`),
  но в сообщении назван deadlock; совпадает ли это с описанной причиной — не проверялось.
- Ответ из git: ветка по умолчанию на GitHub — `main` (`git ls-remote --symref origin HEAD`),
  `origin/dev` целиком входит в `origin/main`, где есть 18 более поздних коммитов. Эта
  спецификация описывает `origin/dev`.
- Почему пул и число полос заданы константой 8, а не `hardware_concurrency()`
  (значение по умолчанию конструктора, `include/mandelbrot_renderer.hpp:14`). Константа
  `THREAD_POOL_SIZE{8}` и `renderer_{THREAD_POOL_SIZE}` есть уже в шаблоне курса
  (`git show 8543c12:include/types.hpp`); причина там не указана.
- Целевые показатели производительности (время кадра при зуме) не заданы.
- Направление мнимой оси вниз (`include/mandelbrot_fractal_utils.hpp:52`) — намеренно
  или нет. Это код шаблона курса, после `8543c12` не менялся.
- Какая версия SFML попадает в контейнер из apt.
