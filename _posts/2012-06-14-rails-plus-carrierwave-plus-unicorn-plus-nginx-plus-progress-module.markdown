---
layout: post
title: "Rails + carrierwave + unicorn + nginx + progress module"
date: 2012-06-14 13:08
comments: true
categories:
- Ubuntu
- nginx
---

No último post eu expliquei como instalar e configurar o nginx com upload e upload progress module. Agora vamos fazer nossa aplicação funcionar com o Carrierwave e Unicorn.

## Configurando o nginx
Primeiro vamos alterar o arquivo de configuração do nginx da nossa aplicação, abra o arquivo `/opt/nginx/conf/sites-available/app.com` no seu editor de textos favorito e adicione o seguinte conteúdo:

{% highlight sh %}
upstream app_unicorn {
  server unix:/tmp/unicorn.app.com.sock fail_timeout=0;
}

upload_progress proxied 1m;

server {
  listen 80;
  server_name arquivos.app.com www.arquivos.app.com;

  if ($host = 'www.arquivos.app.com' ) {
    rewrite  ^/(.*)$  http://arquivos.arquivos.app.com/$1  permanent;
  }

  root /srv/rails/app.com/current/public;

  try_files $uri/index.html $uri @unicorn;

  location @unicorn {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://app_unicorn;
  }

  # Rota para uploads da aplicação
  location /arquivos {
    # Pra onde o nginx envia a requisição quando o upload terminar
    upload_pass @unicorn;

    # Local em que o nginx vai subir o arquivo temporario
    upload_store /srv/rails/app.com/current/public/uploads/tmp 1;

    # Permissão nos arquivos enviados
    upload_store_access user:rw group:rw all:r;

    # Campos passados no request body para o Rails
    upload_set_form_field $upload_field_name[original_filename] "$upload_file_name";
    upload_set_form_field $upload_field_name[content_type] "$upload_content_type";
    upload_set_form_field $upload_field_name[path] "$upload_tmp_path";

    upload_pass_form_field "^X-Progress-ID$|^authenticity_token$";

    upload_cleanup 400 404 499 500-505;

    track_uploads proxied 30s;
  }

  location /uploads {
    # Força download dos arquivos que estiverem dentro de uploads
    if ($request_filename !~* ^.*?\.(jpg)|(png)|(gif)) {
      add_header Content-Type "application/octet-stream";
      add_header Content-Disposition "attachment; filename=$1";
    }
  }

  location ^~ /progress {
    upload_progress_json_output;
    report_uploads proxied;
  }

  client_max_body_size 4G;
  keepalive_timeout 10;
  error_page 500 502 503 504 /500.html;
  location = /500.html {
    root /srv/rails/app.com/current/public;
  }
}
{% endhighlight %}

O arquivo está bem comentado e explicado, tem a configuração do Unicorn e a parte que nos interessa do upload.

A aplicação responde com o progresso do arquivo enviado na url `/progress`, e eu uso a url `/uploads` pra forçar o download dos arquivos que foram enviados.

Mas ainda é necessário criar os diretórios temporários que o nginx usará:

{% highlight sh %}
$ cd /srv/rails/app.com/current/public/uploads/tmp
$ mkdir 0 1 2 3 4 5 6 7 8 9
$ sudo chown -R nginx:nginx /srv/rails/app.com/current/public/uploads/tmp
{% endhighlight %}

## Configurando a aplicação Rails

Agora é a hora de configurar algumas partes da nossa aplicação, primeiro vamos alterar o Model que receberá o upload:

{% highlight ruby %}
# app.com/app/models/arquivo.rb
class Arquivo
  include Mongoid::Document
  # ...

  attr_accessible :arquivo, :original_name, :content_type, :path

  mount_uploader :arquivo, ArquivoUploader

  # ...
end
{% endhighlight %}

Agora criamos o `ArquivoUploader.rb`:

{% highlight ruby %}
# app.com/app/uploaders/arquivo_uploader.rb
# encoding: utf-8
require 'carrierwave/processing/mime_types'
class ArquivoUploader < CarrierWave::Uploader::Base
  include CarrierWave::MimeTypes

  process :set_content_type

  def move_to_store
    true
  end

  # Choose what kind of storage to use for this uploader:
  storage :file

  # Override the directory where uploaded files will be stored.
  # This is a sensible default for uploaders that are meant to be mounted:
  def store_dir
    "#{Rails.root}/public/uploads"
  end
end
{% endhighlight %}

Criamos então uma rota para receber o formulário de arquivos:

{% highlight ruby %}
# app.com/config/routes.rb
resources :arquivos, :only => [:new, :create]
{% endhighlight %}

E a ação correspondente no Controller para salvar o arquivo enviado:

{% highlight ruby %}
# app.com/app/controllers/arquivos_controller.rb
  def create
    @arquivo = Arquivo.new(params[:arquivo])

    # Para receber o arquivo do nginx em produção
    if Rails.env != "development"
      @arquivo.arquivo = ActionDispatch::Http::UploadedFile.new(
        filename: params['arquivo']['arquivo']['original_filename'],
        tempfile: File.open(params['arquivo']['arquivo']['path'])
      )
    end

    if @arquivo.save
      respond_to do |format|
        format.html {
          render :json => @arquivo,
          :content_type => 'text/html',
          :layout => false
        }
        format.json {
          render :json => @arquivo
        }
      end
    else
      render :json => @arquivo.errors, :status => :unprocessable_entity
    end
  end
{% endhighlight %}

A parte mágica aqui é simular um `ActionDispatch::Http::UploadedFile` sendo passado para o Carrierwave. Dessa forma, a aplicação recebe o arquivo do nginx e o Carrierwave move o arquivo do diretório temporário para o `store_dir` configurado anteriormente.

