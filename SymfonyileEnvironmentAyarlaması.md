# Setting the Environment with Symfony

## Configuration Environments
You have only one application, but whether you realize it or not, you need it to behave differently at different times:

While developing, you want to log everything and expose nice debugging tools;
After deploying to production, you want that same application to be optimized for speed and only log errors.
The files stored in config/packages/ are used by Symfony to configure the application services. In other words, you can change the application behavior by changing which configuration files are loaded. That’s the idea of Symfony’s configuration environments.

A typical Symfony application begins with three environments: dev (for local development), prod (for production servers) and test (for automated tests). When running the application, Symfony loads the configuration files in this order (the last files can override the values set in the previous ones):

config/packages/*.yaml (and *.xml and *.php files too);
config/packages/<environment-name>/*.yaml (and *.xml and *.php files too);
config/services.yaml (and services.xml and services.php files too);
config/services_<environment-name>.yaml (and services_<environment-name>.xml and services_<environment-name>.php files too).
Take the framework package, installed by default, as an example:

First, config/packages/framework.yaml is loaded in all environments and it configures the framework with some options;
In the prod environment, nothing extra will be set as there is no config/packages/prod/framework.yaml file;
In the dev environment, there is no file either ( config/packages/dev/framework.yaml does not exist).
In the test environment, the config/packages/test/framework.yaml file is loaded to override some of the settings previously configured in config/packages/framework.yaml.
In reality, each environment differs only somewhat from others. This means that all environments share a large base of common configuration, which is put in files directly in the config/packages/ directory.

> See the configureContainer() method of the Kernel class to learn everything about the loading order of configuration files.

## Selecting the Active Environment
Symfony applications come with a file called .env located at the project root directory. This file is used to define the value of environment variables and it’s explained in detail later in this article.

Open the .env file (or better, the .env.local file if you created one) and edit the value of the APP_ENV variable to change the environment in which the application runs. For example, to run the application in production:
```php 
# .env (or .env.local)
APP_ENV=prod
```
This value is used both for the web and for the console commands. However, you can override it for commands by setting the APP_ENV value before running them:
```php
# Use the environment defined in the .env file
 php bin/console command_name

# Ignore the .env file and run this command in production
 APP_ENV=prod php bin/console command_name
 ```
## Creating a New Environment
The default three environments provided by Symfony are enough for most projects, but you can define your own environments too. For example, this is how you can define a staging environment where the client can test the project before going to production:

Create a configuration directory with the same name as the environment (in this case, config/packages/staging/);
Add the needed configuration files in config/packages/staging/ to define the behavior of the new environment. Symfony loads the config/packages/*.yaml files first, so you only need to configure the differences to those files;
Select the staging environment using the APP_ENV env var as explained in the previous section.

>It’s common for environments to be similar to each other, so you can use symbolic links between config/packages/<environment-name>/ directories to reuse the same configuration.

## Configuration Based on Environment Variables
Using environment variables (or “env vars” for short) is a common practice to configure options that depend on where the application is run (e.g. the database credentials are usually different in production versus your local machine). If the values are sensitive, you can even encrypt them as secrets.

You can reference environment variables using the special syntax %env(ENV_VAR_NAME)%. The values of these options are resolved at runtime (only once per request, to not impact performance).

This example shows how you could configure the database connection using an env var:
```php
# config/packages/doctrine.yaml
doctrine:
    dbal:
        # by convention the env var names are always uppercase
        url: '%env(resolve:DATABASE_URL)%'
    # ...
```

>The values of env vars can only be strings, but Symfony includes some env var processors to transform their contents (e.g. to turn a string value into an integer).

- To define the value of an env var, you have several options:

  * Add the value to a .env file;
  * Encrypt the value as a secret;
  * Set the value as a real environment variable in your shell or your web server.

> Some hosts - like SymfonyCloud - offer easy utilities to manage env vars in production.

 >  Beware that dumping the contents of the $_SERVER and $_ENV variables or outputting the phpinfo() contents will display the values of the environment variables, exposing sensitive information such as the database credentials.
 The values of the env vars are also exposed in the web interface of the Symfony profiler. In practice this shouldn’t be a problem because the web profiler must never be enabled in production.

 ## Configuring Environment Variables in .env Files
 Instead of defining env vars in your shell or your web server, Symfony provides a convenient way to define them inside a .env (with a leading dot) file located at the root of your project.

The .env file is read and parsed on every request and its env vars are added to the $_ENV & $_SERVER PHP variables. Any existing env vars are never overwritten by the values defined in .env, so you can combine both.

For example, to define the DATABASE_URL env var shown earlier in this article, you can add:
```PHP
# .env
DATABASE_URL="mysql://db_user:db_password@127.0.0.1:3306/db_name"
``` 
This file should be committed to your repository and (due to that fact) should only contain “default” values that are good for local development. This file should not contain production values.

In addition to your own env vars, this .env file also contains the env vars defined by the third-party packages installed in your application (they are added automatically by Symfony Flex when installing packages).

## .env File Syntax¶
Add comments by prefixing them with #:
```PHP
# database credentials
DB_USER=root
DB_PASS=pass # this is the secret password
```
Use environment variables in values by prefixing variables with $:
```PHP
DB_USER=root
DB_PASS=${DB_USER}pass # include the user as a password prefix
```
>The order is important when some env var depends on the value of other env vars. In the above example, DB_PASS must be defined after DB_USER. Moreover, if you define multiple .env files and put DB_PASS first, its value will depend on the DB_USER value defined in other files instead of the value defined in this file.

Define a default value in case the environment variable is not set:
```PHP
DB_USER=
DB_PASS=${DB_USER:-root}pass # results in DB_PASS=rootpass
``` 
>New in version 4.4:The support for default values has been introduced in Symfony 4.4.

Embed commands via $() (not supported on Windows):

```PHP 
START_TIME=$(date)
```
> Using $() might not work depending on your shell.

> As a .env file is a regular shell script, you can source it in your own shell scripts:
  ```PHP
  source .env
  ``` 
  ## Overriding Environment Values via .env.local
  If you need to override an environment value (e.g. to a different value on your local machine), you can do that in a .env.local file:

  ```PHP
  # .env.local
DATABASE_URL="mysql://root:@127.0.0.1:3306/my_database_name"
```
This file should be ignored by git and should not be committed to your repository. Several other .env files are available to set environment variables in just the right situation:

- .env: defines the default values of the env vars needed by the application;
- .env.local: overrides the default values for all environments but only on the machine which contains the file. This file should not be committed to the repository and it’s ignored in the test environment (because tests should produce the same results for everyone);
- .env.<environment> (e.g. .env.test): overrides env vars only for one environment but for all machines (these files are committed);
- .env.<environment>.local (e.g. .env.test.local): defines machine-specific env var overrides only for one environment. It’s similar to .env.local, but the overrides only apply to one environment.

Real environment variables always win over env vars created by any of the .env files.

The .env and .env.<environment> files should be committed to the repository because they are the same for all developers and machines. However, the env files ending in .local (.env.local and .env.<environment>.local) should not be committed because only you will use them. In fact, the .gitignore file that comes with Symfony prevents them from being committed.

> Applications created before November 2018 had a slightly different system, involving a .env.dist file. For information about upgrading, see: Nov 2018 Changes to .env & How to Update.

## Configuring Environment Variables in Production
In production, the .env files are also parsed and loaded on each request. So the easiest way to define env vars is by deploying a .env.local file to your production server(s) with your production values.

To improve performance, you can optionally run the dump-env command (available in Symfony Flex 1.2 or later):

```php
# parses ALL .env fileps and dumps their final values to .env.local.php
 composer dump-env prod
 ``` 
After running this command, Symfony will load the .env.local.php file to get the environment variables and will not spend time parsing the .env files.

>Update your deployment tools/workflow to run the dump-env command after each deploy to improve the application performance.

## Encrypting Environment Variables (Secrets)¶
Instead of defining a real environment variable or adding it to a .env file, if the value of a variable is sensitive (e.g. an API key or a database password), you can encrypt the value using the secrets management system.

## Listing Environment Variables¶
Regardless of how you set environment variables, you can see a full list with their values by running:
```php
php bin/console debug:container --env-vars

---------------- ----------------- ---------------------------------------------
 Name             Default value     Real value
---------------- ----------------- ---------------------------------------------
 APP_SECRET       n/a               "471a62e2d601a8952deb186e44186cb3"
 FOO              "[1, "2.5", 3]"   n/a
 BAR              null              n/a
---------------- ----------------- ---------------------------------------------

#you can also filter the list of env vars by name:
 php bin/console debug:container --env-vars foo

#run this command to show all the details for a specific .env var:
 php bin/console debug:container --env-var=FOO
 ```
 >New in version 4.3:The option to debug environment variables was introduced in Symfony 4.3.

 ## Accessing Configuration Parameters

 Controllers and services can access all the configuration parameters. This includes both the parameters defined by yourself and the parameters created by packages/bundles. Run the following command to see all the parameters that exist in your application:

 ```php 
  php bin/console debug:container --parameters
  ```
  In controllers extending from the AbstractController, use the getParameter() helper:
```php
  // src/Controller/UserController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;

class UserController extends AbstractController
{
    // ...

    public function index(): Response
    {
        $projectDir = $this->getParameter('kernel.project_dir');
        $adminEmail = $this->getParameter('app.admin_email');

        // ...
    }
}
``` 
In services and controllers not extending from AbstractController, inject the parameters as arguments of their constructors. You must inject them explicitly because service autowiring doesn’t work for parameters:

```php
# config/services.yaml
parameters:
    app.contents_dir: '...'

services:
    App\Service\MessageGenerator:
        arguments:
            $contentsDir: '%app.contents_dir%'
``` 
If you inject the same parameters over and over again, use the services._defaults.bind option instead. The arguments defined in that option are injected automatically whenever a service constructor or controller action defines an argument with that exact name. For example, to inject the value of the kernel.project_dir parameter whenever a service/controller defines a $projectDir argument, use this:

```php
# config/services.yaml
services:
    _defaults:
        bind:
            # pass this value to any $projectDir argument for any service
            # that's created in this file (including controller arguments)
            $projectDir: '%kernel.project_dir%'

    # ...
    
```
> Read the article about binding arguments by name and/or type to learn more about this powerful feature.

Finally, if some service needs access to lots of parameters, instead of injecting each of them individually, you can inject all the application parameters at once by type-hinting any of its constructor arguments with the

 Symfony\Component\DependencyInjection\ParameterBag\ContainerBagInterface:
```php
// src/Service/MessageGenerator.php
namespace App\Service;

// ...

use Symfony\Component\DependencyInjection\ParameterBag\ContainerBagInterface;

class MessageGenerator
{
    private $params;

    public function __construct(ContainerBagInterface $params)
    {
        $this->params = $params;
    }

    public function someMethod()
    {
        // get any container parameter from $this->params, which stores all of them
        $sender = $this->params->get('mailer_sender');
        // ...
    }
}
``` 
