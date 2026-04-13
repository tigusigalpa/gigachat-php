# GigaChat PHP SDK

![GigaChat PHP SDK](https://i.postimg.cc/66jnmS1y/520455254_d44a0f88_f3d6_4c8d_9127_5d858d726cd6.png)

PHP SDK для Sber GigaChat API с поддержкой Laravel. Streaming и обычные запросы.

[![Latest Version](https://img.shields.io/packagist/v/tigusigalpa/gigachat-php.svg?style=flat-square)](https://packagist.org/packages/tigusigalpa/gigachat-php)
[![PHP Version](https://img.shields.io/packagist/php-v/tigusigalpa/gigachat-php.svg?style=flat-square)](https://packagist.org/packages/tigusigalpa/gigachat-php)
[![License](https://img.shields.io/packagist/l/tigusigalpa/gigachat-php.svg?style=flat-square)](https://packagist.org/packages/tigusigalpa/gigachat-php)

**Язык:** Русский | [English](README-en.md)

**Другие версии:** [Golang SDK](https://pkg.go.dev/github.com/tigusigalpa/gigachat-go)

## Возможности

- Интеграция с GigaChat API
- Управление OAuth-токенами (автообновление)
- Все модели GigaChat (GigaChat, GigaChat-Pro, GigaChat-Max)
- Laravel 8-13 (Service Provider, Facades)
- Диалоги и одиночные запросы
- Streaming-ответы
- Генерация изображений через text2image
- Скачивание и обработка изображений
- Стилизация через системные промпты
- Helper-методы
- Rate limiting middleware
- Artisan-команды

## Установка

### Из Packagist

Через Composer:

```bash
composer require tigusigalpa/gigachat-php
```

### Laravel

Пакет регистрируется автоматически. Опубликуйте конфиг:

```bash
php artisan vendor:publish --tag=gigachat-config
```

## Настройка

### 1. Получение ключей

1. Зарегистрируйтесь в [Sber AI](https://developers.sber.ru/docs/ru/gigachat/quickstart/ind-create-project)
2. Создайте проект, получите **Client ID** и **Client Secret**
3. Сгенерируйте **Authorization Key** (Base64 от "Client ID:Client Secret")

См.: [Создание проекта и получение ключей](https://developers.sber.ru/docs/ru/gigachat/quickstart/ind-create-project)

### 2. Окружение

Добавьте в `.env`:

```env
# Вариант 1: Готовый Authorization Key
GIGACHAT_AUTH_KEY=your_base64_encoded_auth_key

# Вариант 2: Client ID + Secret (auth_key генерируется автоматически)
GIGACHAT_CLIENT_ID=your_client_id
GIGACHAT_CLIENT_SECRET=your_client_secret

# Настройки
GIGACHAT_SCOPE=GIGACHAT_API_PERS
GIGACHAT_DEFAULT_MODEL=GigaChat
GIGACHAT_TEMPERATURE=0.7
GIGACHAT_MAX_TOKENS=1000

# Отключить проверку SSL (при проблемах с сертификатами)
GIGACHAT_CERT_PATH=false
```

## Использование

### Без Laravel

```php
<?php

use Tigusigalpa\GigaChat\Auth\TokenManager;
use Tigusigalpa\GigaChat\GigaChatClient;

// Токен-менеджер
$authKey = base64_encode('your_client_id:your_client_secret');
$tokenManager = new TokenManager($authKey);

// Клиент
$client = new GigaChatClient($tokenManager);

// Доступные модели
$models = $client->models();
print_r($models);

// Отправка сообщения
$messages = [
    ['role' => 'user', 'content' => 'Привет! Как дела?']
];

$response = $client->chat($messages);
echo $response['choices'][0]['message']['content'];
```

### С Laravel

Используйте Facade:

```php
<?php

use Tigusigalpa\GigaChat\Laravel\GigaChat;
use Tigusigalpa\GigaChat\Models\GigaChatModels;

// Вопрос-ответ
$answer = GigaChat::ask('Расскажи анекдот');
echo $answer;

// Список моделей
$models = GigaChat::models();

// С параметрами
$response = GigaChat::chat([
    ['role' => 'user', 'content' => 'Объясни квантовую физику']
], [
    'temperature' => 0.7,
    'max_tokens' => 1000,
    'model' => GigaChatModels::GIGACHAT_2_PRO
]);

echo $response['choices'][0]['message']['content'];
```

### Диалоги

```php
<?php

use Tigusigalpa\GigaChat\Laravel\GigaChat;
use Tigusigalpa\GigaChat\Laravel\GigaChatHelper;

// С системным промптом
$conversation = GigaChatHelper::conversation(
    'Ты полезный помощник программиста',
    'Как создать REST API в Laravel?'
);

$response = GigaChat::chat($conversation);
echo GigaChatHelper::extractContent($response);

// Продолжение диалога
$conversation = GigaChat::continueChat($conversation, 'А как добавить аутентификацию?');
```

### Streaming

```php
<?php

use Tigusigalpa\GigaChat\Laravel\GigaChat;

$messages = [
    ['role' => 'user', 'content' => 'Напиши длинную историю о космосе']
];

// Callback
GigaChat::chatStream($messages, [], function($event, $error) {
    if ($error) {
        echo "Ошибка: " . $error;
        return;
    }
    
    if ($event === '[DONE]') {
        echo "\n✅ Готово!";
        return;
    }
    
    if (isset($event['choices'][0]['delta']['content'])) {
        echo $event['choices'][0]['delta']['content'];
    }
});

// Генератор
$stream = GigaChat::chatStream($messages);
foreach ($stream as $event) {
    if (isset($event['choices'][0]['delta']['content'])) {
        echo $event['choices'][0]['delta']['content'];
    }
}
```

### Eloquent Trait

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Tigusigalpa\GigaChat\Laravel\Traits\HasGigaChat;

class Article extends Model
{
    use HasGigaChat;

    protected $fillable = ['title', 'content', 'category'];

    public function generateSummary(): string
    {
        return $this->summarize('content');
    }

    // Генерация тегов
    public function generateTags(): array
    {
        return $this->generateTags('content', 5);
    }

    // Персонализированный контент
    public function generateRelatedContent(): string
    {
        return $this->generateContent(
            'Создай похожую статью на основе этой',
            ['title', 'category']
        );
    }
}
```

## Модели

Актуальный список: [официальная документация](https://developers.sber.ru/docs/ru/gigachat/models)

### Генерация текста

| Модель             | Описание                                       | Использование                     |
|--------------------|------------------------------------------------|-----------------------------------|
| **GigaChat-2**     | Базовая модель второго поколения               | Общие задачи, диалоги             |
| **GigaChat-2-Pro** | Продвинутая модель с улучшенными возможностями | Сложные задачи, креативное письмо |
| **GigaChat-2-Max** | Максимальная модель для самых сложных задач    | Профессиональные задачи, анализ   |

### Эмбеддинги

| Модель              | Описание                                    | Использование                      |
|---------------------|---------------------------------------------|------------------------------------|
| **Embeddings**      | Базовая модель для векторного представления | Поиск по смыслу, кластеризация     |
| **EmbeddingsGigaR** | Улучшенная модель для создания эмбеддингов  | Точный поиск, семантический анализ |

### Константы моделей

```php
use Tigusigalpa\GigaChat\Models\GigaChatModels;
use Tigusigalpa\GigaChat\Laravel\GigaChat;

$response = GigaChat::chat($messages, [
    'model' => GigaChatModels::GIGACHAT_2_PRO
]);

$generationModels = GigaChatModels::getGenerationModels();
$embeddingModels = GigaChatModels::getEmbeddingModels();

if (GigaChatModels::isValidGenerationModel('GigaChat-2')) {
    // валидна
}
```

## Параметры генерации

```php
use Tigusigalpa\GigaChat\Models\GigaChatModels;

$options = [
    'model' => GigaChatModels::GIGACHAT_2_PRO, // Модель для использования
    'temperature' => 0.7,                      // Креативность (0.0 - 2.0)
    'top_p' => 0.9,                           // Nucleus sampling (0.0 - 1.0)
    'max_tokens' => 1000,                     // Максимальное количество токенов
    'repetition_penalty' => 1.1,              // Штраф за повторения (0.0 - 2.0)
    'update_interval' => 0                    // Интервал обновления для streaming
];

$response = GigaChat::chat($messages, $options);

## Генерация изображений

Встроенная функция text2image. Используйте "нарисуй" в промпте с `function_call: auto`.

### Базовое

```php
<?php

use Tigusigalpa\GigaChat\Auth\TokenManager;
use Tigusigalpa\GigaChat\GigaChatClient;

$tokenManager = new TokenManager($authKey);
$client = new GigaChatClient($tokenManager);

$response = $client->generateImage("Нарисуй красивый закат над морем");

// Извлечь ID изображения
$content = $response['choices'][0]['message']['content'];
if (preg_match('/<img[^>]+src=["']([^"']+)["'][^>]*>/i', $content, $matches)) {
    $fileId = $matches[1];
    $imageData = $client->downloadImage($fileId);
    file_put_contents('sunset.jpg', base64_decode($imageData));
}

### Стилизация

```php
$response = $client->generateImage("Нарисуй розового кота", [
    'system_message' => 'Ты — Василий Кандинский'
]);

$response = $client->generateImage("Нарисуй космический корабль", [
    'system_message' => 'Ты — художник-концептуалист научной фантастики',
    'temperature' => 0.8
]);

### createImage

Генерирует и скачивает в одном вызове:

```php
$result = $client->createImage("Нарисуй футуристический город", [
    'system_message' => 'Ты — архитектор будущего'
]);

// $result: content (base64), file_id, response
file_put_contents('city.jpg', base64_decode($result['content']));

### Laravel

```php
use Tigusigalpa\GigaChat\Laravel\GigaChat;

// drawImage добавляет "Нарисуй" автоматически
$result = GigaChat::drawImage("красивый пейзаж");

// Со стилем художника
$result = GigaChat::drawImageInStyle("портрет кота", "Леонардо да Винчи");

// Извлечь ID изображения
$response = GigaChat::generateImage("Нарисуй дракона");
$imageId = GigaChat::extractImageId($response['choices'][0]['message']['content']);
if ($imageId) {
    $imageData = GigaChat::downloadImage($imageId);
    file_put_contents("dragon.jpg", base64_decode($imageData));
}

```

### Методы изображений

| Метод | Описание | Возвращает |
|--------|-------------|--------|
| `generateImage($prompt, $options)` | Генерация, ответ API | `array` |
| `downloadImage($fileId)` | Скачать по ID | `string` (base64) |
| `createImage($prompt, $options)` | Генерация + скачивание | `array` |
| `drawImage($description, $options)` | Laravel: добавляет "Нарисуй" | `array` |
| `drawImageInStyle($description, $style, $options)` | Laravel: со стилем художника | `array` |
| `extractImageId($content)` | Извлечь ID из HTML | `string\|null` |

### Ошибки изображений

```php
use Tigusigalpa\GigaChat\Exceptions\GigaChatException;
use Tigusigalpa\GigaChat\Exceptions\ValidationException;

try {
    $result = $client->createImage("Нарисуй дракона");
} catch (ValidationException $e) {
    echo "Ошибка валидации: " . $e->getMessage();
} catch (GigaChatException $e) {
    echo "Ошибка генерации: " . $e->getMessage();
}
```

**Важно**: Промпт должен содержать "нарисуй". API вызывает text2image при `function_call: auto`.

## Обработка ошибок

Исключения:

```php
<?php

use Tigusigalpa\GigaChat\Exceptions\GigaChatException;
use Tigusigalpa\GigaChat\Exceptions\AuthenticationException;
use Tigusigalpa\GigaChat\Exceptions\ValidationException;

try {
    $response = GigaChat::chat($messages);
} catch (AuthenticationException $e) {
    // Ошибки авторизации (неверные ключи, истекший токен)
    echo "Ошибка авторизации: " . $e->getMessage();
} catch (ValidationException $e) {
    // Ошибки валидации (неверный формат сообщений)
    echo "Ошибка валидации: " . $e->getMessage();
} catch (GigaChatException $e) {
    // Общие ошибки GigaChat API
    echo "Ошибка GigaChat: " . $e->getMessage();
}
```

### Коды ошибок

#### Авторизация (400-401)

| Код | HTTP | Описание                                          | Решение                                                                        |
|-----|------|---------------------------------------------------|--------------------------------------------------------------------------------|
| 1   | 400  | `scope data format invalid`                       | Проверьте формат поля scope                                                    |
| 4   | 401  | `Can't decode 'Authorization' header`             | Проверьте корректность ключа авторизации                                       |
| 5   | 400  | `scope is empty`                                  | Укажите scope: `GIGACHAT_API_PERS`, `GIGACHAT_API_B2B` или `GIGACHAT_API_CORP` |
| 6   | 401  | `credentials doesn't match db data`               | Перевыпустите ключ авторизации в личном кабинете                               |
| 7   | 401  | `scope from db not fully includes consumed scope` | Укажите корректную версию API в scope                                          |

```php
// Пример обработки ошибок авторизации
try {
    $client = new GigaChatClient($tokenManager);
    $response = $client->chat($messages);
} catch (AuthenticationException $e) {
    $message = $e->getMessage();
    
    if (str_contains($message, 'scope is empty')) {
        echo "Не указан scope. Добавьте GIGACHAT_API_PERS в настройки.";
    } elseif (str_contains($message, 'Authorization')) {
        echo "Неверный ключ авторизации. Проверьте CLIENT_ID и CLIENT_SECRET.";
    } elseif (str_contains($message, 'credentials doesn\'t match')) {
        echo "Ключ не соответствует версии API. Перевыпустите ключ.";
    }
}
```

#### Лимиты (402-403)

| HTTP | Описание            | Причина                   | Решение                                                                       |
|------|---------------------|---------------------------|-------------------------------------------------------------------------------|
| 402  | `Payment Required`  | Закончились токены модели | Проверьте лимит токенов в личном кабинете                                     |
| 403  | `Permission denied` | Нет доступа к методу      | Проверьте тарифный план (например, GET /balance недоступен для pay-as-you-go) |

```php
try {
    $response = $client->chat($messages);
} catch (GigaChatException $e) {
    $code = $e->getCode();
    
    switch ($code) {
        case 402:
            echo "Закончились токены. Пополните баланс или проверьте лимиты.";
            break;
        case 403:
            echo "Нет доступа к этому методу. Проверьте тарифный план.";
            break;
    }
}
```

#### Размер данных (413)

| HTTP | Описание            | Причина                        | Решение                             |
|------|---------------------|--------------------------------|-------------------------------------|
| 413  | `Payload too large` | Превышен размер входных данных | Уменьшите размер промпта или файлов |

```php
try {
    $response = $client->generateImage($longPrompt);
} catch (GigaChatException $e) {
    if ($e->getCode() === 413) {
        echo "Промпт слишком длинный. Сократите текст.";
        // Можно использовать POST /tokens/count для подсчета токенов
    }
}
```

#### Параметры (422)

| HTTP | Описание                                     | Причина                         | Решение                                         |
|------|----------------------------------------------|---------------------------------|-------------------------------------------------|
| 422  | `Requested model does not support functions` | Модель не поддерживает функции  | Используйте другую модель или отключите функции |
| 422  | `system message must be the first message`   | Неверный порядок сообщений      | Системное сообщение должно быть первым          |
| 422  | `Unprocessable Entity`                       | Файл превышает размер контекста | Разделите или сократите файл                    |

```php
try {
    $messages = [
        ['role' => 'user', 'content' => 'Привет'],
        ['role' => 'system', 'content' => 'Ты помощник'], // Неверно!
    ];
    $response = $client->chat($messages);
} catch (GigaChatException $e) {
    if (str_contains($e->getMessage(), 'system message must be the first')) {
        echo "Системное сообщение должно быть первым в списке.";
        
        // Исправляем порядок
        $fixedMessages = [
            ['role' => 'system', 'content' => 'Ты помощник'],
            ['role' => 'user', 'content' => 'Привет'],
        ];
    }
}
```

#### Rate Limit (429)

| HTTP | Описание            | Причина                               | Решение                                       |
|------|---------------------|---------------------------------------|-----------------------------------------------|
| 429  | `Too Many Requests` | Превышен лимит одновременных запросов | Уменьшите частоту запросов, добавьте задержки |

```php
try {
    $response = $client->chat($messages);
} catch (GigaChatException $e) {
    if ($e->getCode() === 429) {
        echo "Слишком много запросов. Подождите и повторите.";
        
        // Добавляем задержку и повторяем
        sleep(2);
        $response = $client->chat($messages);
    }
}
```

#### Сервер (500)

| HTTP | Описание                | Причина                 | Решение                       |
|------|-------------------------|-------------------------|-------------------------------|
| 500  | `Internal Server Error` | Ошибка сервиса GigaChat | Обратитесь в службу поддержки |

```php
try {
    $response = $client->chat($messages);
} catch (GigaChatException $e) {
    if ($e->getCode() === 500) {
        echo "Ошибка сервера GigaChat. Попробуйте позже или обратитесь в поддержку.";
        
        // Логируем для анализа
        error_log("GigaChat 500 error: " . $e->getMessage());
    }
}
```

### Retry-обработчик

```php
<?php

use Tigusigalpa\GigaChat\Laravel\GigaChat;
use Tigusigalpa\GigaChat\Exceptions\GigaChatException;
use Tigusigalpa\GigaChat\Exceptions\AuthenticationException;
use Tigusigalpa\GigaChat\Exceptions\ValidationException;

function handleGigaChatRequest(callable $request): array
{
    $maxRetries = 3;
    $retryDelay = 1; // секунды
    
    for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
        try {
            return $request();
            
        } catch (ValidationException $e) {
            // Ошибки валидации не повторяем
            throw $e;
            
        } catch (AuthenticationException $e) {
            if ($attempt === $maxRetries) {
                throw $e;
            }
            
            // Пытаемся обновить токен
            sleep($retryDelay);
            
        } catch (GigaChatException $e) {
            $code = $e->getCode();
            
            // Повторяем только для определенных ошибок
            if (in_array($code, [429, 500]) && $attempt < $maxRetries) {
                sleep($retryDelay * $attempt); // Экспоненциальная задержка
                continue;
            }
            
            throw $e;
        }
    }
}

// Использование
try {
    $result = handleGigaChatRequest(function() {
        return GigaChat::createImage("Нарисуй кота");
    });
    
    echo "Изображение создано: " . $result['file_id'];
    
} catch (Exception $e) {
    echo "Не удалось создать изображение: " . $e->getMessage();
}
```

### Отладка

```php
try {
    $response = $client->chat($messages);
} catch (GigaChatException $e) {
    error_log('GigaChat Error: ' . json_encode([
        'message' => $e->getMessage(),
        'code' => $e->getCode(),
    ], JSON_UNESCAPED_UNICODE));
}
```

См.: [Ошибки GigaChat API](https://developers.sber.ru/docs/ru/gigachat/api/errors-description)

## Artisan-команды

```bash
# Тестирование подключения к API
php artisan gigachat:test

# Отправка сообщения
php artisan gigachat:chat "Привет, как дела?"

# Отправка с параметрами
php artisan gigachat:chat "Расскажи историю" --model=GigaChat-Pro --temperature=0.8 --max-tokens=500

# Streaming режим
php artisan gigachat:chat "Напиши длинный рассказ" --stream
```

## Rate Limiting

```php
// В routes/api.php
Route::middleware(['gigachat.rate_limit:30,1'])->group(function () {
    Route::post('/chat', [ChatController::class, 'chat']);
});

// Настройка в config/gigachat.php
'rate_limit' => [
    'enabled' => true,
    'max_attempts' => 60,        // Максимум запросов
    'decay_minutes' => 1,        // За период в минутах
],
```

## Тестирование

### Запуск

```bash
# Установка зависимостей для разработки
composer install --dev

# Запуск всех тестов
composer test
# или
php run-tests.php

# Запуск только unit тестов
php run-tests.php --unit

# Запуск с покрытием кода
composer test-coverage
# или
php run-tests.php --coverage
```

### Интеграционные тесты

Для запуска интеграционных тестов с реальным API:

```bash
# Установите переменные окружения
export GIGACHAT_CLIENT_ID=your_client_id
export GIGACHAT_CLIENT_SECRET=your_client_secret
export GIGACHAT_INTEGRATION_TEST=true

# Запустите интеграционные тесты
php run-tests.php --integration
```

### Покрытие

- Генерация текста (chat, streaming)
- Генерация изображений
- Аутентификация (токены, обновление, кеш)
- Laravel (фасады, helpers, service provider)
- Валидация
- Интеграционные тесты

См. [tests/README.md](tests/README.md)

## Примеры

### Чат-бот

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Tigusigalpa\GigaChat\Laravel\GigaChat;

class ChatController extends Controller
{
    public function chat(Request $request)
    {
        $request->validate([
            'message' => 'required|string|max:2000'
        ]);

        try {
            $response = GigaChat::askWithContext(
                'Ты дружелюбный помощник',
                $request->input('message'),
                ['temperature' => 0.7]
            );

            return response()->json([
                'success' => true,
                'reply' => $response
            ]);
        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'error' => $e->getMessage()
            ], 500);
        }
    }
}
```

### Генерация контента

```php
<?php

use Tigusigalpa\GigaChat\Laravel\GigaChat;

class ContentGenerator
{
    public function generateArticle(string $topic, string $style = 'информационный'): string
    {
        return GigaChat::askWithContext(
            "Ты профессиональный копирайтер. Пиши в {$style} стиле.",
            "Напиши статью на тему: {$topic}",
            ['temperature' => 0.8, 'max_tokens' => 1500]
        );
    }

    public function translateText(string $text, string $targetLang = 'английский'): string
    {
        return GigaChat::ask(
            "Переведи следующий текст на {$targetLang} язык:\n\n{$text}",
            ['temperature' => 0.2]
        );
    }

    public function summarizeText(string $text, int $maxWords = 100): string
    {
        return GigaChat::ask(
            "Создай краткое изложение (не более {$maxWords} слов):\n\n{$text}",
            ['temperature' => 0.3, 'max_tokens' => $maxWords * 2]
        );
    }
}
```

### Streaming

```php
<?php

// В контроллере
public function streamChat(Request $request)
{
    $messages = [['role' => 'user', 'content' => $request->input('message')]];

    return response()->stream(function () use ($messages) {
        echo "data: " . json_encode(['type' => 'start']) . "\n\n";
        
        GigaChat::chatStream($messages, [], function($event, $error) {
            if ($error) {
                echo "data: " . json_encode(['type' => 'error', 'error' => $error]) . "\n\n";
                return;
            }
            
            if ($event === '[DONE]') {
                echo "data: " . json_encode(['type' => 'done']) . "\n\n";
                return;
            }
            
            if (isset($event['choices'][0]['delta']['content'])) {
                echo "data: " . json_encode([
                    'type' => 'content',
                    'content' => $event['choices'][0]['delta']['content']
                ]) . "\n\n";
            }
            
            flush();
        });
    }, 200, [
        'Content-Type' => 'text/event-stream',
        'Cache-Control' => 'no-cache',
    ]);
}
```

### Laravel-тесты

```php
<?php

namespace Tests\Feature;

use Tests\TestCase;
use Tigusigalpa\GigaChat\Laravel\GigaChat;

class GigaChatTest extends TestCase
{
    public function test_gigachat_basic_functionality()
    {
        $response = GigaChat::ask('Привет!');
        
        $this->assertNotEmpty($response);
        $this->assertIsString($response);
    }

    public function test_gigachat_with_context()
    {
        $response = GigaChat::askWithContext(
            'Ты математик',
            'Сколько будет 2+2?'
        );
        
        $this->assertStringContainsString('4', $response);
    }
}
```

## FAQ

**Как получить Client ID и Client Secret?**  
Зарегистрируйтесь в [Sber AI](https://developers.sber.ru/docs/ru/gigachat/quickstart/ind-create-project) и создайте проект.

**Ошибка "Invalid token response"?**  
Проверьте Client ID/Secret и доступность сервиса авторизации.

**SSL-сертификаты?**  
Установите `GIGACHAT_CERT_PATH` в путь к сертификату или `false`.

**Production?**  
Да. Настройте SSL и rate limiting.

**Тарифы?**  
[Официальная документация](https://developers.sber.ru/docs/ru/gigachat/api/tariffs)

### SSL-ошибки

```
cURL error 60: SSL certificate problem
```

```bash
# Разработка
GIGACHAT_CERT_PATH=false

# Production
GIGACHAT_CERT_PATH=/path/to/certificate.pem
```

Очистите кэш после изменений:

```bash
php artisan config:clear
php artisan config:cache
```

## Полная конфигурация

`config/gigachat.php`:

```php
<?php

return [
    // Authorization key (Base64(Client ID:Client Secret))
    'auth_key' => env('GIGACHAT_AUTH_KEY', null),

    // Альтернативно, укажите Client ID и Client Secret
    'client_id' => env('GIGACHAT_CLIENT_ID', null),
    'client_secret' => env('GIGACHAT_CLIENT_SECRET', null),

    // Область доступа API: GIGACHAT_API_PERS | GIGACHAT_API_B2B | GIGACHAT_API_CORP
    'scope' => env('GIGACHAT_SCOPE', 'GIGACHAT_API_PERS'),

    // Проверка TLS сертификатов
    'verify' => env('GIGACHAT_CERT_PATH', true),

    // Базовые URI
    'base_uri' => env('GIGACHAT_BASE_URI', 'https://gigachat.devices.sberbank.ru'),
    'oauth_uri' => env('GIGACHAT_OAUTH_URI', 'https://ngw.devices.sberbank.ru:9443'),

    // Модель по умолчанию
    'default_model' => env('GIGACHAT_DEFAULT_MODEL', 'GigaChat'),

    // Параметры генерации по умолчанию
    'default_options' => [
        'temperature' => (float) env('GIGACHAT_TEMPERATURE', 0.7),
        'max_tokens' => (int) env('GIGACHAT_MAX_TOKENS', 1000),
        'top_p' => (float) env('GIGACHAT_TOP_P', 0.9),
        'repetition_penalty' => (float) env('GIGACHAT_REPETITION_PENALTY', 1.1),
    ],

    // Rate limiting настройки
    'rate_limit' => [
        'enabled' => env('GIGACHAT_RATE_LIMIT_ENABLED', true),
        'max_attempts' => (int) env('GIGACHAT_RATE_LIMIT_MAX_ATTEMPTS', 60),
        'decay_minutes' => (int) env('GIGACHAT_RATE_LIMIT_DECAY_MINUTES', 1),
    ],

    // Настройки логирования
    'logging' => [
        'enabled' => env('GIGACHAT_LOGGING_ENABLED', false),
        'channel' => env('GIGACHAT_LOG_CHANNEL', 'default'),
        'level' => env('GIGACHAT_LOG_LEVEL', 'info'),
    ],
];
```

## Требования

- PHP 8.2+
- Laravel 8+ (включая 11, 12)
- Guzzle HTTP 7.8.2+
- Учетные данные GigaChat API

## Лицензия

MIT. См. [LICENSE](LICENSE).

## Ссылки

- [Создание проекта](https://developers.sber.ru/docs/ru/gigachat/quickstart/ind-create-project)
- [Быстрый старт](https://developers.sber.ru/docs/ru/gigachat/quickstart/ind-using-api)
- [Справочник API](https://developers.sber.ru/docs/ru/gigachat/api/reference/rest/gigachat-api)
- [Модели](https://developers.sber.ru/docs/ru/gigachat/models)
- [Тарифы](https://developers.sber.ru/docs/ru/gigachat/api/tariffs)

## Поддержка

- [GitHub Issues](https://github.com/tigusigalpa/gigachat-php/issues)
- [GitHub Discussions](https://github.com/tigusigalpa/gigachat-php/discussions)
- [Документация API](https://developers.sber.ru/docs/ru/gigachat/api/overview)

## Участие

1. Форкните репозиторий
2. Создайте ветку (`git checkout -b feature/name`)
3. Зафиксируйте изменения (`git commit -m 'Add feature'`)
4. Отправьте (`git push origin feature/name`)
5. Откройте Pull Request

Следуйте PSR-12, добавляйте тесты, обновляйте документацию.

## Безопасность

Уязвимости отправляйте на sovletig@gmail.com (не в публичные issues).

## Laravel 12

Полная совместимость:
- Service Provider авторегистрация
- `GigaChat` Facade
- Artisan-команды
- Rate limit middleware
- `HasGigaChat` trait

## Roadmap

- [ ] Кэширование
- [ ] Метрики
- [ ] WebSocket
- [ ] Другие PHP-фреймворки
