---
layout: post
title: "Instalando nginx com upload module e upload progress"
date: 2012-06-13 13:08
comments: true
categories:
- Ubuntu
---

Quando o assunto é upload de arquivos a dor de cabeça é quase inevitável. Algumas semanas atrás precisei construir um sistema para upload de arquivos grandes pra um cliente, e durante o caminho aprendi alguns truques úteis.

Esse artigo faz parte de uma série onde mostrarei como instalar e configurar uma aplicação Rails + Carrierwave com nginx + upload module + upload progress em produção com Unicorn.

## Por que se importar?
Sem o upload module, a aplicação Rails receberá os dados brutos do upload e analisará todo o conteúdo antes de cuidar do arquivo. Para grandes arquivos isso pode ser bem lento, já que o Ruby estará fazendo todo o trabalho.

Com o upload module, o *parsing* do arquivo acontece em C através do nginx antes da sua aplicação em Ruby entrar em ação. O módulo coloca o arquivo em um diretório temporário e retira todos parâmetros *multipart* da requisição POST, os substituindo por parâmetros que você poderá usar na aplicação para pegar o nome e localização do arquivo no disco. Quando a requisição chegar na sua aplicação, todo trabalho pesado já foi concluído e o arquivo está pronto pra ser usado.

O upload progress module informa o progresso do upload, e é um ótimo parceiro do upload module.

## Instalando o nginx com os módulos
Baixe e extraia o código fonte do [nginx][nginx], [upload module][uploadm] e [upload progress module][uploadpm].

Eu não gosto de rodar o nginx com meu usuário ou root, então eu criei um usuário só pra ele:

{% highlight sh %}
$ sudo adduser --system --no-create-home --disabled-login --disabled-password \
--group nginx
{% endhighlight %}

Agora compile o nginx com os módulos, note que eu alterei o *usuário*, *grupo* e *local de instalação* do nginx:

{% highlight sh %}
$ cd <caminho do source do nginx>
$ ./configure --prefix=/opt/nginx --user=nginx --group=nginx --with-http_ssl_module \
--add-module=<caminho do upload module> \
--add-module=<caminho do upload progress module>
$ make
$ sudo make install
{% endhighlight %}

Nota: Se você tiver erros por causa de algum módulo, utilize o seguinte comando para o configure:

{% highlight sh %}
$ CFLAGS=-Wno-unused-but-set-variable ./configure --prefix=/opt/nginx \
--user=nginx --group=nginx --with-http_ssl_module \
--add-module=<caminho do upload module> \
--add-module=<caminho do upload progress module>
{% endhighlight %}

### Crie o arquivo de inicialização do nginx
O Linode tem um bom arquivo para iniciar o nginx:

{% highlight sh %}
$ wget -O init-deb.sh http://library.linode.com/assets/660-init-deb.sh
$ sudo mv init-deb.sh /etc/init.d/nginx
$ sudo chmod +x /etc/init.d/nginx
$ sudo /usr/sbin/update-rc.d -f nginx defaults
$ sudo /etc/init.d/nginx start
{% endhighlight %}



### Pequenas configurações
Eu gosto de manter o nginx funcionando com a mesma configuração de `sites-available` e `sites-enabled` que o Apache usa.
Pra isso, entre no local de instalação do nginx e crie os diretórios:

{% highlight sh %}
$ cd /opt/nginx/conf
$ sudo mkdir sites-enabled
$ sudo mkdir sites-available
{% endhighlight %}

Abra o arquivo `/opt/nginx/conf/nginx.conf` no seu editor de texto favoritos e altere o bloco `http` para ficar parecido com o meu:

{% highlight sh %}
http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;
    gzip on;
    gzip_http_version 1.1;
    gzip_comp_level 2;
    gzip_types    text/plain text/html text/css
                  application/x-javascript text/xml
                  application/xml application/xml+rss
                  text/javascript;

    server {
        listen       80;
        server_name  localhost;

        # If no listen is specified, all IPv4 interfaces on port 80 are listened to.
        # To listen on both IPv4 and IPv6 as well, listen [::] and 0.0.0.0
        # must be specified.
        server_name _;
        return 444;
    }

    include sites-enabled/*;
}
{% endhighlight %}

Agora só nos resta criar o arquivo de configuração da nossa aplicação e criar o link simbólico para ele:

{% highlight sh %}
$ cd /opt/nginx/conf/sites-available
$ sudo touch app.com
$ sudo ln -s /opt/nginx/conf/sites-available/app.com /opt/nginx/conf/sites-enabled/
$ sudo /etc/init.d/nginx restart
{% endhighlight %}

Pronto, nginx instalado com os módulos e devidamente configurado!

[nginx]: http://nginx.org/en/download.html
[uploadm]: http://www.grid.net.ru/nginx/upload.en.html
[uploadpm]: http://wiki.nginx.org/NginxHttpUploadProgressModule
