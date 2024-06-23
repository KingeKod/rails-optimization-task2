# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она работала слишком долго, и не было понятно, закончит ли она вообще работу за какое-то разумное время.

Я решил исправить эту проблему, оптимизировав эту программу.

## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на быстродействие программы я придумал использовать такую метрику: программа должна потреблять < 70Мб памяти при обработке целевого файла data_large.tst в течение всего времени выполнения.

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста в фидбек-лупе позволяет не допустить изменения логики программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`, который позволил мне получать обратную связь по эффективности, сделанных измерений за время, которое бы не превышало 10 секунд.

Вот как я построил `feedback_loop`: провел профилирование данной программы на объеме данных, обработка которых не превышала бы 10 сек. Далее по отчету профилировщика смотрю точки роста, правлю их в коде, тестируем, что ничего не поломалось и повторяем процесс заново. Поскольку с каждой итерацией время выполнения на конкретном объеме может уменьшаться, на каждом этапе корректируется(увеличиается) объем данных при профилировании

Вначале взял файл с 30к строк, так как программа отрабатывала с этим объемом данных за 7.32 секунд.

## Вникаем в детали системы, чтобы найти главные точки роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался несколькими инструментами: memory_profiler,
и ruby-prof в режимах flat, graph и callstack.
valgrind, к сожалению, не удалось применить, так как на процессоре m1 возникли некоторые сложности при сборке образа(

Для начала, рерил попробовать проверить с помощью memory_profiler сколько аллоцируется в целом у нас памяти с 30к строк файла без переписывания на потоковый вход - memory_profiler показал 3.78 GB -  это очень много
Соответственно, как и было написано в задании, я перевел программу на потоковый подход, чтобы файл читался и записывался построчно

Вот какие проблемы удалось найти и решить

### Ваша находка №1
- Я сразу воспользовался отчетами нескольких профилировщиков.
- После увиденного в отчете, что программа потребляет огромное количество памяти при загрузке данных из файла, как и предложено было изначально в задании, переписал програму на построчный подход. Однако такой подход потребовал сильной переделки кода в программе, я попутно сразу переписал методы, где не производительно использовалась работа с массивами и циклами.
- Метрика изменилась максимально сильно! Сразу же после переписывания программы на потоковый и построчную реализацию результат стал невероятно быстрым и результаты были более чем удовлетворяющими) Для 30000 строк программа отработала за долисекунды, поэтому решил попробовать запустить программу для нужного нам файла data_large.txt, и результаты бенчмарка стали таковы:
    MEMORY USAGE: 35 MB
    Программа выполнилась за 8.99 секунд
Однако, как в пред. задании не получилось последовательно отслеживать на каждом шаге производительность программы при изменении конкретного метода, но удивительно, что с таким подходом изменился в корне результат.
- Отчёт профилировщика изменился так же сильно! Получилось так, что переписав программу, ничего дополнительно оптимизировать не пришлось, так как полученная метрика более чем устраивает, а остальная оптимизация сводится к тому, чтобы переделать методы IO: <Class::IO>#foreach, write_to_file; split, ну и немного inject.
По резульатам написал тест  memory_spec.rb, в котором происходит проверка на то, что программа отрабаывает не больше 10 сек, и на то, что программа не потребляет больше 45 Мб памяти при выполнении

### Результаты

Мне удалось обработать необходимый файл data_large.txt. Сколько изначально программа потребляла памяти - сказать затруднительно, но на 30000 строк потребяла 3.78 GB. С 3.78 GB потребление памяти удалось уменьшить до 30 Мб! При обработке всего файла занимает чуть больше, но не выходит за определенный нами в начале бюджет(< 70 Мб).
Конечный результат времени обработки файла data_large.txt при оптимизации по CPU получился - 24.783419 сек.
Конечный результат времени обработки файла data_large.txt при оптимизации по памяти получился - 8.99 секунд

Удивительно, но оптимизация по памяти дала гораздо больший прирост по времени, чем по CPU. Т.е в 2,8 раза быстрее!)