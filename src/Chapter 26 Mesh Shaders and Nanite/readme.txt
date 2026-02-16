Ниже — готовый текст README, который можно просто скопировать в файл, например README_LAB5_ORAL.md. Он построен от общего к частному, и заточен под устную сдачу по вашему ТЗ (Mesh Pipeline + “тяжёлый” меш + DirectStorage).
## Лаба №5 (КГ): Mesh Pipeline + “тяжёлый” меш + DirectStorage — конспект для устной сдачи### ТЗ (что надо показать)- **Добавить в рендер поддержку Mesh Pipeline** и отобразить “тяжёлый” меш.- **Добавить загрузку данных и текстур через DirectStorage**.В этом проекте это уже реализовано как:- Mesh Pipeline: **AS/MS/PS** + StructuredBuffers + (GPU) culling.- Тяжёлый меш: подменяем OBJ на свой и (опционально) DDS-текстуру.- DirectStorage: чтение файла OBJ в память через DirectStorage API (с fallback на обычный ifstream).---## Карта проекта (что за что отвечает)### 1) `NaniteLikeApp` — “сценарий” приложения (вход, управление, загрузка ассетов)- **Где задаётся модель и текстура**: `NaniteLikeApp::BuildMeshletMeshes()`.- Там же: загрузка OBJ (через DirectStorage) → построение meshlet’ов → LOD-иерархия → upload на GPU → загрузка DDS.Ключевой фрагмент:```312:369:d:\KUmp Grafics\5\src\Chapter 26 Mesh Shaders and Nanite\NaniteLike\NaniteLikeApp.cppvoid NaniteLikeApp::BuildMeshletMeshes(){    MeshletMesh mesh;    // Use DirectStorage for fast file loading    bool loaded = MeshletBuilder::LoadOBJWithDirectStorage(        L"OBJ/sword/mygreensword.obj", mesh, mStorageLoader.get());    if (!loaded)    {        // fallback sphere        GeometryGenerator geoGen;        auto sphereData = geoGen.CreateGeosphere(30.0f, 5);        mesh.Name = "Sphere";        MeshletBuilder::BuildFromGeometry(sphereData, mesh);    }    MeshletBuilder::BuildLODHierarchy(mesh);    mMeshletMeshes.push_back(std::move(mesh));    // Upload to GPU    if (!mMeshletMeshes.empty())        mNaniteRenderer->UploadMesh(mCommandList.Get(), mMeshletMeshes[0], 0);    // Texture    if (mNaniteRenderer->LoadTexture(mCommandList.Get(),        L"OBJ/sword/NicoNavarroSword_low_BaseColor.dds"))    {        // ...    }}
Связи:
NaniteLikeApp держит:
mStorageLoader → DirectStorage
mMeshletMeshes → CPU-представление меша (meshlets, bounds, LOD)
mNaniteRenderer → GPU upload + draw
Управление:
M → mNaniteRenderer->ToggleMeshletVisualization()
T → mNaniteRenderer->ToggleTexture()
2) DirectStorageLoader — “быстрый ввод-вывод” (обязательная часть ТЗ)
Цель: показать, что вы грузите данные не обычным fopen/ifstream, а через DirectStorage (с очередью, fence и т.д.).
Главная функция для ТЗ (OBJ читается в RAM):
bool DirectStorageLoader::LoadFileToMemory(    const std::wstring& filename,    std::vector<uint8_t>& outData){    if (!mInitialized)    {        // Fallback to standard file loading        std::ifstream file(filename, std::ios::binary | std::ios::ate);        // ...        return true;    }    // Open file with DirectStorage    // Create DSTORAGE_REQUEST -> EnqueueRequest -> EnqueueSignal -> Submit -> Wait fence    // Check error record    return true;}
Что говорить на защите:
DirectStorage делает загрузку эффективнее за счёт очередей, меньше участия CPU, возможной прямой загрузки в GPU-буфер (в проекте есть и LoadFileToGPUBuffer).
В коде есть fallback, если DirectStorage недоступен (это нормально для демо).
3) MeshletBuilder — “превращаем обычный меш в meshlets”
Тут две части:
Парсинг OBJ (обычный / через DirectStorage в память)
Генерация meshlet’ов через DirectX::ComputeMeshlets
Ключевое место, где появляется meshlet pipeline-логика (meshlets, UniqueVertexIndices, PrimitiveIndices):
bool MeshletBuilder::BuildMeshlets(    const std::vector<XMFLOAT3>& positions,    const std::vector<XMFLOAT3>& normals,    const std::vector<XMFLOAT2>& texCoords,    const std::vector<XMFLOAT3>& tangents,    const std::vector<uint32_t>& indices,    MeshletMesh& outMesh){    // save original mesh    outMesh.Positions = positions;    outMesh.Indices = indices;    // meshlet generation (DirectXMesh)    HRESULT hr = DirectX::ComputeMeshlets(        indices.data(),        indices.size() / 3,        positions.data(),        positions.size(),        nullptr,        dxMeshlets,        uniqueVertexIB,        primitiveIndices,        MAX_MESHLET_VERTICES,        MAX_MESHLET_PRIMITIVES);    // convert dxMeshlets -> outMesh.Meshlets    // convert uniqueVertexIB -> outMesh.UniqueVertexIndices    // convert primitiveIndices -> outMesh.PrimitiveIndices    // bounds    BoundingBox::CreateFromPoints(outMesh.BBox, positions.size(),        positions.data(), sizeof(XMFLOAT3));    BoundingSphere::CreateFromBoundingBox(outMesh.BSphere, outMesh.BBox);    return true;}
Связи переменных (важно понимать):
MeshletMesh.Positions/Normals/TexCoords/Tangents — исходные атрибуты вершин.
MeshletMesh.Indices — исходные индексы (нужны для fallback).
MeshletMesh.Meshlets[] — массив meshlet’ов (offset/count вершин и примитивов).
MeshletMesh.UniqueVertexIndices[] — “словарь” индексов вершин для каждого meshlet.
MeshletMesh.PrimitiveIndices[] — треугольники внутри meshlet (локальные индексы, у вас хранятся как uint8, а на GPU конвертятся в uint32 для StructuredBuffer).
MeshletMesh.MeshletBoundsData[] — bounds для culling.
Bounds и culling (нюанс для ответов):
В ComputeMeshletBounds по сути строится сфера (центр+радиус), а cone-поля инициализируются дефолтно (cone culling фактически “заглушен”, потому что ConeCutoff = 1.0):
outBounds.ConeCutoff = 1.0f; // => cone culling off unless you compute it
4) NaniteRenderer — “GPU upload + выбор пайплайна + draw”
Это то место, которое реально “добавляет поддержку Mesh Pipeline” в рендер.
Важные флаги/переменные (на защите часто спрашивают)
mMeshShadersSupported — есть ли поддержка Mesh Shader на GPU (Tier1+).
mUseMeshShaders — включён ли mesh pipeline или fallback.
mShowMeshletColors — режим визуализации meshlet-ов.
mUseTexture — использовать DDS-текстуру или цвета meshlet-ов.
mTotalMeshlets/mTotalTriangles/... — статистика по загруженному мешу.
Логика выбора пайплайна
void NaniteRenderer::Render(...){    if (mTotalMeshlets == 0) return;    if (mUseMeshShaders && mMeshShaderPSO)        RenderMeshShader(...);    else        RenderFallback(...);}
Upload данных на GPU (StructuredBuffers под MS)
В UploadMesh вы создаёте GPU-буферы:
mMSVertexBuffer (StructuredBuffer<Vertex>)
mMeshletBuffer (StructuredBuffer<Meshlet>)
mMeshletBoundsBuffer (StructuredBuffer<MeshletBounds>)
mUniqueVertexIndicesBuffer
mPrimitiveIndicesBuffer
mInstanceBuffer
(см. UploadMesh — в консоли печатает, сколько вершин/meshlet’ов загрузили).
Шейдеры (что именно является “Mesh Pipeline”)
Файл: Shaders/MeshShader.hlsl
Как данные приходят в шейдер
StructuredBuffers:
Vertices (t0), Meshlets (t1), MeshletBoundsBuffer (t2),
UniqueVertexIndices (t3), PrimitiveIndices (t4), Instances (t5),
gDiffuseMap (t6)
Pass constants b0 содержат:
матрицы gView/gProj/gViewProj,
gFrustumPlanes[6] (для culling),
gMeshletCount, gShowMeshletColors, gUseTexture, …
Где делается culling
В Amplification Shader: тест сферы meshlet-а против 6 плоскостей frustum + опционально cone culling.
Далее — компактируем видимые meshlet-ы и вызываем DispatchMesh только для них.
[numthreads(AS_GROUP_SIZE, 1, 1)]void ASMain(...){    // meshletIndex = gid * AS_GROUP_SIZE + gtid    // frustum test against gFrustumPlanes    // if visible -> atomic push into sharedPayload    DispatchMesh(sharedVisibleCount, 1, 1, sharedPayload);}
Что сказать: “Я делаю GPU culling в AS, и только видимые meshlet-ы отправляю в MS”.
“Тяжёлый меш” — что нужно для зачёта
Что реально требуется
OBJ должен содержать геометрию (строки v и f), желательно триангулированную.
Текстура — DDS (в проекте используется CreateDDSTextureFromFile12).
Где указать пути
В NaniteLikeApp.cpp в BuildMeshletMeshes():
путь к OBJ (строка с LoadOBJWithDirectStorage)
путь к DDS (строка с LoadTexture)
Типовые вопросы на защите (и короткие ответы)
1) “Что такое Mesh Pipeline и чем отличается от VS/PS?”
Mesh Pipeline заменяет классическую обработку вершин на обработку “пачек” геометрии (meshlets):
AS делает отбор (culling/распределение),
MS генерирует вершины и треугольники для растеризации.
Это снижает overhead и помогает с огромными мешами.
2) “Что такое meshlet?”
Маленький кластер треугольников с ограничениями железа.
У вас лимиты: 64 вершины и 124 треугольника (MAX_MESHLET_VERTICES/PRIMITIVES) — они совпадают с тем, что ожидает HLSL.
3) “Где именно вы добавили поддержку Mesh Pipeline?”
В NaniteRenderer:
проверка поддержки mesh shaders,
создание root signature/PSO под AS/MS/PS,
загрузка данных в StructuredBuffer’ы,
Render() выбирает RenderMeshShader().
4) “Где именно используется DirectStorage?”
В MeshletBuilder::LoadOBJWithDirectStorage() → DirectStorageLoader::LoadFileToMemory():
OBJ читается через DirectStorage (очередь + fence), дальше парсится из памяти.
5) “Как делается culling?”
CPU извлекает FrustumPlanes из ViewProj (ExtractFrustumPlanes).
AS в шейдере проверяет sphere bounds meshlet-а относительно 6 плоскостей.
Видимые meshlet-ы компактим и вызываем DispatchMesh только для них.
6) “Текстура как грузится и где применяется?”
DDS грузится в NaniteRenderer::LoadTexture, в шейдере используется gDiffuseMap.
Переключение: T → ToggleTexture().
7) “Что будет, если DirectStorage недоступен?”
Есть fallback на обычный ifstream (в LoadFileToMemory).
Мини-чеклист перед сдачей
Запускается проект, в окне отображается ваш тяжёлый меш.
В консоли видно статистику (meshlets/triangles/visible meshlets).
Можно показать переключение режимов:
M meshlet colors
T texture on/off
OBJ читается через LoadOBJWithDirectStorage (можно прямо ткнуть в код).
Если хочешь, я могу дополнить README под твой формат устной сдачи: “вопрос → где в коде → что сказать”, но я в Ask-режиме не могу править файлы — только дать текст и ссылки на места в коде.
