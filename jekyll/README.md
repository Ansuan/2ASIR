# Jekyll
## Instalación
Instalar los paquetes siguientes:
```sh
sudo apt-get install ruby-full build-essential zlib1g-dev
```

Incluir las variables de ruby:
```sh
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Instalar jekyll:
```sh
gem install jekyll bundler
```

## Creación pagina
Generar el sitio:
```sh
jekyll new sitio
```

Inciar el sitio:
```sh
cd sitio
bundle exec jekyll serve
```
Usar este parametro si se desea ver desde otra IP que no sea localhost:
```sh
bundle exec jekyll serve --host=203.0.113.0
```
