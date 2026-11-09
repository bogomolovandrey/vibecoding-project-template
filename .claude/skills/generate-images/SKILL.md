---
name: generate-images
description: Generate images for app screens using the OpenAI image API, following the project style guide. Use when the user asks to generate, create, or update images for screens, onboarding, banners, or other in-app illustrations.
disable-model-invocation: true
argument-hint: [screens: "все онбординг", "A0", "A0 A1"]
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
---

# Генерация картинок для экранов

Генерируешь картинки для приложения через OpenAI Images API и складываешь их в нужное место в едином стиле.

## Перед первым использованием

Заполни под проект два файла рядом с этим (попроси пользователя описать стиль или прислать референсы):
- [style-guide.md](style-guide.md): стиль, палитра, технические требования. Обязателен для каждого промпта.
- [prompt-templates.md](prompt-templates.md): контекст приложения и что должна передавать картинка на каждом экране.

И раздел «Конфигурация» ниже.

## Конфигурация

- **Ключ API**: `.env`, переменная `OPENAI_IMAGE_API_KEY`
- **Модель**: `.env`, переменная `OPENAI_IMAGE_MODEL` (в моих проектах `gpt-image-1.5`)
- **Папка картинок**: `_путь, например web/public/images/_`
- **Архив заменённых**: `_путь_/archive/`
- **Где лежат тексты экранов**: `_файлы с контентом, из которых берётся смысл экрана_`

## Порядок работы

### Шаг 1. Разобрать запрос из $ARGUMENTS

Определи, каким экранам нужны картинки: конкретные ID (`A0`, `A1 A1.1`) или группы («все онбординг»). Если непонятно, уточни у пользователя.

### Шаг 2. Прочитать контент экранов

Прочитай тексты экранов, чтобы понять смысл каждого.

### Шаг 3. Собрать промпт для каждого экрана

Прочитай style-guide.md и prompt-templates.md (пользователь мог их обновить). Для каждого экрана:

1. Базовое описание стиля из style-guide.md
2. Визуальное описание экрана по prompt-templates.md и тексту экрана
3. Обязательная палитра
4. В конце: `No text, no lettering, no watermarks.` и требования к фону из style-guide.md

Промпт на английском, подробный (150-250 слов), описывает конкретные визуальные элементы, а не абстракции.

### Шаг 4. Сгенерировать

```bash
API_KEY=$(grep OPENAI_IMAGE_API_KEY .env | cut -d= -f2)
MODEL=$(grep OPENAI_IMAGE_MODEL .env | cut -d= -f2)

curl -s -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$MODEL"'",
    "prompt": "<COMPOSED_PROMPT>",
    "size": "1024x1024",
    "background": "transparent",
    "quality": "high"
  }' --max-time 120
```

В ответе `data[0].b64_json`: PNG в base64.

### Шаг 5. Сохранить оригинал и веб-версию

Каждая картинка сохраняется в двух вариантах:
1. **Оригинал** в подпапке `originals/`
2. **Веб-версия** (уменьшенная) в основной папке, её использует приложение

Имя файла: `{section}-{screenId}-{short-description}.png`. Варианты одного экрана: `-v1`, `-v2`.

```bash
mkdir -p <IMAGES_DIR>/<section>/originals
# старую версию в архив
mv <IMAGES_DIR>/<section>/<name>.png <IMAGES_DIR>/archive/<name>_$(date +%Y%m%d_%H%M%S).png 2>/dev/null
# оригинал
echo "$BASE64_DATA" | base64 --decode > <IMAGES_DIR>/<section>/originals/<name>.png
# веб-версия: максимум 480 px по высоте (macOS sips; на Linux: convert -resize x480)
sips -Z 480 <IMAGES_DIR>/<section>/originals/<name>.png --out <IMAGES_DIR>/<section>/<name>.png
```

Размеры API: `1024x1024` (квадрат), `1536x1024` (альбом), `1024x1536` (портрет).

### Шаг 6. Прописать картинку в контент

Обнови поле картинки у экрана в файлах контента (через Edit, не через Bash).

### Шаг 7. Отчёт

- Какие экраны получили картинки
- Пути к файлам
- Какие файлы контента обновлены
- Ошибки (API, таймауты), если были

## Ошибки

- API вернул ошибку: покажи сообщение и переходи к следующему экрану
- Нет ключа: попроси пользователя добавить `OPENAI_IMAGE_API_KEY` в `.env`
- Генерация дольше 120 с: отметь таймаут и переходи к следующему
- Никогда не показывай ключ целиком, максимум первые 8 символов

## Важно

- Генерируй по одной картинке, не параллельно: так не упрёшься в лимиты API
- Проверяй, что файл сохранился и весит больше 10 КБ
