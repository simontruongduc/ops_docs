# Auto deploy Document laravel

## Install deploy and  init source  

Let’s start with installation of Deployer. Run next commands in terminal:
```
curl -LO https://deployer.org/deployer.phar
mv deployer.phar /usr/local/bin/dep
chmod +x /usr/local/bin/dep
```
Next, in your projects directory run:

```
dep init -t Laravel
```

It will create a deploy.php in your source and edit it like this:
```
<?php
namespace Deployer;

require 'recipe/laravel.php';

// Project name
set('application', 'NP_MATION_API');

// Project repository
set('repository', 'newphoria@newphoria.git.backlog.jp:/DEV_MATION_PJT/server-api.git');

// [Optional] Allocate tty for git clone. Default value is false.
set('git_tty', true);

set('keep_releases', 2);

set('git_recursive', false);

// Shared files/dirs between deploys
add('shared_files', [
    '.env',
    'public/.htaccess'
]);
add('shared_dirs', [
    'storage',
    'bootstrap/cache',
]);

// Writable dirs by web server
add('writable_dirs', [
    'bootstrap/cache',
    'storage',
    'storage/app',
    'storage/app/public',
    'storage/framework',
    'storage/framework/cache',
    'storage/framework/sessions',
    'storage/framework/views',
    'storage/logs',
]);

// Hosts

foreach (glob('deploy/config/*.php') as $file) {
    require_once $file;
}

// Tasks

task('build', function () {
    run('cd {{release_path}} && build');
});

task('deploy', [
    //'deploy:info',
    'deploy:prepare',
    'deploy:lock',
    'deploy:release',
    'deploy:update_code',
    'deploy:shared',
    'deploy:writable',
    'deploy:vendors',
    'artisan:storage:link',
    'artisan:view:clear',
    'artisan:cache:clear',
    'artisan:config:cache',
    'artisan:route:cache',
    'deploy:clear_paths',
    'deploy:symlink',
    'deploy:unlock',
    'cleanup',
    'success'
]);
// [Optional] if deploy fails automatically unlock.
after('deploy:failed', 'deploy:unlock');

// Migrate database before symlink new release.

//before('deploy:symlink', 'artisan:migrate');

```

## Make enviroment file and put the ssh key

In your source, make new folders, follow these commands

```
mkdir deploy
cd deploy/
mkdir config
mkdir ec2-keys
```

Move into deploy/config folder, make enviroment file for each enviroment. This is example file of https://smakon.newphoria.net/api
```
<?php
namespace Deployer;

host('40.115.240.156')  //IP of your server
    ->set('http_user', 'apache')
    ->stage('development') //Enviroment name
    ->user('gumi-tu') //SSH user
    ->port(22)
    ->identityFile('deploy/ec2-keys/newphoria.pem') //SSH key using for access into server
    ->forwardAgent(true)
    ->multiplexing(true)
    ->set('branch', '"#gumi_dev"') //Branch name you want to deploy source code
    ->set('deploy_path', '/var/www/html/smakon.newphoria.net/htdocs/smakon-api'); //Folder path in your server 
```

Put your ssh key into deploy/ec2-keys, and set permission for it. Example:
```
chmod -R 400 deploy/ec2-keys/newphoria.pem
```

## Deploy

Make sure you merged all code into the deploy branch (Example: "#gumi_dev") 

After you finish all configs, let deploy, using this command:
```
dep deploy ENVIROMENT_NAME
```

* Note: ENVIROMENT_NAME is name of file in deploy/config folder. Example command: "dep deploy development"

If your deploy successful, it will have an output like this:
```
# Executing task deploy:prepare
# Executing task deploy:lock
# Executing task deploy:release
➤ Executing task deploy:update_code
Cloning into '/var/www/html/smakon.newphoria.net/htdocs/smakon-api/releases/1'...
remote: Enumerating objects: 41, done.
remote: Counting objects: 100% (35/35), done.
remote: Compressing objects: 100% (21/21), done.
remote: Total 21 (delta 14), reused 0 (delta 0)

Receiving objects: 100% (21/21), done.
Resolving deltas: 100% (14/14), completed with 11 local objects.
Checking connectivity... done.
Counting objects: 6467, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (1868/1868), done.
Writing objects: 100% (6467/6467), done.
Total 6467 (delta 4402), reused 6455 (delta 4390)
Connection to 40.115.240.156 closed.
# Ok
# Executing task deploy:shared
# Executing task deploy:writable
# Executing task deploy:vendors
# Executing task artisan:storage:link
# Executing task artisan:view:clear
# Executing task artisan:cache:clear
# Executing task artisan:config:cache
# Executing task artisan:route:cache
# Executing task deploy:clear_paths
# Executing task deploy:symlink
# Executing task deploy:unlock
# Executing task cleanup
Successfully deployed!
```

Now, ssh into your server, go to folder path, you will see 3 folders has been created by deployer, like this:

```
current
shared
releases 
``` 
If this is the first time or you need to update .env file, move into current folder, and edit the .env file:
```
cd current
nano .env
php artisan config:cache
php artisan config:clear
php artisan optimize:clear
```

Almost done.

