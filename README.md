# Карта Docker'a

![Карта Docker](docker_map.jpg)

# `Level D`

## Docker layer

Каждая команда докера - read only слой, Docker использует стратегию cow (copy on write), т.е. при запуске появляется r/w слой и работа идет только с ним, при изменении файлов из базовых слоев, они копируются в r/w слой и используются вместо базовых, значит чем меньше слоев и чем меньше они делают, тем меньше образ

Есть слои, есть и кеш, при билдинге если наш слой не изменился мы будем использовать уже существующий слой из кеша, что ускорит сборку, из этого следует, что команды нужно упорядочить по степени их изменяемости
Например: Сначала билдим зависимости, а потом копируем код, т.к. код меняется чаще зависимостей 

Слои Docker это мощный инструмент, т.к. мы можем использовать их как кеш и не выполнять команду заново, например это позволяет не билдить зависимости, каждый раз попусту гоняя трафик и время цп

## Минимизация размера и ускорение сборки

Для удобного использования в CI/CD, стараются минимизировать размер образа и использовать кэш, т.к. каждый раз качать образы по 1гб и билдить их с самого начала слишком долго, что усложняет доставку и поддержку, кто хочет ждать по 5 минут, прежде чем запустить e2e или integration тесты, которые еще и упадут и все пойдет по кругу.

Тут все просто, меньше слоев - меньше образ, поэтому чем меньше инструкций и чем лучше они написаны, тем лучше, но так же не стоит забывать про кеш и максимизировать его использование

- Для использования кеша, сначала ставим все нужные зависимости
- Минимизируем последствия установки зависимостей и объединяем команды в один слой
```Dockerfile
# Для зависимостей нужны либы? Убери за собой.
RUN \
    savedAptMark="$(apt-mark showmanual)"; \ # Запомним важные пакеты установленные "в ручную", добавь сюда свои пакеты, которые нужны в runtime
    apt-get update && \
    apt-get install -y --no-install-recommends gcc libc6-dev; \ # no-install-recommends, только критически важные пакеты
    pip install --no-cache-dir -r requirements.txt; \ # no-cache-dir не сохранять исходники и wheel
    apt-mark auto '.*' > /dev/null; \ # Пометим все пакеты как установленные автоматически
    apt-mark manual $savedAptMark; \ # Пометим важные пакеты, как установленные в ручную
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \ # Удалим все "auto" пакеты
    rm -rf /var/lib/apt/lists/*; # Удалим кеш apt
```
- Ставим dev зависимости  
Зависимости будут меняться реже чем код, поэтому зачастую будет использоваться кеш, что даст нам скорости при сборке
- Теперь можно копировать код в образ 

# `Level C`

## Multi-stage build



## Да кто такой `Fixuid` и привилегии

По умолчанию Docker работает от root, очевидно это никому не нужно и лучше проверять загружаемые через pull контейнеры, используют ли они root пользователя. 
Контейнер не должен менять свое состояние согласно [12factor](https://12factor.net/), поэтому лучшей практикой будет использовать read-only права для пользователя в контейнере для работы с данными в контейнере. 
Т.к. в контейнере будет свой пользователь, то файлы которые он будет создавать в монтированном томе будут с привилегиями этого юзера, и мы не сможем получить доступ к этим файлам от локального юзера, что создает неудобства. 
Fixuid позволяет запускать контейнер с нужными правами, т.к. при билде проставятся права либо юзера в контейнере, либо рута, если мы не указали явно, то fixuid поможет дать права запускающего юзеру в контейнере и не поломать права на хост машине
Для локальной работы fixuid не нужен, можно запустить контейнер с передачей UID, а с host машиной пускай разбираются DevOps

## ENTRYPOINT VS CMD

Если есть ENTRYPOINT, то все что есть в CMD будет выполняться после него, поэтому CMD может служить для передачи параметров, либо для исполнения действий после ENTRYPOINT

### Почему ENTRYPOINT это плохо

Мы связываем руки людям, которые будут использовать наш образ, т.к. все что будет написано в docker run, попадет туда и если мы качаем пол интернета в ENTRYPOINT, а я хочу зайти в sh, то мне будет очень неприятно, а вообще приложение само должно полностью управлять своим состоянием и само качать пол интернета и делать миграции, поэтому ENTRYPOINT это антипатерн.

И не в коем случае нельзя использовать ENTRYPOINT в shell форме, т.к. он запустит CMD в shell форме, даже если он будет в exec.

# `Level B`

## Shell vs Exec

### Shell

Позволяет использовать все преимущества shell'а, пайпы, подстановку переменных среды и прочую магию, короче привычно и удобно, но надо помнить, что shell создает дочерний процесс при выполнении команд.

### Exec

- Заменяет командную оболочку на заданную программу (не запускает новый процесс), теперь никаких пайпов и env (кроме тех что указаны в Dockerfile).
- Устанавливает redirection для выполняемой программы или для текущей оболочки.
- Если заданы только redirection, то redirection влияют на текущую оболочку без выполнения какой-либо программы.

Для нас это значит, что запущенное приложение будет иметь PID 1 и как следствие будет ловить все сигналы из вне, но так как управление находится у программы, то она и должна обрабатывать все сигналы, иначе они будут проигнорированы.  
Так же, т.к. это PID 1, т.е. Init process, то при его завершении будут убиты все дочернии процессы, поэтому с формой exec в наших руках влясть над всем контейнером 
Но с властью приходят и проблемы, осторожно, сейчас начинается уровень `Dark Internet` - [проблема PID 1 zombie reaping](https://habr.com/ru/company/hexlet/blog/248519/).
