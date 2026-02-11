# День 5. Python и CI/CD

[python] [cicd] [github-actions] [automation]

Автоматизация — это суть DevOps. Если вы делаете что-то руками дважды — напишите скрипт. Если вы делаете это трижды — напишите Пайплайн.
Сегодня мы объединим скриптинг на Python и автоматическую доставку кода (CI/CD).

---

## Python: Клей для DevOps

Bash идеален для простых команд, но ужасен для сложной логики. Python — это стандарт индустрии для автоматизации.

### Hygiene: Virtual Environments
Прежде чем писать код, правило №1: **Никогда не загрязняйте системный Python**.
Всегда используйте `venv`:
```bash
python3 -m venv .venv       # Создать "песочницу"
source .venv/bin/activate   # Войти в неё
pip install requests        # Установить пакеты изолированно
```

### Пример: Health Check
Скрипт, проверяющий сайт (сохраним как `uptime.py`). Если сайт упал — скрипт вернет ошибку (exit code 1).

```python
import sys
import requests

URL = "https://example.com"

try:
    r = requests.get(URL, timeout=5)
    if r.status_code == 200:
        print(f"✅ {URL} is UP")
        sys.exit(0)  # Успех
    else:
        print(f"⚠️ Status: {r.status_code}")
        sys.exit(1)  # Ошибка
except Exception as e:
    print(f"❌ Error: {e}")
    sys.exit(1)
```

---

## CI/CD Pipeline: Конвейер

**CI (Continuous Integration)**: Разработчик пушит код -> Сервер запускает тесты. Если тесты прошли, код вливается в ветку.
**CD (Continuous Delivery)**: После тестов код автоматически деплоится на тестовый сервер или продакшен.

### GitHub Actions
Это встроенный в GitHub инструмент CI/CD. Он настраивается через YAML файлы в папке `.github/workflows/`.

**Анатомия Workflow**:
1.  **Events (Triggers)**: КОГДА запускать? (на `push` или `pull_request`).
2.  **Jobs**: ЧТО делать? (Run tests, Build Docker).
3.  **Steps**: ШАГИ внутри задачи (Checkout code, Run script).

### Практика: Ваш первый Pipeline
Создайте в репозитории файл `.github/workflows/check.yml`.
Этот пайплайн будет запускать наш Python-скрипт при каждом пуше.

```yaml
name: Site Health Check

on:
  push:
    branches: [ "master" ]
  schedule:
    - cron: '*/30 * * * *' # Запускать каждые 30 минут

jobs:
  health-check:
    runs-on: ubuntu-latest  # Виртуальная машина от GitHub
    
    steps:
    # 1. Скачиваем код репозитория
    - uses: actions/checkout@v3
    
    # 2. Ставим Python
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
        
    # 3. Ставим зависимости
    - name: Install dependencies
      run: |
        pip install requests
        
    # 4. Запускаем наш скрипт
    - name: Run Uptime Script
      run: python uptime.py
```

Теперь, если вы запушите этот файл, в вкладке **Actions** на GitHub вы увидите зеленый (или красный) кружок. Вы только что создали своего робота, который работает за вас.
