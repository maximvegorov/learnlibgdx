# Глава 3 Отрисовка 2D-графики

## Оглавление главы

- [Текстуры и SpriteBatch](#текстуры-и-spritebatch)
- [Координаты и камера](#координаты-и-камера)
- [Viewport и изменение размера окна](#viewport-и-изменение-размера-окна)
- [Отрисовка 2D-фигур](#отрисовка-2d-фигур)
- [Полный игровой экран](#полный-игровой-экран)
  - [Проверка и самостоятельная работа](#проверка-и-самостоятельная-работа)

Во второй главе мы разделили приложение на меню и игровой экран. Теперь нарисуем на игровом экране изображения и геометрические фигуры и научимся сохранять их пропорции при изменении размеров окна.

Продолжим работу с проектом `chapter-01/first-game`. В конце главы заменим `PlayScreen.java` в пакете `ru.learnlibgdx.firstgame`. Классы `FirstGame` и `MenuScreen` возьмём из основного примера второй главы. Enter по-прежнему открывает игровой экран, пробел переключает паузу, Escape возвращает в меню.

## Текстуры и SpriteBatch

### От файла изображения к текстуре

**Текстура** — изображение, загруженное в память графического процессора для использования при рисовании. В LibGDX её представляет класс `Texture`. Сам объект текстуры не содержит координат игрового персонажа: одно изображение можно нарисовать в нескольких местах.

Чтобы загрузить PNG, положите его, например, в `assets/images/marker.png` и используйте:

```java
Texture markerTexture = new Texture(Gdx.files.internal("images/marker.png"));
```

Для этого нужны импорты `com.badlogic.gdx.Gdx` и `com.badlogic.gdx.graphics.Texture`. При стандартном запуске проекта через Gradle внутренний путь задаётся относительно каталога ресурсов: префикс `assets/` в строке не нужен. Сохраняйте регистр букв в имени файла и пути. Работа с внутренними файлами описана в [руководстве по файловому API](https://libgdx.com/wiki/file-handling).

Загружайте текстуру один раз при подготовке экрана. Создание `new Texture(...)` в каждом `render()` повторяло бы загрузку изображения и выделение ресурсов.

### Как SpriteBatch рисует изображение

`SpriteBatch` собирает геометрию прямоугольников с текстурами и отправляет её на отрисовку партиями. Благодаря этому несколько вызовов `draw()` могут объединиться в одну отправку графическому процессору. Смена текстуры или заполнение внутреннего буфера приводит к отправке накопленных данных. Устройство этого механизма описано в [руководстве по SpriteBatch](https://libgdx.com/wiki/graphics/2d/spritebatch-textureregions-and-sprites).

Создайте `SpriteBatch` один раз, а в каждом кадре выполняйте:

```java
batch.begin();
batch.draw(markerTexture, 100f, 80f, 64f, 64f);
batch.draw(markerTexture, 200f, 80f, 64f, 64f);
batch.end();
```

Это фрагмент отрисовки, а не полный класс. Переменная `batch` имеет тип `com.badlogic.gdx.graphics.g2d.SpriteBatch`. В примере одна текстура используется дважды. Аргументы после неё задают `x`, `y`, ширину и высоту прямоугольника.

Все вызовы `batch.draw()` должны находиться между `begin()` и `end()`. Для нескольких изображений обычно достаточно одной такой пары. Очищайте кадр до `begin()`, иначе можно стереть уже нарисованную часть сцены.

В обычной 2D-сцене порядок вызовов определяет наложение: сначала рисуем фон, затем объекты, затем элементы поверх них. Если позже добавленный непрозрачный прямоугольник перекрывает предыдущий, предыдущего не будет видно в области пересечения.

### Размер изображения, цвет и фильтрация

У вызова `batch.draw(texture, x, y)` ширина и высота берутся из самой текстуры. Перегрузка с явными размерами позволяет отделить разрешение изображения от размера объекта в игровом мире. Например, PNG размером 128 × 128 пикселей можно нарисовать в прямоугольнике 64 × 64 мировые единицы. Эти перегрузки можно проверить в [исходном коде SpriteBatch](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/g2d/SpriteBatch.java).

Цвет `batch` умножается на цвет текстуры. Белый цвет сохраняет исходное изображение, а другой позволяет его тонировать:

```java
batch.setColor(0.5f, 0.5f, 0.5f, 1f);
batch.draw(markerTexture, 100f, 80f, 64f, 64f);
batch.setColor(1f, 1f, 1f, 1f);
```

Вставляйте этот фрагмент между `begin()` и `end()`. Сброс цвета нужен, чтобы последующие изображения не унаследовали затемнение. Альфа-канал текстуры поддерживает прозрачность; стандартный `SpriteBatch` использует смешивание цветов для её отображения.

При увеличении или уменьшении изображения результат зависит от фильтрации. `Nearest` выбирает ближайший элемент текстуры, а `Linear` интерполирует соседние значения. Для тестовой картинки с чёткими границами явно зададим:

```java
markerTexture.setFilter(Texture.TextureFilter.Nearest,
                        Texture.TextureFilter.Nearest);
```

Первый параметр задаёт фильтр уменьшения, второй — увеличения. Это параметры [GLTexture](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/GLTexture.java), от которого наследуется `Texture`. `Nearest` не гарантирует одинаковую толщину всех увеличенных пикселей при дробном масштабе: для этого дополнительно нужна согласованная настройка масштаба и координат.

### Тестовая текстура без внешнего файла

Чтобы полный пример можно было запустить сразу, создадим картинку 64 × 64 через `Pixmap`: жёлтый квадрат с голубой серединой. `Pixmap` хранит пиксели в памяти и позволяет рисовать простые фигуры. Затем `new Texture(pixmap)` загружает результат на GPU.

После загрузки временный `Pixmap` нужно освободить через `dispose()`. Текстура продолжит существовать отдельно. Созданная таким способом текстура не восстанавливается автоматически после потери графического контекста; здесь используем её для настольного упражнения. Особенности этого способа описаны в [руководстве по Pixmap](https://libgdx.com/wiki/graphics/2d/pixmaps).

Самим `Texture` и `SpriteBatch` тоже нужен `dispose()` в конце использования. Освобождение `SpriteBatch` не освобождает переданные ему текстуры: их жизненным циклом управляет владелец. В нашем примере этим владельцем будет `PlayScreen`.

## Координаты и камера

### Единицы игрового мира

Выберем игровую область **800 × 450 мировых единиц**. Это размеры видимой сцены, а не обязательные размеры окна и не ограничение для всех будущих уровней.

В примере начало координат находится в левом нижнем углу области, ось X направлена вправо, Y — вверх:

```text
(0, 450)                         (800, 450)
    +--------------------------------+
    |                                |
    |             (400, 225)         |
    |                                |
    +--------------------------------+ → X
 (0, 0)                          (800, 0)
    ↑ Y
```

При таком выборе `batch.draw(texture, x, y, width, height)` размещает левый нижний угол прямоугольника в точке `(x, y)`. Чтобы центр квадрата 64 × 64 оказался в центре сцены, его левый нижний угол должен быть:

```java
float x = (800f - 64f) / 2f;  // 368
float y = (450f - 64f) / 2f;  // 193
```

Размеры в мировых единицах позволяют описывать правила независимо от окна. Квадрат шириной 64 единицы остаётся таким для игровой логики и в маленьком, и в большом окне; меняется только его отображение. Различие мировых, экранных и текстурных координат разбирается в [документации координатных систем](https://libgdx.com/wiki/articles/coordinate-systems).

### Ортографическая камера

Для нашей 2D-сцены используем `OrthographicCamera`. Ортографическая проекция не уменьшает объекты с расстоянием так, как перспективная. Камера определяет, какую область мира мы видим, и подготавливает матрицы преобразования координат.

Следующий фрагмент показывает настройку камеры без `Viewport`; в полном примере её размерами будет управлять `Viewport`:

```java
OrthographicCamera camera = new OrthographicCamera();
camera.setToOrtho(false, 800f, 450f);
camera.update();
batch.setProjectionMatrix(camera.combined);
```

Нужен импорт `com.badlogic.gdx.graphics.OrthographicCamera`. Аргумент `false` выбирает направление Y вверх. `setToOrtho()` также помещает камеру в центр указанной области. `camera.combined` объединяет преобразования вида и проекции; передаём её в `SpriteBatch` до `begin()`.

Позиция камеры соответствует центру видимой области. При размерах 800 × 450 и `zoom = 1` положение `(400, 225)` показывает область от `(0, 0)` до `(800, 450)`. После изменения позиции или масштаба нужно обновить матрицы камеры. Свойства и примеры использования приведены в [руководстве по OrthographicCamera](https://libgdx.com/wiki/graphics/2d/orthographic-camera).

Если переместить камеру вправо, неподвижные объекты на изображении сдвинутся влево. Их мировые координаты при этом не меняются. Если установить `camera.zoom = 2f`, при неизменных размерах области камеры будет видно вдвое больше мира по каждой оси, а объекты станут визуально меньше.

### Координаты мыши

У ввода другая система: `Gdx.input.getX()` и `getY()` дают координаты от левого верхнего угла окна, Y растёт вниз. Эти значения нельзя напрямую сравнивать с координатами игровых объектов.

После настройки `Viewport` преобразование можно выполнить так:

```java
// Поле экрана; нужен импорт com.badlogic.gdx.math.Vector2.
private final Vector2 pointer = new Vector2();

// В обработчике ввода, после первоначальной настройки viewport:
pointer.set(Gdx.input.getX(), Gdx.input.getY());
viewport.unproject(pointer);
```

Это два фрагмента для разных мест класса, а не один самостоятельный блок. После `unproject()` вектор содержит мировые координаты. Метод учитывает камеру, масштаб и положение области отрисовки в окне. При `FitViewport` щелчок по полям за пределами игровой области может преобразоваться в точку вне видимого мира: `unproject()` сам не отбрасывает такие события. Контракт преобразования описан в [исходном коде Viewport](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/utils/viewport/Viewport.java).

## Viewport и изменение размера окна

### Выбираем стратегию масштабирования

Камера задаёт вид на мир, а `Viewport` связывает его с прямоугольной областью окна. При изменении размеров окна он рассчитывает экранные границы и обновляет размеры области камеры.

| Класс | Поведение при изменении пропорций окна |
| --- | --- |
| `FitViewport` | Сохраняет пропорции и показывает всю заданную область; могут остаться поля |
| `FillViewport` | Сохраняет пропорции и заполняет окно; часть изображения может оказаться за его границами |
| `StretchViewport` | Заполняет окно, растягивая заданную область; пропорции объектов могут исказиться |
| `ExtendViewport` | Сохраняет пропорции, расширяя видимую область мира по одной оси; базовые размеры задают минимум |
| `ScreenViewport` | При стандартном масштабе одна мировая единица соответствует одной экранной единице; с размером окна меняется видимая область |

Эти стратегии описаны в [руководстве по Viewport](https://libgdx.com/wiki/graphics/viewports). Для нашего примера выберем `FitViewport`: игрок должен видеть всю область 800 × 450, а квадраты должны оставаться квадратами.

Для такой области масштаб равен меньшему из двух отношений:

```text
scale = min(ширина окна / 800, высота окна / 450)
```

Например, получаем следующие размеры области отрисовки:

| Окно | Масштаб | Игровая область в окне | Поля |
| --- | --- | --- | --- |
| 800 × 450 | 1 | 800 × 450 | Нет |
| 800 × 600 | 1 | 800 × 450 | По 75 сверху и снизу |
| 1200 × 450 | 1 | 800 × 450 | По 200 слева и справа |
| 1600 × 900 | 2 | 1600 × 900 | Нет |

Это расчёты для указанных размеров клиентской области, без рамки и заголовка окна. В общем случае итоговые экранные размеры округляются до целых значений.

### Обновление и применение Viewport

Создание и обработка размеров выглядят так:

```java
camera = new OrthographicCamera();
viewport = new FitViewport(800f, 450f, camera);

// В методе resize(int width, int height):
viewport.update(width, height, true);
```

Аргумент `true` центрирует камеру. Для неподвижной сцены это удобно. Если камера следит за персонажем, повторное центрирование при каждом изменении окна будет сбрасывать её позицию: в таком случае используйте `false` и задавайте положение камеры самостоятельно.

Перед рисованием вызываем `viewport.apply()`, затем передаём `camera.combined` в `batch`. `apply()` устанавливает область OpenGL и обновляет камеру; отдельный `camera.update()` сразу после него не нужен. При этом `apply()` не пересчитывает размеры под новое окно: этим занимается `update()`. Различие видно в [реализации Viewport](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/utils/viewport/Viewport.java).

На HiDPI-мониторах логические размеры окна могут отличаться от физических размеров буфера. Используйте размеры, пришедшие в `resize()`, и штатный `Viewport`, не домножая их вручную на масштаб монитора. Особенности LWJGL3 описаны в [руководстве по настройке графики](https://libgdx.com/wiki/graphics/querying-and-configuring-graphics).

## Отрисовка 2D-фигур

### ShapeRenderer: контуры и заливка

Для простых фигур без текстур используем `ShapeRenderer` из пакета `com.badlogic.gdx.graphics.glutils`. Он пригодится для прототипа уровня, полосы здоровья или отладочных границ объектов. В отличие от рисования в `Pixmap`, результат сразу попадает в кадр: создавать промежуточную текстуру не требуется. Примеры использования есть в [руководстве по отрисовке фигур](https://libgdx.com/wiki/graphics/opengl-utils/rendering-shapes).

Как и `SpriteBatch`, объект создаём один раз в конструкторе экрана, используем в каждом кадре и освобождаем в `dispose()`:

```java
// Поле PlayScreen:
private final ShapeRenderer shapes;

// В конструкторе:
shapes = new ShapeRenderer();

// В dispose():
shapes.dispose();
```

Это фрагменты для разных мест класса. Полный вариант приведён ниже. Для контура выбираем `ShapeRenderer.ShapeType.Line`, для заливки — `ShapeRenderer.ShapeType.Filled`. Цвет задаём через `setColor(r, g, b, a)`; здесь используем непрозрачные цвета с `a = 1f`.

После применения viewport передаём фигурам ту же матрицу камеры, что и текстурам:

```java
viewport.apply();
shapes.setProjectionMatrix(camera.combined);

shapes.begin(ShapeRenderer.ShapeType.Filled);
shapes.setColor(0.2f, 0.8f, 0.45f, 1f);
shapes.rect(100f, 80f, 120f, 60f);
shapes.end();

shapes.begin(ShapeRenderer.ShapeType.Line);
shapes.setColor(1f, 1f, 1f, 1f);
shapes.rect(100f, 80f, 120f, 60f);
shapes.end();
```

Получится зелёный прямоугольник с белым контуром. При смене режима завершаем текущую группу через `end()`, затем начинаем следующую. Несколько фигур одного режима можно рисовать внутри одной пары `begin()` / `end()`.

### Линии, прямоугольники, круги и треугольники

| Вызов | Значение аргументов |
| --- | --- |
| `line(x1, y1, x2, y2)` | Координаты двух концов отрезка; в примере используем режим `Line` |
| `rect(x, y, width, height)` | Левый нижний угол, ширина и высота прямоугольника |
| `circle(x, y, radius, segments)` | Центр, радиус и число сегментов окружности |
| `ellipse(x, y, width, height, segments)` | Левый нижний угол и размеры ограничивающего прямоугольника, число сегментов |
| `triangle(x1, y1, x2, y2, x3, y3)` | Координаты трёх вершин треугольника |
| `rectLine(x1, y1, x2, y2, width)` | Отрезок в виде прямоугольника заданной толщины; для сплошной полосы выбираем `Filled` |

Прямоугольник, круг, эллипс и треугольник поддерживают и контур, и заливку. У `circle()` координаты задают **центр**, а у `rect()` и `ellipse()` — **угол**. Толщина `rectLine()` измеряется в мировых единицах, поэтому масштабируется вместе со сценой.

Круг и эллипс приближаются сегментами. В полном примере используем 48 сегментов; попробуйте заменить их на 8 и сравнить границы. Большее значение делает границу менее угловатой, но увеличивает количество геометрии. Перегрузки без `segments` подбирают число сегментов автоматически. Сигнатуры методов и поддерживаемые режимы приведены в [исходном коде ShapeRenderer](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/glutils/ShapeRenderer.java).

### Совместная отрисовка с SpriteBatch

Перед `shapes.begin()` обязательно завершайте отрисовку текстур через `batch.end()`. Перед возвращением к текстурам вызывайте `shapes.end()`. Оба объекта используют собственные буферы геометрии; частое чередование увеличивает число отправок данных на GPU. Это ограничение описано в [руководстве по отрисовке фигур](https://libgdx.com/wiki/graphics/opengl-utils/rendering-shapes).

В нашем экране порядок будет таким: фон и маркеры через `batch`, затем залитые фигуры, затем линии и контуры через `shapes`. Поэтому рамка вокруг движущегося маркера окажется поверх его изображения. Общая `camera.combined` позволит рамке совпадать с маркером при изменении окна.

## Полный игровой экран

Замените содержимое `core/src/main/java/ru/learnlibgdx/firstgame/PlayScreen.java` следующим кодом. Если генератор использовал каталог `src`, сохраните файл в соответствующем пакете этого каталога.

```java
package ru.learnlibgdx.firstgame;

import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.Input;
import com.badlogic.gdx.ScreenAdapter;
import com.badlogic.gdx.graphics.OrthographicCamera;
import com.badlogic.gdx.graphics.Pixmap;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.graphics.glutils.ShapeRenderer;
import com.badlogic.gdx.utils.ScreenUtils;
import com.badlogic.gdx.utils.viewport.FitViewport;

public class PlayScreen extends ScreenAdapter {
    private static final float WORLD_WIDTH = 800f;
    private static final float WORLD_HEIGHT = 450f;
    private static final float MARKER_SIZE = 64f;

    private final FirstGame game;
    private final OrthographicCamera camera;
    private final FitViewport viewport;
    private final SpriteBatch batch;
    private final ShapeRenderer shapes;
    private final Texture markerTexture;
    private final Texture whiteTexture;

    private float elapsed;
    private boolean paused;

    public PlayScreen(FirstGame game) {
        this.game = game;
        camera = new OrthographicCamera();
        viewport = new FitViewport(WORLD_WIDTH, WORLD_HEIGHT, camera);
        batch = new SpriteBatch();
        shapes = new ShapeRenderer();
        markerTexture = createMarkerTexture();
        markerTexture.setFilter(Texture.TextureFilter.Nearest,
                                Texture.TextureFilter.Nearest);
        whiteTexture = createWhiteTexture();
    }

    private Texture createMarkerTexture() {
        Pixmap pixmap = new Pixmap(64, 64, Pixmap.Format.RGBA8888);
        pixmap.setColor(1f, 0.8f, 0.2f, 1f);
        pixmap.fill();
        pixmap.setColor(0.1f, 0.65f, 0.9f, 1f);
        pixmap.fillRectangle(8, 8, 48, 48);
        Texture texture = new Texture(pixmap);
        pixmap.dispose();
        return texture;
    }

    private Texture createWhiteTexture() {
        Pixmap pixmap = new Pixmap(1, 1, Pixmap.Format.RGBA8888);
        pixmap.setColor(1f, 1f, 1f, 1f);
        pixmap.fill();
        Texture texture = new Texture(pixmap);
        pixmap.dispose();
        return texture;
    }

    @Override
    public void show() {
        Gdx.app.log("Play", "Space: pause/resume, Escape: menu");
    }

    @Override
    public void resize(int width, int height) {
        if (width > 0 && height > 0) {
            viewport.update(width, height, true);
        }
    }

    @Override
    public void render(float delta) {
        if (Gdx.input.isKeyJustPressed(Input.Keys.SPACE)) {
            paused = !paused;
        }
        if (!paused) {
            update(Math.min(delta, 0.1f));
        }
        draw();

        if (Gdx.input.isKeyJustPressed(Input.Keys.ESCAPE)) {
            game.showMenu();
            return;
        }
    }

    private void update(float delta) {
        elapsed += delta;
    }

    private void draw() {
        ScreenUtils.clear(0f, 0f, 0f, 1f);
        viewport.apply();
        batch.setProjectionMatrix(camera.combined);

        float x = (WORLD_WIDTH - MARKER_SIZE) / 2f
                + 200f * (float) Math.sin(elapsed);
        float y = (WORLD_HEIGHT - MARKER_SIZE) / 2f;

        batch.begin();
        batch.setColor(0.10f, 0.15f, 0.25f, 1f);
        batch.draw(whiteTexture, 0f, 0f, WORLD_WIDTH, WORLD_HEIGHT);
        batch.setColor(1f, 1f, 1f, 1f);

        batch.draw(markerTexture, 0f, 0f, MARKER_SIZE, MARKER_SIZE);
        batch.draw(markerTexture, WORLD_WIDTH - MARKER_SIZE, 0f,
                   MARKER_SIZE, MARKER_SIZE);
        batch.draw(markerTexture, 0f, WORLD_HEIGHT - MARKER_SIZE,
                   MARKER_SIZE, MARKER_SIZE);
        batch.draw(markerTexture, WORLD_WIDTH - MARKER_SIZE,
                   WORLD_HEIGHT - MARKER_SIZE, MARKER_SIZE, MARKER_SIZE);
        batch.draw(markerTexture, x, y, MARKER_SIZE, MARKER_SIZE);
        batch.end();

        drawShapes(x, y);
    }

    private void drawShapes(float markerX, float markerY) {
        shapes.setProjectionMatrix(camera.combined);

        shapes.begin(ShapeRenderer.ShapeType.Filled);
        shapes.setColor(0.2f, 0.8f, 0.45f, 1f);
        shapes.rect(100f, 80f, 120f, 60f);
        shapes.setColor(1f, 0.55f, 0.2f, 1f);
        shapes.circle(320f, 110f, 30f, 48);
        shapes.setColor(0.7f, 0.4f, 1f, 1f);
        shapes.triangle(410f, 80f, 490f, 80f, 450f, 145f);
        shapes.setColor(0.2f, 0.75f, 0.95f, 1f);
        shapes.ellipse(560f, 85f, 120f, 50f, 48);
        shapes.setColor(1f, 0.8f, 0.2f, 1f);
        shapes.rectLine(100f, 330f, 220f, 360f, 8f);
        shapes.end();

        shapes.begin(ShapeRenderer.ShapeType.Line);
        shapes.setColor(1f, 1f, 1f, 1f);
        shapes.line(100f, 300f, 700f, 300f);
        shapes.rect(100f, 80f, 120f, 60f);
        shapes.circle(320f, 110f, 30f, 48);
        shapes.rect(markerX, markerY, MARKER_SIZE, MARKER_SIZE);
        shapes.end();
    }

    @Override
    public void pause() {
        paused = true;
    }

    @Override
    public void dispose() {
        batch.dispose();
        shapes.dispose();
        markerTexture.dispose();
        whiteTexture.dispose();
    }
}
```

Четыре неподвижных квадрата отмечают углы игрового мира. Пятый движется по горизонтали: его левый край колеблется от 168 до 568 единиц. Пробел останавливает накопление времени, поэтому движущийся квадрат замирает. Ограничение дельты работает так же, как во второй главе.

Под движущимся квадратом расположены зелёный прямоугольник, оранжевый круг, фиолетовый треугольник и голубой эллипс. Над ним — белый отрезок и жёлтая наклонная полоса. Белая рамка следует за квадратом: `drawShapes()` получает те же координаты, что и `batch.draw()`.

Белая текстура 1 × 1 служит заливкой: растягиваем её на всю игровую область и тонируем в тёмно-синий цвет. Затем возвращаем цвет `batch` к белому и рисуем маркеры. В этом примере фон и маркеры используют разные текстуры, поэтому одна пара `begin()` / `end()` не означает одну отправку геометрии на GPU.

`ScreenUtils.clear()` очищает окно чёрным цветом. При обычном состоянии OpenGL без включённого scissor-теста очистка затрагивает весь цветовой буфер, включая поля `FitViewport`. Отрисовка фона через `batch`, напротив, занимает только игровую область. Поэтому поля будут чёрными, а игровая область — синей. Реализацию очистки можно посмотреть в [ScreenUtils](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/utils/ScreenUtils.java).

Конструктор `PlayScreen` здесь вызывается внутри `FirstGame.create()`, когда графический контекст уже доступен. При переходе `Game.setScreen()` вызывает `show()`, а затем `resize()` с текущими размерами, поэтому viewport настроен до первого кадра экрана. При повторном входе из меню он также получает актуальные размеры.

Ресурсы сохраняются при `hide()`: экран из второй главы используется повторно. При завершении `FirstGame.dispose()` вызывает `playScreen.dispose()`, и текстуры, `SpriteBatch` и `ShapeRenderer` освобождаются. У камеры и viewport в этом примере нет ресурсов, требующих `dispose()`.

### Проверка и самостоятельная работа

Из каталога `chapter-01/first-game` выполните `./gradlew build`, затем `./gradlew lwjgl3:run`. В Windows используйте `.\gradlew.bat build` и `.\gradlew.bat lwjgl3:run`.

1. Нажмите Enter. Должны появиться синий фон, четыре квадрата по углам, движущийся квадрат с белой рамкой, цветные фигуры и линии.
2. Изменяйте ширину и высоту окна. Все угловые квадраты должны оставаться видимыми, круг — круглым, а рамка — совпадать с движущимся квадратом. При несовпадении пропорций появятся чёрные поля.
3. Нажмите пробел и измените размер окна. Положение центрального квадрата в мировых координатах сохранится, хотя его экранный размер может измениться.
4. Вернитесь в меню, измените размер окна и снова откройте игровой экран. Проверьте, что область отрисовки соответствует новому окну.
5. Сверните и восстановите игру. Как во второй главе, для продолжения после системной паузы нажмите пробел.

Затем подготовьте собственный квадратный PNG и положите его в `assets/images/marker.png`. В конструкторе замените `markerTexture = createMarkerTexture();` на `markerTexture = new Texture(Gdx.files.internal("images/marker.png"));`. Удалите неиспользуемый метод `createMarkerTexture()`. Остальной код отрисовки и освобождения ресурсов остаётся тем же.

Для сравнения стратегий замените тип поля `FitViewport`, импорт и вызов конструктора на `StretchViewport`. Сделайте окно узким: квадраты вытянутся. Повторите с `FillViewport`: форма сохранится, но часть сцены окажется за границами окна. После эксперимента верните `FitViewport`.

Добавьте белые контуры треугольника и эллипса в группу `ShapeType.Line`, повторив координаты залитых фигур. Затем измените толщину жёлтой полосы с `8f` на `16f`. Для эксперимента с сегментами задайте кругу значение `8` и в заливке, и в контуре, чтобы они совпадали.

Если результат отличается от ожидаемого, проверьте следующие места:

| Симптом | Что проверить |
| --- | --- |
| PNG не найден | Каталог `assets`, относительный путь и регистр имени файла |
| `SpriteBatch.begin must be called before draw` | Все ли вызовы `batch.draw()` стоят между `begin()` и `end()` |
| Маркеры неожиданно затемнены | Возвращается ли цвет `batch` к белому после рисования фона |
| Фигуры смещены относительно текстур | Передаётся ли одна и та же `camera.combined` в `batch` и `shapes` после применения viewport |
| Фигуры перекрываются фоном или рисуются в неожиданном порядке | Завершён ли `batch.end()` до `shapes.begin()` и рисуются ли фигуры после фона |
| Ошибка при начале новой группы фигур | Вызван ли `shapes.end()` перед следующим `shapes.begin(...)` |
| Картинка искажена после изменения окна | Выбран ли `FitViewport` и вызывается ли `viewport.update()` в `resize()` |
| Объекты видны только частично или не там, где ожидалось | Центрирование камеры, её `zoom` и передача `camera.combined` в `batch` |
| После нескольких переходов экран перестаёт работать | Не освобождаются ли повторно используемые текстуры в `hide()` |
