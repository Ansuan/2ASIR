# Composer PHP
## Instalación
Crear un directorio y en su interior un archivo script shell con el que se automatiza la instalación de Composer
```sh
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('sha384', 'composer-setup.php') === '48e3236262b34d30969dca3c37281b3b4bbe3221bda826ac6a9a62d6444cdb0dcd0615698a5cbe587c3f0fe57a54d8f5') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php --install-dir=bin
php -r "unlink('composer-setup.php');"
```
Ejecutar el script. Una vez ejecutado generara un archivo llamado composer.phar. Para verificar la versión y el funcionamiento usar:
```sh
php composer.phar -V
```
## Instalación de paquetes
Para la instalación de paquetes se podra realizar usando ***php composer require monolog/monolog*** o creando un archivo json llamandolo **composer.json** y despues usando ***php composer update*** para la instalación y actualización de paquetes
```json
{
    "require": {
        "monolog/monolog": "1.0.*"
    }
}
```
## Ejemplo de uso
Instalar el paquete **rss-php** con el comando ***php composer.phar require dg/rss-php***
Crear un archivo PHP en el directorio del servidor web como el siguiente. Sera necesario hacer uso de **autoload.php** con el cual podremos hacer uso de las funciones de los paquetes instalados
```php
<?php
require '/home/usuario/composer/bin/vendor/autoload.php';
$url = 'https://elpais.com/rss/elpais/portada.xml';
$rss = Feed::loadRss($url);
echo 'Title: ', $rss->title;
echo 'Description: ', $rss->description;
echo 'Link: ', $rss->link;

foreach ($rss->item as $item) {
	echo 'Title: ', $item->title;
	echo 'Link: ', $item->link;
	echo 'Timestamp: ', $item->timestamp;
	echo 'Description ', $item->description;
	echo 'HTML encoded content: ', $item->{'content:encoded'};
}
?>
```