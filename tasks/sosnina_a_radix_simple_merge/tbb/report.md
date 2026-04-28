# Поразрядная сортировка целых чисел с простым слиянием (TBB-версия)

**Вариант № 17**  
**Студент:** Соснина Александра Антоновна  
**Группа:** 3823Б1ПР1  
**Преподаватель:** Сысоев Александр Владимирович, доцент

---

## Введение

Реализация на **oneTBB** использует `tbb::parallel_for` для двух типов работ: независимая поразрядная сортировка частей и параллельная обработка **пар** на каждом уровне дерева слияний. Планировщик TBB и `simple_partitioner` задают предсказуемое разбиение диапазона итераций, что удобно для воспроизводимости замеров по сравнению с полностью динамическим stealing по умолчанию в некоторых конфигурациях.

Дополнительно в коде заданы **пороги зернистости** (`kMinElementsPerPart`, `kSmallArrayThreshold`, `kLargeArrayThreshold`), чтобы на больших входах не создавать избыточное число мелких частей и уровней merge.

## Постановка задачи

Получить быструю и корректную многопоточную сортировку radix+merge на базе oneTBB, согласованную с общими тестами и с `ppc::util::GetNumThreads()` как верхней границей по параллелизму.

## Связь с кодом

- Файл: `tbb/src/ops_tbb.cpp`.
- Класс: `SosninaATestTaskTBB`.
- Заголовки: `oneapi/tbb/parallel_for.h`, `oneapi/tbb/partitioner.h`.
- Слияние отсортированных половинок: `std::ranges::merge` в буфер результата (как в STL-ветке).

## Описание реализации

### Константы зернистости (фрагмент логики)

В коде используются:

- `kMinElementsPerPart = 4096`;
- `kMinElementsPerPartSmall = 32768` — для `n < kSmallArrayThreshold` стартуем с более крупного минимума части, чтобы на малых и средних массивах не раздувать дерево merge;
- `kSmallArrayThreshold = 1'000'000`;
- `kLargeArrayThreshold = 20'000'000` — на «крупном» входе целимся в порядка **одной части на поток** (через `per_thread_floor`), чтобы снизить глубину merge на perf-тесте с 20 млн элементов.

Базовый минимум размера части:

```text
min_chunk_base = (n < kSmallArrayThreshold) ? kMinElementsPerPartSmall : kMinElementsPerPart
```

Нижняя оценка размера части на поток:

```text
per_thread_floor = (n >= kLargeArrayThreshold)
  ? n / max(1, num_threads)
  : n / max(1, 2 * num_threads)
min_chunk = max(min_chunk_base, per_thread_floor)
max_parts_by_grain = max(1, int(n / min_chunk))
num_parts = min(num_threads, int(n), max_parts_by_grain)
```

Таким образом, `num_parts` ограничен **и** запросом потоков, **и** размером данных, **и** грубой оценкой «не дробить слишком мелко».

### Фаза radix

```cpp
tbb::parallel_for(0, num_parts, [&](int i) {
  std::vector<int> buffer(parts[i].size());
  RadixSortLSD(parts[i], buffer);
}, tbb::simple_partitioner{});
```

`simple_partitioner` фиксирует стратегию разбиения диапазона `[0, num_parts)`.

### Фаза merge

На каждом уровне аналогичный `parallel_for` по числу пар; в лямбде — выделение `next[idx]`, `SimpleMerge`, освобождение исходных векторов пары.

### Проверка

`std::ranges::is_sorted(data)` в конце `RunImpl()`.

### Псевдокод

```text
вычислить num_parts по порогам и GetNumThreads()
разбить data на parts[0..num_parts)

tbb::parallel_for(i in 0..num_parts, simple_partitioner):
    LSD-sort(parts[i])

current <- parts
пока |current| > 1:
    tbb::parallel_for(idx in 0..|current|/2, simple_partitioner):
        next[idx] <- merge(current[2*idx], current[2*idx+1])
    хвост при нечётном |current|
    current <- next
data <- current[0]
```

## Сравнение с OpenMP на уровне идеи

- TBB явно **ограничивает число частей** порогами — на больших `n` это обычно уменьшает глубину merge относительно «голого» `min(threads, n)` в OMP-реализации данной задачи.
- Оба подхода параллелят и radix, и уровни merge; разница в деталях планирования и накладных расходах рантайма.

## Оценка сложности

- По данным: **O(n)** на radix и merge при фиксированной разрядности `int`.
- Практическая масштабируемость: баланс между **толщиной частей** (мало частей — мало параллелизма на radix) и **числом уровней merge** (много мелких частей — много уровней и аллокаций).

## Тестирование

Общий функциональный набор из **48** кейсов в `tests/functional/main.cpp`, сверка с эталоном и проверка отсортированности.

## Экспериментальные результаты (ориентир)

Те же условия, что в сводном отчёте: M2, macOS, Release, `task_run`, 20 млн элементов.

`S = T_seq / T_tbb`, `Eff = S / N`.

| N потоков | Ускорение S | Эффективность Eff |
| --------- | ----------- | ----------------- |
| 2         | 1,91        | 95,5%             |
| 4         | 2,64        | 66%               |
| 8         | 3,05        | 38,1%             |

На использованной платформе TBB в этой серии замеров слегка **опережает** OMP и STL; для отчёта с фиксированными числами лучше приложить вывод своего прогона `ppc_perf_tests`.

## Анализ

- Пороги `kLargeArrayThreshold` / `per_thread_floor` заточены под сценарий perf-теста и объясняют хорошее поведение на **20 млн** элементов.
- На **малых** массивах выгода от TBB относительно OMP может сокращаться из-за накладных расходов задач и того, что число частей принудительно ограничено.

## Вывод

TBB-версия корректна и на рассматриваемой конфигурации показывает **наилучшее** среди OMP/TBB/STL практическое ускорение на крупном входе за счёт сочетания параллельного radix, параллельного merge и настройки зернистости.

## Источники

1. Курс «Параллельное программирование».
2. [oneAPI Threading Building Blocks (oneTBB)](https://www.intel.com/content/www/us/en/developer/tools/oneapi/onetbb.html).
3. [cppreference: execution policies](https://en.cppreference.com/w/cpp/algorithm) (для контекста; в данной задаче `std::execution::par` не используется).
