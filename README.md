# VaahSaaS
> VaahSaaS: Laravel multi-database tenancy based SAAS CMS - Pre-Configured & Ready to Use.


Please consider starring the project to show your :heart: and support.


## Steps to install
Well, this is a pre-configure and ready to install repo for VaahCMS.

#### Step 1:
Download this repository and unzip.

#### Step 2:
Rename `.env.example` to `.env`

#### Step 3:
Run following command:
```bash
composer install
```

#### Step 3:
Publish VaahCms Assets:
```bash
php artisan vendor:publish --provider="WebReinvent\VaahCms\VaahCmsServiceProvider" --tag=assets
```

#### Step:
`php artisan tenancy:install`, this will register necessary `ServiceProviders`, `migrations` and public `tenancy.php` configuration files

#### Step:
Open `tenancy.php` and update the `exempt_domains`, if you are using localhost, make sure you configure VirtualHost


#### Step 4:
Then visit following url to setup the CMS:
```bash
<base-url>/public/vaahcms/setup
```


## Multi-database tenancy setup

https://tenancy.samuelstancl.me/docs/v2/getting-started/

- Install `composer require stancl/tenancy`
- `php artisan tenancy:install`, this will register necessary `ServiceProviders`, `migrations` and public `tenancy.php` configuration files
- In `.env` `DB_CONNECTION` will serve as central database which will store `domain` and `respective` database details
- Note: in `.env` file `DB_USERNAME` should have access to all the database that will be created when new tenants are created.
- In `config/tenancy.php`, add your main domain in following:
```php
...
'exempt_domains' => [ // e.g. domains which host landing pages, sign up pages, etc
         'YourRootDomain.com',
    ],
...
``` 
- Run `php artisan migrate`
- How to create tenancy:
```php
$data = Tenant::new()
            ->withDomains([$request->subdomain])
            ->withData(['plan' => 'free'])
            ->save();
```


## Creating Development Environment on Xampp Windows

- Open `<xampp>\apache\conf\extra\httpd-vhosts.conf` add following code:
```shell
<VirtualHost yourdomain.com>
  DocumentRoot "<xampp>/htdocs/vaahsaas_director/public"
  ServerName yourdomain.com

  <Directory "<xampp>/htdocs/vaahsaas_director/public">
    Options Indexes FollowSymLinks
    AllowOverride All
    Order allow,deny
    Allow from all
  </Directory>
</VirtualHost>

<VirtualHost test.yourdomain.com>
  DocumentRoot "<xampp>/htdocs/vaahsaas_director/public"
  ServerName test.yourdomain.com

  <Directory "<xampp>/htdocs/vaahsaas_director/public">
    Options Indexes FollowSymLinks
    AllowOverride All
    Order allow,deny
    Allow from all
  </Directory>
</VirtualHost>
```

- Then open `notepad` as administrator `C:\Windows\System32\drivers\etc\host`
- Add following lines:
```shell
127.0.0.1 yourdomain.com
127.0.0.1 test.yourdomain.com
```
- You may see some error`test.yourdomain.com` like `Tenancy Not Identified` if you have already 
not created any tenancy for the domain.


## CPanel Database CRUD operations
CPanel does not allow `mysql_query` to create `database` and this conflicts with `stancl/tenancy`.
Hence we need to install following package:
```shell
composer require webreinvent/laravel-cpanel 
```
This is allow us to do CRUD operation for CPanel Databases.

After the installation, we need to create `DatabaseManager` and register it in `config/tenancy.php`, code of that will be following:
```php
<?php

declare(strict_types=1);

namespace App\TenantDatabaseManagers; //You should correct this namespace based on where the file is located

use Illuminate\Contracts\Config\Repository;
use Illuminate\Database\DatabaseManager as IlluminateDatabaseManager;
use Stancl\Tenancy\Contracts\TenantDatabaseManager;
use WebReinvent\CPanel\CPanel;

class CPanelMySQLDatabaseManager implements TenantDatabaseManager
{
    /** @var \Illuminate\Database\Connection */
    protected $database;

    public function __construct(Repository $config, IlluminateDatabaseManager $databaseManager)
    {
        $this->database = $databaseManager->connection($config['tenancy.database_manager_connections.mysql']);
    }

    public function createDatabase(string $name): bool
    {
        /*$charset = $this->database->getConfig('charset');
        $collation = $this->database->getConfig('collation');
        return $this->database->statement("CREATE DATABASE `$name` CHARACTER SET `$charset` COLLATE `$collation`");*/

        $cpanel = new CPanel();
        $response = $cpanel->createDatabase($name);

        if(isset($response['status']) && $response['status'] == 'success')
        {
            $cpanel->setAllPrivilegesOnDatabase(env('DB_USERNAME'), $name);

            return true;
        } else{
            return false;
        }

    }

    public function deleteDatabase(string $name): bool
    {
        //return $this->database->statement("DROP DATABASE `$name`");

        $cpanel = new CPanel();
        $response = $cpanel->deleteDatabase($name);
        if(isset($response['status']) && $response['status'] == 'success')
        {
            return true;
        } else{
            return false;
        }

    }

    public function databaseExists(string $name): bool
    {
        return (bool) $this->database->select("SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA WHERE SCHEMA_NAME = '$name'");
    }
}

```

After this register it in `config/tenancy.php`:
```php
...
    'database_managers' => [
        // Tenant database managers handle the creation & deletion of tenant databases.
        'sqlite' => Stancl\Tenancy\TenantDatabaseManagers\SQLiteDatabaseManager::class,
        //'mysql' => Stancl\Tenancy\TenantDatabaseManagers\MySQLDatabaseManager::class, // You should use this for localhost
        'mysql' => App\TenantDatabaseManagers\CPanelMySQLDatabaseManager::class, // Correct the namespace
        'pgsql' => Stancl\Tenancy\TenantDatabaseManagers\PostgreSQLDatabaseManager::class,
    ],
..
```



## Setup wildcard sub domains on CPanel for Laravel

- Login `CPanel >> Subdomains`, enter `*` in subdomain, in `domain` choose `top level domain`, 
remove `_wildcard_` from `Document Root`.

After this if you visit any subdomain it will show SSL error, to resolve it we need to install
`wildcard ssl`. 
- Visit `https://www.sslforfree.com/` and enter `*.yourdomain.com` and click `Create Free Certificate`
- Add `A` record to domain as per the instruction. On next page it will show all there SSL Certificate strings.
- Login `CPanel >> SSL/TLS >> Manage SSL sites >> Install an SSL Website` and choose `*.yourdomain.com` and enter SSL Certificate string to respective section and save.
- Now try to visit any sub domain all of them should be working.
- Before deploying VaahSaas, create a folder `vaahsaas` and move all files and folder in that except `public`
- Rename `public` folder to `public_html` and change the content to following:
```php
require __DIR__.'/../vaahsaas/vendor/autoload.php';
$app = require_once __DIR__.'/../vaahsaas/bootstrap/app.php';
```
- So in root folder you will have two folder `vaahsaas` and `public_html` which can been deployed to the `root` of cpanel

 

## Support us

[WebReinvent](https://www.webreinvent.com) is a web agency based in Delhi, India. You'll find an overview of all our open source projects [on github](https://github.com/webreinvent).

## License

The MIT License (MIT). Please see [License File](LICENSE) for more information.
