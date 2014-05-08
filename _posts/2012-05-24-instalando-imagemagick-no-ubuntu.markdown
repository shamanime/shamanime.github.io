---
layout: post
title: "Instalando ImageMagick no Ubuntu"
date: 2012-05-24 13:07
comments: true
categories:
- Ubuntu
---

ImageMagick é um utilitário que uso em muitos projetos, e sempre foi problema pra ser instalado.

Finalmente consegui instalar sem complicações!

### Instalando as dependências...
Está usando Ubuntu 12.04 desktop? Então primeiro instale o `aptitude` com `sudo apt-get install aptitude`!

{% highlight sh %}
$ sudo aptitude install libperl-dev
$ sudo aptitude install libmagickwand-dev
$ sudo aptitude install libmagickcore-dev libmagick++-dev
{% endhighlight %}

### … e agora o ImageMagick!

{% highlight sh %}
$ wget http://www.imagemagick.org/download/ImageMagick-6.7.7-0.tar.gz
$ tar -vzxf ImageMagick-6.7.7-0.tar.gz
$ cd ImageMagick-6.7.7-0
$ ./configure
$ make
$ sudo make install
$ sudo ldconfig /usr/local/lib
{% endhighlight %}

Agora você pode instalar suas gemas em paz! ;-)

