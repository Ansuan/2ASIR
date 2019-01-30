# Codeigniter

## Instalación

Para la instalación tendremos que ir a su web oficial , descargar la ultima versión y descomprimirla 

Una vez que tengamos el directorio tendremos que introducirlo en ***/var/www/html***

## Funcionamiento MVC (Basico)

En el siguiente manual se usará los archivos por defecto de codeignaiter y se da por sabida la estructura de los directorio y archivos.

Para empezar editaremos el fichero ***CI/application/controllers/Welcome.php***
```php
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

class Welcome extends CI_Controller {
    public function index()//Funcion por defecto
    {
    	$this->load->view('welcome_message');//vista por defecto
    }
    public function pipo() //Esta seria la nueva funcion 
    {
	    $this->load->view('pipo');//aqui le indicamos la vista que tiene que cargar
    }
}
```
Ahora en las vistas tendremos que agregar pipo ***CI/application/views/pipo.php***
```html
<html>
	<head>
		<title>Tutorial</title>
	</head>
​	<body>
		Esto seria la prueba del tutorial
​	</body>
</html>
```


Para comprobar que funciona necesitamos un navegador y la siguiente url

[http://localhost/tutorial/index.php/Welcome/pipo](http://192.168.1.155/tutorial/index.php/Welcome/pipo)

Si quisiéramos acceder a la pagina por defecto que nos ofrece seria la siguiente

http://localhost/tutorial/index.php/Welcome/index

## Cargar un archivo usando file_get_content

***CI/application/controllers/Welcome.php***

```php
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

class Welcome extends CI_Controller {
    public function index() //funcion por defecto
    {
        $this->load->view('welcome_message');//Muestra la vista Welcome.php
        }
    public function pipo()//Funcion en la que bamos a usar file_get_content
    {
        $ar['contenido'] = file_get_contents('/etc/passwd',false);
        //metemos en un array el contenido del fichero /etc/passwd
        $this->load->view('pipo', $ar);
        //cargamos el contendo en la vista
    }

}
```

***CI/application/views/pipo.php***
```html
<?php
defined('BASEPATH') OR exit('No direct script access allowed');
?><!DOCTYPE html>
<html lang="en">
	<head></head>
	<title></title>
​	<body>
​		<?php echo $contenido ?>
​	</body>
</html>
```
Para probar su funcionamiento usaremos la siguiente URL

http://localhost/CI/index.php/Welcome/pipo

## Contador de visitas con REDIS

Usaremos el codigo anterior añadiendo lo siguiente en el controlador pero esta vez en la funcion index
```php
public function index()
	{
		//$this->load->view('welcome_message',$data);
		$this->load->driver('cache');
		$data['visitas'] = $this->cache->redis->increment('visitas');
		$this->load->view('welcome_message',$data);
  }
```
En la vista añadimos lo siguiente.

```php
(<?=$visitas?>)
```

Una vez terminado con el código tendremos que iniciar redis

![img](https://lh4.googleusercontent.com/vviblqnMG49lmYlRMs7ey7_H9cgcXMxWJheqjXQ-egH-f0nELvbt7Myl2_mpzU6C-HPOLn3jdoLdBci49fWZP6VayyAI86G_dUhc7qIV7MFPk66K-Cki3Jw1_h9StoAerdIzahXn)

Cuando ya lo tengamos iniciado probamos con la siguiente url

http://localhost/CI/index.php/Welcome/index


### Uso de base de datos
Editar el archivo ***CI/application/config/database.php*** para introducir los parametros para acceder a la base de datos
```php
<?php
defined('BASEPATH') OR exit('No direct script access allowed');
$active_group = 'default';
$query_builder = TRUE;

$db['default'] = array(
	'dsn'	=> '',
	'hostname' => 'localhost',
	'username' => 'root',
	'password' => 'root',
	'database' => 'prueba',
	'dbdriver' => 'mysqli',
	'dbprefix' => '',
	'pconnect' => FALSE,
	'db_debug' => (ENVIRONMENT !== 'production'),
	'cache_on' => FALSE,
	'cachedir' => '',
	'char_set' => 'utf8',
	'dbcollat' => 'utf8_general_ci',
	'swap_pre' => '',
	'encrypt' => FALSE,
	'compress' => FALSE,
	'stricton' => FALSE,
	'failover' => array(),
	'save_queries' => TRUE
);
```
Crear un modelo nuevo en ***CI/application/models/News_model.php***
```php
<?php
class News_model extends CI_Model {

    public function __construct(){
        $this->load->database();
    }

    public function get_news($slug = FALSE){
        if ($slug === FALSE){
            $query = $this->db->get('news');
            return $query->result_array();
        }

        $query = $this->db->get_where('news', array('slug' => $slug));
        return $query->row_array();
    }

    public function set_news(){
        $this->load->helper('url');

        $slug = url_title($this->input->post('title'), 'dash', TRUE);

        $data = array(
            'title' => $this->input->post('title'),
            'slug' => $slug,
            'text' => $this->input->post('text')
        );
        return $this->db->insert('news', $data);
    }
}
?>
```
Crear un nuevo controlador en ***CI/application/controllers/News.php***
```php
<?php
class News extends CI_Controller {
  public function __construct(){
    parent::__construct();
    $this->load->model('news_model');
  }
    public function index(){
        $data['news'] = $this->news_model->get_news();
        $data['title'] = 'News archive';
        $this->load->view('news/index', $data);
    }
    public function view($slug = NULL)
    {
        $data['news_item'] = $this->news_model->get_news($slug);
        if (empty($data['news_item']))
        {
                //show_404();
                $data['news'] = $this->news_model->get_news();
                $data['title'] = 'News archive';
                $this->load->view('news/index', $data);
        }
        $data['title'] = $data['news_item']['title'];
        $this->load->view('news/view', $data);
    }
    public function create(){
        $this->load->helper('form');
        $this->load->library('form_validation');
        $data['title'] = 'Create a news item';
        $this->form_validation->set_rules('title', 'Title', 'required');
        $this->form_validation->set_rules('text', 'text', 'required');
        if ($this->form_validation->run() === FALSE)
        {
            $this->load->view('news/create');
        }
        else
        {
            $this->news_model->set_news();
            $this->load->view('news/success');
        }
    }
}
?>
```
Crear una nueva vista en ***CI/application/views/news/index.php*** para mostrar todas las noticias
```html
<h2><?php echo $title ?></h2>
<?php foreach ($news as $news_item): ?>
    <h3><?php echo $news_item['title'] ?></h3>
    <div class="main">
            <?php echo $news_item['text'] ?>
    </div>
    <p><a href="<?php echo $news_item['slug'] ?>">View article</a></p>
<?php endforeach ?>
```
Crear una nueva vista en ***CI/application/views/news/view.php*** para permitir visualizar una sola noticia
```php
<?php
echo '<h2>'.$news_item['title'].'</h2>';
echo $news_item['text'];
?>
```
Crear una nueva vista en ***CI/application/views/news/create.php*** para insertar en bd
```html
<h2><?php echo $title ?></h2>
<?php echo validation_errors(); ?>
<?php echo form_open('news/create') ?>
    <label for="title">Title</label>
    <input type="input" name="title" /><br />
    <label for="text">Text</label>
    <textarea name="text"></textarea><br />
    <input type="submit" name="submit" value="Create news item" />
</form>
```

### Uso de helper
Crear un nuevo helper en ***CI/application/helpers/site_helper.php*** en el que se incluira la funcion de CURL por ello se requerira la instalación de la clase ***CURL*** siendo desde Ubuntu ***apt install php-curl***
```php
<?php
 function getPage($url,$method,$params) {
    $headers=array(
       'Content-Type: application/json',
    );
    $ch = curl_init();
    
    curl_setopt($ch, CURLOPT_USERAGENT, true);

    curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
    curl_setopt($ch, CURLOPT_URL, $url);
    if($method=="POST") {
      curl_setopt($ch, CURLOPT_CUSTOMREQUEST, "POST");
      curl_setopt($ch, CURLOPT_POSTFIELDS,$params);
    }
    if($method=="DELETE" || $method=="PUT") {
      curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
      curl_setopt($ch, CURLOPT_POSTFIELDS, $params);
    }
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);

    $result = curl_exec($ch);
    curl_close($ch);
    return $result;
 }
 function getSslPage($url) {
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, FALSE);
    curl_setopt($ch, CURLOPT_USERAGENT, true);
    # # curl_setopt($ch, CURLOPT_HEADER, false);
    curl_setopt($ch, CURLOPT_HEADER, 'Content-Type: application/json');
    curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_REFERER, $url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);
    $result = curl_exec($ch);
    curl_close($ch);
    return $result;
 }
?>
```
Añadir en ***CI/application/config/autoload.conf*** 
```php
$autoload['helper'] = array(site);
```
Añadir en el controlador ***CI/application/controllers/Welcome.php*** haciendo uso del helper de la clase CURL para obtener el JSON, decodificarlo para guardarlo en un array y pasarlo a la vista
```php
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

class Welcome extends CI_Controller {

        public function index()
        {
                $this->load->helper('site');
                $url='http://';
                $data['obj']=json_decode(getSslPage($url),true);
                $this->load->view('welcome_message',$data);
        }
}
```
Añadir en la vista ***CI/application/views/welcome_message.php***  un foreach para recorrer el array pasado
```php
<?php
foreach ($obj as $k => $v){
        echo $v['id'] . '<br>';
        echo $v['name'] . '<br>';
}
?>
```
