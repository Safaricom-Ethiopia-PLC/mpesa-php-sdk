# [mpesa-php-sdk] M-Pesa SDK for PHP

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://mit-license.org/Safaricom-Ethiopia-PLC)

## Overview

Easily integrate M-PESA APIs into your PHP or Laravel applications using the `Safaricom-Ethiopia-PLC/mpesa-php-sdk` package.

## Features

- **Authentication**: Generate access tokens effortlessly.
- **STK Push (NI Push)**: Initiate customer-to-business payments with USSD prompts.
- **C2B Support**: Register URLs and handle customer payments.
- **B2C Payouts**: Process business-to-customer disbursements.
- **Modular Design**: Well-structured and extensible codebase.
- **Error Handling**: Robust, developer-friendly error management.

---

## Requirements

- PHP 7.4 or higher
- cURL extension enabled
- Composer

## Installation

- ### Using Composer

  Install the SDK via Composer:

  ```bash
  composer require Safaricom-Ethiopia-PLC/mpesa-php-sdk:dev-main
  ```

- ### Using a Custom Repository

  Since this package is hosted on GitHub, add the repository to your `composer.json`:

  ```json
  "repositories": [
      {
          "type": "vcs",
          "url": "https://github.com/Safaricom-Ethiopia-PLC/mpesa-php-sdk.git"
      }
  ],
  "require": {
      "Safaricom-Ethiopia-PLC/mpesa-php-sdk": "dev-main"
  },


  "minimum-stability": "dev"
  ```

  Then run:

  ```bash
  composer update
  ```

- ### Laravel (Optional)

  If using Laravel, publish the configuration file (if supported by your SDK):

  ```bash
  php artisan vendor:publish --tag=mpesa-config
  ```

## Configuration Guide

- ### In Core PHP

  The `Authentication.php` file centralizes configuration. Customize it with:

  ```php
  $config = [
      'base_url' => 'https://apisandbox.safaricom.et/mpesa/',
      'consumer_key' => 'your_consumer_key',
      'consumer_secret' => 'your_consumer_secret',
  ];
  $securityCredential = 'your_security_credential';
  ```

- ### In Laravel

  Configuration is managed via `config/mpesa.php` in Laravel projects. After publishing the config file with `php artisan vendor:publish --tag=mpesa-config`, customize it with your credentials. Use environment variables in your `.env` file:

  ```env
  APP_ENV=development #production
  DEV_MPESA_BASE_URL=https://apisandbox.safaricom.et/
  DEV_MPESA_CONSUMER_KEY=your_consumer_key
  DEV_MPESA_CONSUMER_SECRET=your_consumer_secret
  DEV_SECURITY_CREDENTIAL=your_security_credential

  PROD_MPESA_BASE_URL=https://apisandbox.safaricom.et/
  PROD_MPESA_CONSUMER_KEY=your_consumer_key
  PROD_MPESA_CONSUMER_SECRET=your_consumer_secret
  PROD_SECURITY_CREDENTIAL=your_security_credential
  ```

  In a Laravel controller:

  ```php
  <?php

  namespace App\Http\Controllers;

  use Mpesa\Sdk\Authentication as MpesaAuth;
  use Mpesa\Sdk\Client;
  use Illuminate\Support\Facades\Cache;
  use Illuminate\Support\Facades\Log;

  class MpesaController extends Controller
  {
      private function getClient(): Client
      {
          $env = env('APP_ENV', 'development');
          $config = config("mpesa.{$env}", [
              'base_url' => 'https://apisandbox.safaricom.et/mpesa/',
              'consumer_key' => 'YOUR_CONSUMER_KEY',
              'consumer_secret' => 'YOUR_CONSUMER_SECRET',
          ]);
          return new Client(array_merge($config, [
              'timeout' => 1, 'retries' => 5, 'retry_delay' => 10,
          ]));
      }

      private function authenticate(Client $client): string
      {
          $auth = new MpesaAuth($client);
          $token = $auth->generateToken();
          Log::info("M-Pesa Auth Success", ['token' => $token->accessToken]);
          return $token->accessToken;
      }

      private function getAccessToken(): string
      {
          return Cache::remember('mpesa_access_token', 3599, fn() => $this->authenticate($this->getClient()));
      }

      public function testSdk()
      {
          try {
              return response()->json(['token' => $this->getAccessToken()]);
          } catch (\Exception $e) {
              return response()->json(['error' => $e->getMessage()], 500);
          }
      }
  }
  ```

  Api Route

  ```php
  Route::get('/test-mpesa', [MpesaController::class, 'testSdk']);

  Route::prefix('mpesa')->group(function () {
      Route::get('/auth', [MpesaController::class, 'getToken']);
      Route::post('/balance', [MpesaController::class, 'checkBalance']);
      Route::post('/b2c/payout', [MpesaController::class, 'b2cPayout']);
      Route::post('/c2b/register', [MpesaController::class, 'c2bRegister']);
      Route::post('/c2b/simulate', [MpesaController::class, 'simulateTransaction']);
      Route::post('/stk/push', [MpesaController::class, 'stkPush']);
      Route::post('/reverse', [MpesaController::class, 'reverseTransaction']);
      Route::post('/status', [MpesaController::class, 'transactionStatus']);
  });

  ```

- ### Notes

  - **Service Classes**: Each service class (e.g., `StkPushService`) encapsulates an API operation. Copy the full implementations from your project files.

  - **Logging**: Logs are written to the `logs/` directory. Ensure it exists and is writable.
  - **UUID**: Some examples use `ramsey/uuid`. Install it with:

  ```bash
  composer require ramsey/uuid
  ```

  - **Sandbox vs Production**: Update `base_url` in `Authentication.php` for production use:

  ```php
  'base_url' => 'https://apisandbox.safaricom.et/mpesa'
  ```

## Usage

This SDK provides a `Client` class for API interactions and service classes for each M-PESA API endpoint. Below are usage examples for authentication and various API operations.

### 1. Authentication

All API requests require an access token. Use `Authentication.php` to authenticate a `Client` instance:

```php
require __DIR__ . '/vendor/autoload.php';
require __DIR__ . '/Authentication.php'; // Adjust path as needed

use Mpesa\Sdk\Client;

$client = getClient();
authenticate($client, __DIR__ . '/logs/auth.log');

// The client is now ready for API requests
```

### 2. STK Push Integration

Initiate an STK Push payment request using `StkPushService`:

```php
require __DIR__ . '/vendor/autoload.php';
require __DIR__ . '/Authentication.php';
require __DIR__ . '/StkPushService.php';

use Ramsey\Uuid\Uuid;

$service = new StkPushService();
$service->run();
```

### 3. Account Balance Query

Check your account balance using `AccountBalanceService`:

```php
require __DIR__ . '/vendor/autoload.php';
require __DIR__ . '/Authentication.php';
require __DIR__ . '/AccountBalanceService.php';

$service = new AccountBalanceService();
$service->run();
```

### 4. C2B URL Registration

Register C2B callback URLs using `C2BRegisterService`:

```php
require __DIR__ . '/vendor/autoload.php';
require __DIR__ . '/Authentication.php';
require __DIR__ . '/C2BRegisterService.php';

$service = new C2BRegisterService();
$service->run();
```

### 5. B2C Payout

Initiate a B2C payout using `B2CPayOutService`:

```php
require __DIR__ . '/vendor/autoload.php';
require __DIR__ . '/Authentication.php';
require __DIR__ . '/B2CPayOutService.php';

$service = new B2CPayOutService();
$service->run();

```

### 6. Transaction Status Query

Query transaction status using `TransactionStatusService`:

```php
require __DIR__ . '/vendor/autoload.php';
require __DIR__ . '/Authentication.php';
require __DIR__ . '/TransactionStatusService.php';

$service = new TransactionStatusService();
$service->run();
```

### 7. Transaction Reversal

Reverse a transaction using `TransactionReversalService`:

```php
require __DIR__ . '/vendor/autoload.php';
require __DIR__ . '/Authentication.php';
require __DIR__ . '/TransactionReversalService.php';

$service = new TransactionReversalService();
$service->run();
```

### 8. B2C Simulation

Simulate a B2C transaction using `SimulateTransactionService`:

```php
require __DIR__ . '/vendor/autoload.php';
require __DIR__ . '/Authentication.php';
require __DIR__ . '/SimulateTransactionService.php';

$service = new SimulateTransactionService();
$service->run();
```

## Contributing

Contributions are welcome! Please follow these steps:

1. **Fork the repository** on GitHub.
2. **Clone your fork** to your local machine.

   ```bash
   git clone https://github.com/Safaricom-Ethiopia-PLC/mpesa-php-sdk.git
   ```

3. **Create a new feature branch**.

   ```bash
   git checkout -b feature/<your-feature-name>
   ```

4. **Make your changes** and commit them.

   ```bash
   git commit -am "Add new <short feature description>"
   ```

5. **Push your branch** to your fork.

   ```bash
   git push origin feature/<your-feature-name>
   ```

6. **Open a pull request** from your branch to the main repository.

## License

This project is licensed under the [MIT License](https://mit-license.org/Safaricom-Ethiopia-PLC). See the [LICENSE](LICENSE) file for more details.

---

_Happy coding with M-Pesa PHP SDK!_
