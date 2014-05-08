---
layout: post
title: "TS3 Server no Ubuntu"
date: 2012-05-29 13:07
comments: true
categories:
- Ubuntu
---

Um pequeno tutorial sobre como instalar o TeamSpeak 3 no Ubuntu 12.04 LTS.

## Crie o usuário
Você precisa criar um usuário sobre o qual o TeamSpeak 3 irá rodar:

{% highlight sh %}
$ sudo adduser --disabled-login teamspeak
{% endhighlight %}

## Faça o download e extraia
Agora faça o download do software (64 bits no meu caso) e descompacte:

{% highlight sh %}
$ wget http://teamspeak.gameserver.gamed.de/ts3/releases/3.0.5/teamspeak3-server_linux-amd64-3.0.5.tar.gz
$ tar -zxf teamspeak3-server_linux-amd64-3.0.5.tar.gz
{% endhighlight %}

Mova os arquivos para um lugar mais apropriado, e mude as permissões da pasta:

{% highlight sh %}
$ sudo mv teamspeak3-server_linux-amd64 /opt/ts3
$ sudo chown -R teamspeak /opt/ts3
{% endhighlight %}

## Adicione o script de inicialização
Se você olhar dentro da pasta `/opt/ts3`, irá encontrar um script para iniciar/desligar o servidor chamado `ts3server_startscript.sh`.

Crie o seguinte arquivo `/etc/init.d/teamspeak` com seu editor favorito e adicione o seguinte conteúdo:

{% highlight sh %}
#! /bin/sh
### BEGIN INIT INFO
# Provides:          teamspeak
# Required-Start:    networking
# Required-Stop:
# Default-Start:     2 3 4 5
# Default-Stop:      S 0 1 6
# Short-Description: TeamSpeak Server Daemon
# Description:       Starts/Stops/Restarts the TeamSpeak Server Daemon
### END INIT INFO

set -e

PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
DESC="TeamSpeak Server"
NAME=teamspeak
USER=teamspeak
DIR=/opt/ts3
DAEMON=$DIR/ts3server_startscript.sh
#PIDFILE=/var/run/$NAME.pid
SCRIPTNAME=/etc/init.d/$NAME

# Gracefully exit if the package has been removed.
test -x $DAEMON || exit 0

cd $DIR
sudo -u teamspeak ./ts3server_startscript.sh $1
{% endhighlight %}

Modifique as permissões do arquivo e inicie o servidor:

{% highlight sh %}
$ sudo chmod 755 /etc/init.d/teamspeak
$ /etc/init.d/teamspeak start
{% endhighlight %}

Pronto! Lembre-se de anotar o token para ganhar permissões em seu primeiro acesso!

Se você possuir um arquivo `license.dat` é só colar na pasta `/opt/ts3` e reiniciar o servidor ;-)

