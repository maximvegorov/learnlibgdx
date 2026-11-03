# Глава 6 Камера в 2D-игре

## Оглавление главы

- [Координаты игрового мира и экрана](#координаты-игрового-мира-и-экрана)
- [Камера и Viewport](#камера-и-viewport)
- [Следование за игроком](#следование-за-игроком)
- [Зум камеры](#зум-камеры)
- [Следование со смещением и стили камеры](#следование-со-смещением-и-стили-камеры)

В третьей главе камера показывала всю небольшую сцену. Теперь сделаем уровень больше окна и позволим игроку перемещаться по нему. Камера будет выбирать, какую часть уровня показать, насколько приблизить изображение и сколько пространства оставить перед персонажем.

Продолжим проект `chapter-01/first-game`. Заменим только `PlayScreen.java` в пакете `ru.learnlibgdx.firstgame` модуля `core`. `FirstGame` и загрузку ресурсов оставим из главы 4, меню — из главы 5. По-прежнему нужны области `marker` и `white` в загруженном атласе. Новых зависимостей и ресурсов эта глава не требует.

## Координаты игрового мира и экрана

### Уровень существует независимо от окна

**Мировые координаты** описывают положение объектов внутри игры. В нашем примере уровень занимает прямоугольник от `(0, 0)` до `(3200, 1800)`. Ось X направлена вправо, ось Y — вверх. Игрок имеет размер `64 × 64` мировых единицы, а его позиция хранит центр изображения.

Единицу мира выбирает разработчик. Можно считать одну клетку одной единицей или использовать более крупные числа, как в этой книге. Размер PNG не определяет размер объекта в мире: вызов `batch.draw(region, x, y, 64f, 64f)` задаёт размеры изображения при рисовании независимо от исходного разрешения текстуры.

Размер видимой области при обычном масштабе будет `800 × 450` мировых единиц. Если камера стоит в точке `(1600, 900)`, она видит участок:

```text
X: от 1200 до 2000
Y: от  675 до 1125
```

Эти числа рассчитаны для камеры без поворота и с `zoom = 1`. Перемещение камеры не меняет координаты игрока, препятствий и границ уровня. Оно меняет их положение на экране.

### Координаты ввода

У `Gdx.input.getX()` и `Gdx.input.getY()` начало координат находится в левом верхнем углу окна, Y направлен вниз. Это другая система координат, чем выбранная нами система мира. Кроме того, камера может быть смещена, а вокруг изображения могут быть полосы. Поэтому одного переворота Y недостаточно.

Для перевода координат мыши в мир используйте `viewport.unproject()`. Метод изменяет переданный вектор. Обратное преобразование `viewport.project()` возвращает экранные координаты с Y вверх; их нельзя без преобразования сравнивать с координатами ввода. Различия систем описаны в [документации Coordinate systems](https://libgdx.com/wiki/articles/coordinate-systems).

Например, этот фрагмент можно выполнять после обновления камеры. Поле `mouseWorld` типа `com.badlogic.gdx.math.Vector2` создайте один раз:

```java
private final Vector2 mouseWorld = new Vector2();

// В обработке ввода, после применения актуального состояния камеры:
if (Gdx.input.justTouched()) {
    mouseWorld.set(Gdx.input.getX(), Gdx.input.getY());
    viewport.unproject(mouseWorld);
    Gdx.app.log("World", mouseWorld.toString());
}
```

Это дополнительный фрагмент для эксперимента, в полный класс ниже он не включён. Вызывайте его после `viewport.apply()` в `draw()`: тогда преобразование соответствует изображению текущего кадра. Здесь камера смотрит на плоскость XY без наклона.

`unproject()` не отбрасывает нажатия на полосы вокруг игрового изображения. Для игровых действий сначала проверьте, что указатель попадает в прямоугольник viewport. Его положение задаётся относительно нижнего края окна, поэтому верхнюю границу для координат ввода вычисляем отдельно:

```java
int x = Gdx.input.getX();
int y = Gdx.input.getY();
int top = Gdx.graphics.getHeight()
        - viewport.getScreenY() - viewport.getScreenHeight();
boolean inside = x >= viewport.getScreenX()
        && x < viewport.getScreenX() + viewport.getScreenWidth()
        && y >= top
        && y < top + viewport.getScreenHeight();

if (inside) {
    mouseWorld.set(x, y);
    viewport.unproject(mouseWorld);
}
```

Используйте размеры окна в той же системе, что и ввод; не подменяйте их произвольно размерами framebuffer на HiDPI-дисплее. `Viewport` применяет экранный прямоугольник через `HdpiUtils`, а преобразования координат учитывают сохранённые границы viewport. Это видно в [реализации Viewport](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/utils/viewport/Viewport.java).

## Камера и Viewport

### Два разных назначения

`OrthographicCamera` задаёт положение и ортографическую проекцию. В такой проекции размер изображения не уменьшается из-за удаления объекта по глубине, как в перспективе. Для нашей плоской сцены все объекты рисуются на XY, камера не вращается.

`Viewport` определяет, как область камеры разместить внутри окна при разных размерах и пропорциях. Его `worldWidth` и `worldHeight` — размеры области просмотра до учёта зума, а не размеры всего уровня. Основные свойства камеры разобраны в [руководстве Orthographic camera](https://libgdx.com/wiki/graphics/2d/orthographic-camera).

Для уровня `3200 × 1800` создадим:

```java
camera = new OrthographicCamera();
viewport = new FitViewport(800f, 450f, camera);
camera.position.set(1600f, 900f, 0f);
```

Стратегию масштабирования выбирают под правила игры:

| Viewport | Поведение при изменении пропорций окна |
| --- | --- |
| `FitViewport` | Сохраняет пропорции и всю заданную область; свободное место остаётся полосами |
| `FillViewport` | Сохраняет пропорции и заполняет окно; часть заданной области может оказаться за его краями |
| `StretchViewport` | Заполняет окно растяжением; пропорции объектов могут исказиться |
| `ExtendViewport` | Сохраняет пропорции и расширяет видимую область по одной оси; можно задать максимальные размеры |
| `ScreenViewport` | При стандартном `unitsPerPixel = 1` размер области мира соответствует размеру окна в экранных единицах |

В этой главе используем `FitViewport`, чтобы изменение окна само по себе не давало игроку больший обзор. Описание стратегий есть в [официальном руководстве Viewports](https://libgdx.com/wiki/graphics/viewports).

### Изменение окна и порядок отрисовки

В `resize()` передадим новые размеры viewport:

```java
@Override
public void resize(int width, int height) {
    if (width > 0 && height > 0) {
        viewport.update(width, height, false);
    }
}
```

Третий аргумент `false` сохраняет позицию камеры. В меню мы передавали `true`, чтобы поставить камеру в центр области интерфейса. Для следования за игроком такое центрирование после изменения окна было бы нежелательным.

Порядок одного кадра:

1. Обработать управление и обновить позицию игрока.
2. Обновить зум, позицию камеры и ограничения по границам уровня.
3. Вызвать `viewport.apply()` и передать `camera.combined` в `SpriteBatch`.
4. Нарисовать мир в мировых координатах.

`viewport.apply()` устанавливает экранную область и вызывает обновление камеры. Если между изменением камеры и `apply()` нужно выполнить `unproject()`, сначала вызовите `camera.update()` самостоятельно. Матрицы должны соответствовать текущим `position` и `zoom`.

Не вычитайте позицию камеры из координат каждого спрайта вручную: это преобразование уже выполняет матрица `camera.combined`.

## Следование за игроком

### Центр игрока и центр кадра

Простейшее следование — присваивание позиции игрока камере:

```java
camera.position.set(player.x, player.y, 0f);
```

Здесь `player` хранит **центр**. При рисовании от него вычитаем половину размера изображения. Если в вашей игре хранится левый нижний угол, целью камеры станет `(playerX + width / 2, playerY + height / 2)`.

Камера обновляется после движения игрока, иначе изображение будет отставать на один кадр даже при точном следовании.

### Ограничение краями уровня

Если просто поставить камеру на персонажа у левой границы, половина кадра окажется за пределами карты. Для камеры без поворота половины видимой ширины и высоты равны:

```java
float halfWidth = camera.viewportWidth * camera.zoom / 2f;
float halfHeight = camera.viewportHeight * camera.zoom / 2f;
```

При уровне шире кадра центр камеры по X должен находиться между `halfWidth` и `LEVEL_WIDTH - halfWidth`. По Y действуют такие же ограничения. У края игрок перестаёт находиться в центре экрана — это ожидаемый результат.

Если кадр шире уровня, выполнить оба ограничения невозможно. В примере ниже камера в таком случае ставится в центр уровня. Пространство за картой остаётся видимым. Другой вариант — запретить слишком сильное отдаление.

### Полный игровой экран

Класс сразу содержит четыре режима следования и зум; их алгоритмы разберём в следующих разделах. Начальный режим — точное следование.

- WASD или стрелки — движение.
- Q — приблизить, E — отдалить, R — вернуть масштаб `1`.
- Цифры 1–4 в верхнем ряду — сменить стиль камеры.
- Пробел — пауза, Escape — меню.

Замените `PlayScreen.java`:

```java
package ru.learnlibgdx.firstgame;

import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.Input;
import com.badlogic.gdx.ScreenAdapter;
import com.badlogic.gdx.graphics.OrthographicCamera;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.graphics.g2d.TextureAtlas;
import com.badlogic.gdx.graphics.g2d.TextureRegion;
import com.badlogic.gdx.math.MathUtils;
import com.badlogic.gdx.math.Vector2;
import com.badlogic.gdx.utils.ScreenUtils;
import com.badlogic.gdx.utils.viewport.FitViewport;

public class PlayScreen extends ScreenAdapter {
    private static final float VIEW_WIDTH = 800f;
    private static final float VIEW_HEIGHT = 450f;
    private static final float LEVEL_WIDTH = 3200f;
    private static final float LEVEL_HEIGHT = 1800f;
    private static final float PLAYER_SIZE = 64f;
    private static final float SPEED = 300f;
    private static final float FOLLOW_RATE = 6f;
    private static final float LOOK_AHEAD = 180f;
    private static final float MIN_ZOOM = 0.5f;
    private static final float MAX_ZOOM = 2f;

    private enum CameraStyle { LOCKED, SMOOTH, LOOK_AHEAD, DEAD_ZONE }

    private final FirstGame game;
    private final OrthographicCamera camera = new OrthographicCamera();
    private final FitViewport viewport =
            new FitViewport(VIEW_WIDTH, VIEW_HEIGHT, camera);
    private final SpriteBatch batch = new SpriteBatch();
    private final TextureRegion markerRegion;
    private final TextureRegion whiteRegion;
    private final Vector2 player =
            new Vector2(LEVEL_WIDTH / 2f, LEVEL_HEIGHT / 2f);
    private final Vector2 movement = new Vector2();

    private CameraStyle style = CameraStyle.LOCKED;
    private float facingX = 1f;
    private boolean paused;

    public PlayScreen(FirstGame game, TextureAtlas atlas) {
        this.game = game;
        markerRegion = requireRegion(atlas, "marker");
        whiteRegion = requireRegion(atlas, "white");
        camera.position.set(player.x, player.y, 0f);
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
        Gdx.app.log("Play", "WASD/arrows: move; Q/E/R: zoom; "
                + "1-4: camera; Space: pause; Escape: menu");
    }

    @Override
    public void resize(int width, int height) {
        if (width > 0 && height > 0) {
            viewport.update(width, height, false);
            clampCamera();
            camera.update();
        }
    }

    @Override
    public void render(float delta) {
        if (Gdx.input.isKeyJustPressed(Input.Keys.ESCAPE)) {
            game.showMenu();
            return;
        }
        if (Gdx.input.isKeyJustPressed(Input.Keys.SPACE)) {
            paused = !paused;
        }
        selectCameraStyle();
        if (!paused) {
            float dt = Math.min(delta, 0.1f);
            updatePlayer(dt);
            updateZoom(dt);
            updateCamera(dt);
        }
        draw();
    }

    private void selectCameraStyle() {
        if (Gdx.input.isKeyJustPressed(Input.Keys.NUM_1)) style = CameraStyle.LOCKED;
        if (Gdx.input.isKeyJustPressed(Input.Keys.NUM_2)) style = CameraStyle.SMOOTH;
        if (Gdx.input.isKeyJustPressed(Input.Keys.NUM_3)) style = CameraStyle.LOOK_AHEAD;
        if (Gdx.input.isKeyJustPressed(Input.Keys.NUM_4)) style = CameraStyle.DEAD_ZONE;
    }

    private void updatePlayer(float dt) {
        movement.set(0f, 0f);
        if (down(Input.Keys.A, Input.Keys.LEFT)) movement.x -= 1f;
        if (down(Input.Keys.D, Input.Keys.RIGHT)) movement.x += 1f;
        if (down(Input.Keys.S, Input.Keys.DOWN)) movement.y -= 1f;
        if (down(Input.Keys.W, Input.Keys.UP)) movement.y += 1f;
        if (movement.x != 0f) facingX = Math.signum(movement.x);
        movement.nor().scl(SPEED * dt);
        player.add(movement);
        float half = PLAYER_SIZE / 2f;
        player.x = MathUtils.clamp(player.x, half, LEVEL_WIDTH - half);
        player.y = MathUtils.clamp(player.y, half, LEVEL_HEIGHT - half);
    }

    private boolean down(int first, int second) {
        return Gdx.input.isKeyPressed(first) || Gdx.input.isKeyPressed(second);
    }

    private void updateZoom(float dt) {
        float direction = 0f;
        if (Gdx.input.isKeyPressed(Input.Keys.Q)) direction -= 1f;
        if (Gdx.input.isKeyPressed(Input.Keys.E)) direction += 1f;
        camera.zoom *= (float) Math.exp(direction * dt);
        camera.zoom = MathUtils.clamp(camera.zoom, MIN_ZOOM, MAX_ZOOM);
        if (Gdx.input.isKeyJustPressed(Input.Keys.R)) camera.zoom = 1f;
    }

    private void updateCamera(float dt) {
        float targetX = player.x;
        float targetY = player.y;

        if (style == CameraStyle.LOOK_AHEAD) {
            targetX += facingX * LOOK_AHEAD;
        } else if (style == CameraStyle.DEAD_ZONE) {
            float halfZoneX = camera.viewportWidth * camera.zoom * 0.15f;
            float halfZoneY = camera.viewportHeight * camera.zoom * 0.15f;
            targetX = followOutsideZone(player.x, camera.position.x, halfZoneX);
            targetY = followOutsideZone(player.y, camera.position.y, halfZoneY);
        }

        targetX = clampCenter(targetX, LEVEL_WIDTH,
                camera.viewportWidth * camera.zoom / 2f);
        targetY = clampCenter(targetY, LEVEL_HEIGHT,
                camera.viewportHeight * camera.zoom / 2f);

        if (style == CameraStyle.SMOOTH || style == CameraStyle.LOOK_AHEAD) {
            float alpha = 1f - (float) Math.exp(-FOLLOW_RATE * dt);
            camera.position.x += (targetX - camera.position.x) * alpha;
            camera.position.y += (targetY - camera.position.y) * alpha;
        } else {
            camera.position.set(targetX, targetY, 0f);
        }
        clampCamera();
    }

    private static float followOutsideZone(float playerCoordinate,
                                           float cameraCoordinate,
                                           float halfZone) {
        if (playerCoordinate < cameraCoordinate - halfZone) {
            return playerCoordinate + halfZone;
        }
        if (playerCoordinate > cameraCoordinate + halfZone) {
            return playerCoordinate - halfZone;
        }
        return cameraCoordinate;
    }

    private static float clampCenter(float value, float levelSize, float halfView) {
        if (halfView * 2f >= levelSize) return levelSize / 2f;
        return MathUtils.clamp(value, halfView, levelSize - halfView);
    }

    private void clampCamera() {
        camera.position.x = clampCenter(camera.position.x, LEVEL_WIDTH,
                camera.viewportWidth * camera.zoom / 2f);
        camera.position.y = clampCenter(camera.position.y, LEVEL_HEIGHT,
                camera.viewportHeight * camera.zoom / 2f);
    }

    private void draw() {
        ScreenUtils.clear(0f, 0f, 0f, 1f);
        viewport.apply();
        batch.setProjectionMatrix(camera.combined);
        batch.begin();
        batch.setColor(0.10f, 0.15f, 0.25f, 1f);
        batch.draw(whiteRegion, 0f, 0f, LEVEL_WIDTH, LEVEL_HEIGHT);
        batch.setColor(0.20f, 0.28f, 0.38f, 1f);
        for (int x = 0; x <= LEVEL_WIDTH; x += 100) {
            batch.draw(whiteRegion, x, 0f, 2f, LEVEL_HEIGHT);
        }
        for (int y = 0; y <= LEVEL_HEIGHT; y += 100) {
            batch.draw(whiteRegion, 0f, y, LEVEL_WIDTH, 2f);
        }
        batch.setColor(1f, 0.65f, 0.2f, 1f);
        batch.draw(markerRegion, 0f, 0f, PLAYER_SIZE, PLAYER_SIZE);
        batch.draw(markerRegion, LEVEL_WIDTH - PLAYER_SIZE, 0f,
                PLAYER_SIZE, PLAYER_SIZE);
        batch.draw(markerRegion, 0f, LEVEL_HEIGHT - PLAYER_SIZE,
                PLAYER_SIZE, PLAYER_SIZE);
        batch.draw(markerRegion, LEVEL_WIDTH - PLAYER_SIZE,
                LEVEL_HEIGHT - PLAYER_SIZE, PLAYER_SIZE, PLAYER_SIZE);
        batch.setColor(1f, 1f, 1f, 1f);
        batch.draw(markerRegion, player.x - PLAYER_SIZE / 2f,
                player.y - PLAYER_SIZE / 2f, PLAYER_SIZE, PLAYER_SIZE);
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

Сетка и четыре угловых маркера служат ориентирами: по ним видно движение камеры. Нормализация `movement` сохраняет одинаковую скорость по прямой и по диагонали. Это свободное движение по плоскости без физики и столкновений; проверяются только границы уровня.

Ограничение `dt` до `0.1` секунды защищает учебный пример от большого скачка после задержки кадра, но отбрасывает часть прошедшего времени. Это не замена полноценному управлению шагом симуляции. На паузе останавливаются игрок, слежение и зум. После сворачивания окна продолжение выполняется пробелом.

Экран владеет `SpriteBatch`, а атласом владеет прежний `AssetManager`. Поэтому освобождаем только `batch`. У `OrthographicCamera` и `FitViewport` нет требующих освобождения через `dispose()` собственных графических ресурсов.

## Зум камеры

### Что означает camera.zoom

У ортографической камеры увеличение `zoom` показывает больше мира и делает объекты меньше на экране. Уменьшение `zoom` приближает изображение. При неизменном viewport:

| `zoom` | Видимая область нашего мира | Размер объектов относительно `zoom = 1` |
| --- | --- | --- |
| `0.5` | `400 × 225` | Вдвое крупнее |
| `1` | `800 × 450` | Исходный |
| `2` | `1600 × 900` | Вдвое мельче |

Формула — `viewportWidth * zoom` и `viewportHeight * zoom`. Эти множители используются при построении проекции в [исходном коде OrthographicCamera](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/OrthographicCamera.java). Размер самого уровня и скорость игрока при зуме не меняются.

В нашем `updateZoom()` масштаб изменяется умножением на `exp(direction * dt)`. При удержании E одинаковые интервалы времени увеличивают масштаб в одинаковое число раз; Q выполняет обратное изменение. Это выбранный для примера алгоритм управления, а не обязательное правило LibGDX. Диапазон `[0.5, 2]` не позволяет получить нулевой или отрицательный масштаб.

Изменять `camera.position.z` для такого приближения не нужно: ортографическая проекция не даёт перспективного увеличения. Меняйте `zoom`, затем обновляйте матрицы камеры.

### Зум и границы карты

Зум обновляется **до** следования и ограничения камеры. Например, при `zoom = 2` половина видимой ширины равна `800`, поэтому центр камеры не должен подходить к левому краю ближе этого расстояния. Проверка границ, рассчитанная только для `zoom = 1`, показала бы пространство вне уровня.

В коде ограничивается и целевая позиция, и итоговая позиция после сглаживания. Второе ограничение нужно, если новый масштаб сделал прежнее положение недопустимым. Рядом с границей это может вызвать резкое смещение: удержание кадра внутри уровня здесь важнее плавности.

Для полного запрета выхода кадра за прямоугольный уровень можно ограничить максимальный масштаб значением:

```text
min(LEVEL_WIDTH / camera.viewportWidth,
    LEVEL_HEIGHT / camera.viewportHeight)
```

Формула предполагает камеру без поворота. У нас этот предел равен `4`, а выбранный максимум `2` оставляет запас.

### Интерфейс должен иметь свою камеру

Если нарисовать счёт или панель здоровья той же матрицей, что и мир, интерфейс будет перемещаться и масштабироваться вместе с ним. Для неподвижного HUD заведите отдельные камеру и viewport, например `FitViewport(800, 450)`, с масштабом `1`.

После рисования мира завершите `batch`, примените viewport интерфейса, установите его матрицу и начните новый проход рисования. В `resize()` обновляйте оба viewport: игровой — с `false`, интерфейсный — с `true`. Перед следующим рисованием мира снова применяйте игровой viewport. Меню из главы 5 уже использует независимую камеру, поэтому изменение игрового зума на него не влияет.

## Следование со смещением и стили камеры

Камера определяет, какую информацию игрок получает вовремя. Один и тот же персонаж может требовать разных правил кадрирования в исследовании, прыжке или погоне. Следующие режимы — наши алгоритмы поверх `OrthographicCamera`; готовых режимов с такими именами в её API нет.

### Точное следование — клавиша 1

Камера сразу получает целевую позицию. Игрок остаётся в центре кадра, пока не сработает ограничение уровня. Этот режим удобно использовать для проверки движения и координат. Он также повторяет каждое небольшое перемещение персонажа, что не всегда приятно при тряске или частых прыжках.

### Плавное следование — клавиша 2

Камера за кадр проходит только часть расстояния до цели:

```java
float alpha = 1f - (float) Math.exp(-FOLLOW_RATE * dt);
camera.position.x += (targetX - camera.position.x) * alpha;
camera.position.y += (targetY - camera.position.y) * alpha;
```

Постоянный коэффициент вроде `0.1f` в каждом кадре давал бы разную скорость приближения при разной частоте кадров. Экспоненциальный коэффициент учитывает прошедшее время: для неподвижной цели оставшаяся ошибка после времени `t` равна начальной ошибке, умноженной на `exp(-FOLLOW_RATE * t)`. При `FOLLOW_RATE = 6` за полсекунды исчезает примерно 95% исходного расстояния.

Это точное свойство формулы для неизменной цели. Для движущейся цели результат также зависит от дискретных обновлений её положения; ограничение `dt` дополнительно меняет соответствие реальному времени. Чем больше `FOLLOW_RATE`, тем быстрее реакция. Маленькое значение создаёт заметное отставание.

При телепортации или появлении игрока обычно стоит сразу установить `camera.position` в новую целевую точку, ограничить её картой и обновить камеру. Иначе она будет долго пролетать через промежуточные участки уровня.

### Взгляд вперёд — клавиша 3

В примере камера стремится к точке на `180` единиц впереди по горизонтали:

```java
targetX = player.x + facingX * LOOK_AHEAD;
targetY = player.y;
```

Когда игрок движется вправо, камера смещается вправо, поэтому игрок оказывается левее центра и видит больше пространства перед собой. `facingX` запоминает последнее ненулевое направление горизонтального ввода. При остановке смещение сохраняется; при развороте меняется цель, а камера плавно переходит к ней.

Для постоянного вертикального смещения можно задать `targetY = player.y + 80f`: тогда над персонажем будет больше обзора. Это изменение цели в `updateCamera()`, перед её ограничением границами.

В другом варианте величина упреждения зависит от скорости: `offsetX = velocityX * lookAheadSeconds`. Например, при скорости `300` единиц в секунду и времени `0.6` секунды получится смещение `180`. Для такого режима берите фактическую скорость и ограничивайте максимальное смещение. Наш класс использует направление ввода, поэтому у стены продолжает смотреть в сторону нажатой клавиши.

Смещение в примере измеряется в мировых единицах. При отдалении оно занимает меньшую долю экрана. Если требуется постоянная доля кадра, вычисляйте его через видимую ширину, например `camera.viewportWidth * camera.zoom * 0.2f`.

### Мёртвая зона — клавиша 4

**Мёртвая зона** — прямоугольник вокруг центра кадра, внутри которого движение игрока не сдвигает камеру. Выйдя за его границу, игрок сдвигает камеру настолько, чтобы снова оказаться на границе зоны.

В нашем режиме полуширина зоны составляет 15% видимой ширины, полувысота — 15% видимой высоты. Значит, вся зона занимает центральные 30% кадра по каждой оси. При `zoom = 1` это прямоугольник `240 × 135` мировых единиц. Проверяем центр игрока, а не края спрайта.

Метод `followOutsideZone()` отдельно обрабатывает обе оси. Пока координата игрока внутри допустимого интервала, он возвращает прежнюю координату камеры. Справа от зоны новая позиция камеры равна `playerCoordinate - halfZone`, слева — `playerCoordinate + halfZone`.

Этот режим в примере не использует сглаживание: граница зоны соблюдается сразу, если ограничения карты позволяют. Если сглаживать движение к полученной цели, персонаж временно сможет находиться за пределами зоны. Такой вариант допустим, но это уже другое поведение.

### Как выбрать поведение

| Задача сцены | Возможная отправная точка |
| --- | --- |
| Свободное движение с видом сверху | Точное или плавное следование по двум осям |
| Платформер с частыми короткими прыжками | Горизонтальное упреждение и более высокая мёртвая зона по Y |
| Комната, целиком помещающаяся в кадре | Фиксированная камера в центре комнаты |
| Быстрое движение вперёд | Смещение по направлению и скорости движения |
| Сцена с несколькими важными объектами | Цель в центре их общего прямоугольника и зум, вмещающий его с отступом |

Это варианты проектирования, а не обязательные жанровые правила. Для платформера можно обновлять вертикальную цель только при приземлении или выходе за заданный порог. Для комнаты — переключать целевую точку после пересечения прохода. Начинайте с одного правила и проверяйте, успевает ли игрок заметить препятствия.

### Самостоятельная проверка

Запустите проект командой `./gradlew lwjgl3:run` из каталога `first-game` или `.\gradlew.bat lwjgl3:run` в Windows. Нажмите Enter в меню.

1. В режиме 1 пройдите к каждой границе уровня. Камера должна остановиться раньше игрока, а игрок — остаться целиком внутри карты.
2. В центре уровня нажмите 2, пробегите по диагонали и отпустите управление. Камера должна плавно догнать остановившегося игрока.
3. Нажмите 3 и несколько раз смените направление по X. Сравните количество видимого пространства перед персонажем и за ним.
4. Нажмите 4. Небольшие перемещения внутри центральной зоны не должны двигать сетку; при выходе из зоны сетка начнёт смещаться.
5. Удерживайте Q, затем E. Проверьте пределы зума и нажмите R. Повторите у угла карты: за границами уровня не должно появляться пустое пространство при разрешённом диапазоне масштаба.
6. Измените пропорции окна. Объекты сохранят форму, вокруг изображения могут появиться чёрные полосы, позиция камеры не должна сброситься к началу уровня.
7. Проверьте пробел, возврат в меню и повторный вход. Меню сохраняет свой масштаб; игровой экран сохраняет положение игрока, стиль камеры и зум.

Если камера не движется, проверьте передачу `camera.combined` в `batch`. Если щелчки попадают не туда, проверьте использование именно игрового `viewport.unproject()` и актуальность матриц. Если у края карты появляется пустота только после отдаления, проверьте множитель `camera.zoom` в расчёте границ.
