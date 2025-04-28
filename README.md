# Using Proxies in Laravel

[![Bright Data Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.com/)

This guide explains how to configure and implement proxies in a Laravel project for unblocked web scraping and geographic access control.

- [What Is a Laravel Proxy?](#what-is-a-laravel-proxy)
- [Use Cases for Proxies in Laravel](#use-cases-for-proxies-in-laravel)
- [Using a Proxy in Laravel: Step-By-Step Guide](#using-a-proxy-in-laravel-step-by-step-guide)
- [Advanced Use Cases](#advanced-use-cases)
- [Use a Bright Data Proxy in Laravel](#use-a-bright-data-proxy-in-laravel)
- [Extra: Symfony's `HttpClient` Proxy Integration](#extra-symfonys-httpclient-proxy-integration)

## What Is a Laravel Proxy?

A Laravel proxy functions as a middleman between your Laravel application and an external server. It allows you to programmatically direct your server's traffic [through a proxy server](https://brightdata.com/blog/proxy-101/what-is-proxy-server) to conceal your IP address.

Here's how proxies work in Laravel:

1. Laravel initiates an HTTP request using an HTTP client library with proxy configuration.
2. The request travels through the proxy server.
3. The proxy transmits it to the destination server.
4. The destination server sends a response back to the proxy.
5. The proxy relays the response to Laravel.

As a result, the destination server sees the request originating from the proxy's IP—not your Laravel server's address. This mechanism enables bypassing geographic restrictions, enhancing anonymity, and handling rate limitations.

## Use Cases for Proxies in Laravel

Proxies in Laravel serve numerous purposes, but these three are particularly common:

- **Web scraping**: Implement proxies to prevent IP blocks, circumvent rate limits, or avoid other restrictions when creating a web scraping API. For additional information, read our tutorial on [web scraping with Laravel](https://brightdata.com/blog/web-data/web-scraping-with-laravel).
- **Bypassing rate limits on third-party APIs**: Alternate between proxy IPs to remain within API usage quotas and prevent throttling.
- **Accessing geo-restricted content**: Choose proxy servers in specific locations to use services only available in certain countries.

For additional examples, consult our guide on [web data and proxy use cases](https://brightdata.com/use-cases).

## Using a Proxy in Laravel: Step-By-Step Guide

In this section, we'll demonstrate how to incorporate a proxy in Laravel using the [default HTTP client](https://laravel.com/docs/master/http-client). We'll also address proxy integration with the [Symfony `HttpClient`](https://symfony.com/doc/current/http_client.html) library later in this article.

> **Note**:
> 
> Laravel's HTTP client is built on Guzzle, so you might want to review our [Guzzle proxy integration guide](https://brightdata.com/blog/how-tos/proxy-with-guzzle).

To illustrate the integration, we'll establish a `GET` `/api/v1/get-ip` endpoint that:

1. Sends a `GET` request to [`https://httpbin.io/ip`](https://httpbin.io/ip) using the configured proxy.
2. Extracts the exit IP address from the response.
3. Returns that IP to the client calling the Laravel endpoint.

If everything is properly configured, the IP returned by the API will match the proxy's IP address.

### Step #1: Project Setup

If you already have a Laravel application configured, you can skip ahead to Step #2.

Otherwise, follow these instructions to create a new Laravel project. Open your terminal and execute the following Composer [`create-command`](https://getcomposer.org/doc/03-cli.md#create-project) to initialize a fresh Laravel project:

```sh
composer create-project laravel/laravel laravel-proxies-application
```

This command will create a new Laravel project in a directory named `laravel-proxies-application`. Open this folder with your preferred PHP IDE.

At this point, the directory should contain the standard Laravel file structure:

![The Laravel projext structure](https://media.brightdata.com/2025/04/The-Laravel-projext-structure.png)

### Step #2: Define the Test API Endpoint

In the project directory, execute the following [Artisan command](https://laravel.com/docs/11.x/artisan) to generate a new controller:

```sh
php artisan make:controller IPController
```

This will create a file named `IPController.php` in the `/app/Http/Controllers` directory with this default content:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class IPController extends Controller
{
    //
}
```

Now, add the `getIP()` method below to `IPController.php`:

```php
public function getIP(): JsonResponse
{
    // make a GET request to the "/ip" endpoint to get the IP of the server
    $response = Http::get('https://httpbin.io/ip'); 

    // retrieve the response data
    $responseData = $response->json();

    // return the response data 
    return response()->json($responseData);
}
```

This method utilizes Laravel's `Http` client to retrieve your IP address from `https://httpbin.io/ip` and returns it as JSON.

Remember to include these two imports:

```php
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Http;
```

Since you want your Laravel application to provide stateless APIs, enable API routing with the [`install:api`](https://laravel.com/docs/master/routing#api-routes) Artisan command:

```sh
php artisan install:api
```

To expose this method through an API endpoint, add the following route to the `routes/api.php` file:

```php
use App\Http\Controllers\IPController;

Route::get('/v1/get-ip', [IPController::class, 'getIP']);
```

Your new API endpoint will be accessible at:

```php
/api/v1/get-ip
```

**Note**: Remember that all Laravel APIs are available under the `/api` path by default.

Time to test the `/api/v1/get-ip` endpoint!

Launch the Laravel development server by running:

```sh
php artisan serve
```

Your server should now be listening locally on port `8000`.

Make a `GET` request to the `/api/v1/get-ip` endpoint using cURL:

```sh
curl -X GET 'http://localhost:8000/api/v1/get-ip' 
```

**Note**: On Windows, substitute `curl` with `curl.exe`. Learn more in our guide on [how to send GET requests with cURL](https://brightdata.com/faqs/curl/curl-get-requests).

You should receive a response similar to:

```json
{
  "origin": "45.89.222.18"
}
```

This response matches exactly what the `/ip` endpoint of HttpBin produces, confirming that your Laravel API works correctly. Specifically, the IP address shown is your machine's public IP.

### Step #3: Retrieve a Proxy

To use a proxy in your Laravel application, you first need access to a functioning proxy server.

A typical proxy URL follows this format:

```
<protocol>://<host>:<port>
```

Where:

- `protocol` is the protocol needed to connect to the proxy server (e.g., `http`, `https`, `socks5`)
- `host` is the IP address or domain of the proxy server
- `port` is the port used to route the traffic

For this example, assume your proxy URL is:

```
http://66.29.154.103:3128
```

Store this in a variable inside the `getIP()` method:

```php
$proxyUrl = 'http://66.29.154.103:3128';
```

### Step #4: Integrate the Proxy in `Http`

Incorporating a proxy into Laravel using the `Http` client requires minimal additional configuration:

```php
$proxyUrl = 'http://66.29.154.103:3128';

$response = Http::withOptions([
    'proxy' => $proxyUrl
])->get('https://httpbin.io/ip');

$responseData = $response->json();
```

You need to pass the proxy URL as an option using the [`withOptions()`](https://laravel.com/docs/11.x/http-client#guzzle-options) method. This instructs Laravel's HTTP client to route the request through the specified proxy server, using [Guzzle's `proxy` option](https://docs.guzzlephp.org/en/stable/request-options.html#proxy).

### Step #5: Put It All Together

Your final Laravel API logic with proxy integration should now look like this:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Http;

class IPController extends Controller
{

    public function getIP(): JsonResponse
    {
        // the URL of the proxy server
        $proxyUrl = 'http://66.29.154.103:3128'; // replace with your proxy URL

        // make a GET request to /ip endpoint through the proxy server
        $response = Http::withOptions([
            'proxy' => $proxyUrl
        ])->get('https://httpbin.io/ip');

        // retrieve the response data
        $responseData = $response->json();

        // return the response data
        return response()->json($responseData);
    }
}
```

Test it by running Laravel locally:

```sh
php artisan serve
```

And connect to the `/api/v1/get-ip` endpoint again:

```sh
curl -X GET 'http://localhost:8000/api/v1/get-ip' 
```

This time, the output should look something like:

```json
{
  "origin": "66.29.154.103"
}
```

The `"origin"` field displays the IP address of the proxy server, confirming that your actual IP is now concealed behind the proxy.

> **Warning**:
> 
> Free proxy servers are typically unstable or short-lived. By the time you try this, the example proxy may no longer be operational. If necessary, replace the `$proxyUrl` with a currently active one before testing.

If you encounter SSL errors while making the request, refer to the troubleshooting tips provided in the advanced use cases section below.

## Advanced Use Cases

You've just mastered the basics of proxy integration with Laravel, but there are additional advanced scenarios to consider.

### Proxy Authentication

Premium proxies often require authentication to ensure only authorized users can access them. Without proper credentials, you'll encounter this error:

```
cURL error 56: CONNECT tunnel failed, response 407
```

The URL of an authenticated proxy typically follows this format:

```
<protocol>://<username>:<password>@<host>:<port>
```

Where `username` and `password` are your authentication credentials.

Laravel's `Http` class (which uses Guzzle under the hood) fully supports authenticated proxies. Little modification is needed—simply include the authentication credentials directly in the proxy URL:

```php
$proxyUrl = '<protocol>://<username>:<password>@<host>:<port>';
```

For example:

```php
// authenticated proxy with username and password
$proxyUrl = 'http://<username>:<password>@<host>:<port>';

$response = Http::withOptions([
    'proxy' => $proxyUrl
])->get('https://httpbin.io/ip');
```

Replace the value of `$proxyUrl` with a valid authenticated proxy URL.

`Http` will now direct the traffic to the configured authenticated proxy server!

### Avoid SSL Certificate Issues

When configuring a proxy with Laravel's `Http` client, your requests might fail due to SSL certificate verification errors like:

```
cURL error 60: SSL certificate problem: self-signed certificate in certificate chain
```

This typically occurs when the proxy server uses a self-signed [SSL certificate](https://www.cloudflare.com/learning/ssl/what-is-an-ssl-certificate/).

If you trust the proxy server and you're only testing locally or in a secure environment, you can disable SSL verification:

```php
$response = Http::withOptions([
    'proxy' => $proxyUrl,
    'verify' => false, // disable SSL certificate verification
])->get('https://httpbin.io/ip');
```

> **Warning**:
> 
> Disabling SSL verification makes you vulnerable to man-in-the-middle attacks. Use this option only in trusted environments.

Alternatively, if you have the [proxy server's certificate file](https://docs.brightdata.com/general/account/ssl-certificate) (e.g., `proxy-ca.crt`), you can use it for SSL verification:

```php
$response = Http::withOptions([
    'proxy' => $proxyUrl,
    'verify' => storage_path('certs/proxy-ca.crt'), // Path to the CA bundle
])->get('https://httpbin.io/ip');
```

Ensure the `proxy-ca.crt` file is stored in a secure and accessible directory (e.g., `storage/certs/`), and Laravel has permission to read it.

With either approach implemented, the SSL verification error caused by the proxy should be resolved.

### Proxy Rotation

If you repeatedly use the same proxy server, the target website will likely eventually detect and block that proxy's IP address. To prevent this, you can [rotate your proxy servers](https://brightdata.com/blog/how-tos/how-to-rotate-an-ip-address)—using a different one for each request.

Here are the steps to rotate proxies in Laravel:

1. Create an array containing multiple proxy URLs
2. Randomly select one before each request
3. Set the selected proxy in the HTTP client configuration

Implement this with the following code:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Http;

function getRandomProxyUrl(): string
{
    // a list of valid proxy URLs (replace with your proxy URLs)
    $proxies = [
        '<protocol_1>://<proxy_host_1>:<port_1>', 
        // ...
        '<protocol_n>://<proxy_host_n>:<port_n>',
    ];

    // return a proxy URL randomly picked from the list
    return $proxies[array_rand($proxies)];
}

class IPController extends Controller
{
    public function getIP(): JsonResponse
    {
        // the URL of the proxy server
        $proxyUrl = getRandomProxyUrl();
        // make a GET request to /ip endpoint through the proxy server
        $response = Http::withOptions([
            'proxy' => $proxyUrl
        ])->get('https://httpbin.io/ip');

        // retrieve the response data
        $responseData = $response->json();

        // return the response data
        return response()->json($responseData);
    }
}
```

This snippet demonstrates how to randomly select a proxy from a list to implement proxy rotation. While effective, this approach has limitations:

1. You must maintain a collection of reliable proxy servers, which typically comes at a cost.
2. For effective rotation, the proxy pool must be sufficiently large. Without enough proxies, the same servers will be used repeatedly, potentially leading to detection and blocking.

To overcome these challenges, consider using [Bright Data's rotating proxy network](https://brightdata.com/solutions/rotating-proxies).

## Use a Bright Data Proxy in Laravel

Follow these steps to implement Bright Data's residential proxies with Laravel.

If you don't have an account yet, [sign up with Bright Data](https://brightdata.com/cp/start). If you already have one, log in to access your user dashboard:

![The Bright Data dashboard](https://media.brightdata.com/2025/04/The-Bright-Data-dashboard.png)

After logging in, click the "Get proxy products" button:

![Clicking the "Get proxy products" button](https://media.brightdata.com/2025/04/Clicking-the-Get-proxy-products-button.png)

You'll be directed to the "Proxies & Scraping Infrastructure" page:

![The "Proxies & Scraping Infrastructure" page](https://media.brightdata.com/2025/04/The-Proxies-Scraping-Infrastructure-page-1.png)

In the table, locate the "Residential" row and click on it:

![Clicking the "residential" row](https://media.brightdata.com/2025/04/Clicking-the-residential-row.png)

You'll be taken to the residential proxy page:

![The "residential" page](https://media.brightdata.com/2025/04/The-residential-page.png)

For first-time users, follow the setup wizard to configure the proxy service according to your requirements. For additional assistance, feel free to [contact their 24/7 support team](https://brightdata.com/contact).

On the "Overview" tab, find your proxy's host, port, username, and password:

![The proxy credentials](https://media.brightdata.com/2025/04/The-proxy-credentials.png)

Use these details to construct your proxy URL:

```php
$proxyUrl = 'http://<brightdata_proxy_username>:<brightdata_proxy_password>@<brightdata_proxy_host>:<brightdata_proxy_port>';
```

Replace the placeholders (`<brightdata_proxy_username>`, `<brightdata_proxy_password>`, `<brightdata_proxy_host>`, `<brightdata_proxy_port>`) with your actual proxy credentials.

Make sure to toggle the "Off" switch to enable the proxy product, and complete the remaining setup instructions:

![Clicking the activation toggle](https://media.brightdata.com/2025/04/Clicking-the-activation-toggle.png)

With your proxy URL configured, you can now integrate it into Laravel using the `Http` client. Here's the code to send a request via Bright Data's rotating residential proxy in Laravel:

```php
public function getIp()
{
    // TODO: replace the placeholders with your Bright Data's proxy info
    $proxyUrl = 'http://<brightdata_proxy_username>:<brightdata_proxy_password>@<brightdata_proxy_host>:<brightdata_proxy_port>';

    // make a GET request to "/ip" through the proxy
    $response = Http::withOptions([
        'proxy' => $proxyUrl,
    ])->get('https://httpbin.org/ip');

    // get the response data
    $responseData = $response->json();

    return response()->json($responseData);
}
```

Each time you execute this script, you'll observe a different exit IP.

## [Extra] Symfony's `HttpClient` Proxy Integration

If you prefer Symfony's `HttpClient` component over Laravel's default HTTP client `Http`, follow these instructions to implement proxy integration with `HttpClient` in Laravel.

First, install the Symfony HTTP client package via Composer:

```sh
composer require symfony/http-client
```

Next, you can utilize Symfony's `HttpClient` with a proxy as follows:

```php
<?php

namespace App\Http\Controllers;

use Symfony\Contracts\HttpClient\HttpClientInterface;
use Illuminate\Http\JsonResponse;

class IpController extends Controller
{
    // where to store the HttpClient instance
    private $client;

    public function __construct(HttpClientInterface $client)
    {
        // initialize the HttpClient private instance
        $this->client = $client;
    }

    public function getIp(): JsonResponse
    {
          // your proxy URL
          $proxyUrl = 'http://66.29.154.103:3128'; // replace with your proxy URL

          // make a GET request to the "/ip" endpoint through the proxy
          $response = $this->client->request('GET', 'https://httpbin.io/ip', [
              'proxy' => $proxyUrl,
          ]);

          // parse the response JSON and return it
          $responseData = $response->toArray();
          return response()->json($responseData);
    }
}
```

This configuration allows you to use Symfony's `HttpClient` to send requests through a proxy.

## Conclusion

Free proxy services can be unreliable and potentially risky. For consistent performance, security, and scalability, you need a trusted proxy provider. Save time and effort by choosing the [market-leading proxy provider](https://brightdata.com/blog/proxy-101/best-proxy-providers), Bright Data.

Create an account and begin testing our proxies for free today!