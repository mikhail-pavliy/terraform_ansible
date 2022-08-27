# Разаработка ansible-роли для установки WordPress с использованием инфраструктуры, поднятой при помощи манифестов терраформа.

1. Корректировка манифестов терраформ

Так как у WordPress в стандартной конфигурации нет специального пути вида /health, то для проверки “здоровья” узлов в ```attached_target_group``` в файле ```lb.tf``` надо заменить блок ```healthcheck``` на следующий:

```ruby
  healthcheck {
      name = "tcp"
      tcp_options {
        port = 80
      }
    }
```
И в файл ```output.tf``` добавим вывод второго IP, он нам понадобится:
```ruby
output "vm_linux_2_public_ip_address" {
  description = "Virtual machine IP"
  value = yandex_compute_instance.wp-app-2.network_interface[0].nat_ip_address
}
```
Применим изменения при помощи команды:

```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/terraform$ terraform apply --auto-approve
```

2. Разработка плейбука

Прежде всего создадим каталог ansible. Желательно сделать это на одном уровне с каталогом terraform:

```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform$ mkdir ansible
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform$ ls -lh
итого 16K
drwxrwxr-x 2 mikhail mikhail 4,0K авг 27 18:52 ansible
-rw-rw-r-- 1 mikhail mikhail 1,2K авг 27 18:50 README.md
drwxrwxr-x 4 mikhail mikhail 4,0K авг 27 18:51 terraform
```

Перейдем в каталог ansible и создадим необходимую структуру подкаталогов
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ mkdir -p environments/prod/group_vars
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ mkdir files
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ mkdir templates
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ mkdir playbooks
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ mkdir roles
```
Далее мы создадим файл ```ansible.cfg ``` с описанием настроек Ansible.
```ruby
[defaults]
# Откуда брать инвентори по-умолчанию
inventory = ./environments/prod/inventory
# Под каким пользователем подключаться к хостам
remote_user = ubuntu
# Где брать приватный ключ для подключения к хостам
private_key_file = ~/.ssh/id_rsa.pub
# Выключение проверки SSH Host-keys
host_key_checking = False
# Выключение использования *.retry-файлов
retry_files_enabled = False
# Местонахождение ролей
roles_path = ./roles
```
Опишем в файле ```environments/prod/inventory``` IP хостов, на которые мы будем ставить WordPress.
```ruby
[wp_app]
app ansible_host=51.250.95.144
app2 ansible_host=51.250.30.47
```
Далее, нам надо создать файл с групповыми переменными для группы хостов ```wp_app``` в каталоге ```environments/prod/group_vars```
```ruby
wordpress_db_name: db
wordpress_db_user: user
wordpress_db_password: password
wordpress_db_host: c-c9qmemrl7qvbh0bdoou2.rw.mdb.yandexcloud.net
```
В каталоге ```playbooks``` создадим файл ```install.yml```. И наш плейбук будет начинаться с установки необходимых зависимостей, которые понадобятся WordPress-у.
```ruby
- hosts: wp_app
  become: yes

  tasks:
    - name: Update apt-get repo and cache
      apt: update_cache=yes force_apt_get=yes cache_valid_time=3600

    - name: install dependencies
      package:
        name:
          - apache2
          - ghostscript
          - libapache2-mod-php
          - mysql-server
          - php
          - php-bcmath
          - php-curl
          - php-imagick
          - php-intl
          - php-json
          - php-mbstring
          - php-mysql
          - php-xml
          - php-zip
        state: present
```		
Запустим плейбук и убедимся, что подключение к удаленным хостам прошло успешно и установка пакетов прошла без проблем:
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ ansible-playbook playbooks/install.yml
PLAY [wp_app] ************************************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************************
ok: [app2]
ok: [app]

TASK [Update apt-get repo and cache] *************************************************************************************************************************************************
ok: [app2]
ok: [app]

TASK [install dependencies] **********************************************************************************************************************************************************
changed: [app2]
changed: [app]

PLAY RECAP ***************************************************************************************************************************************************************************
app                        : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
app2                       : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 

```
Следующий наш шаг - это создание на хостах каталога, куда будет установлены WordPress и скачивание дистрибутива WordPress с разархивированием в этот каталог.
Добавим следующие шаги в наш плейбук
```ruby
- name: Create the installation directory
      file:
        path: /srv/www
        owner: www-data
        state: directory

    - name: Install WordPress
      unarchive:
        src: https://wordpress.org/latest.tar.gz
        dest: /srv/www
        owner: www-data
        remote_src: yes
```
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ ansible-playbook playbooks/install.yml
PLAY [wp_app] ************************************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************************
ok: [app2]
ok: [app]

TASK [Update apt-get repo and cache] *************************************************************************************************************************************************
ok: [app2]
ok: [app]

TASK [install dependencies] **********************************************************************************************************************************************************
ok: [app2]
ok: [app]

TASK [Create the installation directory] *********************************************************************************************************************************************
changed: [app2]
changed: [app]

TASK [Install WordPress] *************************************************************************************************************************************************************
changed: [app2]
changed: [app]

PLAY RECAP ***************************************************************************************************************************************************************************
app                        : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
app2                       : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```
В каталоге ```templates``` создадим файл ```wordpress.conf.j2``` со следующим содержимым:
```
<VirtualHost *:80>
    DocumentRoot /srv/www/wordpress
    <Directory /srv/www/wordpress>
        Options FollowSymLinks
        AllowOverride Limit Options FileInfo
        DirectoryIndex index.php
        Require all granted
    </Directory>
    <Directory /srv/www/wordpress/wp-content>
        Options FollowSymLinks
        Require all granted
    </Directory>
</VirtualHost>
```
После того, как мы подготовили эту конфигурацию, добавим шаг в плейбук:
```ruby
  - name: Copy file with owner and permissions
      template:
        src: ../templates/wordpress.conf.j2
        dest: /etc/apache2/sites-available/wordpress.conf
        owner: "www-data"
```
После того как в конфигурации Apache появилась описание нашего сайта, необходимо выполнить ряд дополнительных шагов:

включить в Apache модуль rewrite
“активировать” наш сайт
“деактивировать” сайт-заглушку (“It Works”)
Задачу с включением модуля мы можем решить при помощи ansible-модуля ```apache2_module```. Для остальных задач придется задействовать ansible-модуль ```shell```.

Добавим новые задачи в плейбук:
```ruby
- name: Enable URL rewriting
      apache2_module:
        name: rewrite
        state: present

    - name: Enable the site
      shell: a2ensite wordpress

    - name: Disable the default “It Works” site
      shell: a2dissite 000-default

    - name: Reload service apache2
      service:
        name: apache2
        state: reloaded
```
Последняя задача, которую надо решить в этом плейбуке, это подключить WordPress к базе данных MySql.
Для этого нам понадобится установить на хосты корневой сертификат для соединения с базой данных по SSL. И нам нужно будет внести изменения в конфигурацию уже самого WordPress.
Начнем с сертификата. Сохраним его в каталоге ```files```:		
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" -O ./files/root.crt
--2022-08-27 20:21:42--  https://storage.yandexcloud.net/cloud-certs/CA.pem
Распознаётся storage.yandexcloud.net (storage.yandexcloud.net)… 213.180.193.243, 2a02:6b8::1d9
Подключение к storage.yandexcloud.net (storage.yandexcloud.net)|213.180.193.243|:443... соединение установлено.
HTTP-запрос отправлен. Ожидание ответа… 200 OK
Длина: 3579 (3,5K) [application/x-x509-ca-cert]
Сохранение в: ‘./files/root.crt’

./files/root.crt                              100%[===============================================================================================>]   3,50K  --.-KB/s    за 0s      

2022-08-27 20:21:42 (572 MB/s) - ‘./files/root.crt’ сохранён [3579/3579]
```
Далее, в каталоге ```templates``` создадим файл ```wp-config.php.j2``` со следующим содержимым:
```ruby
<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the installation.
 * You don't have to use the web site, you can copy this file to "wp-config.php"
 * and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * MySQL settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://wordpress.org/support/article/editing-wp-config-php/
 *
 * @package WordPress
 */

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', '{{ wordpress_db_name }}' );

/** MySQL database username */
define( 'DB_USER', '{{ wordpress_db_user }}' );

/** MySQL database password */
define( 'DB_PASSWORD', '{{ wordpress_db_password }}' );

/** MySQL hostname */
define( 'DB_HOST', '{{ wordpress_db_host }}' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

/**#@+
 * Authentication unique keys and salts.
 *
 * Change these to different unique phrases! You can generate these using
 * the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}.
 *
 * You can change these at any point in time to invalidate all existing cookies.
 * This will force all users to have to log in again.
 *
 * @since 2.6.0
 */
define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );

/**#@-*/

/**
 * WordPress database table prefix.
 *
 * You can have multiple installations in one database if you give each
 * a unique prefix. Only numbers, letters, and underscores please!
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 *
 * For information on other constants that can be used for debugging,
 * visit the documentation.
 *
 * @link https://wordpress.org/support/article/debugging-in-wordpress/
 */
define( 'WP_DEBUG', true );
define( 'WP_DEBUG_LOG', true );

/* Add any custom values between this line and the "stop editing" line. */



/* That's all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
        define( 'ABSPATH', __DIR__ . '/' );
}

define('MYSQL_CLIENT_FLAGS', MYSQLI_CLIENT_SSL);

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';
```
Запустим плейбук еще раз
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ ansible-playbook playbooks/install.yml
```
И попробуем открыть страницу в браузере обращаясь к IP балансировщика

![image](https://github.com/mikhail-pavliy/terraform_ansible/blob/main/Screenshot_20220827_202827.png)

3. Преобразование плейбука в роль

Пришло время решить основную задачу - на основе разработанного плейбука создать роль.
Подготовим структуру каталогов:
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ mkdir roles/wordpress
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ mkdir roles/wordpress/defaults
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ mkdir roles/wordpress/files
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ mkdir roles/wordpress/meta
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ mkdir roles/wordpress/tasks
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ mkdir roles/wordpress/templates
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ tree roles/wordpress/
```
Переменные, которые нам необходимы, опишем в файле ```defaults/main.yml``` и укажем некие абстрактные значения.
Реальные же значения переменных задаются в описании групповых переменных.
```ruby
wordpress_db_name: "mysql_db_name"
wordpress_db_user: "mysql_username"
wordpress_db_password: "mysql_user_password"
wordpress_db_host: "mysql_fqdn"
```
В файле ```meta/main.yml``` укажем общие сведения о нашей роли
```ruby
---
galaxy_info:
  role_name: wordpress
  author: sablin
  description: Wordpress CMS System
  license: "license (MIT)"

  min_ansible_version: 2.9

  galaxy_tags:
    - linux
    - server
    - web
    - wordpress
    - cms

  platforms:
    - name: Ubuntu
      versions:
        - Focal
```
Далее, в каталог роли ```files``` скопируем корневой сертификат.
А в каталог роли templates скопируем шаблоны ```wordpress.conf.j2 ```и ```wp-config.php.j2.```
И нам осталось перенести шаги плейбука в файл ```tasks/main.yml```.
```ruby
---
- name: Update apt-get repo and cache
  apt: update_cache=yes force_apt_get=yes cache_valid_time=3600

- name: install dependencies
  package:
    name:
      - apache2
      - ghostscript
      - libapache2-mod-php
      - mysql-server
      - php
      - php-bcmath
      - php-curl
      - php-imagick
      - php-intl
      - php-json
      - php-mbstring
      - php-mysql
      - php-xml
      - php-zip
    state: present

- name: Create the installation directory
  file:
    path: /srv/www
    owner: www-data
    state: directory

- name: Install WordPress
  unarchive:
    src: https://wordpress.org/latest.tar.gz
    dest: /srv/www
    owner: www-data
    remote_src: yes

- name: Copy file with owner and permissions
  template:
    src: wordpress.conf.j2
    dest: /etc/apache2/sites-available/wordpress.conf

- name: Enable URL rewriting
  apache2_module:
    name: rewrite
    state: present

- name: Enable the site
  shell: a2ensite wordpress

- name: Disable the default “It Works” site
  shell: a2dissite 000-default

- name: Reload service apache2
  service:
    name: apache2
    state: reloaded

- name: Install mysql cert
  copy:
    src: ./files/root.crt
    dest: /usr/local/share/ca-certificates/root.crt

- name: Update cert index
  shell: /usr/sbin/update-ca-certificates

- name: Configure WordPress
  template:
    src: wp-config.php.j2
    dest: "/srv/www/wordpress/wp-config.php"
    owner: "www-data"
```
структура каталогов роли должна выглядеть так:
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ tree roles/wordpress/
roles/wordpress/
├── defaults
│   └── main.yml
├── files
│   └── root.crt
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    ├── wordpress.conf.j2
    └── wp-config.php.j2
```
Основной плейбук теперь будет выглядеть следующим образом:
```ruby
- hosts: wp_app
  become: yes

  roles:
    - role: wordpres
```
Запустим еще раз плейбук, чтобы убедиться, что ничего не сломалось:
```ruby	
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ ansible-playbook playbooks/install.yml
PLAY [wp_app] ************************************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************************
ok: [app]
ok: [app2]

TASK [wordpress : Update apt-get repo and cache] *************************************************************************************************************************************
ok: [app2]
ok: [app]

TASK [wordpress : install dependencies] **********************************************************************************************************************************************
ok: [app2]
ok: [app]

TASK [wordpress : Create the installation directory] *********************************************************************************************************************************
ok: [app]
ok: [app2]

TASK [wordpress : Install WordPress] *************************************************************************************************************************************************
ok: [app2]
ok: [app]

TASK [wordpress : Copy file with owner and permissions] ******************************************************************************************************************************
ok: [app]
ok: [app2]

TASK [wordpress : Enable URL rewriting] **********************************************************************************************************************************************
ok: [app]
ok: [app2]

TASK [wordpress : Enable the site] ***************************************************************************************************************************************************
changed: [app]
changed: [app2]

TASK [wordpress : Disable the default “It Works” site] *******************************************************************************************************************************
changed: [app]
changed: [app2]

TASK [wordpress : Reload service apache2] ********************************************************************************************************************************************
changed: [app2]
changed: [app]

TASK [wordpress : Install mysql cert] ************************************************************************************************************************************************
ok: [app]
ok: [app2]

TASK [wordpress : Update cert index] *************************************************************************************************************************************************
changed: [app2]
changed: [app]

TASK [wordpress : Configure WordPress] ***********************************************************************************************************************************************
ok: [app2]
ok: [app]

PLAY RECAP ***************************************************************************************************************************************************************************
app                        : ok=13   changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
app2                       : ok=13   changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```
Если плейбук отработал без ошибок, то теперь можно пройти на сайт и закончить его настройку

![image](https://github.com/mikhail-pavliy/terraform_ansible/blob/main/Screenshot_20220827_205752.png)

![image](https://github.com/mikhail-pavliy/terraform_ansible/blob/main/Screenshot_20220827_205956.png)

```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/09-terraform/ansible$ tree
.
├── ansible.cfg
├── environments
│   └── prod
│       ├── group_vars
│       │   └── wp_app
│       └── inventory
├── playbooks
│   └── install.yml
└── roles
    └── wordpress
        ├── defaults
        │   └── main.yml
        ├── files
        │   └── root.crt
        ├── meta
        │   └── main.yml
        ├── tasks
        │   └── main.yml
        └── templates
            ├── wordpress.conf.j2
            └── wp-config.php.j2
```