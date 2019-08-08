---
layout: post
title: Rsyslog и Graylog
description: Как с помощью rsyslog сливать логи в graylog и не облажаться.
categories: [devops]
tags: [rsyslog, graylog, logging]
language: ru
---

# Rsyslog и Graylog

Обычно, если речь заходит о современных системах сбора, хранения и анализа логов, на ум сразу приходят Graylog или ELK. В нашей компании выбор пал на Graylog (так как, в отличие от ELK или Splunk, мы были с ним немного знакомы).

И так, graylog установлен и запущен (в docker-контейнере), надо кормить его логами. Для этого у нас есть несколько путей

1. Агенты на местах, которые будут читать лог файлы на сервере, форматировать их в удобный для лог-сервера вид и отправлять дальше. Сюда можно отнести beats и NXLog. Конфигурацию для них можно централизованно разливать с помощью collector sidecar.

2. Другой популярной альтернативой является передача syslog сообщений по сети и форматирование их удобный для анализа вид уже на сервере. 

3. Сливать уже отформатированные в GELF формат сообщения.

У каждого из этих подходов нашлись свои минусы, которые вынудили рассмотреть альтернативное решение. Поговорим об этих минусах.

## Почему мы отказались от syslog'а

Изначальная идея заключалась в том, что бы отправлять данные через syslog протокол и разбирать их на индексированные поля средствами Graylog через регулярные выражения, grok паттерны и тп. Отпрвлялись данные rsyslog'ом, в основном потому, что на debian он уже есть. На этом пути мы столкнулись с непредвиденной, но важной трудностью: даже с нашим довольно небольшим (~40 rps) количеством сообщений Graylog периодически вставал. Вот прямо как описано [тут](https://github.com/Graylog2/graylog2-server/issues/4326). Аналогичные топики на форуме грейлога обычно заканчиваются решением в духе "вам нужен кластер помощнее", текущий не справляется с нагрузкой.

Кроме того, syslog не умеет отправлять мультистроковые сообщения (php-fpm-slowlog, mysql slowlog, stacktrace'ы, специфичные логи приложений), что здорово мешает их анализировать. Из не очень понятных для меня соображений, это запрещено и команда graylog не считает это проблемой. [1](https://github.com/Graylog2/graylog2-server/issues/5971), [2](https://github.com/Graylog2/graylog2-server/issues/5225), [3](https://github.com/Graylog2/graylog2-server/issues/4536) и это не полный список issues на эту тему. 

Все это привело нас к мысли, что разбивать сообщения на поля нужно на местах, а в грейлог отправлять уже отформатированные данные.

## Почему мы отказались от агентов

И так, агенты. Это хорошее решение, которое позволяет достаточно быстро направить красивые отформатированные логи на сервер. Но, к сожалению, тот же beats умеет читать логи только из сторонних файлов, но не из сокета. 
Так же я не разобрался (допускаю, что как-то это все-таки можно сделать), как изменить конфигурацию beats для того или иного сервиса. Например, в nginx access_log мы добавили время ответа upstream сервера, но заставить beats учесть эти изменения у меня так и не вышло.

## Почему мы используем rsyslog

На rsyslog как средство доставки сообщений в graylog мы стали смотреть, потому что в Debian он уже есть, умеет кучу всего и неплохо документирован. Пока мы предпринимали неудачный заход номер один c plain syslog сообщениями, удалось неплохо разобраться в синтаксисе rsyslog и том, как он работает. Так что проделывать то же самое ради beats, который, к тому же, не умеет ряд нужных нам вещей (типа чтения логов из сокета) не очень хотелось.

## Как подружить rsyslog и gelf.

Для начала, rsyslog не умеет прямо совсем из коробки конвертировать сообщения в GELF. Но умеет конвертировать их в JSON и имеет довольно развитую систему шаблонов. Поэтому с помощью вот такого трюка можно конвертировать в gelf обычные syslog сообщения.

```
template(name="gelf" type="list") {
  constant(value="{\"version\":\"1.1\",")
  constant(value="\"host\":\"")                  property(name="hostname")
  constant(value="\",\"short_message\":\"")      property(name="msg" format="json")
  constant(value="\",\"timestamp\":")            property(name="timegenerated" dateformat="unixtimestamp")
  constant(value=",\"_application_name\":\"")    property(name="app-name")
  constant(value="\",\"_facility\":\"")          property(name="syslogfacility-text")
  constant(value="\",\"level\":\"")              property(name="syslogseverity")
  constant(value="\"}")
}

call graylog
ruleset(name="graylog") {
  action(
    type="omfwd"
    Target="graylog.cetera.ru"
    Port="12201"
    Protocol="tcp"
    KeepAlive="on"
    template="gelf"
    StreamDriver="gtls"
    StreamDriverMode="1"
    StreamDriverAuthMode="x509/name"
    StreamDriverPermittedPeers="graylog.cetera.ru"
    TCP_FrameDelimiter="0"
  )
}
```

Возможно, вы наткнетесь на [вот такую](https://www.rsyslog.com/doc/master/tutorials/gelf_forwarding.html) документацию Rsyslog и заметите несколько отличий:

1. Поле timestamp в документации Rsyslog взято в кавычки, но современный стандарт gelf требует, чтобы это было число. Поэтому кавычки лишние.
2. Добавлены поля _application_name и _facility. Потому что мы можем это сделать ))
3. Используется не udp, а tcp. Пожалуй, самый контринтуитивный момент - документация явно сообщает нам, что "_We now have a syslog forwarding action. This uses the omfwd module. Please note that the case above only works for UDP transport. When using TCP, Graylog expects a Nullbyte as message delimiter. This is currently not possible with rsyslog_". Но на самом деле все уже давно возможно - вот этот момент [описан в документации Rsyslog](https://www.rsyslog.com/doc/master/configuration/modules/omfwd.html#tcp-framedelimiter), вот [issue](https://github.com/rsyslog/rsyslog/issues/1602), которое сделало это возможным.
4. Используется шифрование через сертификаты.

Так что на самом деле, rsyslog вполне в состоянии отправлять GELF сообщения, делать это с современными версиями Graylog (проверено на Graylog 2.5), с шифрованием и по TCP протоколу. Надеюсь, и документация Rsyslog скоро прольем свет на этот момент.

### Продвинутая индексация и форматирование сообщений.

Но раз мы используем graylog, хочется иметь не только логи, но и возможность их как-то анализировать. Для этого нам нужны gelf сообщения с дополнительными полями. Чтобы приготовить их, восползуемся киллер фичей rsyslog'а - [модулем mmnormalize](https://www.rsyslog.com/doc/master/configuration/modules/mmnormalize.html) и [liblognorm](https://github.com/rsyslog/liblognorm/blob/master/doc/configuration.rst). Важная ремарка - здесь и далее речь о liblognorm версии 2.

Вкратце, liblognorm позволяет очень быстро парсить сообщения с grok-подобных патернов, но без использования регулярных выражений. Соответвенно, шаблоны не такие гибкие как в grok, но их хватает. Mmnormalize позволяет связывать правила liblognorm и шаблоны rsyslog.

Для этого используем немного другой шаблон и, как следствие, другой ruleset.

```
template(name="gelf-ext" type="list") {
  constant(value="{\"version\":\"1.1\",")
  constant(value="\"host\":\"")                  property(name="hostname")
  constant(value="\",\"short_message\":\"")      property(name="msg" format="json")
  constant(value="\",\"timestamp\":")            property(name="timegenerated" dateformat="unixtimestamp")
  constant(value=",\"_application_name \":\"")   property(name="app-name")
  constant(value="\",\"level\":\"")              property(name="syslogseverity")
  constant(value="\",")                          property(name="$!all-json" position.from="2")
}

ruleset(name="graylog-ext") {
  action(
    type="omfwd"
    Target="graylog.cetera.ru"
    Port="12201"
    Protocol="tcp"
    KeepAlive="on"
    template="gelf-ext"
    StreamDriver="gtls"
    StreamDriverMode="1"
    StreamDriverAuthMode="x509/name"
    StreamDriverPermittedPeers="graylog.cetera.ru"
    TCP_FrameDelimiter="0"
  )
}
```

Основное отличие шаблона **graylog-ext** от **graylog** - использование строковой переменной $!all-json. В нее с помощью правил liblognorm мы поместим json-строку со всеми полями, которые были в сообщении. Ruleset **graylog-ext** отличается от **graylog** только использованием другого шаблона.

Теперь разберем, как готовить разбитые на поля gelf сообщения на нескольких примерах. Начнем с простого.

#### Proftpd xferlog

Каждое сообщение в xferlog - данные о работе с одним файлом с помощью proftpd демона. Подробнее о формате xferlog можно почитать [тут](https://linux.die.net/man/5/xferlog).

Само сообщение выглядит так `Sun Dec 30 01:38:57 2018 0 80.237.119.90 22062 /var/www/site/path/to/dir/file.php b _ o r username sftp 0 * c`. Отсюда нам хочется вытащить IP клиента, имя залогиненного пользователя, протокол, тип передачи и тд.

Вот правило liblognorm, которое это делает:

```
version=2

rule=proftpd,info,gelf:%-:word% %_timestamp:date-rfc3164{"format":"timestamp-unix"}% %-:number% %_transfer_time_:number{"format":"number"}% %_client_ip:word% %-:number% %_file_name:word% %_transfer_type:word% %-:word% %_direction:word% %_access_mode:word% %_user_name:word% %_service_name:word% %-:word% %-:word% %_completion_status:word%
```

Конструкция **%-:word%** записывает в специальную переменную **-** последовательность непробельных символов, а сама переменная **-** нужна для создания безымянного поля, которое не попадет в итоговый json.
Другой примечательной конструкцией тут является **%_transfer_time_:number{"format":"number"}%**. С еë помощью, еы записываем в json поле "\_transfer\_time" значение как число, а не как строку. Позднее это приведет к тому, что elasticsearch посчитает это числовым полем и мы сможем по этому полю построить в graylog, например, график средних значений.

Свяжем все это воедино вместе с нашим ruleset'ом **graylog-ext**.

```
input(
  type="imfile"
  File="/var/log/proftpd/xferlog"
  Tag="proftpd"
  Severity="info"
  Facility="local7"
  ruleset="proftpd"
)

ruleset(name="proftpd") {
  action(type="mmnormalize" rulebase="/etc/rsyslog.d/rules/proftpd_xfer.rb")
  call graylog-ext
}
```

С помощью модуля [imfile](https://www.rsyslog.com/doc/master/configuration/modules/imfile.html) будем читать сообщения из файла /var/log/proftpd/xferlog. В явном виде добавим имя приложения с помощью директивы tag, укажем severity и facility. Самое главное, все сообщения должны так же пройти через ruleset proftpd.

Далее, в самом ruleset'е вызовем модуль **mmnormalize** и укажем путь к файлу с правилами liblognorm. Это сформирует строку $!all-json. Затем передадим эту строку в ruleset **graylog-ext**. Этот ruleset, в свою очередь, сформирует gelf сообщение и отправит его в graylog.

Перейдем к более сложному примеру.

## PHP-FPM Slowlog

Slowlog - это многострочное сообщение со стектрейсом. Например, вот такое.
```
[02-Jan-2019 02:10:20]  [pool pool_name] pid 13776
script_filename = /var/www/site/path/to/dir/file.php
[0x00007fc26e017500] curl_exec() /var/www/site/path/to/dir/file.php:351
[0x00007fc26e017430] client() /var/www/site/www/local/widget/scripts/service.php:301
[0x00007fc26e0171f0] requestPV() /var/www/site/www/local/widget/scripts/service.php:148
[0x00007fc26e017160] getPVFile() /var/www/site/www/local/widget/scripts/service.php:34
[0x00007fc26e0170c0] getPV() /var/www/site/www/local/widget/scripts/service.php:11
```

Посмотрим, как конвертировать его можно сконфертировать в gelf. 
```
version=2

rule=php-fpm,info,gelf:\\n[%date:char-to:]%]  [pool %pool_name:char-to:]%] pid %-:number%\\nscript_filename = %script_filename:string-to{"extradata":"\\n"}%\\n%full_message:rest%
```

Символ перевода строки в правилах liblognorm записывается как _\\n_. Также заслуживают внимания две конструкции:
* [%date:char-to:]%] - тут в поле date мы запишем все символы до следующего символа **]**. Аналогично работает конструкция string-to. Разница в том, что здесь мы в качестве границы указываем не отдельный символ, а строку.
* %full_message:rest% - с помощью типа поля rest мы в full_message поместим все оставшееся сообщение.

Свяжем все это воедино.


```
input(
  type="imfile"
  file="/var/www/*/logs/*log.slow"
  startmsg.regex="^$"
  tag="php-fpm"
  readTimeout="10"
  ruleset="php-fpm-slow"
  addMetadata="off"
)

ruleset(name="php-fpm-slow") {
  action(type="mmnormalize" rulebase="/etc/rsyslog.d/rules/php_fpm_slow.rb")
  set $!full_message = replace($!full_message, '\\n', "\n");
  call graylog-ext
}
```

Обратите внимание на выражение `set $!full_message = replace($!full_message, '\\n', "\n");`. С его помощью мы убираем экранирование символа перевода строки и в грейлоге поле full_message отразится как обычное многострочное.

Напоследок, разберем другой хитрый пример.

## Nginx error_log

В лог ошибок nginx могут попадать как сообщения от nginx. так и сообщения от fastcgi серверов, которые nginx проксирует. Кроме того, сообщения могут быть как многострочные, так и однострачные. И, вишенка на торте, nginx сначала пишет тело сообщение произвольной длины, а потом добавляет мета информацию. Здесь мы также посмотрим, как с помощью одного rulebase'а применить к сообщениям разные правила. 
Примеры сообщений:

```
 2019/08/07 11:01:52 [error] 22493#22493: *2117401 FastCGI sent in stderr: "PHP message: [2019-08-07 11:01:52] production.ERROR: Error Processing Request {"userId":2,"email":"user@domain.tld","exception":"[object] (Exception(code: 1): Error Processing Request at /var/www/site/www/app/Http/Controllers/HomeController.php:26)
[stacktrace]
#0 [internal function]: App\\Http\\Controllers\\HomeController->index()
#1 /var/www/site/www/vendor/laravel/framework/src/Illuminate/Routing/Controller.php(54): call_user_func_array(Array, Array)
#2 /var/www/site/www/vendor/laravel/framework/src/Illuminate/Routing/ControllerDispatcher.php(45): Illuminate\\Routing\\Controller->callAction('index', Array)
#3 /var/www/site/www/vendor/laravel/framework/src/Illuminate/Routing/Route.php(212): Illuminate\\Routing\\ControllerDispatcher->dispatch(Object(Illuminate\\Routing\\Route), Object(App\\Http\\Controllers\\HomeController), 'index')
#4 /var/www/site/www/vendor/laravel/framework/src/Illuminate/Routing/Route.php(169): Illuminate\\Routing\\Rout" while reading response header from upstream, client: 188.126.90.15, server: host, request: "GET / HTTP/2.0", upstream: "fastcgi://unix:/var/run/php/site.sock:", host: "site"
 24079#24079: *7858927 open() "/var/www/site/www/cache/js/s1/_catalog/page_a772ecd0357d0d28edb5730c1c1fdbfc/page_a772ecd0357d0d28edb5730c1c1fdbfc_v1.js,q15428907836424.pagespeed.jm.ICbv86Z6iJ.js" failed (2: No such file or directory), client: 66.249.66.54, server: site, request: "GET /cache/js/s1/_catalog/page_a772ecd0357d0d28edb5730c1c1fdbfc/page_a772ecd0357d0d28edb5730c1c1fdbfc_v1.js,q15428907836424.pagespeed.jm.ICbv86Z6iJ.js HTTP/1.1", host: "site", referrer: "https://site/catalog/path/to/dir/"
```

И наш rulebase файл.

```
version=2

type=@value:%..:char-to:,%
type=@value:"%..:char-to:"%"

prefix= %-:word% %-:word% [error] %-:number%#%-:number%: *%-:word% 

rule=gelf,monolog,nginx,error:FastCGI sent in stderr: "PHP message: [%-:char-to:]%] %full_message:string-to{"extradata":"\" while reading response header from upstream"}%" while reading response header from upstream, %
    {"name":"meta", "type":"repeat",
    "parser":[
               {"type":"char-to", "name":"key", "extradata":":"},
               {"type":"literal", "text":": "},
               {"type":"@value",  "name":"value"}
             ],
    "while":[
               {"type":"literal", "text":", "}
            ]
    }%

rule=gelf,nginx,error:%full_message:string-to{"extradata":", client:"}%, %
    {"name":"meta", "type":"repeat",
    "parser":[
               {"type":"char-to", "name":"key", "extradata":":"},
               {"type":"literal", "text":": "},
               {"type":"@value",  "name":"value"}
             ],
    "while":[
               {"type":"literal", "text":", "}
            ]
    }%
```

Здесь мы использовали возможность создать произвольный тип - @value. Это последовательность символов, которая оканчиваетс запятой или двойной кавычкой. Также использован prefix - эта директива позволяет задать общий префикс для всех правил, которые следует за ней. Обратите внимание на [пробел в начале сообщения](https://www.rsyslog.com/log-normalization-and-the-leading-space/). И, наконец, используется тип поля repeat - специальный парсер, который позволяет в цикле обработать повторяющиеся значения. Без него тут не обойтись, потому что nginx вставляет не все ключи в метаданные сообщения. Скажем, реферера может не быть и тогда в лог он не попадет совсем; та же история с ключом upstream. И заранее мы не знаем какие ключи тут окажутся. Уверенность есть только в ключе client.

Все ключи и значения, изввлеченные repeat парсером попадут внутрь ключа meta. Получится примерно вот такая конструкция.
```
{
	"tags": ["gelf","nginx","error"],
	"full_message": "...",
	"meta": {
		"client": "...",
		"request": "...",
		...
	}
}
```

Это не хорошо, так как gelf допускает только строки и числа в качестве значений ключей. Но и эта проблема разрешима.

```
input(
  type="imuxsock"
  Socket="/run/nginx.syslog"
  ruleset="nginx"
)

ruleset(name="nginx-error-log") {
    action(type="mmnormalize" rulebase="/etc/rsyslog.d/rules/nginx_error.rb")
    set $!full_message = replace($!full_message, '#012', "\n");
    foreach ($.i in $!meta) do {
        if ($.i!key == 'host') then {
            set $!_vhost = $.i!value;
        } else if ($.i!key == 'client') then {
            set $!_client_ip = $.i!value;
        } else if ($.i!key == 'server') then {
            set $!_server = $.i!value;
        } else if ($.i!key == 'request') then {
            set $!_http_request = $.i!value;
        } else if ($.i!key == 'upstream') then {
            set $!_upstream = $.i!value;
        }
    }
    unset $!meta;
  }
  call graylog-ext
}
```

Для начала, поскольку мы читаем данные не из файла, а из сокета. Перевод строки это не _\\n_, а _#012_. И с помощью скриптового языка rsyslog aka [rainer script](https://www.rsyslog.com/doc/master/rainerscript/index.html) разберем содержимое поля meta, вытащив на первый уровень все нужные нам поля и удалив meta из $!all-json.

## Заключение

Rsyslog очень мощное средство анализа логов, которое вопреки собственной документации, отлично работает с graylog'ом. Зная синтаксис liblognorm, вы можете решить почти любую задачу (из того, с чем столкнулся я - невозможность вытащить дату из логов как timestamp, если она не в предопределнном формате. Но [надежда жива](https://github.com/rsyslog/liblognorm/issues/189)).
