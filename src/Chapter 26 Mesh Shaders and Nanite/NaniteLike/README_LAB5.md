# Лабораторная работа №5 — Mesh Shaders (NaniteLike)

Инструкция по импорту своей модели в проект **NaniteLike** для 5-й лабы по компьютерной графике.

---

## Отличия от стандартной инструкции (Microsoft D3D12MeshShaders)

В стандартной инструкции используется Microsoft D3D12MeshShaders с **D3D12WavefrontConverter** (OBJ → BIN).  
В проекте **NaniteLike** другая архитектура:

| Microsoft D3D12MeshShaders | NaniteLike (этот проект) |
|---------------------------|---------------------------|
| Конвертация OBJ → BIN через D3D12WavefrontConverter | **OBJ загружается напрямую** во время выполнения |
| Менять имя bin-файла в D3D12MeshletRender.cpp (строка 15) | **Менять путь к OBJ** в NaniteLikeApp.cpp (строка 318) |
| Папка Assets для bin-файлов | Папка **OBJ** для obj-файлов |

**D3D12WavefrontConverter и bin-файлы в этом проекте не используются.**

---

## Пошаговая инструкция

### 1. Подготовка модели в Blender

1. Скачайте модель из интернета и откройте в Blender.
2. Если модель состоит из нескольких объектов — объедините их: **Object Mode → Select All → Object → Join**.
3. Триангулируйте геометрию: **Edit Mode → Face → Triangulate Faces**.
4. Экспортируйте в OBJ:
   - **File → Export → Wavefront (.obj)**
   - Включите: **Selection Only** (если нужно), **Triangulate Faces**, **Include UVs**, **Include Normals**
   - Сохраните как `my_model.obj`

### 2. Размещение файлов в проекте

Создайте папку для своей модели в `OBJ`:

```
NaniteLike/
├── OBJ/
│   └── MyModel/           ← создайте папку с именем модели
│       ├── my_model.obj   ← ваш OBJ-файл
│       └── texture.dds    ← (опционально) текстура в формате DDS
```

**Важно:** путь к файлам задаётся относительно рабочей директории. При сборке папка `OBJ` автоматически копируется в `bin/Debug/`, поэтому модели будут находиться и при запуске из Visual Studio, и при запуске exe напрямую.

### 3. Изменение кода в Visual Studio

Откройте файл **`NaniteLikeApp.cpp`** и измените **две строки**:

#### Строка ~318 — путь к OBJ-модели:

```cpp
// Было:
bool loaded = MeshletBuilder::LoadOBJWithDirectStorage(
    L"OBJ/StGilesLychGate01/StGilesLychGate02.obj", mesh, mStorageLoader.get());

// Стало (пример для my_model.obj):
bool loaded = MeshletBuilder::LoadOBJWithDirectStorage(
    L"OBJ/MyModel/my_model.obj", mesh, mStorageLoader.get());
```

#### Строка ~356 — путь к текстуре (если есть):

```cpp
// Было:
if (mNaniteRenderer->LoadTexture(mCommandList.Get(), L"OBJ/StGilesLychGate01/wssop-5brlm.dds"))

// Стало (если у вас есть текстура):
if (mNaniteRenderer->LoadTexture(mCommandList.Get(), L"OBJ/MyModel/texture.dds"))
```

Если текстуры нет — можно оставить старый путь или закомментировать блок загрузки текстуры. Без текстуры модель будет отображаться в режиме meshlet-цветов или серым цветом.

### 4. Сборка и запуск

1. В Visual Studio: **ПКМ по проекту NaniteLike → Set as Startup Project**
2. **Build → Build Solution** (или F7)
3. **Debug → Start Debugging** (или F5)

Если всё настроено верно, должна отобразиться ваша модель.

---

## Управление в приложении

| Клавиша | Действие |
|---------|----------|
| **WASD** | Движение камеры |
| **Q / E** | Вниз / вверх |
| **Shift** | Ускоренное движение |
| **Мышь** | Поворот камеры |
| **M** | Переключение визуализации meshlet-цветов |
| **T** | Переключение текстуры / meshlet-цветов |

---

## Возможные проблемы

### «OBJ not found, using generated sphere»

- OBJ-файл не найден. Проверьте:
  1. Путь в коде совпадает с реальным расположением файла.
  2. Файл лежит в `NaniteLike/OBJ/...` (относительно папки проекта).
  3. При запуске exe из папки `bin/Debug/` — рабочая директория может быть другой. В таком случае либо запускайте из Visual Studio, либо скопируйте папку `OBJ` в `bin/Debug/`.

### «Кракозябра из треугольников» вместо модели

- Часто связано с некорректной геометрией OBJ:
  1. Убедитесь, что в Blender всё объединено в один объект и триангулировано.
  2. Попробуйте другую модель (желательно изначально в OBJ).
  3. Проверьте, что экспорт из Blender выполнен с опциями **Triangulate Faces**, **Include UVs**, **Include Normals**.

### Текстура не загружается

- Поддерживается только формат **DDS**. Конвертируйте PNG/JPG в DDS (например, через [DirectXTex](https://github.com/microsoft/DirectXTex) или онлайн-конвертеры).

---

## Структура проекта (кратко)

| Файл / папка | Назначение |
|--------------|------------|
| `NaniteLikeApp.cpp` | Точка входа, загрузка модели (строки 318, 356) |
| `MeshletBuilder.cpp` | Загрузка OBJ и построение meshlet-ов |
| `NaniteRenderer.cpp` | Рендеринг через Mesh Shader pipeline |
| `OBJ/` | Папка с OBJ-моделями и текстурами |
| `Shaders/MeshShader.hlsl` | Mesh Shader (AS + MS + PS) |

---

## Итог

Для сдачи лабы в проекте NaniteLike нужно:

1. Подготовить OBJ в Blender (объединить, триангулировать, экспорт).
2. Положить OBJ (и при необходимости DDS-текстуру) в `OBJ/ВашаПапка/`.
3. Изменить путь к OBJ в `NaniteLikeApp.cpp` (строка ~318).
4. При наличии текстуры — изменить путь к DDS (строка ~356).
5. Собрать и запустить проект.

Конвертер D3D12WavefrontConverter и bin-файлы в этом проекте **не используются**.
