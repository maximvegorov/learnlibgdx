# Глава 5 Шрифты в LibGDX

## Оглавление главы

- [TrueType и BitmapFont](#truetype-и-bitmapfont)
- [Свой FontGenerator](#свой-fontgenerator)
- [Размеры и качество текста](#размеры-и-качество-текста)

В четвёртой главе мы вынесли изображения в атлас и организовали загрузку ресурсов. Теперь добавим в меню название игры и подсказки на русском языке. Заодно разберёмся, почему один и тот же шрифт может выглядеть чётко в маленьком окне и размыто после увеличения.

Продолжим проект `my-game`: подключим расширение FreeType, добавим файл шрифта и класс `FontGenerator`, заменим `MenuScreen.java`. Оба Java-класса относятся к пакету `ru.learnlibgdx.mygame` в модуле `core`. `MyGame`, `LoadingScreen` и `PlayScreen` остаются в варианте из главы 4. Enter открывает игру, пробел переключает паузу, Escape возвращает в меню.

## TrueType и BitmapFont

### От описания букв к изображению

**Глиф** — графическое представление символа. Для русского меню нам нужны глифы кириллицы, цифр и используемых знаков препинания.

TrueType-шрифт, обычно файл `.ttf`, содержит описание контуров и метрик. При растеризации контуры превращаются в пиксели выбранного размера. В нашем приложении эту работу выполняет расширение `gdx-freetype`: его `FreeTypeFontGenerator` создаёт объект `BitmapFont`. Во время отрисовки текст уже состоит из изображений глифов, размещённых на текстурах. Общий подход описан в [руководстве по FreeType](https://libgdx.com/wiki/extensions/gdx-freetype).

Есть два удобных пути подготовки шрифта:

| Подход | Что положить в `assets` | Когда подготавливаются изображения глифов |
| --- | --- | --- |
| Готовый растровый шрифт | Описание `.fnt` и все PNG-страницы | До запуска игры, в инструменте подготовки шрифтов |
| Генерация из TrueType | Исходный `.ttf` | Во время подготовки ресурсов приложения |

В обоих случаях текст можно рисовать через `BitmapFont.draw()`. Файл `.ttf` не передают конструктору `BitmapFont` вместо `.fnt`: сначала нужен генератор. Возможности растровых шрифтов и инструменты их подготовки перечислены в [документации Bitmap fonts](https://libgdx.com/wiki/graphics/2d/fonts/bitmap-fonts).

### Готовый растровый шрифт

Например, [Hiero](https://libgdx.com/wiki/tools/hiero) позволяет заранее выбрать шрифт, размер и набор символов, затем сохранить описание и изображения. При экспорте включите русские буквы, в том числе `Ё` и `ё`. Для обычного `BitmapFont` используйте текстовый формат AngelCode BMFont.

Если результат экспорта находится в `assets/fonts`, загрузка выглядит так:

```java
BitmapFont font = new BitmapFont(Gdx.files.internal("fonts/menu.fnt"));

// Во время отрисовки, с уже настроенной матрицей SpriteBatch:
batch.begin();
font.draw(batch, "Привет, игрок!", 32f, 400f);
batch.end();

// Когда шрифт окончательно больше не нужен:
font.dispose();
```

Нужны импорты `com.badlogic.gdx.Gdx` и `com.badlogic.gdx.graphics.g2d.BitmapFont`; `batch` имеет тип `SpriteBatch`. Это схема жизненного цикла, а не содержимое одного метода: создание выполняется один раз, рисование — по кадрам, освобождение — при завершении работы владельца. Изображения страниц загружаются по описанию `.fnt`, поэтому одного этого файла недостаточно.

В главе 4 мы познакомились с `AssetManager`. Он умеет загружать готовый `.fnt` как `BitmapFont`; в таком случае освобождением управляет менеджер. В примере выше шрифт создан напрямую, и его освобождает создавший код. Самый короткий вариант `new BitmapFont()` использует встроенный растровый шрифт, но для русского интерфейса подготовим собственный набор глифов. Конструкторы и правила владения текстурами приведены в [исходном коде BitmapFont](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/g2d/BitmapFont.java).

### Подключение FreeType для Desktop

Для основного примера выберем генерацию из TTF. В существующий блок `dependencies` файла `core/build.gradle` добавьте:

```groovy
api "com.badlogicgames.gdx:gdx-freetype:$gdxVersion"
```

В блок `dependencies` файла `lwjgl3/build.gradle` добавьте:

```groovy
implementation "com.badlogicgames.gdx:gdx-freetype-platform:$gdxVersion:natives-desktop"
```

Первая зависимость предоставляет Java API, вторая — нативную библиотеку для Desktop. Используйте то же значение `gdxVersion`, что и для остальных модулей LibGDX. Если FreeType уже был выбран при создании проекта, не дублируйте строки. После изменения обновите Gradle-проект в IDE. Координаты зависимостей приведены в [официальном руководстве по Gradle](https://libgdx.com/wiki/articles/dependency-management-with-gradle#freetypefont-gradle).

Подготовьте обычный TTF со статическим начертанием и поддержкой кириллицы, который разрешено распространять вместе с игрой. Скопируйте его в `assets/fonts/ui.ttf`; `ui.ttf` здесь — выбранное нами имя файла. Сохраните рядом необходимые по лицензии уведомления. Структура ресурсов будет такой:

```text
assets/
├── atlases/
│   ├── game.atlas
│   └── game.png
└── fonts/
    └── ui.ttf
```

Это ручной шаг: расширение не скачивает шрифт и не создаёт `ui.ttf`. В коде внутренний путь будет `fonts/ui.ttf`, без префикса `assets/`.

## Свой FontGenerator

### Задача класса

`FontGenerator` — наш вспомогательный класс, который использует библиотечный `FreeTypeFontGenerator`. Он будет хранить настройки одного начертания, создавать шрифты нужных размеров и повторно возвращать уже созданные объекты.

Для меню выберем два размера: 40 для заголовка и 24 для подсказок. Набор символов зададим явно: латинские буквы, цифры, русские буквы и несколько знаков. Включение символа в набор не добавляет его в исходный TTF: соответствующий глиф должен быть в самом файле.

Ключом кэша станет размер. Это подходит, пока файл шрифта, набор символов, обводка и фильтры одинаковы для всех запросов. Если позже добавить разные начертания или эффекты, их тоже потребуется учитывать в ключе либо разделить по экземплярам генератора.

### Реализация

Создайте `FontGenerator.java`:

```java
package ru.learnlibgdx.mygame;

import com.badlogic.gdx.files.FileHandle;
import com.badlogic.gdx.graphics.Color;
import com.badlogic.gdx.graphics.Texture.TextureFilter;
import com.badlogic.gdx.graphics.g2d.BitmapFont;
import com.badlogic.gdx.graphics.g2d.freetype.FreeTypeFontGenerator;
import com.badlogic.gdx.graphics.g2d.freetype.FreeTypeFontGenerator.FreeTypeFontParameter;
import com.badlogic.gdx.utils.Disposable;

import java.util.HashMap;
import java.util.Map;

public final class FontGenerator implements Disposable {
    private static final String CHARACTERS =
            "\u0000 ABCDEFGHIJKLMNOPQRSTUVWXYZ"
            + "abcdefghijklmnopqrstuvwxyz0123456789"
            + "АБВГДЕЁЖЗИЙКЛМНОПРСТУФХЦЧШЩЪЫЬЭЮЯ"
            + "абвгдеёжзийклмнопрстуфхцчшщъыьэюя"
            + ".,:;!?-–—()[]«»\"'/\\+*=_%№…";

    private final FreeTypeFontGenerator generator;
    private final Map<Integer, BitmapFont> fonts = new HashMap<>();
    private boolean disposed;

    public FontGenerator(FileHandle file) {
        generator = new FreeTypeFontGenerator(file);
    }

    public BitmapFont get(int size) {
        if (disposed) {
            throw new IllegalStateException("FontGenerator is disposed");
        }
        if (size <= 0) {
            throw new IllegalArgumentException("Font size must be positive");
        }

        BitmapFont cached = fonts.get(size);
        if (cached != null) {
            return cached;
        }

        FreeTypeFontParameter parameter = new FreeTypeFontParameter();
        parameter.size = size;
        parameter.characters = CHARACTERS;
        parameter.color = Color.WHITE;
        parameter.minFilter = TextureFilter.Linear;
        parameter.magFilter = TextureFilter.Linear;
        parameter.incremental = false;

        BitmapFont font = generator.generateFont(parameter);
        font.setUseIntegerPositions(false);
        fonts.put(size, font);
        return font;
    }

    @Override
    public void dispose() {
        if (disposed) {
            return;
        }
        disposed = true;
        for (BitmapFont font : fonts.values()) {
            font.dispose();
        }
        fonts.clear();
        generator.dispose();
    }
}
```

`\u0000` в начале набора позволяет генератору подготовить глиф замены для отсутствующих символов. Пробел включён отдельно. `incremental = false` означает, что нужные глифы создаются при вызове `generateFont()`. При `incremental = true` возможна генерация по мере обращения к символам; тогда нельзя освобождать библиотечный генератор, пока шрифт ещё может запросить новый глиф. Параметры описаны в [FreeTypeFontGenerator](https://github.com/libgdx/libgdx/blob/master/extensions/gdx-freetype/src/com/badlogic/gdx/graphics/g2d/freetype/FreeTypeFontGenerator.java).

В нашем классе генератор остаётся живым, чтобы можно было запросить другой размер. Все созданные шрифты принадлежат `FontGenerator`. Получивший их экран **не вызывает `dispose()` у каждого шрифта отдельно**. Также не следует менять масштаб общего объекта без учёта других пользователей: два вызова `get(24)` возвращают одну и ту же ссылку.

Вызывайте конструктор, `get()` и `dispose()` в потоке приложения, где доступен графический контекст. Не создавайте экземпляр в статическом поле до запуска LibGDX и не переносите `get()` в произвольный фоновый поток: генерация включает создание текстур. Требования к потоку графических операций объясняются в [документации Threading](https://libgdx.com/wiki/app/threading).

### Меню с русским текстом

Замените `MenuScreen.java` следующим кодом:

```java
package ru.learnlibgdx.mygame;

import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.Input;
import com.badlogic.gdx.ScreenAdapter;
import com.badlogic.gdx.graphics.Color;
import com.badlogic.gdx.graphics.OrthographicCamera;
import com.badlogic.gdx.graphics.g2d.BitmapFont;
import com.badlogic.gdx.graphics.g2d.GlyphLayout;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.utils.ScreenUtils;
import com.badlogic.gdx.utils.viewport.FitViewport;

public class MenuScreen extends ScreenAdapter {
    private static final float UI_WIDTH = 800f;
    private static final float UI_HEIGHT = 450f;

    private final MyGame game;
    private final OrthographicCamera camera;
    private final FitViewport viewport;
    private final SpriteBatch batch;
    private final FontGenerator fonts;
    private final BitmapFont titleFont;
    private final BitmapFont textFont;
    private final GlyphLayout layout = new GlyphLayout();

    public MenuScreen(MyGame game) {
        this.game = game;
        camera = new OrthographicCamera();
        viewport = new FitViewport(UI_WIDTH, UI_HEIGHT, camera);
        batch = new SpriteBatch();
        fonts = new FontGenerator(Gdx.files.internal("fonts/ui.ttf"));
        titleFont = fonts.get(40);
        textFont = fonts.get(24);
    }

    @Override
    public void resize(int width, int height) {
        if (width > 0 && height > 0) {
            viewport.update(width, height, true);
        }
    }

    @Override
    public void render(float delta) {
        ScreenUtils.clear(0.10f, 0.15f, 0.25f, 1f);
        viewport.apply();
        batch.setProjectionMatrix(camera.combined);

        batch.begin();
        titleFont.setColor(Color.WHITE);
        drawCentered(titleFont, "My Game", 330f);
        textFont.setColor(0.75f, 0.9f, 1f, 1f);
        drawCentered(textFont, "Нажмите Enter, чтобы играть", 245f);
        textFont.setColor(Color.WHITE);
        drawCentered(textFont, "Пробел — пауза, Escape — меню", 195f);
        batch.end();

        if (Gdx.input.isKeyJustPressed(Input.Keys.ENTER)) {
            game.showPlay();
            return;
        }
    }

    private void drawCentered(BitmapFont font, String text, float y) {
        layout.setText(font, text);
        font.draw(batch, layout, (UI_WIDTH - layout.width) / 2f, y);
    }

    @Override
    public void dispose() {
        batch.dispose();
        fonts.dispose();
    }
}
```

`GlyphLayout` вычисляет расположение глифов и ширину строки. Поэтому заголовок центрируется по измеренному тексту, а не по количеству букв. Пробел, `Ш` и `i` могут иметь разную ширину; кроме того, расстояние между некоторыми парами букв зависит от кернинга. После `setColor()` заново вызываем `layout.setText()`, чтобы раскладка получила текущий цвет. Эти операции описаны в [исходном коде GlyphLayout](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/g2d/GlyphLayout.java).

В используемой системе координат Y направлен вверх. Параметр `y` у `BitmapFont.draw()` задаёт положение верха заглавных букв первой строки; это не нижний край текстового прямоугольника. Базовая линия находится примерно на `font.getCapHeight()` ниже, а отдельные глифы могут выступать за эти границы. Для компоновки используйте метрики и `GlyphLayout`, а не предположение, что размер 24 означает ровно 24 единицы высоты любой строки.

В этой версии шрифты создаются синхронно в конструкторе меню после загрузки атласа. Прогресс `AssetManager` из главы 4 их не учитывает. Два небольших набора удобны для учебного примера; при большом числе шрифтов подготовку стоит включить в загрузку ресурсов. Для этого существуют `FreeTypeFontGeneratorLoader` и `FreetypeFontLoader`, описанные в [разделе загрузки TTF через AssetManager](https://libgdx.com/wiki/managing-your-assets#loading-a-ttf-using-the-assethandler). Не регистрируйте один и тот же объект одновременно у менеджера и у нашего владельца шрифтов.

Меню хранится в `MyGame` между переходами. Поэтому в `hide()` ничего не освобождаем. При закрытии приложения `MyGame.dispose()` из главы 4 вызовет `menuScreen.dispose()`, а тот освободит пакет рисования, созданные шрифты и генератор.

## Размеры и качество текста

### Размер генерации и масштаб отрисовки

Разделяйте три величины:

1. **Размер генерации** — `FreeTypeFontParameter.size`, размер растеризации в пикселях.
2. **Масштаб данных шрифта** — `font.getData().setScale(...)`, который изменяет размеры и размещение уже готовых глифов при рисовании.
3. **Масштаб камеры и Viewport** — преобразование единиц интерфейса в экранные координаты.

Например, генерация при размере 24 и последующий `setScale(2f)` растянут готовые изображения. Генерация при размере 48 заново растеризует контуры, поэтому результат может быть качественнее. `setScale()` не обращается к TTF и не добавляет детализацию. `setScale(1f)` возвращает масштаб данных к исходному значению.

В нашем меню камера не приближается, а `FitViewport` сохраняет область 800 × 450. При обычном соотношении экранных координат и физических пикселей 1:1 окно 1600 × 900 увеличит все элементы вдвое. Шрифты, созданные при 24 и 40, тоже увеличатся вдвое. Изменение размера окна само по себе не запускает генерацию новых глифов. Поведение `FitViewport` описано в [руководстве по Viewport](https://libgdx.com/wiki/graphics/viewports).

Для интерфейса удобно выбрать несколько размеров под роли: подписи, основной текст, заголовок. Не запрашивайте новый размер на каждом кадре анимации: наш кэш хранит каждый созданный вариант до `dispose()`. Число размеров и набор символов напрямую влияют на объём ресурсов.

### Фильтры и положение глифов

| Настройка | Результат | Когда попробовать |
| --- | --- | --- |
| `Nearest` | Выбирает ближайший тексель, при увеличении видны ступеньки | Пиксельный шрифт при целочисленном масштабе |
| `Linear` | Интерполирует соседние тексели, при увеличении смягчает края | Сглаженный интерфейсный шрифт |
| Мипмапы с подходящим min-фильтром | Используют уменьшенные уровни текстуры | Заметное уменьшение текста, после проверки результата |

В нашем генераторе установлены `Linear` для уменьшения и увеличения. Это разумная исходная настройка для меню, но она не восстанавливает детали маленького растрового изображения. Сильное уменьшение тоже может ухудшать читаемость; для мелкой подписи лучше подготовить подходящий размер, чем сжимать большой заголовок. Проблемы масштабирования обсуждаются в [руководстве FreeType](https://libgdx.com/wiki/extensions/gdx-freetype).

У `FreeTypeFontParameter` есть `genMipMaps`. Если включаете его, выбирайте min-фильтр, использующий мипмапы, например `TextureFilter.MipMapLinearLinear`; один этот фильтр без создания уровней недостаточен. Для текущего меню мипмапы не включаем.

`setUseIntegerPositions(false)` в нашем классе разрешает дробные позиции глифов. Это полезно для центрирования и плавного движения. При `true` позиции округляются, но в координатах рисования, а не обязательно в физических пикселях дисплея. Если `Viewport` затем применит дробный масштаб, округление само по себе не гарантирует чёткость. Для пиксельного текста согласуйте и положение, и масштаб с экранной сеткой. Реализация округления находится в [BitmapFontCache](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/g2d/BitmapFontCache.java).

### HiDPI и размер интерфейса

На HiDPI-дисплее логическая ширина окна может отличаться от ширины буфера отрисовки в физических пикселях. Для LWJGL3 это зависит от `HdpiMode`. Размеры буфера доступны через `Gdx.graphics.getBackBufferWidth()` и `getBackBufferHeight()`. Не следует считать, что `getWidth()` всегда возвращает физические пиксели. Различие объясняется в [документации графической конфигурации](https://libgdx.com/wiki/graphics/querying-and-configuring-graphics).

Для меню с камерой без дополнительного zoom можно оценить плотность так:

```text
физических пикселей на единицу UI =
    физическая ширина области Viewport / мировая ширина Viewport
```

Здесь нужна ширина самой области рисования без боковых полос. При логических координатах ширину `viewport.getScreenWidth()` можно перевести в физические пиксели, умножив на отношение `getBackBufferWidth() / getWidth()` с вычислением в `float` и при ненулевой ширине окна.

Если на одну единицу UI приходится два физических пикселя, для условного размера 24 можно сгенерировать шрифт при размере 48 и назначить ему масштаб `0.5f`. Он займёт примерно прежнее место в интерфейсе, но будет использовать более подробную растеризацию. Метрики могут немного измениться из-за округления и хинтинга, поэтому раскладку нужно пересчитать. Это стратегия адаптации, а не автоматическое поведение приведённого `MenuScreen`.

Для её реализации храните отдельные варианты с учётом плотности и обновляйте их при смене выбранного уровня масштаба. Не применяйте такой `setScale()` к общему шрифту, если другие надписи ожидают его исходный размер. Для небольшого проекта можно начать с фиксированных размеров нашей главы и проверить их на целевых экранах.

### Читаемость и масштабируемые надписи

Начертание и контраст не менее важны, чем фильтр. Проверяйте мелкий текст на настоящем игровом фоне, с цифрами, знаками и буквами `Ё`, `й`, `щ`. Светлая надпись теряется на светлой картинке; подложка, тень или обводка помогают отделить её от фона. У генератора есть параметры `borderWidth`, `borderColor`, `shadowOffsetX` и `shadowOffsetY`. Подбирайте их для конкретного размера: толстая обводка может закрыть просветы маленьких букв.

Для текста, который должен сильно увеличиваться или вращаться, существует другой подход — **distance field fonts**. Текстура хранит сведения о расстоянии до контура, а специальный шейдер восстанавливает границу. Это требует соответствующей подготовки шрифта и отрисовки; переключение обычного PNG на `Linear` не превращает его в distance field. Подход и пример реализации приведены в [официальном руководстве Distance field fonts](https://libgdx.com/wiki/graphics/2d/fonts/distance-field-fonts). Для нашего меню достаточно обычных растровых глифов подходящего размера.

### Проверка и самостоятельная работа

Из каталога `my-game` запустите `./gradlew lwjgl3:run` или `.\gradlew.bat lwjgl3:run` в Windows PowerShell.

1. После загрузки должны появиться заголовок «My Game» и две подсказки. Проверьте русские буквы и длинное тире.
2. Нажмите Enter, затем Escape. Меню должно вернуться с тем же текстом; повторная генерация шрифтов при переходе не нужна.
3. Измените размер окна. Надписи должны оставаться по центру области интерфейса. Сравните чёткость в окне 800 × 450 и 1600 × 900 с учётом масштаба дисплея.
4. Временно замените заголовок на «Ёжик, йод, щука — 0123456789». Проверьте, что у исходного TTF есть все нужные глифы.
5. Сравните два варианта заголовка: `fonts.get(20)` с `setScale(2f)` и `fonts.get(40)` без изменения масштаба. В каждом запуске меняйте только `titleFont`, затем верните основной пример.
6. Попробуйте `Nearest` вместо `Linear`, перезапустив приложение для повторной генерации. Сравните результат при разных размерах окна.

Если текст выглядит неправильно, начните с этих проверок:

| Симптом | Что проверить |
| --- | --- |
| Не найден `fonts/ui.ttf` | Есть ли файл в `assets/fonts`, совпадают ли имя и регистр |
| Ошибка загрузки нативной библиотеки FreeType | Подключены ли `natives-desktop`, совпадают ли версии зависимостей |
| Русские буквы заменены квадратами или отсутствуют | Есть ли они в TTF и в `CHARACTERS`; сохранён ли Java-файл в UTF-8 |
| Текст размыт после увеличения окна | Насколько растянут исходный растр; нужен ли более крупный размер генерации |
| Текст обрезан у края | Учтены ли метрики, высота раскладки и область `Viewport` |
| После возврата в меню текст пропадает | Не освобождаются ли шрифты в `hide()` или отдельно от их владельца |
| Память растёт при изменении размеров | Не создаётся ли бесконечное число вариантов в кэше |

В качестве упражнения добавьте надпись «Пауза» в `PlayScreen` из главы 4. Заранее определите владельца шрифта: если экран создаёт собственный `FontGenerator`, он же должен его освобождать; если шрифт общий для меню и игры, перенесите владельца в `MyGame` и освобождайте после обоих экранов.
