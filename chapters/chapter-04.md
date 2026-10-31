# Глава 4 Ресурсы и экран загрузки

## Оглавление главы

- [Атласы текстур](#атласы-текстур)
- [Загрузка ресурсов через AssetManager](#загрузка-ресурсов-через-assetmanager)
- [Экран загрузки](#экран-загрузки)
- [Освобождение ресурсов](#освобождение-ресурсов)

В третьей главе игровой экран сам создавал текстуры и освобождал их. Когда одни изображения нужны нескольким экранам, такой подход усложняет управление ресурсами: приходится решать, кто загружает картинку и кто имеет право её уничтожить. Теперь перенесём загрузку в общий `AssetManager`, объединим изображения в атлас и добавим отдельный экран ожидания.

Продолжим работу с проектом `chapter-01/first-game`. Подготовим файлы атласа, заменим `FirstGame.java` и `PlayScreen.java`, добавим `LoadingScreen.java`. Все три класса относятся к пакету `ru.learnlibgdx.firstgame` в модуле `core`. `MenuScreen` из второй главы и настольный запуск остаются прежними. Порядок переходов будет таким:

```text
загрузка → меню ⇄ игровой экран
```

Enter открывает игру из меню, пробел переключает паузу, Escape возвращает в меню. Ресурсы загружаются один раз при запуске приложения.

## Атласы текстур

### Зачем объединять изображения

**Атлас текстур** хранит несколько изображений на общей текстуре или на нескольких текстурах, называемых страницами. В LibGDX результат упаковки состоит из текстового файла `.atlas` с описанием областей и изображений страниц, обычно PNG. При распространении игры нужны и описание, и все страницы.

В предыдущем примере фон и маркер использовали разные текстуры. Размещая их на одной странице, мы уменьшаем количество переключений текстур при рисовании. Атлас с несколькими страницами не гарантирует одну отправку геометрии: переход на другую страницу тоже меняет текстуру. Назначение упаковки описано в [руководстве TexturePacker](https://libgdx.com/wiki/tools/texture-packer).

Не стоит собирать все изображения большой игры в один атлас только ради уменьшения числа файлов. В нашем проекте удобно хранить вместе фон-заливку и маркер, поскольку они нужны одновременно. В игре с несколькими уровнями можно выделить общий атлас интерфейса и отдельные атласы уровней.

### Подготовка и упаковка

Создайте в каталоге `first-game` папку `art/atlas`. Подготовьте в любом графическом редакторе два PNG:

- `marker.png` — квадрат 64 × 64 пикселя, например жёлтый с голубой серединой, как в третьей главе;
- `white.png` — полностью белое непрозрачное изображение 1 × 1 пиксель для заливки.

Если выполняли упражнение с собственным PNG в третьей главе, скопируйте его в `art/atlas/marker.png`. Исходники будем хранить вне `assets`, а в ресурсы игры поместим результат упаковки:

```text
first-game/
├── art/atlas/
│   ├── marker.png
│   ├── white.png
│   └── pack.json
└── assets/atlases/
    ├── game.atlas
    └── game.png
```

Последние два файла создаст упаковщик. Для этих маленьких изображений хватит одной страницы. В `art/atlas/pack.json` запишите:

```json
{
  "rotation": false,
  "stripWhitespaceX": false,
  "stripWhitespaceY": false,
  "paddingX": 2,
  "paddingY": 2,
  "duplicatePadding": true,
  "filterMin": "Nearest",
  "filterMag": "Nearest"
}
```

Отключаем поворот изображений и обрезку прозрачных краёв, чтобы дальше рисовать области обычным `batch.draw(region, ...)` без поправок на упаковку. Отступы с копиями граничных пикселей помогают избежать захвата соседней картинки при фильтрации. Для этого примера выбираем `Nearest`. Настройки `pack.json` разобраны в [документации упаковщика](https://libgdx.com/wiki/tools/texture-packer#configuration).

Для запуска используем `TexturePacker` из библиотеки `gdx-tools`. В `lwjgl3/build.gradle` добавьте следующие блоки, сохранив существующие настройки:

```groovy
configurations {
    texturePacker
}

dependencies {
    texturePacker "com.badlogicgames.gdx:gdx-tools:$gdxVersion"
}

tasks.register('packAssets', JavaExec) {
    classpath = configurations.texturePacker
    mainClass = 'com.badlogic.gdx.tools.texturepacker.TexturePacker'
    workingDir = rootProject.projectDir
    args 'art/atlas', 'assets/atlases', 'game'
}
```

`gdxVersion` — то же свойство версии, которое используется в зависимостях проекта из первой главы. Отдельная конфигурация `texturePacker` нужна только задаче упаковки и не добавляет инструмент в зависимости самой игры. `JavaExec` запускает указанный Java-класс с заданными аргументами и рабочим каталогом; см. [документацию Gradle](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.JavaExec.html). Три аргумента упаковщика — входной каталог, выходной каталог и имя атласа без расширения; их обработку можно проверить в [исходном коде TexturePacker](https://github.com/libgdx/libgdx/blob/master/extensions/gdx-tools/src/com/badlogic/gdx/tools/texturepacker/TexturePacker.java).

Из каталога `first-game` выполните:

```bash
./gradlew lwjgl3:packAssets
```

В Windows PowerShell используйте `.\gradlew.bat lwjgl3:packAssets`. Проверьте наличие `assets/atlases/game.atlas` и изображения страницы. После изменения исходных PNG повторяйте упаковку перед запуском игры. В этом упражнении задача запуска не вызывает упаковщик автоматически.

### Получение области атласа

`TextureAtlas` представляет загруженный атлас, а `TextureRegion` — прямоугольную область текстуры. Область не копирует пиксели и не создаёт отдельную текстуру. У `TextureAtlas.AtlasRegion` дополнительно есть сведения об упаковке. Устройство этих объектов описано в исходном коде [TextureRegion](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/g2d/TextureRegion.java) и [TextureAtlas](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/g2d/TextureAtlas.java).

Когда атлас уже загружен, область `marker.png` из нашего входного каталога можно найти по имени `marker`:

```java
TextureRegion marker = atlas.findRegion("marker");
if (marker == null) {
    throw new IllegalStateException("Region not found: marker");
}

// Внутри пары batch.begin() / batch.end():
batch.draw(marker, 100f, 80f, 64f, 64f);
```

Здесь `atlas` имеет тип `TextureAtlas`, `batch` — `SpriteBatch`; фрагмент не является самостоятельным классом. `findRegion()` возвращает `null`, если имя не найдено. Найдите нужные области один раз при подготовке экрана и сохраните ссылки в полях, как в полном примере ниже.

Правило «имя файла без расширения» требует уточнения для других наборов ресурсов: имя может включать путь вложенного каталога, а при стандартном `useIndexes: true` числовой суффикс вроде `_0` сохраняется отдельно как индекс. Например, для `walk_0.png` используют `atlas.findRegion("walk", 0)`. Эти правила описаны в [разделах о структуре каталогов и индексах TexturePacker](https://libgdx.com/wiki/tools/texture-packer). Имена `marker.png` и `white.png` в нашем примере не требуют такой обработки.

## Загрузка ресурсов через AssetManager

### Очередь и готовые объекты

`AssetManager` хранит загруженные ресурсы, использует загрузчики для разных типов и учитывает зависимости между ними. Например, для атласа нужно загрузить изображения его страниц. В стандартном менеджере уже есть загрузчики текстур, атласов, растровых шрифтов, звуков и музыки. Общая схема приведена в [руководстве по AssetManager](https://libgdx.com/wiki/managing-your-assets).

Минимальная последовательность выглядит так:

```java
AssetManager assets = new AssetManager();
assets.load("atlases/game.atlas", TextureAtlas.class);

// Далее в каждом кадре экрана загрузки:
if (assets.update()) {
    TextureAtlas atlas = assets.get("atlases/game.atlas", TextureAtlas.class);
    // Теперь атлас доступен; можно подготовить игровой экран.
}
```

Нужны импорты `com.badlogic.gdx.assets.AssetManager` и `com.badlogic.gdx.graphics.g2d.TextureAtlas`. Создание менеджера и вызов `load()` выполняются один раз, а проверка `update()` — по кадрам. Это схема; полный код ниже также обрабатывает ошибку загрузки.

| Метод | Что делает |
| --- | --- |
| `load(path, type)` | Добавляет запрос в очередь |
| `update()` | Продвигает загрузку; возвращает `true`, когда очередь и текущие задачи обработаны |
| `getProgress()` | Возвращает прогресс обработки от `0` до `1` |
| `isLoaded(path, type)` | Проверяет наличие уже загруженного ресурса |
| `get(path, type)` | Возвращает загруженный объект; сам загрузку не запускает |
| `finishLoading()` | Блокирует вызывающий поток до завершения загрузки |

Поведение методов можно проверить в [исходном коде AssetManager](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/assets/AssetManager.java). До готовности ресурса вызывать `get()` нельзя: будет исключение. Идентификатором служит путь, поэтому будем хранить его в одной константе. При стандартном разрешении внутренних файлов путь задаётся относительно `assets`, без префикса `assets/`.

В очередь достаточно поставить `game.atlas`. Его страницы автоматически добавляются как зависимости загрузчиком [TextureAtlasLoader](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/assets/loaders/TextureAtlasLoader.java). Не нужно отдельно вызывать `load()` для `game.png`. В готовую сборку при этом должны попасть оба файла.

### Что загружается в фоне

Асинхронная загрузка не означает, что все операции выполняются вне потока отрисовки. Например, [TextureLoader](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/assets/loaders/TextureLoader.java) подготавливает данные изображения в рабочем потоке, а создание графической текстуры завершает в потоке отрисовки. Поэтому в нашем приложении вызываем `update()` из `render()`, где доступен графический контекст.

Вызов `finishLoading()` в экране загрузки остановил бы подготовку следующих кадров до завершения работы. По той же причине здесь не нужен цикл `while (!assets.update())`: он удерживал бы управление внутри одного кадра.

У метода есть вариант `update(int millis)`, который старается продвигать очередь в пределах указанного времени. Это не строгий предел длительности кадра: отдельная синхронная операция может занять больше времени. В упражнении достаточно одного `update()` за кадр.

## Экран загрузки

### Ресурсы самого индикатора

Экран загрузки должен уметь рисовать до готовности игрового атласа. Для полосы используем `ShapeRenderer`: он рисует геометрические фигуры, в нашем случае два прямоугольника. Ему не нужен PNG, но он создаёт собственные графические ресурсы, которые позднее освободим. API приведён в [исходном коде ShapeRenderer](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/glutils/ShapeRenderer.java).

Полоса занимает 400 × 24 единицы в знакомой области 800 × 450. Серый прямоугольник обозначает полную длину, голубой — текущий прогресс. Если файл не загрузится, нарисуем красную полосу и выведем причину в журнал. Перехода в меню при ошибке не будет.

### Главный класс игры

Замените `FirstGame.java` следующим кодом:

```java
package ru.learnlibgdx.firstgame;

import com.badlogic.gdx.Game;
import com.badlogic.gdx.assets.AssetManager;
import com.badlogic.gdx.graphics.g2d.TextureAtlas;

public class FirstGame extends Game {
    public static final String GAME_ATLAS = "atlases/game.atlas";

    private AssetManager assets;
    private LoadingScreen loadingScreen;
    private MenuScreen menuScreen;
    private PlayScreen playScreen;

    @Override
    public void create() {
        assets = new AssetManager();
        assets.load(GAME_ATLAS, TextureAtlas.class);
        loadingScreen = new LoadingScreen(this, assets);
        setScreen(loadingScreen);
    }

    public void onAssetsLoaded() {
        TextureAtlas atlas = assets.get(GAME_ATLAS, TextureAtlas.class);
        playScreen = new PlayScreen(this, atlas);
        menuScreen = new MenuScreen(this);
        showMenu();

        assets.setErrorListener(null);
        loadingScreen.dispose();
        loadingScreen = null;
    }

    public void showMenu() {
        setScreen(menuScreen);
    }

    public void showPlay() {
        setScreen(playScreen);
    }

    @Override
    public void dispose() {
        setScreen(null);
        if (loadingScreen != null) loadingScreen.dispose();
        if (menuScreen != null) menuScreen.dispose();
        if (playScreen != null) playScreen.dispose();
        assets.dispose();
    }
}
```

Менеджер создаётся в `create()` и принадлежит игре. `static final` используется только для строки пути. Игровой экран появляется после загрузки, поэтому его конструктор получает готовый атлас. Если окно закрыли во время загрузки, `menuScreen` и `playScreen` ещё равны `null`, а `loadingScreen` всё равно будет освобождён.

После успешного перехода экран загрузки больше не используется: освобождаем его сразу и обнуляем поле. Меню и игровой экран, напротив, сохраняются для повторных переходов.

### Код LoadingScreen

Создайте `LoadingScreen.java`:

```java
package ru.learnlibgdx.firstgame;

import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.ScreenAdapter;
import com.badlogic.gdx.assets.AssetManager;
import com.badlogic.gdx.graphics.OrthographicCamera;
import com.badlogic.gdx.graphics.glutils.ShapeRenderer;
import com.badlogic.gdx.utils.ScreenUtils;
import com.badlogic.gdx.utils.viewport.FitViewport;

public class LoadingScreen extends ScreenAdapter {
    private final FirstGame game;
    private final AssetManager assets;
    private final OrthographicCamera camera;
    private final FitViewport viewport;
    private final ShapeRenderer shapes;
    private boolean failed;

    public LoadingScreen(FirstGame game, AssetManager assets) {
        this.game = game;
        this.assets = assets;
        camera = new OrthographicCamera();
        viewport = new FitViewport(800f, 450f, camera);
        shapes = new ShapeRenderer();

        assets.setErrorListener((asset, error) -> {
            failed = true;
            Gdx.app.error("Loading", "Cannot load: " + asset.fileName, error);
        });
    }

    @Override
    public void resize(int width, int height) {
        if (width > 0 && height > 0) {
            viewport.update(width, height, true);
        }
    }

    @Override
    public void render(float delta) {
        boolean finished = false;
        if (!failed) {
            finished = assets.update();
        }

        ScreenUtils.clear(0.04f, 0.05f, 0.08f, 1f);
        viewport.apply();
        shapes.setProjectionMatrix(camera.combined);

        shapes.begin(ShapeRenderer.ShapeType.Filled);
        shapes.setColor(0.2f, 0.2f, 0.25f, 1f);
        shapes.rect(200f, 213f, 400f, 24f);
        if (failed) {
            shapes.setColor(0.9f, 0.2f, 0.2f, 1f);
            shapes.rect(200f, 213f, 400f, 24f);
        } else {
            shapes.setColor(0.1f, 0.65f, 0.9f, 1f);
            shapes.rect(200f, 213f, 400f * assets.getProgress(), 24f);
        }
        shapes.end();

        if (finished && !failed) {
            game.onAssetsLoaded();
            return;
        }
    }

    @Override
    public void dispose() {
        shapes.dispose();
    }
}
```

Проверяем `failed` после `update()`: именно во время этого вызова обработчик может сообщить об ошибке. Если установлен слушатель ошибок, одной готовности очереди недостаточно для перехода. В этом примере после сбоя дальнейшую загрузку останавливаем, оставляя окно открытым для просмотра журнала. Исправьте ресурс и перезапустите приложение; повторную попытку отдельной кнопкой здесь не реализуем.

`getProgress()` отражает обработку запросов и зависимостей, а не прочитанные байты или оставшееся время. Небольшой атлас загрузится быстро, и полоса может быть видна всего несколько кадров. Это нормально; искусственная задержка не требуется.

Переключение выполняется после `shapes.end()`. Вызов `setScreen()` сам вызовет `hide()` старого экрана, затем `show()` и `resize()` нового. Он не вызывает `dispose()` старого экрана — поэтому освобождение явно написано в `FirstGame`. Этот порядок определён в [реализации Game](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/Game.java).

### Игровой экран с общим атласом

Замените `PlayScreen.java` следующим вариантом. Камера, движение и управление сохраняются из третьей главы, но вместо самостоятельного создания текстур экран получает две области атласа:

```java
package ru.learnlibgdx.firstgame;

import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.Input;
import com.badlogic.gdx.ScreenAdapter;
import com.badlogic.gdx.graphics.OrthographicCamera;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.graphics.g2d.TextureAtlas;
import com.badlogic.gdx.graphics.g2d.TextureRegion;
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
    private final TextureRegion markerRegion;
    private final TextureRegion whiteRegion;

    private float elapsed;
    private boolean paused;

    public PlayScreen(FirstGame game, TextureAtlas atlas) {
        this.game = game;
        markerRegion = requireRegion(atlas, "marker");
        whiteRegion = requireRegion(atlas, "white");
        camera = new OrthographicCamera();
        viewport = new FitViewport(WORLD_WIDTH, WORLD_HEIGHT, camera);
        batch = new SpriteBatch();
    }

    private static TextureRegion requireRegion(TextureAtlas atlas, String name) {
        TextureRegion region = atlas.findRegion(name);
        if (region == null) {
            throw new IllegalStateException("Region not found: " + name);
        }
        return region;
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
        batch.draw(whiteRegion, 0f, 0f, WORLD_WIDTH, WORLD_HEIGHT);
        batch.setColor(1f, 1f, 1f, 1f);
        batch.draw(markerRegion, 0f, 0f, MARKER_SIZE, MARKER_SIZE);
        batch.draw(markerRegion, WORLD_WIDTH - MARKER_SIZE, 0f,
                   MARKER_SIZE, MARKER_SIZE);
        batch.draw(markerRegion, 0f, WORLD_HEIGHT - MARKER_SIZE,
                   MARKER_SIZE, MARKER_SIZE);
        batch.draw(markerRegion, WORLD_WIDTH - MARKER_SIZE,
                   WORLD_HEIGHT - MARKER_SIZE, MARKER_SIZE, MARKER_SIZE);
        batch.draw(markerRegion, x, y, MARKER_SIZE, MARKER_SIZE);
        batch.end();
    }

    @Override
    public void pause() {
        paused = true;
    }

    @Override
    public void dispose() {
        batch.dispose();
    }
}
```

Поиск отсутствующей области — другая ошибка, чем сбой загрузки файла. Атлас может успешно загрузиться, но не содержать `marker`. В таком случае `requireRegion()` остановит подготовку экрана с понятным исключением. Проверяйте имена исходных PNG и повторно запускайте упаковщик.

## Освобождение ресурсов

### Кто владеет объектом

Сборщик мусора Java не освобождает за нас все ресурсы графического драйвера. Для объектов LibGDX, реализующих `Disposable`, владелец должен организовать вызов `dispose()`. Общий принцип разобран в [руководстве по управлению памятью](https://libgdx.com/wiki/articles/memory-management).

В нашем примере ответственность распределена так:

| Объект | Владелец | Когда освобождается |
| --- | --- | --- |
| `AssetManager` | `FirstGame` | При завершении приложения |
| Атлас и загруженные через менеджер страницы | `AssetManager` | При освобождении менеджера |
| `ShapeRenderer` | `LoadingScreen` | После перехода в меню или при закрытии окна во время загрузки |
| `SpriteBatch` | `PlayScreen` | При освобождении игрового экрана |
| `TextureRegion` | Поле игрового экрана, ссылающееся на область общей текстуры | Отдельного `dispose()` нет |

Экран заимствует области атласа. Поэтому в его `dispose()` нет ни `atlas.dispose()`, ни `markerRegion.getTexture().dispose()`: ресурсом управляет менеджер. Ручное освобождение такой текстуры оставило бы в менеджере и других экранах ссылки на непригодный ресурс.

Если же создать атлас напрямую через `new TextureAtlas(...)`, минуя менеджер, обязанность вызвать `atlas.dispose()` будет у создавшего его владельца. Не смешивайте эти две схемы для одного объекта. `SpriteBatch.dispose()` освобождает собственные ресурсы пакета рисования, но не используемые им текстуры.

### Выгрузка отдельных ресурсов

Когда уровень окончательно закрыт и его изображения больше никому не нужны, ресурс можно выгрузить по пути:

```java
assets.unload(FirstGame.GAME_ATLAS);
```

Это пример API, **его не нужно добавлять в `hide()` нашего игрового экрана**. Мы возвращаемся к тому же `PlayScreen`, и его поля должны продолжать ссылаться на живые текстуры. Для выгрузки уровня сначала прекратите его отрисовку и удалите ссылки на его ресурсы; при следующем входе заново загрузите атлас и создайте области.

Менеджер ведёт собственные счётчики ссылок: он учитывает обработанные запросы `load()` и зависимости ресурсов, а не количество Java-переменных, ссылающихся на объект. Повторные запросы `load()` для одного ресурса увеличивают счётчик после обработки. Для уже загруженного ресурса без ожидающих запросов `unload()` уменьшает счётчик и освобождает ресурс, когда тот достигает нуля. Если запрос ещё находится в очереди, `unload()` может отменить его. Вызов `get()` лишь возвращает объект и не добавляет владение. Поэтому нельзя бездумно вызывать `load()` при каждом `show()` экрана. Правила счётчиков приведены в [руководстве AssetManager](https://libgdx.com/wiki/managing-your-assets#disposing-assets), а обработка очереди — в [реализации unload()](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/assets/AssetManager.java).

`clear()` освобождает содержимое менеджера и очищает очередь, оставляя сам менеджер пригодным к дальнейшему использованию. `dispose()` дополнительно завершает его работу. При выходе из игры достаточно `assets.dispose()`; предварительный `clear()` не нужен. Эти операции могут ожидать завершения уже выполняющейся работы, поэтому не следует использовать их как способ мгновенно отменить загрузку в каждом кадре.

В `FirstGame.dispose()` сначала снимаем текущий экран через `setScreen(null)`, затем освобождаем экраны и последним — менеджер. Не рассчитываем на `Game.dispose()` как на автоматическое освобождение всех экранов: его стандартная реализация только скрывает текущий экран.

### Самостоятельная проверка

После упаковки запустите `./gradlew lwjgl3:run` из каталога `first-game` или `.\gradlew.bat lwjgl3:run` в Windows.

1. Дождитесь синего меню и нажмите Enter. Должны появиться четыре угловых маркера и один движущийся, как в третьей главе.
2. Проверьте пробел, Escape, повторный вход и изменение размера окна. Возврат из меню не должен запускать загрузку заново.
3. Закройте игру. Временно измените `GAME_ATLAS` на `"atlases/missing.atlas"` и запустите её снова. Должны появиться красная полоса и сообщение с причиной ошибки в консоли. Верните правильный путь.
4. Измените цвета `marker.png`, повторите `lwjgl3:packAssets` и перезапустите игру. У всех пяти маркеров должно измениться изображение.
5. Сверните и восстановите окно во время игры. Как в главах 2 и 3, движение останется на паузе до нажатия пробела.

Если результат отличается от ожидаемого, проверьте следующие места:

| Симптом | Что проверить |
| --- | --- |
| Не найден `.atlas` или PNG страницы | Выполнена ли упаковка, все ли выходные файлы находятся в `assets/atlases`, совпадает ли регистр пути |
| `Asset not loaded` | Не вызывается ли `get()` до завершения загрузки |
| `Region not found` | Есть ли в атласе области `marker` и `white`; перепакованы ли изображения |
| Полоса не двигается | Вызывается ли `update()` каждый кадр; нет ли в журнале ошибки |
| Экран загрузки почти незаметен | Для маленького атласа это ожидаемое поведение |
| После возврата из меню пропадают изображения | Не освобождаются ли атлас или его страницы в `hide()` либо вручную из экрана |
| Изменённый PNG не виден в игре | Повторно ли запущен упаковщик и перезапущено ли приложение |
