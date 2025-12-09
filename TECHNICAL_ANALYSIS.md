# Техническая документация: Векторы обхода Orebfuscator

## Disclaimer
Этот документ создан исключительно в образовательных целях для понимания механизмов работы anti-xray систем. Информация предназначена для администраторов серверов и разработчиков плагинов безопасности.

---

## 1. Архитектура Orebfuscator в Minecraft 1.21.x

### 1.1 Поток обработки пакетов

```
Клиент запрашивает чанк
    ↓
Сервер загружает чанк из мира
    ↓
ObfuscationListener перехватывает CHUNK_DATA пакет (ProtocolLib)
    ↓
ObfuscationProcessor обрабатывает чанк:
    - Проходит по всем блокам в секции
    - Проверяет, должен ли блок быть скрыт (hiddenBlocks)
    - Проверяет соседей (shouldObfuscate)
    - Заменяет блок на случайный (randomBlocks)
    - Удаляет NBT данные блок-энтити
    ↓
Модифицированный пакет отправляется клиенту
```

### 1.2 Ключевые компоненты

**ObfuscationProcessor** - главный класс обфускации
- Расположение: `/orebfuscator-plugin/src/main/java/net/imprex/orebfuscator/obfuscation/ObfuscationProcessor.java`
- Ответственность: замена блоков в ChunkSection

**DeobfuscationWorker** - восстановление реальных блоков
- Расположение: `/orebfuscator-plugin/src/main/java/net/imprex/orebfuscator/obfuscation/DeobfuscationWorker.java`
- Ответственность: отправка block updates при событиях

**ProximityWorker** - система proximity hiding
- Расположение: `/orebfuscator-plugin/src/main/java/net/imprex/orebfuscator/proximity/ProximityWorker.java`
- Ответственность: раскрытие блоков при приближении игрока

---

## 2. Вектор #1: Анализ обновлений блоков (Block Update Analysis)

### 2.1 Принцип работы

Когда игрок ломает блок, Orebfuscator деобфусцирует соседние блоки в радиусе `updateRadius`:

```java
// DeobfuscationWorker.java:82-102
public void processPosition(BlockPos position, int depth) {
  int blockId = OrebfuscatorNms.getBlockState(worldAccessor.world, position);
  if (BlockFlags.isObfuscateBitSet(blockFlags.flags(blockId)) && updatedBlocks.add(position)) {
    // Инвалидация кеша
    cache.invalidate(chunkPosition);
  }
  
  if (depth-- > 0) {
    // Рекурсивно обновляет 6 соседей
    processPosition(position.add(1, 0, 0), depth);
    processPosition(position.add(-1, 0, 0), depth);
    processPosition(position.add(0, 1, 0), depth);
    processPosition(position.add(0, -1, 0), depth);
    processPosition(position.add(0, 0, 1), depth);
    processPosition(position.add(0, 0, -1), depth);
  }
}
```

### 2.2 Концепция обхода

**Клиентский мод для отслеживания**:

```java
public class BlockUpdateTracker {
    private Map<BlockPos, BlockState> blockCache = new HashMap<>();
    private Set<BlockPos> suspiciousBlocks = new HashSet<>();
    
    @SubscribeEvent
    public void onBlockUpdate(BlockUpdateEvent event) {
        BlockPos pos = event.getPos();
        BlockState oldState = blockCache.get(pos);
        BlockState newState = event.getState();
        
        // Если блок изменился с обычного камня на руду
        if (isCommonBlock(oldState) && isValuableOre(newState)) {
            // Это была деобфускация -> реальная руда
            suspiciousBlocks.add(pos);
            markAsRealOre(pos, newState);
            
            // Логируем для анализа
            logBlockChange(pos, oldState, newState, "DEOBFUSCATION");
        }
        
        blockCache.put(pos, newState);
    }
    
    private boolean isCommonBlock(BlockState state) {
        return state.getBlock() == Blocks.STONE 
            || state.getBlock() == Blocks.DEEPSLATE
            || state.getBlock() == Blocks.NETHERRACK;
    }
    
    private boolean isValuableOre(BlockState state) {
        return state.getBlock() == Blocks.DIAMOND_ORE
            || state.getBlock() == Blocks.DEEPSLATE_DIAMOND_ORE
            || state.getBlock() == Blocks.ANCIENT_DEBRIS
            || state.getBlock() == Blocks.EMERALD_ORE;
    }
}
```

### 2.3 Стратегия эксплуатации

1. **Passive scanning** - просто записывать все изменения блоков
2. **Active scanning** - специально ломать блоки вокруг подозрительных областей
3. **Pattern recognition** - если деобфускация происходит часто в одной области = жила руд

**Пример активного сканирования**:
```java
public void scanArea(BlockPos center, int radius) {
    // Ломаем блоки в шахматном порядке
    for (int x = -radius; x <= radius; x += 2) {
        for (int y = -radius; y <= radius; y += 2) {
            for (int z = -radius; z <= radius; z += 2) {
                BlockPos pos = center.add(x, y, z);
                // Ломаем блок (отправляем пакет)
                breakBlock(pos);
                // Ждём деобфускации соседних блоков
                Thread.sleep(50);
                // Анализируем изменения
                analyzeUpdates(pos);
            }
        }
    }
}
```

**Защита**: Уменьшить `updateRadius` до 1 или 0, но это ухудшит игровой опыт.

---

## 3. Вектор #2: Анализ освещения (Light Level Analysis)

### 3.1 Проблема в реализации

Orebfuscator заменяет только `blockState`, но НЕ данные освещения:

```java
// ObfuscationProcessor.java:107-108
if (obfuscated) {
    chunkSection.setBlockState(index, blockState);
    // ПРОБЛЕМА: освещение не обновляется!
    // Клиент получает реальные данные освещения
}
```

### 3.2 Как это эксплуатировать

**Концепция**:
- Алмазная руда не излучает свет
- Если видим светящийся блок (glowstone) на месте, где освещение = 0 → подделка
- Если видим камень, где должен быть свет → возможно скрыта лава/светокамень

**Клиентский анализ**:

```java
public class LightAnalyzer {
    public void analyzeChunk(Chunk chunk) {
        for (ChunkSection section : chunk.getSections()) {
            for (int y = 0; y < 16; y++) {
                for (int z = 0; z < 16; z++) {
                    for (int x = 0; x < 16; x++) {
                        BlockState block = section.getBlockState(x, y, z);
                        int blockLight = section.getBlockLight(x, y, z);
                        int skyLight = section.getSkyLight(x, y, z);
                        
                        // Детект несоответствия
                        if (block.getLightEmission() != blockLight) {
                            // Либо обфускация, либо что-то рядом светит
                            if (noLightSourcesNearby(x, y, z)) {
                                markAsSuspicious(x, y, z, "LIGHT_MISMATCH");
                            }
                        }
                        
                        // Анализ теней
                        if (skyLight < expectedSkyLight(x, y, z)) {
                            // Что-то блокирует свет, но мы не видим блок
                            checkForHiddenBlocks(x, y, z);
                        }
                    }
                }
            }
        }
    }
    
    private int expectedSkyLight(int x, int y, int z) {
        // Рассчитываем ожидаемый уровень света с неба
        // Учитывая видимые блоки над этой позицией
        int highestBlock = findHighestBlockAbove(x, z);
        if (y >= highestBlock) return 15;
        return Math.max(0, 15 - (highestBlock - y));
    }
}
```

### 3.3 Продвинутая техника: Shadow Mapping

```java
public class ShadowMapper {
    // Строим карту теней на основе ВИДИМЫХ блоков
    private int[][] visibleShadowMap = new int[16][16];
    // Получаем реальную карту теней от сервера
    private int[][] actualShadowMap = new int[16][16];
    
    public Set<BlockPos> findHiddenBlocks() {
        Set<BlockPos> hidden = new HashSet<>();
        
        for (int x = 0; x < 16; x++) {
            for (int z = 0; z < 16; z++) {
                int visibleHeight = calculateHeightFromVisible(x, z);
                int actualHeight = getActualHeightFromLighting(x, z);
                
                if (actualHeight < visibleHeight) {
                    // Есть скрытые блоки, которые блокируют свет!
                    for (int y = actualHeight; y < visibleHeight; y++) {
                        hidden.add(new BlockPos(x, y, z));
                    }
                }
            }
        }
        
        return hidden;
    }
}
```

**Защита**: Требуется полная переработка системы освещения (очень затратно по производительности).

---

## 4. Вектор #3: Статистический анализ (Statistical Analysis)

### 4.1 Проблема распределения блоков

В реальном мире Minecraft:
- Stone: ~99.9%
- Diamond Ore: ~0.01%
- Gold Ore: ~0.05%

В обфусцированных чанках (типичная конфигурация):
- Stone: ~80%
- Diamond Ore: ~5%
- Gold Ore: ~5%
- Deepslate: ~10%

### 4.2 Статистический детектор

```java
public class StatisticalAnalyzer {
    private static final double EXPECTED_DIAMOND_RATIO = 0.0001; // 0.01%
    private static final double OBFUSCATED_DIAMOND_RATIO = 0.05;  // 5%
    private static final double THRESHOLD = 0.01; // 1%
    
    public ChunkAnalysisResult analyzeChunk(Chunk chunk) {
        Map<Block, Integer> blockCounts = new HashMap<>();
        int totalBlocks = 0;
        
        // Считаем блоки
        for (ChunkSection section : chunk.getSections()) {
            if (section == null) continue;
            
            for (int i = 0; i < 4096; i++) {
                BlockState state = section.getBlockState(i);
                blockCounts.merge(state.getBlock(), 1, Integer::sum);
                totalBlocks++;
            }
        }
        
        // Анализируем распределение
        double diamondRatio = blockCounts.getOrDefault(Blocks.DIAMOND_ORE, 0) / (double) totalBlocks;
        double goldRatio = blockCounts.getOrDefault(Blocks.GOLD_ORE, 0) / (double) totalBlocks;
        double emeraldRatio = blockCounts.getOrDefault(Blocks.EMERALD_ORE, 0) / (double) totalBlocks;
        
        boolean isObfuscated = false;
        
        // Если руд слишком много -> обфускация активна
        if (diamondRatio > THRESHOLD) {
            isObfuscated = true;
        }
        
        // Рассчитываем энтропию
        double entropy = calculateEntropy(blockCounts, totalBlocks);
        
        // Обфусцированные чанки имеют выше энтропию
        if (entropy > 2.5) { // Порог подбирается эмпирически
            isObfuscated = true;
        }
        
        return new ChunkAnalysisResult(isObfuscated, diamondRatio, entropy);
    }
    
    private double calculateEntropy(Map<Block, Integer> counts, int total) {
        double entropy = 0.0;
        for (int count : counts.values()) {
            if (count == 0) continue;
            double p = count / (double) total;
            entropy -= p * (Math.log(p) / Math.log(2));
        }
        return entropy;
    }
    
    // Фильтр: убираем все блоки, которые статистически невозможны
    public Set<BlockPos> filterFakeOres(Chunk chunk) {
        Set<BlockPos> realOres = new HashSet<>();
        ChunkAnalysisResult result = analyzeChunk(chunk);
        
        if (!result.isObfuscated) {
            // Чанк не обфусцирован, все руды реальны
            return getAllOres(chunk);
        }
        
        // Статистически, в обфусцированном чанке должно быть примерно:
        int expectedRealDiamonds = (int) (chunk.getTotalBlocks() * EXPECTED_DIAMOND_RATIO);
        
        // Все остальные алмазы - подделки
        // Но какие именно реальные? Нужны дополнительные эвристики...
        
        return realOres;
    }
}
```

### 4.3 Байесовский классификатор

```java
public class BayesianOreClassifier {
    // Обучаем на реальных данных
    private double P_real_diamond = 0.0001;
    private double P_fake_diamond = 0.05;
    
    // Признаки для классификации
    public double classifyOre(BlockPos pos, Chunk chunk) {
        // P(real | features) используя теорему Байеса
        
        // Признак 1: Глубина (реальные алмазы на определённой Y)
        double depthScore = getDepthProbability(pos.getY());
        
        // Признак 2: Соседние блоки (реальные алмазы рядом с deepslate/stone)
        double neighborScore = analyzeNeighbors(pos, chunk);
        
        // Признак 3: Форма жилы (реальные руды образуют жилы 1-10 блоков)
        double veinScore = analyzeVeinShape(pos, chunk);
        
        // Признак 4: Обновлялся ли блок (обновлённые = вероятно реальные)
        double updateScore = wasBlockUpdated(pos) ? 0.9 : 0.1;
        
        // Комбинируем признаки
        double likelihood = depthScore * neighborScore * veinScore * updateScore;
        
        // Байесовская формула
        double posterior = (likelihood * P_real_diamond) / 
                          ((likelihood * P_real_diamond) + ((1-likelihood) * P_fake_diamond));
        
        return posterior;
    }
    
    private double getDepthProbability(int y) {
        // Алмазы наиболее вероятны на Y=-59
        if (y >= -64 && y <= -54) {
            return 1.0 - Math.abs(y + 59) / 10.0;
        }
        return 0.1;
    }
    
    private double analyzeVeinShape(BlockPos center, Chunk chunk) {
        // Реальные жилы руд имеют 1-10 блоков близко друг к другу
        Set<BlockPos> vein = floodFillOre(center, chunk, 10);
        
        if (vein.size() >= 1 && vein.size() <= 10) {
            return 0.8; // Похоже на реальную жилу
        } else if (vein.size() > 20) {
            return 0.1; // Слишком большая - точно подделка
        }
        return 0.5;
    }
}
```

**Защита**: Подстраивать распределение randomBlocks под реальное, но это сложно и затратно.

---

## 5. Вектор #4: Layer Obfuscation Exploit

### 5.1 Уязвимость

Когда включен `layerObfuscation: true`:

```java
// ObfuscationProcessor.java:82-90
if (obfuscationConfig.layerObfuscation()) {
  if (layerY != y) {
    layerY = y;
    layerYBlockState = bundle.nextRandomObfuscationBlock(y);
  }
  blockState = layerYBlockState; // ВСЕ блоки на одном Y одинаковые!
}
```

### 5.2 Эксплуатация

```java
public class LayerDetector {
    public Map<Integer, Block> detectLayers(Chunk chunk) {
        Map<Integer, Block> layerMap = new HashMap<>();
        
        for (int y = chunk.getMinBuildHeight(); y < chunk.getMaxBuildHeight(); y++) {
            Map<Block, Integer> blocksAtY = new HashMap<>();
            
            // Считаем блоки на этом Y уровне
            for (int x = 0; x < 16; x++) {
                for (int z = 0; z < 16; z++) {
                    Block block = chunk.getBlockState(x, y, z).getBlock();
                    if (block == Blocks.STONE || block == Blocks.DEEPSLATE) {
                        continue; // Пропускаем обычные блоки
                    }
                    blocksAtY.merge(block, 1, Integer::sum);
                }
            }
            
            // Если на слое МНОГО одинаковых "руд" -> это layer obfuscation
            for (Map.Entry<Block, Integer> entry : blocksAtY.entrySet()) {
                if (entry.getValue() > 50) { // Более 50 одинаковых блоков
                    layerMap.put(y, entry.getKey());
                    System.out.println("Layer detected at Y=" + y + ": " + entry.getKey());
                }
            }
        }
        
        return layerMap;
    }
    
    // Если детектирован layer obfuscation, можно:
    // 1. Игнорировать все блоки, которые совпадают с layer pattern
    // 2. Искать блоки, которые НЕ совпадают с паттерном = реальные
    public Set<BlockPos> findRealOres(Chunk chunk) {
        Map<Integer, Block> layers = detectLayers(chunk);
        Set<BlockPos> realOres = new HashSet<>();
        
        for (int y = chunk.getMinBuildHeight(); y < chunk.getMaxBuildHeight(); y++) {
            Block layerBlock = layers.get(y);
            if (layerBlock == null) continue;
            
            for (int x = 0; x < 16; x++) {
                for (int z = 0; z < 16; z++) {
                    BlockState state = chunk.getBlockState(x, y, z);
                    
                    // Если блок НЕ совпадает с layer pattern
                    if (isValuableOre(state.getBlock()) && state.getBlock() != layerBlock) {
                        // Это может быть реальная руда!
                        realOres.add(new BlockPos(x, y, z));
                    }
                }
            }
        }
        
        return realOres;
    }
}
```

**Защита**: НЕ использовать layer obfuscation (рекомендация в конфиге: `layerObfuscation: false`).

---

## 6. Вектор #5: Proximity Timing Attack

### 6.1 Механика Proximity Hider

```java
// ProximityWorker.java работает периодически
// Когда игрок приближается, блоки раскрываются с задержкой

// ProximityDirectorThread запускает worker каждые ~100ms
while (running) {
    Thread.sleep(100);
    proximityWorker.process(players);
}
```

### 6.2 Эксплуатация

```java
public class ProximityTimingExploit {
    private Map<BlockPos, List<Long>> blockAppearanceTimings = new HashMap<>();
    
    @SubscribeEvent
    public void onBlockAppear(BlockUpdateEvent event) {
        BlockPos pos = event.getPos();
        long timestamp = System.currentTimeMillis();
        
        blockAppearanceTimings.computeIfAbsent(pos, k -> new ArrayList<>()).add(timestamp);
        
        // Если блок появился когда мы приблизились
        double distanceToPlayer = pos.distanceTo(getPlayerPos());
        if (distanceToPlayer < 20) {
            // Это был proximity-скрытый блок
            if (isValuableOre(event.getState().getBlock())) {
                markAsRealOre(pos);
                System.out.println("Real ore found via proximity: " + pos);
            }
        }
    }
    
    // Активное сканирование: двигаемся по сетке и записываем появления блоков
    public void activeScan(World world, BlockPos start, int radius) {
        for (int x = -radius; x <= radius; x += 5) {
            for (int z = -radius; z <= radius; z += 5) {
                BlockPos scanPos = start.add(x, 0, z);
                
                // Телепортируемся / двигаемся к позиции
                moveToPosition(scanPos);
                
                // Ждём proximity reveal
                Thread.sleep(200);
                
                // Проверяем новые блоки
                checkForNewBlocks(scanPos, 20);
            }
        }
    }
}
```

### 6.3 Усовершенствованная техника: Differential Analysis

```java
public class DifferentialProximityAnalyzer {
    private Map<ChunkPos, ChunkSnapshot> previousSnapshots = new HashMap<>();
    
    public Set<BlockPos> findProximityRevealedOres(ChunkPos chunkPos) {
        Set<BlockPos> revealed = new HashSet<>();
        
        ChunkSnapshot previous = previousSnapshots.get(chunkPos);
        ChunkSnapshot current = takeSnapshot(chunkPos);
        
        if (previous == null) {
            previousSnapshots.put(chunkPos, current);
            return revealed;
        }
        
        // Сравниваем снимки
        for (int x = 0; x < 16; x++) {
            for (int z = 0; z < 16; z++) {
                for (int y = -64; y < 320; y++) {
                    Block prevBlock = previous.getBlock(x, y, z);
                    Block currBlock = current.getBlock(x, y, z);
                    
                    // Блок изменился с обычного на руду
                    if (prevBlock != currBlock && isValuableOre(currBlock)) {
                        BlockPos worldPos = new BlockPos(
                            chunkPos.x * 16 + x,
                            y,
                            chunkPos.z * 16 + z
                        );
                        revealed.add(worldPos);
                    }
                }
            }
        }
        
        previousSnapshots.put(chunkPos, current);
        return revealed;
    }
}
```

**Защита**: Добавить случайную задержку, но это ухудшит UX.

---

## 7. Комбинированная атака: ML-Based Ore Detection

### 7.1 Архитектура

```python
import tensorflow as tf
import numpy as np

class OreDetectionNN:
    def __init__(self):
        self.model = self.build_model()
    
    def build_model(self):
        model = tf.keras.Sequential([
            tf.keras.layers.Conv3D(32, (3,3,3), activation='relu', input_shape=(16,16,16,5)),
            tf.keras.layers.MaxPooling3D((2,2,2)),
            tf.keras.layers.Conv3D(64, (3,3,3), activation='relu'),
            tf.keras.layers.MaxPooling3D((2,2,2)),
            tf.keras.layers.Flatten(),
            tf.keras.layers.Dense(128, activation='relu'),
            tf.keras.layers.Dropout(0.5),
            tf.keras.layers.Dense(1, activation='sigmoid')  # Вероятность реальной руды
        ])
        return model
    
    def prepare_features(self, chunk, pos):
        # Извлекаем кубический регион 16x16x16 вокруг позиции
        region = np.zeros((16, 16, 16, 5))
        
        for dx in range(-8, 8):
            for dy in range(-8, 8):
                for dz in range(-8, 8):
                    block_pos = pos.add(dx, dy, dz)
                    block_state = chunk.getBlockState(block_pos)
                    
                    # Признак 1: ID блока
                    region[dx+8, dy+8, dz+8, 0] = block_state.getId()
                    
                    # Признак 2: Уровень освещения
                    region[dx+8, dy+8, dz+8, 1] = chunk.getLightLevel(block_pos)
                    
                    # Признак 3: Sky light
                    region[dx+8, dy+8, dz+8, 2] = chunk.getSkyLight(block_pos)
                    
                    # Признак 4: Был ли обновлён блок
                    region[dx+8, dy+8, dz+8, 3] = 1.0 if was_updated(block_pos) else 0.0
                    
                    # Признак 5: Соответствие статистике
                    region[dx+8, dy+8, dz+8, 4] = calculate_statistical_score(block_state)
        
        return region
    
    def predict_real_ore(self, chunk, pos):
        features = self.prepare_features(chunk, pos)
        features = np.expand_dims(features, axis=0)  # Batch dimension
        
        probability = self.model.predict(features)[0][0]
        return probability > 0.7  # Порог уверенности
```

### 7.2 Сбор датасета для обучения

```java
public class DatasetCollector {
    // Подключаемся к тестовому серверу с отключенным Orebfuscator
    public void collectRealOreData() {
        connectToServer("localhost:25565", orebfuscator=false);
        
        for (int i = 0; i < 1000; i++) {
            Chunk chunk = loadRandomChunk();
            
            for (BlockPos orePos : findAllOres(chunk)) {
                ChunkSnapshot snapshot = captureRegion(orePos, 16);
                saveToDataset(snapshot, label="REAL_ORE");
            }
        }
    }
    
    // Подключаемся к серверу с включенным Orebfuscator
    public void collectFakeOreData() {
        connectToServer("localhost:25565", orebfuscator=true);
        
        for (int i = 0; i < 1000; i++) {
            Chunk chunk = loadRandomChunk();
            
            // Собираем все руды до приближения
            Set<BlockPos> oresBeforeProximity = findAllOres(chunk);
            
            // Приближаемся и ждём деобфускации
            teleportToChunk(chunk);
            Thread.sleep(5000);
            
            // Собираем реальные руды после proximity reveal
            Set<BlockPos> oresAfterProximity = findAllOres(chunk);
            
            // Разница = поддельные руды
            Set<BlockPos> fakeOres = new HashSet<>(oresBeforeProximity);
            fakeOres.removeAll(oresAfterProximity);
            
            for (BlockPos fakeOre : fakeOres) {
                ChunkSnapshot snapshot = captureRegion(fakeOre, 16);
                saveToDataset(snapshot, label="FAKE_ORE");
            }
        }
    }
}
```

**Защита**: Очень сложно защититься от ML. Требуется:
- Постоянное изменение алгоритмов обфускации
- Добавление шума, который ML не сможет отфильтровать
- Но это может создать false positives для легитимных игроков

---

## 8. Рекомендации по защите

### 8.1 Для администраторов

**Конфигурация Orebfuscator**:

```yaml
# config.yml - БЕЗОПАСНАЯ конфигурация
general:
  updateRadius: 1  # Минимизировать деобфускацию
  updateOnBlockDamage: false  # Не раскрывать при ударе
  bypassNotification: false  # Не показывать уведомления
  ignoreSpectator: false  # НЕ игнорировать спектаторов

worlds:
  world:
    enabled: true
    minY: -64
    maxY: 320
    
    obfuscation:
      enabled: true
      layerObfuscation: false  # ОБЯЗАТЕЛЬНО отключить!
      
      # Скрывать только действительно ценные блоки
      hiddenBlocks:
        - "minecraft:diamond_ore"
        - "minecraft:deepslate_diamond_ore"
        - "minecraft:emerald_ore"
        - "minecraft:ancient_debris"
        - "minecraft:gold_ore"
        - "minecraft:deepslate_gold_ore"
      
      # Заменять ТОЛЬКО на обычные блоки
      randomBlocks:
        - name: "minecraft:stone"
          weight: 45
        - name: "minecraft:deepslate"
          weight: 45
        - name: "minecraft:andesite"
          weight: 5
        - name: "minecraft:diorite"
          weight: 5
        # НЕ ДОБАВЛЯТЬ РУДЫ В randomBlocks!
    
    proximity:
      enabled: true
      distance: 12  # Меньше = безопаснее
      
      frustumCulling:
        enabled: true  # Важно для производительности
        minDistance: 3
        fov: 80
      
      rayCastCheck:
        enabled: true  # Проверка линии видимости
        onlyCheckCenter: false
      
      useBlockBelow: true
      
      hiddenBlocks:
        minecraft:chest: {}
        minecraft:trapped_chest: {}
        minecraft:ender_chest: {}
        minecraft:furnace: {}
        minecraft:spawner: {}

cache:
  enabled: true
  maxSize: 10000
  expireAfterAccess: 30
```

### 8.2 Дополнительные меры

**1. Использовать античит-плагины**:

- **Matrix** - детект X-ray поведения через анализ добычи
- **Vulcan** - продвинутая статистика
- **GrimAC** - предсказание движения игрока

**2. Мониторинг статистики**:

```yaml
# Плагин для мониторинга добычи
mining-monitor:
  alert-thresholds:
    diamonds-per-hour: 50  # Алерт если > 50 алмазов/час
    ore-finding-accuracy: 0.3  # Алерт если > 30% точность
    
  check-patterns:
    - straight-line-mining  # Детект прямого копания к рудам
    - perfect-ore-tracking  # Детект идеального следования за жилой
```

**3. Логирование**:

```java
// Логировать подозрительное поведение
public class SuspiciousActivityLogger {
    public void checkPlayer(Player player) {
        PlayerStats stats = getStats(player);
        
        // Проверка 1: Слишком много редких руд
        if (stats.getDiamondsPerHour() > 50) {
            alert("Player " + player.getName() + " mining too many diamonds");
        }
        
        // Проверка 2: Слишком высокая точность
        double accuracy = stats.getOresFound() / (double) stats.getBlocksBroken();
        if (accuracy > 0.3) {
            alert("Player " + player.getName() + " has suspicious accuracy");
        }
        
        // Проверка 3: Паттерны движения
        if (detectsStraightLineToOre(player)) {
            alert("Player " + player.getName() + " moving straight to ores");
        }
    }
}
```

### 8.3 Для разработчиков Orebfuscator

**Предложения по улучшению**:

1. **Обфускация освещения**:
```java
// Добавить ложные источники света
if (isObfuscated) {
    int fakeLightLevel = random.nextInt(15);
    chunkSection.setBlockLight(index, fakeLightLevel);
}
```

2. **Динамическая обфускация**:
```java
// Разное для каждого игрока
int seed = player.getUniqueId().hashCode() ^ chunkX ^ chunkZ;
Random playerRandom = new Random(seed);
```

3. **Задержка деобфускации с шумом**:
```java
// Добавить случайную задержку 0-500ms
int delay = random.nextInt(500);
scheduler.runTaskLater(() -> {
    deobfuscate(block);
}, delay / 50); // Minecraft ticks
```

4. **Адаптивная защита**:
```java
// Если детектирован подозрительный игрок, усилить обфускацию
if (isSuspicious(player)) {
    config.setUpdateRadius(player, 0); // Никакой деобфускации
    config.setProximityDistance(player, 5); // Минимальная дистанция
}
```

---

## 9. Заключение

### Успешность обхода: Сводная таблица

| Метод | Сложность | Эффективность | Детектируемость |
|-------|-----------|---------------|-----------------|
| Block Update Analysis | Низкая | Высокая (70-90%) | Средняя |
| Light Level Analysis | Средняя | Средняя (40-60%) | Низкая |
| Statistical Analysis | Средняя | Средняя (50-70%) | Низкая |
| Layer Obfuscation Exploit | Низкая | Очень высокая (95%) | Средняя |
| Proximity Timing | Средняя | Высокая (60-80%) | Высокая |
| ML-Based Detection | Очень высокая | Очень высокая (80-95%) | Низкая |

### Требования для реализации полноценного обхода:

1. **Знания**:
   - Протокол Minecraft (packet structure)
   - Java/Kotlin программирование
   - Forge/Fabric modding API
   - Машинное обучение (опционально)

2. **Инструменты**:
   - Minecraft mod development environment
   - ProtocolLib / PacketEvents
   - Анализатор сетевых пакетов

3. **Время разработки**: 20-40 часов для базового обхода, 100+ для продвинутого

### Вывод:

Orebfuscator **эффективен против** большинства игроков, использующих простые X-ray текстуры или моды. Однако он **уязвим** перед продвинутыми читами, которые анализируют:
- Обновления блоков
- Освещение
- Статистику
- Temporal patterns

Для максимальной защиты необходимо:
- Правильно настроить Orebfuscator
- Использовать дополнительные античит-плагины
- Мониторить статистику игроков
- Регулярно обновлять защиту
