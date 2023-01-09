# Laravel

> [https://g-ernaelsten.developpez.com/tutoriels/PostgREST/#LIV](https://g-ernaelsten.developpez.com/tutoriels/PostgREST/#LIV)

By adding the `symfony/http-client` reference, we can easily access to the APIs.

```bash
composer require symfony/http-client
```

```php
// j'instancie le client
$client = HttpClient::create();
// je récupère le token
$token = $client->request('POST', 'http://localhost:3000/rpc/login', [
            'body' => ["pseudo" => "jhon", "pass" => "bonjour"]
        ]);

// je crée un ensemble d'options, avec le token
$options = [
           'headers' => [
                'Prefer' => 'count=exact',
                'Authorization' => 'Bearer ' . $token->toArray()[0]['token']]
        ];
// j'appelle mon API, avec le token en options
$response = $client->request('GET', 'http://localhost:3000/film', $options);
// j'affiche le résultat sous forme de tableau
dd($response->toArray());
```
