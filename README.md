**RL-Scalper-Bot**

Проект: **RL-Scalper-Bot** — это модульная Python-платформа для разработки, тестирования и развёртывания алгоритмического торгового бота на основе обучения с подкреплением (Reinforcement Learning). Бот генерирует торговые сигналы (покупка/продажа/отсутствие сигнала) в режиме скальпинга/интрадей и не исполняет ордера автоматически.

---

## 📂 Структура проекта

```bash
rl-scalper-bot/
├── data/                       # Сырые и обработанные данные
│   ├── raw/                    # Исторические минутные/тиковые бары
│   └── processed/              # Подготовленные окна для обучения и тестирования
│
├── configs/                    # Конфигурационные файлы (YAML/JSON)
│   ├── data_pipeline.yaml      # Источники данных, символы, периоды
   ├── env.yaml                # Параметры Gym-окружения (окно, комиссии)
   ├── training.yaml           # Гиперпараметры обучения (алгоритм, LR, γ)
   └── inference.yaml          # Настройки сервиса инференса (пороги, логирование)

├── src/
│   ├── data_pipeline/          # Сбор, очистка и фичеризация данных
│   │   ├── fetcher.py          # API-клиент для загрузки баров
│   │   ├── cleaner.py          # Обработка пропусков и артефактов
│   │   └── featurizer.py       # Индикаторы и нормализация
│   │
│   ├── env/                    # OpenAI Gym-окружение
│   │   ├── trading_env.py      # Класс Env с методами step, reset, reward
│   │   └── wrappers.py         # (опционально) обёртки для векторизации, логирования
│   │
│   ├── agents/                 # RL-агенты и базовые политики
│   │   ├── base_agent.py       # Интерфейс: train, act, save, load
│   │   ├── ppo_agent.py        # Реализация агента PPO (Stable Baselines3)
│   │   └── sac_agent.py        # Опционально: SAC / TD3 для off-policy
│   │
│   ├── backtester/             # Модуль бэктестинга
│   │   └── backtest.py         # Прогон модели и расчет метрик (PnL, Sharpe)
│   │
│   ├── inference/              # Сервис онлайн-инференса
│   │   ├── api.py              # FastAPI/Flask endpoint `/signal`
│   │   └── model_loader.py     # Загрузка сохранённой модели
│   │
│   ├── monitoring/             # Логирование и метрики
│   │   ├── logger.py           # Настройка логгера (file/console)
│   │   └── metrics.py          # Подготовка метрик для Grafana/Prometheus
│   │
│   └── utils/                  # Утилитарные модули
│       ├── config.py           # Загрузка и валидация конфигов
│       ├── dates.py            # Работа с таймзонами и календарём торгов
│       └── serializers.py      # Сериализация и десериализация моделей
│
├── experiments/                # Логи экспериментов и сохранённые модели
├── docker/                     # Dockerfile и манифесты Kubernetes
│   ├── Dockerfile.base
│   ├── Dockerfile.training
│   └── k8s.yaml
│
├── tests/                      # Unit и интеграционные тесты (pytest)
├── scripts/                    # Утилиты для запуска (ETL, обучение, бэктест)
│   ├── run_data_pipeline.py
│   ├── train_agent.py
│   ├── backtest_report.py
│   └── serve_inference.py
│
├── .github/ci/                 # CI/CD (GitHub Actions)
│   └── python-ci.yaml
├── requirements.txt            # Основные зависимости
├── README.md                   # Документация (вы здесь)
└── LICENSE                     # Лицензия проекта
```

---

## 🚀 Быстрый старт

1. Клонируйте репозиторий и перейдите в директорию:

   ```bash
   git clone https://github.com/your-repo/rl-scalper-bot.git
   cd rl-scalper-bot
   ```

2. Создайте и активируйте виртуальное окружение:

   ```bash
   python3 -m venv venv
   source venv/bin/activate  # Linux/Mac
   venv\\Scripts\\activate  # Windows
   ```

3. Установите зависимости:

   ```bash
   pip install -r requirements.txt
   ```

4. Настройте конфиги в папке `configs/` под свои тикеры, API-ключи и гиперпараметры.

5. Запустите обработку данных:

   ```bash
   python scripts/run_data_pipeline.py --config configs/data_pipeline.yaml
   ```

6. Обучите RL-агента:

   ```bash
   python scripts/train_agent.py --config configs/training.yaml
   ```

7. Проведите бэктест и оцените метрики:

   ```bash
   python scripts/backtest_report.py --config configs/env.yaml
   ```

8. Запустите сервис инференса локально:

   ```bash
   python scripts/serve_inference.py --config configs/inference.yaml
   ```

---

## 🏗 Подробная архитектура

### 1. Data Pipeline

* **fetcher.py**: загружает исторические минуты/тики через API (CCXT, Polygon, OANDA).
* **cleaner.py**: удаляет выбросы, заполняет пропуски, синхронизирует метки времени.
* **featurizer.py**: считает технические индикаторы (SMA, EMA, RSI, MACD), объёмные (VWAP, OBV), нормализует данные (скользящий Z-score).

Все скрипты настраиваются через `configs/data_pipeline.yaml` и сохраняют результат в `data/processed/`.

### 2. Gym-окружение (`src/env/`)

Класс `TradingEnv(gym.Env)`:

* **observation\_space**: массив последних N баров × M признаков.
* **action\_space**: {0: нейтрально, 1: сигнал на покупку, 2: сигнал на продажу}.
* **step()**: обновление состояния, расчёт PnL с учётом комиссий/проскальзывания, штрафы за риск.
* **reset()**: начало нового эпизода (новый торговый день или окно).

### 3. RL-агенты (`src/agents/`)

* **BaseAgent**: общий интерфейс (`train()`, `act()`, `save()`, `load()`).
* **PPOAgent**: обёртка над Stable Baselines3 PPO.
* **SACAgent**: опционально используется для off-policy алгоритмов (SAC/TD3).

Настройка через `configs/training.yaml`. Логи и чекпойнты сохраняются в `experiments/`.

### 4. Бэктестинг (`src/backtester/`)

Скрипт `backtest.py` прогоняет сохранённую модель по историческим данным, собирает метрики:

* CAGR, Sharpe Ratio, Max Drawdown, Win Rate,
* Средняя прибыль на сделку.

Результаты экспортируются в CSV/графики.

### 5. Сервис инференса (`src/inference/`)

FastAPI/Flask endpoint `/signal`:

* Получает свежий бар (или batch) в JSON,
* Загружает модель через `model_loader.py`,
* Возвращает JSON с сигналом {-1, 0, 1} и вероятностями.

### 6. Мониторинг и логирование (`src/monitoring/`)

* **logger.py**: настраивает вывод в файл и консоль.
* **metrics.py**: собирает и экспортирует метрики в Prometheus.
* Дашборд: Grafana для PnL-кривых, частоты сигналов, latency.

### 7. Инфраструктура и CI/CD

* **Docker**: образы для обучения (`Dockerfile.training`) и инференса (`Dockerfile.base`).
* **Kubernetes**: `docker/k8s.yaml` для развёртывания сервисов.
* **GitHub Actions**: тестирование, линтеры, сборка и пуш Docker-образов.

---

## 🔧 Тестирование

* Unit-тесты для `data_pipeline`, `env` и `agents` в папке `tests/` (pytest).
* Smoke-тест обучения: короткая сессия тренировки на dummy-данных.

---

## 🤝 Вклад

1. Форкните репозиторий
2. Создайте ветку `feature/your-feature`
3. Реализуйте изменения и покрытие тестами
4. Откройте Pull Request

---

## 📄 Лицензия

Проект распространяется под лицензией MIT. Подробнее в файле `LICENSE`.
