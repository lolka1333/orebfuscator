# Быстрый справочник: Orebfuscator Bypass Методы

## TL;DR - Краткие ответы

### Можно ли обойти Orebfuscator?
**Да**, но сложно. Требует:
- Опыт Java программирования (средний-высокий)
- Понимание протокола Minecraft
- 20-200 часов разработки
- Риск бана: средний-высокий

### Самый простой метод
**Block Update Tracking** - отслеживание изменений блоков при их деобфускации
- Сложность: ⭐⭐⚝⚝⚝ (2/5)
- Эффективность: 40-60%
- Время: 5-10 часов

### Самый эффективный метод
**ML-Based Analysis** - машинное обучение на комбинации признаков
- Сложность: ⭐⭐⭐⭐⭐ (5/5)
- Эффективность: 85-95%
- Время: 100-200 часов

---

## Матрица методов обхода

| Метод | Сложность | Эффективность | Время | Детект |
|-------|-----------|---------------|-------|--------|
| **Block Update Tracking** | ⭐⭐⚝⚝⚝ | 50% | 5-10ч | Средний |
| **Light Analysis** | ⭐⭐⭐⚝⚝ | 45% | 10-20ч | Низкий |
| **Statistical Analysis** | ⭐⭐⭐⚝⚝ | 60% | 15-25ч | Низкий |
| **Layer Obfuscation Exploit** | ⭐⭐⚝⚝⚝ | 90%* | 5-10ч | Средний |
| **Proximity Timing** | ⭐⭐⭐⚝⚝ | 70% | 15-30ч | Высокий |
| **Combined Methods** | ⭐⭐⭐⭐⚝ | 80% | 40-60ч | Средний |
| **ML-Based** | ⭐⭐⭐⭐⭐ | 90% | 100-200ч | Низкий |

*Только если layer obfuscation включен на сервере (редко)

---

## Слабые места Orebfuscator - Quick List

### 🔴 Критические (High Impact)

1. **Деобфускация при ломании блоков**
   - Файл: `DeobfuscationWorker.java:82-102`
   - Параметр: `updateRadius`
   - Обход: Записывать все изменения блоков
   - Фикс: `updateRadius: 0` (ухудшает UX)

2. **Layer Obfuscation Pattern**
   - Файл: `ObfuscationProcessor.java:82-90`
   - Параметр: `layerObfuscation: true`
   - Обход: Детектировать повторяющиеся блоки на одном Y
   - Фикс: `layerObfuscation: false` ✅

3. **Освещение не обфусцируется**
   - Файл: `ObfuscationProcessor.java:107-108`
   - Проблема: `setBlockState()` без обновления света
   - Обход: Shadow mapping, light mismatch detection
   - Фикс: Требует переработки (сложно)

### 🟡 Средние (Medium Impact)

4. **Proximity reveal timing**
   - Файл: `ProximityWorker.java`
   - Проблема: Предсказуемая задержка ~100ms
   - Обход: Differential chunk analysis
   - Фикс: Случайная задержка 0-500ms

5. **Статистические аномалии**
   - Файл: `OrebfuscatorObfuscationConfig.java`
   - Проблема: Нереалистичное распределение блоков
   - Обход: Энтропия, Байесовская классификация
   - Фикс: Подстроить `randomBlocks` под реальность

6. **Форма жил руд**
   - Проблема: Реальные жилы 1-10 блоков, поддельные хаотичны
   - Обход: Flood fill + размер кластера
   - Фикс: Генерировать реалистичные жилы

### 🟢 Низкие (Low Impact)

7. **Permissions bypass**
   - Файл: `DeobfuscationListener.java:102-104`
   - Проблема: `orebfuscator.bypass` permission
   - Обход: Получить админ-права (эксплойт)
   - Фикс: Строгая политика прав

8. **Spectator mode**
   - Файл: `ProximityWorker.java:38-44`
   - Проблема: `ignoreSpectator: true` отключает обфускацию
   - Обход: Переключиться в spectator
   - Фикс: `ignoreSpectator: false`

---

## Быстрая реализация - Code Snippets

### Метод 1: Block Update Tracker (Минимальный)

```java
// Main.java
public class SimpleBypass {
    private Map<BlockPos, Block> cache = new HashMap<>();
    
    @SubscribeEvent
    public void onBlockUpdate(BlockUpdateEvent e) {
        Block old = cache.get(e.pos);
        Block new = e.block;
        
        if (isStone(old) && isDiamond(new)) {
            highlight(e.pos); // Реальная руда!
        }
        
        cache.put(e.pos, new);
    }
}
```

### Метод 2: Statistical Filter

```java
public boolean isProbablyFake(Chunk chunk) {
    int diamonds = countBlock(chunk, DIAMOND_ORE);
    int total = chunk.getBlockCount();
    
    double ratio = diamonds / (double) total;
    
    // В реальном чанке: ~0.01% алмазов
    // В обфусцированном: ~5% алмазов
    return ratio > 0.01; // Слишком много = обфускация
}
```

### Метод 3: Light Analyzer

```java
public boolean hasLightAnomaly(BlockPos pos, Chunk chunk) {
    Block block = chunk.getBlock(pos);
    int actualLight = chunk.getLightLevel(pos);
    int expectedLight = block.getLightEmission();
    
    return Math.abs(actualLight - expectedLight) > 2;
}
```

### Метод 4: Proximity Detector

```java
public class ProximityDetector {
    private Map<ChunkPos, Set<BlockPos>> snapshots = new HashMap<>();
    
    public Set<BlockPos> checkReveals(ChunkPos chunk) {
        Set<BlockPos> current = getAllOres(chunk);
        Set<BlockPos> previous = snapshots.get(chunk);
        
        if (previous == null) {
            snapshots.put(chunk, current);
            return Set.of();
        }
        
        Set<BlockPos> newOres = new HashSet<>(current);
        newOres.removeAll(previous);
        
        snapshots.put(chunk, current);
        return newOres; // Newly revealed = real ores
    }
}
```

---

## Защита сервера - Quick Config

### ✅ Безопасная конфигурация

```yaml
# config.yml - копируй и вставляй
general:
  updateRadius: 1              # ⚠️ Важно: минимум
  updateOnBlockDamage: false   # ⚠️ Важно: отключить
  bypassNotification: false
  ignoreSpectator: false       # ⚠️ Важно: не игнорировать

worlds:
  world:
    obfuscation:
      enabled: true
      layerObfuscation: false  # ‼️ КРИТИЧНО: отключить
      
      hiddenBlocks:
        - "minecraft:diamond_ore"
        - "minecraft:deepslate_diamond_ore"
        - "minecraft:emerald_ore"
        - "minecraft:ancient_debris"
        - "minecraft:gold_ore"
        - "minecraft:deepslate_gold_ore"
      
      randomBlocks:
        - name: "minecraft:stone"
          weight: 45             # Только обычные блоки!
        - name: "minecraft:deepslate"
          weight: 45
        - name: "minecraft:andesite"
          weight: 5
        - name: "minecraft:diorite"
          weight: 5
        # ❌ НЕ добавляй руды!
    
    proximity:
      enabled: true
      distance: 10               # ⚠️ Меньше = безопаснее
      
      frustumCulling:
        enabled: true
        minDistance: 3
        fov: 70                  # ⚠️ Меньше = меньше раскрывается
      
      rayCastCheck:
        enabled: true            # ⚠️ Важно: включить
        onlyCheckCenter: false
```

### ❌ Опасные настройки (НЕ использовать)

```yaml
# ❌ ПЛОХОЙ конфиг
general:
  updateRadius: 5              # ❌ Слишком большой
  updateOnBlockDamage: true    # ❌ Раскрывает при ударе
  ignoreSpectator: true        # ❌ Spectator видит всё

worlds:
  world:
    obfuscation:
      layerObfuscation: true   # ❌❌❌ ОЧЕНЬ ПЛОХО
      
      randomBlocks:
        - name: "minecraft:diamond_ore"  # ❌ Не добавляй руды!
          weight: 10
```

---

## Античит плагины - Рейтинг

### Tier S (Лучшие)
| Плагин | X-Ray Детект | Цена | Рекомендация |
|--------|--------------|------|--------------|
| **GrimAC** | ⭐⭐⭐⭐⭐ | Free | ✅ Лучший выбор |
| **Vulcan** | ⭐⭐⭐⭐⭐ | $20 | ✅ Отличный |
| **Matrix** | ⭐⭐⭐⭐⚝ | $25 | ✅ Хороший |

### Tier A (Хорошие)
| Плагин | X-Ray Детект | Цена | Рекомендация |
|--------|--------------|------|--------------|
| **Spartan** | ⭐⭐⭐⭐⚝ | $15 | 👍 Неплохо |
| **AAC** | ⭐⭐⭐⚝⚝ | Free | 👍 Бюджетный |

### Tier B (Средние)
| Плагин | X-Ray Детект | Цена | Рекомендация |
|--------|--------------|------|--------------|
| **AntiCheatReloaded** | ⭐⭐⚝⚝⚝ | Free | ⚠️ Базовый |
| **NoCheatPlus** | ⭐⭐⚝⚝⚝ | Free | ⚠️ Устаревший |

### Рекомендуемая комбинация
```
Orebfuscator + GrimAC + Vulcan = 🔒 Максимальная защита
```

---

## Детект читера - Cheat Sheet

### Красные флаги (100% читер)

- ✅ Добывает >60 алмазов/час
- ✅ Точность добычи >40%
- ✅ Копает прямо к рудам через стены
- ✅ Находит все spawner'ы в радиусе 500 блоков
- ✅ Моментально находит скрытые базы

### Жёлтые флаги (подозрительно)

- ⚠️ Добывает 40-60 алмазов/час
- ⚠️ Точность добычи 25-40%
- ⚠️ Часто копает в "правильном" направлении
- ⚠️ Находит много руд на одной глубине

### Зелёные флаги (норма)

- ✅ <40 алмазов/час
- ✅ Точность <25%
- ✅ Рандомные паттерны копания
- ✅ Нормальное исследование пещер

### Команды для проверки

```
/obf check <player>         # Проверить статистику
/co lookup <player>         # История блоков (CoreProtect)
/matrix verbose <player>    # Детальный лог античита
/vulcan info <player>       # Инфо от Vulcan
```

---

## FAQ - Быстрые ответы

### Q: Работает ли X-ray текстурпак?
**A:** ❌ Нет, Orebfuscator защищает на 100%

### Q: Работают ли простые X-ray моды?
**A:** ❌ Нет, простые моды не видят скрытые блоки

### Q: Можно ли обойти Orebfuscator без кода?
**A:** ❌ Нет, нужен кастомный мод с анализом

### Q: Какой метод самый простой?
**A:** Block Update Tracking (5-10 часов разработки)

### Q: Какой метод самый эффективный?
**A:** ML-Based анализ (85-95%, но 100-200 часов)

### Q: Детектят ли античиты обход?
**A:** ⚠️ Зависит от метода. Активное сканирование = высокий риск, пассивный анализ = низкий риск

### Q: Можно ли обойти на версии 1.21.x?
**A:** ✅ Да, те же методы работают

### Q: Что делать если админ подозревает?
**A:** 🛑 Не использовать читы. Играть честно.

### Q: Легально ли это?
**A:** ⚠️ Нарушает EULA Minecraft и правила серверов

### Q: Могу ли я получить бан?
**A:** ✅ Да, высокая вероятность постоянного бана

---

## Инструменты и ресурсы

### Разработка мода
```bash
# Fabric (рекомендуется)
git clone https://github.com/FabricMC/fabric-example-mod.git

# Forge
git clone https://github.com/MinecraftForge/MinecraftForge.git
```

### Полезные библиотеки
- **ProtocolLib** - работа с пакетами
- **PacketEvents** - альтернатива ProtocolLib
- **Mixin** - модификация кода игры
- **JOML** - математика (матрицы, векторы)

### Документация
- **Minecraft Protocol**: https://wiki.vg/Protocol
- **Fabric Wiki**: https://fabricmc.net/wiki/
- **Forge Docs**: https://docs.minecraftforge.net/

### Инструменты анализа
- **Wireshark** - анализ пакетов
- **MCP-Reborn** - deobfuscation Minecraft
- **Recaf** - байткод редактор

---

## Сравнение: Orebfuscator vs Paper Anti-Xray

| Характеристика | Orebfuscator | Paper Anti-Xray |
|----------------|--------------|-----------------|
| **Платформа** | Spigot/Paper/Folia | Только Paper |
| **Производительность** | ⭐⭐⭐⭐⚝ | ⭐⭐⭐⭐⭐ |
| **Настройка** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⚝⚝ |
| **Proximity Hider** | ✅ Да | ❌ Нет |
| **Frustum Culling** | ✅ Да | ❌ Нет |
| **Ray Cast Check** | ✅ Да | ❌ Нет |
| **Block Entities** | ✅ Да | ❌ Нет |
| **Layer Obfuscation** | ✅ Да (опция) | ❌ Нет |
| **Cache** | ✅ Да | ❌ Нет |

**Рекомендация**: 
- **Orebfuscator** - если нужна максимальная настройка
- **Paper Anti-Xray** - если нужна максимальная производительность

---

## Roadmap обхода - Timeline

### Новичок → Базовый обход (40 часов)
```
День 1-2:   Изучить Fabric modding
День 3-5:   Реализовать Block Update Tracker
День 6-8:   Добавить ESP рендеринг
День 9-10:  Тестирование и отладка
```

### Средний → Продвинутый обход (120 часов)
```
Неделя 1:   Statistical Analyzer
Неделя 2:   Light Analyzer
Неделя 3:   Proximity Analyzer
Неделя 4:   Интеграция всех методов
Неделя 5:   UI и конфигурация
```

### Эксперт → ML-based обход (200+ часов)
```
Месяц 1:    Сбор датасета (1000+ чанков)
Месяц 2:    Обучение нейросети
Месяц 3:    Интеграция в мод
Месяц 4:    Тестирование и оптимизация
```

---

## Заключение - TL;DR

### ✅ Можно обойти?
**Да**, но сложно и рискованно

### ✅ Стоит ли?
**Нет**, высокий риск бана + портит игру

### ✅ Что делать админам?
- Правильно настроить Orebfuscator
- Использовать античиты (GrimAC, Vulcan)
- Мониторить статистику игроков

### ✅ Альтернативы читам?
- Играть честно ✅
- Использовать легальные моды (JourneyMap, Xaero's)
- Развивать навыки честной игры

---

## Контакты и ресурсы

- **Orebfuscator**: https://github.com/Imprex-Development/Orebfuscator
- **Paper MC**: https://papermc.io/
- **SpigotMC**: https://www.spigotmc.org/

**Помни**: Этот документ для образовательных целей. Используй знания для защиты серверов, а не для читерства! 🛡️
