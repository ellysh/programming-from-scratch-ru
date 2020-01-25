## Условные операторы

Мы уже встречались с условными конструкциями, когда знакомились с утилитой `find` и логическими операторами. В языке Bash есть и другие формы ветвления. Каждая из них предназначена для определённой цели. Рассмотрим инструкции `if` и `case`, которые часто оказываются полезны при разработке скриптов.

### Оператор if

Мы руководствуемся разными требованиями, когда пишем однострочную команду и скрипт. В первом случае важна компактность. Согласитесь, что короткую команду проще набирать и шанс допустить ошибку меньше. В случае скриптов, важнее наглядность и удобство чтения.

Операторы `&&` и `||` хорошо подходят для однострочных команд. Но когда речь заходит о скриптах, у них появляются более удобные альтернативы. На самом деле всё зависит от конкретного случая. Иногда `&&` и `||` отлично вписываются в код скрипта, а иногда их следует заменить на более удобные конструкции. Рассмотрим эти случаи подробнее.

Ещё раз обратимся к нашему скрипту для резервного копирования из листинга 3-9. Его первая строка выглядит следующим образом:
{line-numbers: false, format: Bash}
```
bsdtar -cjf "$1".tar.bz2 "$@" && echo "bsdtar - OK" > results.txt || { echo "bsdtar - FAILS" > results.txt ; exit 1 ; }
```

Мы разбили вызовы утилит `bsdtar` и `cp` на две отдельные команды. Это помогло, но лишь отчасти. Вызов `bsdtar` всё ещё слишком длинный, из-за чего его неудобно читать и изменять. Это верный сигнал, что при его написании было принято неверное техническое решение.

Распишем алгоритм вызова `bsdtar` по шагам:

1. Выполнить архивирование и сжатие всех файлов и каталогов из переменной `$@`.

2. Если команда завершилась успешно, записать в лог-файл строку "bsdtar - OK".

3. Если при работе `bsdtar` произошла ошибка, записать в лог-файл строку "bsdtar - FAILS" и завершить работу скрипта.

Вопросы вызывает третий пункт. Обратите внимание, что при успешной завершении `bsdtar` выполняется только одно действие. В случае же ошибки - действий два и они объединены в [**блок**](https://www.gnu.org/software/bash/manual/html_node/Command-Grouping.html) с помощью фигурных скобок. 

Конструкция `if` была введена как раз для удобства работы с блоками команд. В общем случае она выглядит следующим образом:
{line-numbers: false}
```
if УСЛОВИЕ
then
    ДЕЙСТВИЕ
fi
```

Если требуется записать `if` в одну строку, после `if УСЛОВИЕ` и `then ДЕЙСТВИЕ` следует поставить точки с запятой:
{line-numbers: false}
```
if УСЛОВИЕ; then ДЕЙСТВИЕ; fi
```

УСЛОВИЕ и ДЕЙСТВИЕ в операторе `if` представляют собой команду или блок команд. Если УСЛОВИЕ завершилось успешно с кодом 0, будут выполнены команды, соответствующие ДЕЙСТВИЮ.

Рассмотрим простой пример:
{line-numbers: false, format: Bash}
```
if cmp file1.txt file2.txt &> /dev/null
then
    echo "Файлы file1.txt и file2.txt идентичны"
fi
```

В качестве УСЛОВИЯ мы вызываем утилиту `cmp`. Она побайтово сравнивает содержимое двух файлов, переданных на вход. Если они отличаются, `cmp` печатает в стандартный поток вывода позицию первого различающегося символа. В данном случае нас интересует только код возврата утилиты. Поэтому мы перенаправляем её вывод в файл [`/dev/null`](https://ru.wikipedia.org/wiki//dev/null). Запись в `dev/null` всегда происходит успешно, а все записанные данные удаляются.

Итак, если файлы `file1.txt` и `file2.txt` имеют одинаковое содержимое, команда `echo` выведет соответствующее сообщение на экран.

Если в случае невыполнения условия требуется выполнить какие-то действия, воспользуйтесь конструкцией `if-else`:
{line-numbers: false}
```
if УСЛОВИЕ
then
    ДЕЙСТВИЕ 1
else
    ДЕЙСТВИЕ 2
fi
```

Запись `if-else` в одну строку выглядит так:
{line-numbers: false}
```
if УСЛОВИЕ; then ДЕЙСТВИЕ 1; else ДЕЙСТВИЕ 2; fi
```

В этой конструкции блок команд, обозначенный как ДЕЙСТВИЕ 2, будет выполняться если УСЛОВИЕ вернёт код ошибки отличный от 0. В противном случае исполнится ДЕЙСТВИЕ 1.

Воспользуемся конструкцией `if-else`, чтобы добавить в наш пример сравнения файлов вывод сообщения об их различии:
{line-numbers: false, format: Bash}
```
if cmp file1.txt file2.txt &> /dev/null
then
    echo "Файлы file1.txt и file2.txt идентичны"
else
    echo "Файлы file1.txt и file2.txt различаются"
fi
```

Вернёмся к нашему скрипту резервного копирования. Как мы выяснили, если при выполнении кого-то условия следует выполнить блок команд, предпочтительнее использовать конструкцию `if`, а не операторы `&&` и `||`. В нашем случае блок команд включает вывод сообщения об ошибке в лог-файл и вызов `exit`.

Перепишем вызов `bsdtar` с использованием `if`:
{line-numbers: false, format: Bash}
```
if bsdtar -cjf "$1".tar.bz2 "$@"
then
    echo "bsdtar - OK" > results.txt
else
    echo "bsdtar - FAILS" > results.txt
    exit 1
fi
```

Согласитесь, что теперь читать код стало намного удобнее. На самом деле мы можем его упростить. Применим технику [**раннего возврата**](https://habr.com/ru/post/348074/) и заменим конструкцию `if-else` на `if`:
{line-numbers: false, format: Bash}
```
if ! bsdtar -cjf "$1".tar.bz2 "$@"
then
    echo "bsdtar - FAILS" > results.txt
    exit 1
fi

echo "bsdtar - OK" > results.txt
```

Что изменилось? Поведение нашего кода осталось без изменений: в случае ошибки будет выведено сообщение "bsdtar - FAILS" и вызовется `exit`, а в противном случае произойдёт вывод "bsdtar - OK". Идея раннего возврата заключается в том, чтобы в случае ошибки завершить программу как можно раньше. Благодаря этому, мы избегаем многократных вложений операторов `if`.

Рассмотрим пример. Представьте, что у нас есть алгоритм состоящий из пяти действий. Каждое последующее должно выполняться только при успешном завершении предыдущего. Мы можем реализовать этот алгоритм с помощью конструкции `if` таким образом:
{line-numbers: false}
```
if ДЕЙСТВИЕ 1
then
    if ДЕЙСТВИЕ 2
    then
        if ДЕЙСТВИЕ 3
        then
            if ДЕЙСТВИЕ 4
            then
                ДЕЙСТВИЕ 5
            fi
        fi
    fi
fi
```

Такая программа выглядит запутанной. Добавьте в неё выводы сообщений об ошибках во всех `else` случаях и читать её станет ещё сложнее. Ранний возврат решает именно эту проблему. Перепишем наш алгоритм с применением этой техники:
{line-numbers: false}
```
if ! ДЕЙСТВИЕ 1
then
    # обработка ошибки
fi

if ! ДЕЙСТВИЕ 2
then
    # обработка ошибки
fi

if ! ДЕЙСТВИЕ 3
then
    # обработка ошибки
fi

if ! ДЕЙСТВИЕ 4
then
    # обработка ошибки
fi

ДЕЙСТВИЕ 5
```

I> Строка скрипта начинающаяся с символа решётка `#` является [**комментарием**](https://ru.wikipedia.org/wiki/Комментарии_(программирование)). Это значит, что она будет проигнорированна интерпретатором.

Обратите внимание, что применение конструкции `if` в данном случае оправданно, только если обработка ошибки состоит из блока команд. Если же достаточно просто вызова `exit`, предпочтительнее использовать операторы `&&` и `||`:
{line-numbers: false}
```
ДЕЙСТВИЕ 1 || exit 1
ДЕЙСТВИЕ 2 || exit 1
ДЕЙСТВИЕ 3 || exit 1
ДЕЙСТВИЕ 4 || exit 1
ДЕЙСТВИЕ 5
```

Листинг 3-10 демонстрирует скрипт резервного копирования, переписанный с использованием конструкции `if`.

{caption: "Листинг 3-10. Скрипт с ранним возвратом", line-numbers: true, format: Bash}
![`make-backup-if.sh`](code/BashScripting/make-backup-if.sh)

### Оператор [[

Мы познакомились с оператором `if`. В качестве условия в нём вызывается какая-то команда Bash или сторонняя утилита. Мы уже знаем, как работать с файловой системой. Поэтому способ комбинации `if`, например, с командой `grep` достаточно очевиден.

При вызове утилиты `grep` в качестве условия конструкции `if` используйте параметр `-q`. Благодаря ему, `grep` не будет ничего выводить в stdout, а вместо этого вернёт код 0 при первом вхождении искомой строки или шаблона. Например:
{line-numbers: false, format: Bash}
```
if grep -q -R "General Public License" /usr/share/doc/bash
then
    echo "Bash распространяется под лицензией GPL"
fi
```

Что делать, если нам требуется проверять условия не связанные с файловой системой, а, например, сравнивать строки или числа? Для работы со строками в Bash есть специальный оператор `[[`. Обратите внимание, что это не внешняя утилита, а [**зарезервированное слово**](https://ru.wikipedia.org/wiki/Зарезервированное_слово) интерпретатора.

W> Оператора `[[` нет в Bourne shell. Если в вашем случае важна POSIX совместимость, вам придётся воспользоваться устаревшим оператором [`test`](http://mywiki.wooledge.org/BashFAQ/031) или его синонимом `[`. Никогда не используйте его в Bash! Возможности `test` ограничены, а правильные способы применения неочевидны.

Начнём с простого примера использования `[[`. Предположим, что нам надо сравнить две строки. В этом случае условие `if` будет выглядеть следующим образом:
{line-numbers: false, format: Bash}
```
if [[ abc = abc ]]
then
    echo "Строки равны"
fi
```

Выполнив этот код, вы увидите сообщение, что строки равны. Подобная проверка не слишком полезна. Намного чаще приходится сравнивать значение какой-нибудь переменной (например, `$var`) со строкой. В этом случае проверка будет выглядеть так:
{line-numbers: false, format: Bash}
```
if [[ $var = abc ]]
then
    echo "Переменная равна строке abc"
fi
```

Выражения допустимые в операторе `[[` приведены в таблице 3-1.

{caption: "Таблица 3-11. Выражения оператора `[[`", width: "100%"}
| Тип данных | Выражение | Описание | Пример |
| --- | --- | --- | --- |
|        | > | Строка больше в порядке ASCII кодов. | [[ b > a ]] && echo "ASCII код b больше, чем a" |
| Строки | < | Строка меньше в порядке ASCII кодов. | [[ ab < ac ]] && echo "ASCII код b меньше, чем c" |
|        | = или == | Строки равны. | [[ abc = abc ]] && echo "Строки равны" |
|        | != | Строки не равны. | [[ abc != ab ]] && echo "Строки не равны" |
|  | | | |

### Оператор ((