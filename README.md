# Моделирование пользовательской активности на основе пуассоновских процессов

Исходный код и результаты экспериментов к выпускной квалификационной работе
«Методы моделирования пользовательской активности на основе пуассоновских процессов»
(СПбГУ МКН, программа ВМ.5889 «Разработка ПО и науки о данных»).

Работа сравнивает лестницу моделей для прогноза дневной интенсивности покупок в
e-commerce: от простого пуассоновского базлайна до процессов Хоукса
(Scaled-baseline и Joint) и градиентного бустинга, а также исследует
кросс-канальную структуру самовозбуждения (search / cat → cart / order).

## Структура репозитория

```
src/
  diploma_baselines/        базовые модели и пайплайн экспериментов
    models/                 poisson, rolling/seasonal poisson, personalized gamma-poisson, hawkes
    pipeline.py             сборка эксперимента (train/test split, метрики, артефакты)
    metrics.py              NLL и сопутствующие метрики
    data.py                 загрузка дневной сетки, окно анализа
    plots.py                визуализации
    feature_research.py     отбор/исследование признаков
    excitation_research.py  исследование самовозбуждения
  diploma_experimental/     экспериментальные модели
    gbdt.py                 градиентный бустинг
    pipeline.py

scripts/
  compute/                  скрипты, считающие эксперименты (run_*.py)
  plots/                    скрипты построения графиков для работы/слайдов
  archive/                  отложенные/старые скрипты

diploma/
  reports/                  markdown-отчёты и артефакты по каждому эксперименту
  references/               вспомогательные материалы
  NN_*.md                   сводные отчёты по главам (по порядку лестницы моделей)

data/                       данные (см. раздел «Данные»; в репо только сжатый .gz)

notebooks/
  quickstart_hawkes.ipynb   распаковка данных и запуск Хоукс-модели «из коробки»
```

## Установка

Требуется Python 3.14. Зависимости: `numpy`, `pandas`, `scipy`, `scikit-learn`,
`matplotlib`.

```bash
python -m venv .venv
source .venv/bin/activate
pip install numpy pandas scipy scikit-learn matplotlib
```

Версии, на которых получены результаты работы: numpy 2.3.5, pandas 2.3.3,
scipy 1.16.3, scikit-learn 1.8.0, matplotlib 3.10.8.

## Данные

Эксперименты работают с дневной сеткой активности когорты из 10 000 пользователей.
В репозитории лежит сжатая версия (~13 МБ):
`data/processed/orbitals/dayuses_cohort_10000_seed42_daily_grid.csv.gz`.
Распаковать перед запуском (или воспользоваться ноутбуком, см. ниже):

```bash
gunzip -k data/processed/orbitals/dayuses_cohort_10000_seed42_daily_grid.csv.gz
```

Одна строка — это `(user_id, event_date)`; колонки:

| колонка                                                         | смысл                                                                                            |
| ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `user_id`                                                            | идентификатор пользователя (псевдоним)                              |
| `event_date`                                                         | календарный день                                                                       |
| `search`, `cat`                                                    | число событий поиска / просмотра категории                        |
| `searches`                                                           | число поисковых сессий                                                            |
| `has_*_to_cart`, `has_*_to_ord`                                    | флаги переходов воронки                                                          |
| `search_to_cart`, `search_to_ord`, `cat_to_cart`, `cat_to_ord` | переходы по воронке                                                                  |
| `to_cart`, `to_ord`                                                | добавления в корзину / заказы за день (целевые величины) |
| `gmv`                                                                | оборот за день                                                                            |

Целевая переменная по умолчанию — `to_ord` (число заказов за день).
Окно анализа: `2025-01-15 … 2025-09-30`, доля train — `0.8`.

Путь к данным переопределяется флагом `--data-path` у любого скрипта.

## Быстрый старт

Ноутбук [`notebooks/quickstart_hawkes.ipynb`](notebooks/quickstart_hawkes.ipynb)
распаковывает данные, прогоняет главную модель работы (Scaled-baseline Hawkes)
и показывает результат — удобно для первого запуска.

## Запуск и проверка

Скрипты запускаются из корня репозитория. Каждый сам добавляет корень в `sys.path`
и пишет артефакты (json со сводкой, графики) в `--output-dir`.

Главная модель работы (Scaled-baseline Hawkes, полураспады 1 и 3 дня):

```bash
python scripts/compute/run_experimental_1_hawkes.py
```

Скрипт печатает сводку в stdout (json) и сам создаёт папку
`diploma/reports/experimental_1_hawkes/`, куда складывает артефакты. Все параметры
(`--target-col`, `--train-ratio`, `--analysis-start/end`, регуляризация и т.д.)
имеют значения по умолчанию, воспроизводящие числа из работы.

Готовые отчёты всех экспериментов уже лежат в `diploma/reports/`. Повторный запуск
скрипта пересоздаёт соответствующую папку байт-в-байт — после прогона `git status`
остаётся чистым, то есть результаты в репозитории получены именно этим кодом на
этих данных.

Ключевые скрипты лестницы моделей:

| модель                   | скрипт                                                              |
| ------------------------------ | ------------------------------------------------------------------------- |
| Poisson (глобальный) | `scripts/compute/run_poisson_baseline.py`                               |
| Rolling seasonal Poisson       | `scripts/compute/run_rolling_seasonal_poisson_baseline.py`              |
| Personalized gamma-Poisson     | `scripts/compute/run_personalized_rolling_seasonal_poisson_baseline.py` |
| Scaled-baseline Hawkes         | `scripts/compute/run_experimental_1_hawkes.py`                          |
| Joint Hawkes                   | `scripts/compute/run_joint_hawkes_ch8.py`                               |
| GBDT                           | `scripts/compute/run_experimental_2_gbdt.py`                            |
| Сводка лестницы  | `scripts/plots/run_ladder_summary.py`                                   |

Ожидаемые test NLL (target `to_ord`): Global ≈ 0.4608, Rolling seasonal ≈ 0.4576,
Personalized GP ≈ 0.4096, Scaled-baseline Hawkes ≈ 0.3958, Joint Hawkes ≈ 0.3952, GBDT ≈ 0.3881.

Прочие скрипты `scripts/compute/run_*.py` считают отдельные эксперименты
(кросс-канальная матрица, profile likelihood, доверительные интервалы,
жизненный цикл пользователя и т.д.); номера в именах файлов (`_ch15`, `_ch19`, …)
соответствуют главам/разделам отчётов в `diploma/reports/`.
