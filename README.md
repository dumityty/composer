<img src="http://159.65.210.101/php-actions.png" align="right" alt="PHP Actions for Github" />

Use the Composer CLI in your Github Actions.
============================================

Composer is a tool for dependency management in PHP. It allows you to declare the libraries your project depends on, and it will manage (install/update) them for you.

If you are running tests like [PHPUnit][php-actions-phpunit], [phpspec][php-actions-phpspec] or [Behat][php-actions-behat] in your Github actions, chances are you will need to install your project's dependencies using Composer.

An example repository has been created at https://github.com/php-actions/example-composer to show how to use this action in a real project. The repository also depends on a private dependency and uses SSH keys for authentication.

Usage
-----

Create your Github Workflow configuration in `.github/workflows/ci.yml` or similar.

```yaml
name: CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - uses: php-actions/composer@v6
    # ... then your own project steps ...
```

### Version numbers

This action is released with semantic version numbers, but also tagged so the latest major release's tag always points to the latest release within the matching major version.

Please feel free to use `uses: php-actions/composer@v6` to always run the latest version of v6, or `uses: php-actions/composer@v6.0.0` to specify the exact release.


Running custom commands
-----------------------

By default, adding `- uses: php-actions/composer@v6` into your workflow will run `composer install`, as `install` is the default command name. The `install` command will be provided with a default set of arguments (see below).

You can issue custom commands by passing a `command` input, like so:

```yaml
jobs:
  build:

    ...

    - name: Install dependencies
      uses: php-actions/composer@v6
      with:
        command: your-command-here
```

Passing arguments
-----------------

Any arbitrary arguments can be passed to composer by using the `args` input, however there are a few inputs pre-configured to handle common arguments. All inputs are optional. Please see the following list:

+ `interaction` - Whether to ask any interactive questions - yes / no (default `no`)
+ `dev` - Whether to install dev packages - yes / no (default `yes`)
+ `progress` - Whether to output download progress - yes / no (default `no`)
+ `quiet` - Whether to suppress all messages - yes / no (default `no`)
+ `args` - Optional arguments to pass - no constraints (default _empty_)
+ `only_args` - Only run the desired command with this args. Ignoring all other provided arguments(default _empty_)
+ `php_version` - Choose which version of PHP you want to use - x.y (default `latest`) (e.g. `7.4.29`, `8.2`, or any version listed on https://www.php.net/releases/index.php)
+ `version` - Choose which version of Composer you want to use - default `latest` (e.g. `1.x`, `1.10.26`, `2.x`, `2.2.x`, or any exact version listed on https://getcomposer.org/download/)
+ `memory_limit` - Sets the composer memory limit - (default _empty_)
+ `container_workdir` - Sets the aplication workdir inside container - (default /app)

There are also SSH input available: `ssh_key`, `ssh_key_pub` and `ssh_domain` that are used for depending on private repositories. See below for more information on usage.

Example of a yaml config that does not want to install dev packages, and passes the `--profile` and `--ignore-platform-reqs` arguments:

```yaml
jobs:
  build:

    ...

    - name: Install dependencies
      uses: php-actions/composer@v6
      with:
        dev: no
        args: --profile --ignore-platform-reqs
```

Using different versions of PHP or Composer
-------------------------------------------

This action runs on a custom base image, available at https://github.com/php-actions/php-build which allows for switching the active PHP version on-the-fly, and this repository allows switching of Composer versions on-the-fly.

Use the following inputs to run a specific PHP/Composer version combination:

+ `php_version` Available versions: `7.1`, `7.2`, `7.3`, `7.4`, `8.0`, `8.1` (default: `latest` aka: `8.1`)
+ `version` Available versions: `latest`, `preview`, `snapshot`, `1.x`, `2.x`, `2.2.x` or an exact version like `2.5.8` (default: `latest`)

Make sure to put the PHP version number in quotes, otherwise YAML will interpret e.g. `8.0` as `8` which means latest 8.x, not 8.0.

Example configuration that runs Composer version 1 on PHP version 7.1:
```yaml
jobs:
  build:

    ...

    - name: Install dependencies
      uses: php-actions/composer@v6
      with:
        php_version: "7.1"
        version: 1
```

Execute composer install in a different folder
-------------------------------------------

```yaml
    - name: Install dependencies
      uses: "php-actions/composer@v6"
      env:
        COMPOSER: "composer.json"
      with:
        php_version: "5.6.40"
        version: "2.2"
        args: "--ignore-platform-reqs --optimize-autoloader"
        working_dir: "my/different/folder"
```

Including PHP Extensions
-------------------------------------------

This action includes the [extensions that Composer suggests](https://github.com/composer/composer/blob/master/composer.json#L44) by default. To include additional PHP extensions in your action steps, set the `php_extensions` input with any of the [supported extension names](https://github.com/mlocati/docker-php-extension-installer#supported-php-extensions) separated by spaces.

Example configuration that runs Composer version 2 on PHP version 7.4 with the Redis and Exif extensions enabled:

```yaml
jobs:
  build:

    ...

    - name: Install dependencies
      uses: php-actions/composer@v6
      with:
        php_version: "7.4"
        php_extensions: redis exif
        version: 2.x
```

Caching dependencies for faster builds
--------------------------------------

Github actions supports dependency caching, allowing Composer downloads to be cached between workflows, as long as the `composer.lock` file has not changed. This produces much faster builds, as the `composer install` command does not have to download files over the network at all if the cache is valid.

Example workflow (taken from https://github.com/PhpGt/Database):

```yaml
name: CI

on: [push]

jobs:
  build:
    runs-on: [ubuntu-latest]
    
    steps:
    - uses: actions/checkout@v3

    - name: Cache Composer dependencies
      uses: actions/cache@v3
      with:
        path: /tmp/composer-cache
        key: ${{ runner.os }}-${{ hashFiles('**/composer.lock') }}
      
    - uses: php-actions/composer@v6

    ...      
```

In the example above, the "key" is passed to the Cache action that consists of a hash of the composer.lock file. This means that as long as the contents of composer.lock doesn't change between workflows, the Composer cache directory will be persisted between workflows.

Installing private repositories
-------------------------------

To install from a private repository, SSH authentication must be used. Generate an SSH key pair for this purpose and add it to your private repository's configuration, preferable with only read-only privileges. On Github for instance, this can be done by using [deploy keys][deploy-keys].

If using multiple private repos, Github does not allow to share deploy keys across repos, so you can use a normal user account that has access to all the private repos and [add the SSH key][add-ssh-key-git] to that user.

Add the key pair to your project using  [Github Secrets][secrets], and pass them into the `php-actions/composer` action by using the `ssh_key` and `ssh_key_pub` inputs. If your private repository is stored on another server than github.com, you also need to pass the domain via `ssh_domain`. If the private repository is configured to use a non-standard SSH port, you can configure this by passing `ssh_port`.

Example yaml, showing how to pass secrets:

```yaml
jobs:
  build:

    ...

    - name: Install dependencies
      uses: php-actions/composer@v6
      with:
        ssh_key: ${{ secrets.ssh_key }}
        ssh_key_pub: ${{ secrets.ssh_key_pub }}
```

There is an example repository available for reference at https://github.com/php-actions/example-composer that uses a private dependency. Check it out for a live working project.

### HTTP basic authentication

It's recommended to use SSH keys for authentication, but sometimes HTTP basic authentication is the only tool available at the time. In order to use this authentication mechanism as securely as possible, please follow these steps:

1) Create a [personal access token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) for the Github account you wish to authenticate with.

2) Create a new GitHub Secret called `PAT` with a value of personal access token.

3) Pass this secret to COMPOSER_AUTH variable:

```yaml
jobs:
  build:

    ...

    - name: Install dependencies
      uses: php-actions/composer@v6
      env:
        COMPOSER_AUTH: '{"github-oauth": {"github.com": "${{ secrets.PAT }}"} }'
```

4) Now, any connections Composer makes to GitHub.com will use your HTTP basic auth credentials, which is essentially the same as being logged in as you, so your private repositories will now be available to Composer.

***

If you found this repository helpful, please consider [sponsoring the developer][sponsor].

[php-actions-phpunit]: https://github.com/marketplace/actions/phpunit-php-actions 
[php-actions-phpspec]: https://github.com/marketplace/actions/phpspec-php-actions 
[php-actions-behat]: https://github.com/marketplace/actions/behat-php-actions 
[deploy-keys]: https://docs.github.com/en/developers/overview/managing-deploy-keys
[add-ssh-key-git]: https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account
[secrets]: https://docs.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets
[sponsor]: https://github.com/sponsors/g105b
