# Исправление ошибки публикации пакета на TestPyPI

## 1. Причина возникновения ошибки

При выполнении GitHub Actions на этапе публикации пакета на TestPyPI возникает ошибка `403 Forbidden`.

В каталоге `dist/` содержатся сборки двух пакетов:

- `math-utils` версии `0.1.0`;
- `utils-subaru13` версии `0.1.0`.

При попытке публикации пакета `math-utils` сервер отклоняет запрос из-за отсутствия прав доступа. В результате выполнение сценария CI/CD прерывается.

Предположительная причина заключается в наличии старых файлов сборки в репозитории и отсутствии очистки каталога `dist/` перед выполнением новой сборки.

## 2. Удаление старых файлов сборки

Для удаления ранее созданных сборок необходимо выполнить в терминале PowerShell следующие команды (Если у Вас Windows):

```
Remove-Item -Recurse -Force dist
```

При наличии каталога `build/` его также необходимо удалить:

```
Remove-Item -Recurse -Force build
```

## 3. Настройка файла .gitignore

В корневом каталоге проекта необходимо создать или дополнить файл `.gitignore` следующим содержимым:

```
dist/
build/
*.egg-info/
__pycache__/
.pytest_cache/
.venv/
```

Это позволит исключить из системы контроля версий автоматически создаваемые файлы сборки, кэш и виртуальное окружение.

Для исключения уже отслеживаемых каталогов необходимо выполнить команду:

```
git rm -r --cached --ignore-unmatch dist build
```

## 4. Проверка параметров пакета

В файле `setup.py` необходимо проверить название и версию публикуемого пакета:

```
name="utils-subaru13",
version="0.1.0",
```


## 5. Изменение конфигурации GitHub Actions

В файле `.github/workflows/main.yml` необходимо добавить очистку каталогов перед каждой сборкой пакета. Также следует добавить проверку содержимого каталога `dist/` перед публикацией.

Исправленное содержимое файла:

```
name: Python package CI/CD

on:
  push:
    branches: [ main ]
    tags: [ 'v*.*.*' ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install build pytest numpy

      - name: Run tests
        run: pytest -q

      - name: Clean old distributions
        run: rm -rf dist build *.egg-info

      - name: Build package
        run: python -m build

  publish-to-testpypi:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Clean old distributions
        run: rm -rf dist build *.egg-info

      - name: Build package
        run: |
          python -m pip install --upgrade pip build
          python -m build

      - name: Check distributions
        run: ls -la dist/

      - name: Publish to TestPyPI
        uses: pypa/gh-action-pypi-publish@release/v1
        with:
          repository-url: https://test.pypi.org/legacy/
          password: ${{ secrets.TEST_PYPI_API_TOKEN }}
          packages-dir: dist/
          attestations: false
```

Параметр `attestations: false` добавлен для устранения предупреждения, связанного с использованием API-токена вместо Trusted Publishing.

## 6. Проверка локальной сборки

Для проверки корректности сборки необходимо выполнить следующие команды:

```
python -m pip install --upgrade build
python -m build
```

После выполнения сборки в каталоге `dist/` должны находиться только следующие файлы:

```
dist/
├── utils_subaru13-0.1.0-py3-none-any.whl
└── utils_subaru13-0.1.0.tar.gz
```

Файлы, относящиеся к пакету `math-utils`, должны отсутствовать.

## 7. Отправка изменений в удалённый репозиторий

После внесения изменений необходимо выполнить следующие команды:

```
git add .
git commit -m "Fix TestPyPI publishing"
git push origin main
```

При отправке изменений в ветку `main` автоматически запускаются тестирование и сборка пакета. Публикация выполняется только при отправке тега.

## 8. Повторный запуск публикации

Если тег `v0.1.0` уже существует, но соответствующая версия пакета ещё не опубликована, для учебного проекта допускается пересоздание тега:

```
git tag -d v0.1.0
git push origin --delete v0.1.0
git tag v0.1.0
git push origin v0.1.0
```

Если тег ранее не создавался, достаточно выполнить:

```
git tag v0.1.0
git push origin v0.1.0
```

Если версия `0.1.0` уже опубликована, необходимо изменить версию пакета в `setup.py`, например на `0.1.1`, сохранить изменения в репозитории и создать новый тег:

```
git add setup.py
git commit -m "Bump version to 0.1.1"
git push origin main
git tag v0.1.1
git push origin v0.1.1
```

## 9. Проверка API-токена

На основании журнала выполнения GitHub Actions установлено, что TestPyPI распознаёт учётную запись, но отклоняет попытку публикации пакета `math-utils`.

На данном этапе пересоздание API-токена не требуется.

Если после удаления старых сборок ошибка `403 Forbidden` сохранится, необходимо проверить:

- наличие прав на публикацию пакета `utils-subaru13`;
- область действия API-токена;
- принадлежность токена сервису TestPyPI, а не основному PyPI;
- наличие секрета `TEST_PYPI_API_TOKEN` в разделе `Settings → Secrets and variables → Actions`;
- доступность выбранного названия пакета.

## 10. Проверка результата

После отправки тега необходимо перейти во вкладку Actions репозитория и проверить выполнение заданий `build-and-test` и `publish-to-testpypi`.

При успешной публикации пакет будет доступен по адресу:

[https://test.pypi.org/project/utils-subaru13/](https://test.pypi.org/project/utils-subaru13/)

Для проверки установки пакета используется команда:

```
python -m pip install --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ utils-subaru13
```

Дополнительный индекс позволяет устанавливать зависимости, отсутствующие на TestPyPI.

## 11. Анализ остальных предупреждений

| Сообщение                               | Причина                                                               |
| --------------------------------------- | --------------------------------------------------------------------- |
| `Unable to find image locally`          | Отсутствие локального Docker-образа. Образ загружается автоматически. |
| `long_description missing`              | Отсутствие подробного описания пакета.                                |
| `long_description_content_type missing` | Не указан формат описания пакета.                                     |
| `Trusted Publishing`                    | Используется API-токен вместо механизма доверенной публикации.        |
| `403 Forbidden`                         | Отсутствие прав на публикацию пакета.                                 |

Основной причиной остановки процесса публикации является ошибка `403 Forbidden`. Остальные сообщения не препятствуют выполнению сборки и публикации.

Предложенные изменения направлены на исключение устаревших файлов сборки, предотвращение их повторного попадания в репозиторий и обеспечение корректной публикации пакета. Окончательная работоспособность конфигурации определяется по результатам повторного запуска GitHub Actions.

Уточнить причину ошибки 403Сократить повторяющиеся инструкции