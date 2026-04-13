# GigaChat PHP SDK

![GigaChat PHP SDK](https://i.postimg.cc/66jnmS1y/520455254_d44a0f88_f3d6_4c8d_9127_5d858d726cd6.png)

PHP SDK for Sber GigaChat API with Laravel support. Works with streaming and regular requests.

[![Latest Version](https://img.shields.io/packagist/v/tigusigalpa/gigachat-php.svg?style=flat-square)](https://packagist.org/packages/tigusigalpa/gigachat-php)
[![PHP Version](https://img.shields.io/packagist/php-v/tigusigalpa/gigachat-php.svg?style=flat-square)](https://packagist.org/packages/tigusigalpa/gigachat-php)
[![License](https://img.shields.io/packagist/l/tigusigalpa/gigachat-php.svg?style=flat-square)](https://packagist.org/packages/tigusigalpa/gigachat-php)

**Language:** English | [Русский](README.md)

**Other versions:** [Golang SDK](https://pkg.go.dev/github.com/tigusigalpa/gigachat-go)

## Features

- GigaChat API integration
- OAuth token management (automatic refresh)
- All GigaChat models (GigaChat, GigaChat-Pro, GigaChat-Max)
- Laravel 8-13 support (Service Provider, Facades)
- Conversations and single requests
- Streaming responses
- Image generation via text2image
- Image download and processing
- Style transfer through system prompts
- Helper methods
- Rate limiting middleware
- Artisan commands

## Installation

### From Packagist

Install via Composer:

```bash
composer require tigusigalpa/gigachat-php
```

### Laravel

The package auto-registers in Laravel. Publish the config:

```bash
php artisan vendor:publish --tag=gigachat-config
```

## Configuration

### 1. Get credentials

1. Register at [Sber AI](https://developers.sber.ru/docs/ru/gigachat/quickstart/ind-create-project)
2. Create a project, get **Client ID** and **Client Secret**
3. Generate **Authorization Key** (Base64 of "Client ID:Client Secret")

See: [Creating a project and getting keys](https://developers.sber.ru/docs/ru/gigachat/quickstart/ind-create-project)

### 2. Environment

Add to `.env`:

```env
# Option 1: Ready Authorization Key
GIGACHAT_AUTH_KEY=your_base64_encoded_auth_key

# Option 2: Client ID + Secret (auth_key generated automatically)
GIGACHAT_CLIENT_ID=your_client_id
GIGACHAT_CLIENT_SECRET=your_client_secret

# Settings
GIGACHAT_SCOPE=GIGACHAT_API_PERS
GIGACHAT_DEFAULT_MODEL=GigaChat
GIGACHAT_TEMPERATURE=0.7
GIGACHAT_MAX_TOKENS=1000

# Disable SSL verification (for certificate issues)
GIGACHAT_CERT_PATH=false
```

## Usage

### Without Laravel

```php
<?php

use Tigusigalpa\GigaChat\Auth\TokenManager;
use Tigusigalpa\GigaChat\GigaChatClient;

// Token manager
$authKey = base64_encode('your_client_id:your_client_secret');
$tokenManager = new TokenManager($authKey);

// Client
$client = new GigaChatClient($tokenManager);

// Available models
$models = $client->models();
print_r($models);

// Send message
$messages = [
    ['role' => 'user', 'content' => 'Hello! How are you?']
];

$response = $client->chat($messages);
echo $response['choices'][0]['message']['content'];
```

### With Laravel

Use the Facade:

```php
<?php

use Tigusigalpa\GigaChat\Laravel\GigaChat;
use Tigusigalpa\GigaChat\Models\GigaChatModels;

// Simple Q&A
$answer = GigaChat::ask('Tell me a joke');
echo $answer;

// List models
$models = GigaChat::models();

// With parameters
$response = GigaChat::chat([
    ['role' => 'user', 'content' => 'Explain quantum physics']
], [
    'temperature' => 0.7,
    'max_tokens' => 1000,
    'model' => GigaChatModels::GIGACHAT_2_PRO
]);

echo $response['choices'][0]['message']['content'];
```

### Conversations

```php
<?php

use Tigusigalpa\GigaChat\Laravel\GigaChat;
use Tigusigalpa\GigaChat\Laravel\GigaChatHelper;

// With system prompt
$conversation = GigaChatHelper::conversation(
    'You are a helpful programming assistant',
    'How to create REST API in Laravel?'
);

$response = GigaChat::chat($conversation);
echo GigaChatHelper::extractContent($response);

// Continue conversation
$conversation = GigaChat::continueChat($conversation, 'How to add authentication?');
```

### Streaming

```php
<?php

use Tigusigalpa\GigaChat\Laravel\GigaChat;

$messages = [
    ['role' => 'user', 'content' => 'Write a long story about space']
];

// Callback
GigaChat::chatStream($messages, [], function($event, $error) {
    if ($error) {
        echo "Error: " . $error;
        return;
    }
    
    if ($event === '[DONE]') {
        echo "\n✅ Done!";
        return;
    }
    
    if (isset($event['choices'][0]['delta']['content'])) {
        echo $event['choices'][0]['delta']['content'];
    }
});

// Generator
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

    // Generate tags
    public function generateTags(): array
    {
        return $this->generateTags('content', 5);
    }

    // Personalized content
    public function generateRelatedContent(): string
    {
        return $this->generateContent(
            'Create a similar article based on this one',
            ['title', 'category']
        );
    }
}
```

## Models

Current models: [official documentation](https://developers.sber.ru/docs/ru/gigachat/models)

### Text generation

| Model              | Description                               | Usage                           |
|--------------------|-------------------------------------------|---------------------------------|
| **GigaChat-2**     | Base second-generation model              | General tasks, dialogues        |
| **GigaChat-2-Pro** | Advanced model with enhanced capabilities | Complex tasks, creative writing |
| **GigaChat-2-Max** | Maximum model for the most complex tasks  | Professional tasks, analysis    |

### Embeddings

| Model               | Description                            | Usage                             |
|---------------------|----------------------------------------|-----------------------------------|
| **Embeddings**      | Base model for vector representation   | Semantic search, clustering       |
| **EmbeddingsGigaR** | Enhanced model for creating embeddings | Precise search, semantic analysis |

### Model constants

```php
use Tigusigalpa\GigaChat\Models\GigaChatModels;
use Tigusigalpa\GigaChat\Laravel\GigaChat;

$response = GigaChat::chat($messages, [
    'model' => GigaChatModels::GIGACHAT_2_PRO
]);

$generationModels = GigaChatModels::getGenerationModels();
$embeddingModels = GigaChatModels::getEmbeddingModels();

if (GigaChatModels::isValidGenerationModel('GigaChat-2')) {
    // valid
}
```

## Generation Parameters

```php
use Tigusigalpa\GigaChat\Models\GigaChatModels;

$options = [
    'model' => GigaChatModels::GIGACHAT_2_PRO, // Model to use
    'temperature' => 0.7,                      // Creativity (0.0 - 2.0)
    'top_p' => 0.9,                           // Nucleus sampling (0.0 - 1.0)
    'max_tokens' => 1000,                     // Maximum number of tokens
    'repetition_penalty' => 1.1,              // Repetition penalty (0.0 - 2.0)
    'update_interval' => 0                    // Update interval for streaming
];

$response = GigaChat::chat($messages, $options);
```

## Image Generation

GigaChat has built-in text2image. Use "нарисуй" (draw) in the prompt with `function_call: auto`.

### Basic

```php
<?php

use Tigusigalpa\GigaChat\Auth\TokenManager;
use Tigusigalpa\GigaChat\GigaChatClient;

$tokenManager = new TokenManager($authKey);
$client = new GigaChatClient($tokenManager);

$response = $client->generateImage("Нарисуй красивый закат над морем");

// Extract image ID
$content = $response['choices'][0]['message']['content'];
if (preg_match('/<img[^>]+src=["\']([^"\']+)["\'][^>]*>/i', $content, $matches)) {
    $fileId = $matches[1];
    $imageData = $client->downloadImage($fileId);
    file_put_contents('sunset.jpg', base64_decode($imageData));
}
```

### Stylization

```php
$response = $client->generateImage("Нарисуй розового кота", [
    'system_message' => 'You are Wassily Kandinsky'
]);

$response = $client->generateImage("Нарисуй космический корабль", [
    'system_message' => 'You are a concept artist for science fiction',
    'temperature' => 0.8
]);
```

### createImage

Generates and downloads in one call:

```php
$result = $client->createImage("Нарисуй футуристический город", [
    'system_message' => 'You are an architect of the future'
]);

// $result: content (base64), file_id, response
file_put_contents('city.jpg', base64_decode($result['content']));
```

### Laravel

```php
use Tigusigalpa\GigaChat\Laravel\GigaChat;

// drawImage adds "Нарисуй" automatically
$result = GigaChat::drawImage("beautiful landscape");

// With artist style
$result = GigaChat::drawImageInStyle("portrait of a cat", "Leonardo da Vinci");

// Extract image ID
$response = GigaChat::generateImage("Нарисуй dragon");
$imageId = GigaChat::extractImageId($response['choices'][0]['message']['content']);
if ($imageId) {
    $imageData = GigaChat::downloadImage($imageId);
    file_put_contents("dragon.jpg", base64_decode($imageData));
}
```

### Image Methods

| Method | Description | Returns |
|--------|-------------|--------|
| `generateImage($prompt, $options)` | Generate, return API response | `array` |
| `downloadImage($fileId)` | Download by ID | `string` (base64) |
| `createImage($prompt, $options)` | Generate + download | `array` |
| `drawImage($description, $options)` | Laravel: adds "Нарисуй" | `array` |
| `drawImageInStyle($description, $style, $options)` | Laravel: with artist style | `array` |
| `extractImageId($content)` | Extract ID from HTML | `string\|null` |

### Image Errors

```php
use Tigusigalpa\GigaChat\Exceptions\GigaChatException;
use Tigusigalpa\GigaChat\Exceptions\ValidationException;

try {
    $result = $client->createImage("Нарисуй дракона");
} catch (ValidationException $e) {
    // Validation errors (empty prompt, etc.)
    echo "Validation error: " . $e->getMessage();
} catch (GigaChatException $e) {
    // API errors or image ID extraction issues
    echo "Generation error: " . $e->getMessage();
}
```

## Testing

### Running Tests

```bash
# Install development dependencies
composer install --dev

# Run all tests
composer test
# or
php run-tests.php

# Run only unit tests
php run-tests.php --unit

# Run with code coverage
composer test-coverage
# or
php run-tests.php --coverage
```

### Integration Tests

To run integration tests with real API:

```bash
# Set environment variables
export GIGACHAT_CLIENT_ID=your_client_id
export GIGACHAT_CLIENT_SECRET=your_client_secret
export GIGACHAT_INTEGRATION_TEST=true

# Run integration tests
php run-tests.php --integration
```

### Coverage

- Text generation (chat, streaming)
- Image generation
- Authentication (tokens, refresh, cache)
- Laravel (facades, helpers, service provider)
- Validation
- Integration tests

See [tests/README.md](tests/README.md)

## Prompt Examples

```php
// Good prompts (contain "нарисуй")
$client->generateImage("Нарисуй закат в горах");
$client->generateImage("Нарисуй портрет кота в стиле ренессанса");
$client->generateImage("Нарисуй абстрактную композицию");

// Stylization through system_message
$client->generateImage("Нарисуй цветы", [
    'system_message' => 'You are Claude Monet, painting in impressionist style'
]);

$client->generateImage("Нарисуй робота", [
    'system_message' => 'You are a concept artist for science fiction films'
]);
```

**Note**: Prompts must contain "нарисуй" (draw). The API calls text2image when `function_call: auto` is set.

## Error Handling

Exceptions:

```php
<?php

use Tigusigalpa\GigaChat\Exceptions\GigaChatException;
use Tigusigalpa\GigaChat\Exceptions\AuthenticationException;
use Tigusigalpa\GigaChat\Exceptions\ValidationException;

try {
    $response = GigaChat::chat($messages);
} catch (AuthenticationException $e) {
    // Authentication errors (invalid keys, expired token)
    echo "Authentication error: " . $e->getMessage();
} catch (ValidationException $e) {
    // Validation errors (invalid message format)
    echo "Validation error: " . $e->getMessage();
} catch (GigaChatException $e) {
    // General GigaChat API errors
    echo "GigaChat error: " . $e->getMessage();
}
```

### Error Codes

#### Authentication (400-401)

| Code | HTTP | Description                                       | Solution                                                                      |
|------|------|---------------------------------------------------|-------------------------------------------------------------------------------|
| 1    | 400  | `scope data format invalid`                       | Check scope field format                                                      |
| 4    | 401  | `Can't decode 'Authorization' header`             | Check authorization key correctness                                           |
| 5    | 400  | `scope is empty`                                  | Specify scope: `GIGACHAT_API_PERS`, `GIGACHAT_API_B2B` or `GIGACHAT_API_CORP` |
| 6    | 401  | `credentials doesn't match db data`               | Reissue authorization key in personal account                                 |
| 7    | 401  | `scope from db not fully includes consumed scope` | Specify correct API version in scope                                          |

```php
// Example of handling authentication errors
try {
    $client = new GigaChatClient($tokenManager);
    $response = $client->chat($messages);
} catch (AuthenticationException $e) {
    $message = $e->getMessage();
    
    if (str_contains($message, 'scope is empty')) {
        echo "Scope not specified. Add GIGACHAT_API_PERS to settings.";
    } elseif (str_contains($message, 'Authorization')) {
        echo "Invalid authorization key. Check CLIENT_ID and CLIENT_SECRET.";
    } elseif (str_contains($message, 'credentials doesn\'t match')) {
        echo "Key doesn't match API version. Reissue the key.";
    }
}
```

#### Limits (402-403)

| HTTP | Description         | Cause                  | Solution                                                             |
|------|---------------------|------------------------|----------------------------------------------------------------------|
| 402  | `Payment Required`  | Model tokens exhausted | Check token limit in personal account                                |
| 403  | `Permission denied` | No access to method    | Check tariff plan (e.g., GET /balance unavailable for pay-as-you-go) |

```php
try {
    $response = $client->chat($messages);
} catch (GigaChatException $e) {
    $code = $e->getCode();
    
    switch ($code) {
        case 402:
            echo "Tokens exhausted. Top up balance or check limits.";
            break;
        case 403:
            echo "No access to this method. Check tariff plan.";
            break;
    }
}
```

#### Payload (413)

| HTTP | Description         | Cause                    | Solution                   |
|------|---------------------|--------------------------|----------------------------|
| 413  | `Payload too large` | Input data size exceeded | Reduce prompt or file size |

```php
try {
    $response = $client->generateImage($longPrompt);
} catch (GigaChatException $e) {
    if ($e->getCode() === 413) {
        echo "Prompt too long. Shorten the text.";
        // You can use POST /tokens/count to count tokens
    }
}
```

#### Parameters (422)

| HTTP | Description                                  | Cause                           | Solution                                 |
|------|----------------------------------------------|---------------------------------|------------------------------------------|
| 422  | `Requested model does not support functions` | Model doesn't support functions | Use different model or disable functions |
| 422  | `system message must be the first message`   | Wrong message order             | System message must be first             |
| 422  | `Unprocessable Entity`                       | File exceeds context size       | Split or shorten the file                |

```php
try {
    $messages = [
        ['role' => 'user', 'content' => 'Hello'],
        ['role' => 'system', 'content' => 'You are assistant'], // Wrong!
    ];
    $response = $client->chat($messages);
} catch (GigaChatException $e) {
    if (str_contains($e->getMessage(), 'system message must be the first')) {
        echo "System message must be first in the list.";
        
        // Fix the order
        $fixedMessages = [
            ['role' => 'system', 'content' => 'You are assistant'],
            ['role' => 'user', 'content' => 'Hello'],
        ];
    }
}
```

#### Rate Limit (429)

| HTTP | Description         | Cause                             | Solution                             |
|------|---------------------|-----------------------------------|--------------------------------------|
| 429  | `Too Many Requests` | Concurrent request limit exceeded | Reduce request frequency, add delays |

```php
try {
    $response = $client->chat($messages);
} catch (GigaChatException $e) {
    if ($e->getCode() === 429) {
        echo "Too many requests. Wait and retry.";
        
        // Add delay and retry
        sleep(2);
        $response = $client->chat($messages);
    }
}
```

#### Server (500)

| HTTP | Description             | Cause                  | Solution        |
|------|-------------------------|------------------------|-----------------|
| 500  | `Internal Server Error` | GigaChat service error | Contact support |

```php
try {
    $response = $client->chat($messages);
} catch (GigaChatException $e) {
    if ($e->getCode() === 500) {
        echo "GigaChat server error. Try later or contact support.";
        
        // Log for analysis
        error_log("GigaChat 500 error: " . $e->getMessage());
    }
}
```

### Retry Handler

```php
<?php

use Tigusigalpa\GigaChat\Laravel\GigaChat;
use Tigusigalpa\GigaChat\Exceptions\GigaChatException;
use Tigusigalpa\GigaChat\Exceptions\AuthenticationException;
use Tigusigalpa\GigaChat\Exceptions\ValidationException;

function handleGigaChatRequest(callable $request): array
{
    $maxRetries = 3;
    $retryDelay = 1; // seconds
    
    for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
        try {
            return $request();
            
        } catch (ValidationException $e) {
            // Don't retry validation errors
            throw $e;
            
        } catch (AuthenticationException $e) {
            if ($attempt === $maxRetries) {
                throw $e;
            }
            
            // Try to refresh token
            sleep($retryDelay);
            
        } catch (GigaChatException $e) {
            $code = $e->getCode();
            
            // Retry only for specific errors
            if (in_array($code, [429, 500]) && $attempt < $maxRetries) {
                sleep($retryDelay * $attempt); // Exponential backoff
                continue;
            }
            
            throw $e;
        }
    }
}

// Usage
try {
    $result = handleGigaChatRequest(function() {
        return GigaChat::createImage("Draw a cat");
    });
    
    echo "Image created: " . $result['file_id'];
    
} catch (Exception $e) {
    echo "Failed to create image: " . $e->getMessage();
}
```

### Debugging

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

See: [GigaChat API errors](https://developers.sber.ru/docs/ru/gigachat/api/errors-description)

## Artisan Commands

```bash
# Test API connection
php artisan gigachat:test

# Send message
php artisan gigachat:chat "Hello, how are you?"

# Send with parameters
php artisan gigachat:chat "Tell a story" --model=GigaChat-Pro --temperature=0.8 --max-tokens=500

# Streaming mode
php artisan gigachat:chat "Write a long story" --stream
```

## Rate Limiting

```php
// In routes/api.php
Route::middleware(['gigachat.rate_limit:30,1'])->group(function () {
    Route::post('/chat', [ChatController::class, 'chat']);
});

// Configuration in config/gigachat.php
'rate_limit' => [
    'enabled' => true,
    'max_attempts' => 60,        // Maximum requests
    'decay_minutes' => 1,        // Per period in minutes
],
```

## Examples

### Chatbot

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
                'You are a friendly assistant',
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

### Content Generator

```php
<?php

use Tigusigalpa\GigaChat\Laravel\GigaChat;

class ContentGenerator
{
    public function generateArticle(string $topic, string $style = 'informational'): string
    {
        return GigaChat::askWithContext(
            "You are a professional copywriter. Write in {$style} style.",
            "Write an article on the topic: {$topic}",
            ['temperature' => 0.8, 'max_tokens' => 1500]
        );
    }

    public function translateText(string $text, string $targetLang = 'English'): string
    {
        return GigaChat::ask(
            "Translate the following text to {$targetLang}:\n\n{$text}",
            ['temperature' => 0.2]
        );
    }

    public function summarizeText(string $text, int $maxWords = 100): string
    {
        return GigaChat::ask(
            "Create a brief summary (no more than {$maxWords} words):\n\n{$text}",
            ['temperature' => 0.3, 'max_tokens' => $maxWords * 2]
        );
    }
}
```

### Laravel Tests

```php
<?php

namespace Tests\Feature;

use Tests\TestCase;
use Tigusigalpa\GigaChat\Laravel\GigaChat;

class GigaChatTest extends TestCase
{
    public function test_gigachat_basic_functionality()
    {
        $response = GigaChat::ask('Hello!');
        
        $this->assertNotEmpty($response);
        $this->assertIsString($response);
    }

    public function test_gigachat_with_context()
    {
        $response = GigaChat::askWithContext(
            'You are a mathematician',
            'What is 2+2?'
        );
        
        $this->assertStringContainsString('4', $response);
    }
}
```

## FAQ

**How to get Client ID and Client Secret?**  
Register at [Sber AI](https://developers.sber.ru/docs/ru/gigachat/quickstart/ind-create-project) and create a project.

**"Invalid token response" error?**  
Check Client ID/Secret and authorization service availability.

**Custom SSL certificates?**  
Set `GIGACHAT_CERT_PATH` to certificate path or `false` to disable.

**Production ready?**  
Yes. Configure SSL and rate limiting properly.

**Pricing?**  
[Official documentation](https://developers.sber.ru/docs/ru/gigachat/api/tariffs)

### SSL Errors

```
cURL error 60: SSL certificate problem
```

```bash
# Development
GIGACHAT_CERT_PATH=false

# Production
GIGACHAT_CERT_PATH=/path/to/certificate.pem
```

Clear cache after changes:

```bash
php artisan config:clear
php artisan config:cache
```

## Full Configuration

`config/gigachat.php`:

```php
<?php

return [
    // Authorization key (Base64(Client ID:Client Secret))
    'auth_key' => env('GIGACHAT_AUTH_KEY', null),

    // Alternatively, specify Client ID and Client Secret
    'client_id' => env('GIGACHAT_CLIENT_ID', null),
    'client_secret' => env('GIGACHAT_CLIENT_SECRET', null),

    // API access scope: GIGACHAT_API_PERS | GIGACHAT_API_B2B | GIGACHAT_API_CORP
    'scope' => env('GIGACHAT_SCOPE', 'GIGACHAT_API_PERS'),

    // TLS certificate verification
    'verify' => env('GIGACHAT_CERT_PATH', true),

    // Base URIs
    'base_uri' => env('GIGACHAT_BASE_URI', 'https://gigachat.devices.sberbank.ru'),
    'oauth_uri' => env('GIGACHAT_OAUTH_URI', 'https://ngw.devices.sberbank.ru:9443'),

    // Default model
    'default_model' => env('GIGACHAT_DEFAULT_MODEL', 'GigaChat'),

    // Default generation parameters
    'default_options' => [
        'temperature' => (float) env('GIGACHAT_TEMPERATURE', 0.7),
        'max_tokens' => (int) env('GIGACHAT_MAX_TOKENS', 1000),
        'top_p' => (float) env('GIGACHAT_TOP_P', 0.9),
        'repetition_penalty' => (float) env('GIGACHAT_REPETITION_PENALTY', 1.1),
    ],

    // Rate limiting settings
    'rate_limit' => [
        'enabled' => env('GIGACHAT_RATE_LIMIT_ENABLED', true),
        'max_attempts' => (int) env('GIGACHAT_RATE_LIMIT_MAX_ATTEMPTS', 60),
        'decay_minutes' => (int) env('GIGACHAT_RATE_LIMIT_DECAY_MINUTES', 1),
    ],

    // Logging settings
    'logging' => [
        'enabled' => env('GIGACHAT_LOGGING_ENABLED', false),
        'channel' => env('GIGACHAT_LOG_CHANNEL', 'default'),
        'level' => env('GIGACHAT_LOG_LEVEL', 'info'),
    ],
];
```

## Requirements

- PHP 8.2+
- Laravel 8+ (including 11, 12)
- Guzzle HTTP 7.8.2+
- GigaChat API credentials

## License

MIT. See [LICENSE](LICENSE).

## Links

- [Create Project](https://developers.sber.ru/docs/ru/gigachat/quickstart/ind-create-project)
- [Quick Start](https://developers.sber.ru/docs/ru/gigachat/quickstart/ind-using-api)
- [API Reference](https://developers.sber.ru/docs/ru/gigachat/api/reference/rest/gigachat-api)
- [Models](https://developers.sber.ru/docs/ru/gigachat/models)
- [Pricing](https://developers.sber.ru/docs/ru/gigachat/api/tariffs)

## Support

- [GitHub Issues](https://github.com/tigusigalpa/gigachat-php/issues)
- [GitHub Discussions](https://github.com/tigusigalpa/gigachat-php/discussions)
- [API Documentation](https://developers.sber.ru/docs/ru/gigachat/api/overview)

## Contributing

1. Fork the repository
2. Create feature branch (`git checkout -b feature/name`)
3. Commit changes (`git commit -m 'Add feature'`)
4. Push (`git push origin feature/name`)
5. Open Pull Request

Follow PSR-12, add tests, update docs.

## Security

Report vulnerabilities to sovletig@gmail.com (not public issues).

## Laravel 12

Fully compatible:
- Service Provider auto-registers
- `GigaChat` Facade
- Artisan commands
- Rate limit middleware
- `HasGigaChat` trait

## Roadmap

- [ ] Response caching
- [ ] Metrics
- [ ] WebSocket
- [ ] Other PHP frameworks
