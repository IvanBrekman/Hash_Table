# HashTable
Хеш-таблица с закрытой адресацией для хранения строк и дальнейшего поиска по таблице.
<br><br><br>

## Введение
Для ознакомления со статьей читатель должен иметь базовые представления о хеш-таблицах, хеш-функциях, синтаксисе языка `С` и ассемблера `NASM`.

Целью данной работы является разработка словаря для хранения массива строк и дальнейшего поиска по словарю. В качестве словаря будет использоваться хеш-таблица со списками, реализованными под хранение строк.

После реализации структуры данных предстоит провести анализ и выбрать хеш-функцию, которая лучше всего подойдет для нашего usecase. Затем нашей задачей будет замерить время работы хеш-функции и попытаться максимально ее оптимизировать, стараясь при этом сохранить переносимость программы на другие устройства.

Приятного чтения, и да прибудет с вами Котик!

![image](https://sun1-17.userapi.com/impg/ParUUs2WiFcnxg14QOSIok7RerIBC1HosEMjUg/snp42wDVjSI.jpg?size=200x0&quality=88&crop=5,4,890,890&sign=1b61ea57e100650d16e7f9cc5d4b1fea&c_uniq_tag=o5bASAgP_RWciRPxU2cCCZCFXGnNvufio-_m3OZmUno&ava=1)
<br><br><br>

## Реализация таблицы
Хеш-таблица не самая простая структура данных, которая работает с объектами (например со списками) через указатели. Да и сами строки в сущности являются указателями, которые надо создавать, хранить, возможно, очищать, не говоря уже о том, что в процессе оптимизаций можно запросто что-то сломать или испортить и потратить часы на дебаг очередного `Segmentation Fault`.

Предостерегая эти ошибки, сделаем жертву временем Богу Дебага, написав защищенные таблицу и список. Это означает, что при реализации будут добавлены поля и функции, которые отвечают за проверку корректного состояния структур данных и вывода подробной информации обо всех полях объекта в случае ошибки или при желании.

Из этих соображений имеем следующую структуру таблицы.
```c++
typedef unsigned long long  ull;
typedef validate_level      validate;

const long unsigned INIT_CANARY  = 0x5AFEA2EA; // SAFE AREA

struct HashInfo {
    const char* type = nullptr;
    const char* name = nullptr;
    const char* file = nullptr;
    const char* func = nullptr;
          int   line = 0;
};

struct HashTable {
    ull        _lcanary = INIT_CANARY;
    validate   _vlevel  = validate::NO_VALIDATE;
    HashInfo*  _info    = (HashInfo*) poisons::UNINITIALIZED_PTR;

    ull (*_hash) (char* string) = (ull (*) (char*)) poisons::UNINITIALIZED_PTR;

    List*   data        = (List*)poisons::UNINITIALIZED_PTR;
    int     size        = poisons::UNINITIALIZED_INT;
    int     capacity    = poisons::UNINITIALIZED_INT;

    ull        _rcanary = INIT_CANARY;
};
```

Для удобства определяем несколько `typedef`, а все поля проинициализируем специальными ядовитыми значениями, чтобы в случае ошибок можно было увидеть, менялись ли значения этих полей с начальных.

Канареек поставим от случайного выхода за границы массива и порчи данных структуры (вовсе не для него).

<p align="center" width="100%">
  <img src="https://cartoonresearch.com/wp-content/uploads/2018/12/putty-tat-tweety-344.jpg"> 
</p>
<br>

Также добавим в структуру уровень валидации для управления проверками при работе таблицы, поле со структурой информации о таблице (откуда она, как называется и пр.) и поле для хеш-функции, с помощью которой таблица будет работать (чтобы не нагружать функции доп аргументом в виде хеш-функции).

Из функций наша таблица будет иметь:
1. Конструктор и деструктор.
```c++
HashTable* table_ctor(HashInfo*   info                      = nullptr,
                      ull       (*hash_func) (char* string) = default_hash,
                      validate    level                     = VALIDATE_LEVEL,
                      int         capacity                  = CAPACITY_VALUES[0]
                     );
HashTable* table_dtor(HashTable* table);
```

2. Функции для проверок (валидации) таблицы, а также вывод информации о таблице (дамп).
```c++
hashtable_errors table_error     (const HashTable*       table);
const char*      table_error_desc(const hashtable_errors error_code);

int table_dump(const HashTable* table, const char* reason, FILE* log=stdout);
```

3. Необходимые для работы функции вставки, поиска, а также для увлечения размера, чтобы хеш-функции лучше работали.
```c++
int table_add (      HashTable* table, char* string);
int table_find(const HashTable* table, char* string);

int table_rehash(    HashTable* table, int new_capacity=OLD_CAPACITY);
int get_next_capacity(int capacity);
```
<br><br><br>

## Выбор хеш-функции
### Введение
Основа хеш-таблицы - хеш-функция. Хеш-функции в идеале должны для одинаковых объектов выдавать одинаковый ключ, а для разных - разный, однако на практике случаются коллизии (совпадение хешей), поэтому наша задача выбрать хеш-функцию, которая будет иметь меньше всего коллизий.

Для анализа этого введем коэффициент коллизии `collision coef = elems / lists`, где `elems` - кол-во всех элементов в таблице, `lists` - кол-во списков, в которых есть хотя бы 1 элемент. Чем меньше этот коэффициент - тем лучше.

Выбирать будем из 6 функций:

| Название | Описание |
|:----------------:|:---------:|
| constant_hash | возвращает константу |
| first_letter_hash | возвращает ascii-код первого символа слова |
| symbol_sum_hash | возвращает сумму ascii-кодов символов слова |
| string_len_hash | возвращает длину слова |
| roll_hash | полиномиальный хеш |
| crc32_hash | реализация crc32 |

<br>

### Реализация
Будем использовать функцию `test_func_collision`, которая будет создавать таблицу с указанной функцией, заполнять ее данными из указанного файла и считать коэффициент коллизий, выводя результат тестирования в консоль. Также эта функция будет вносить информацию о загруженности таблицы в специальный файл, по которому позже будут строиться графики.

Для замера возьмем текст Шекспира "Ромео и Джульетта". Предварительно с помощью скрипта очистим файл от всех знаков препинания, приведем все слова к нижнему регистру и удалим все повторы, так как наша задача сейчас - оценить качество распределения слов хеш-функцией.

```c++
int test_func_collision(const char* filename, const HashFunc* hash, const char* save_file, int save_from, int save_to) {
    ASSERT_IF(VALID_PTR(filename), "Invalid filename ptr", 0);
    ASSERT_IF(VALID_PTR(hash),     "Invalid hash ptr",     0);

    HashTable*   table   = CREATE_TABLE(table, hash->func, validate::WEAK_VALIDATE, CAPACITY_VALUES[1]);
    LoadContext* context = load_strings_to_table(table, filename, 1);

    CollisionData* data  = get_collision_info(table);
    
    FILE* dest = open_file(save_file, "a");

    save_to = (save_to == OLD_CAPACITY) ? table->capacity : save_to;

    fprintf(dest, "%s;", hash->func_name);
    for (int i = save_from; i < save_to; i++) {
        List* lst = table->data + i;

        fprintf(dest, "%d", lst->size);
        if (i + 1 < save_to) fprintf(dest, " ");
    }
    fprintf(dest, ";%.3lf\n", data->coef);

    printf("=============== Collision test ===============\n");
    printf("func name:     '%s'\n",   hash->func_name);
    printf("collision coef: %lf\n\n", data->coef);
    printf("finds:          %d\n",  context->finds);
    printf("inserts:        %d\n",  context->inserts);
    printf("fi coef:        %lf\n", (double)context->finds / context->inserts);
    printf("==============================================\n\n");

    FREE_PTR(data->data,       int);
    FREE_PTR(context->storage, char);

    FREE_PTR(data,    CollisionData);
    FREE_PTR(context, LoadContext);

    table_dtor(table);
    close_file(dest);

    return 1;
}
```
<br>

### Результаты
При запуске программы получаем результаты:
```
=============== Collision test ===============
func name:     'Constant hash'
collision coef: 3743.000000

finds:          0
inserts:        3743
fi coef:        0.000000
==============================================

=============== Collision test ===============
func name:     'First symbol hash'
collision coef: 149.720000

finds:          0
inserts:        3743
fi coef:        0.000000
==============================================

=============== Collision test ===============
func name:     'Symbols sum hash'
collision coef: 5.191401

finds:          0
inserts:        3743
fi coef:        0.000000
==============================================

=============== Collision test ===============
func name:     'String len hash'
collision coef: 233.937500

finds:          0
inserts:        3743
fi coef:        0.000000
==============================================

=============== Collision test ===============
func name:     'Roll hash'
collision coef: 1.895190

finds:          0
inserts:        3743
fi coef:        0.000000
==============================================

=============== Collision test ===============
func name:     'Crc32 hash'
collision coef: 1.417266

finds:          0
inserts:        3743
fi coef:        0.000000
==============================================
```

Также имеем визуализацию этих данных в графиках и таблице:

На графиках числа слева означают кол-во элементов в списке, а значения снизу - индекс списка в таблице.
Также последний график отображает значения коэффициентов коллизий для каждой функции (число снизу - номер графика соответствующей функции).
![image](https://github.com/IvanBrekman/Hash_Table/blob/main/data/images/graphs_no_asm.png)

| Название | Описание | Коэффициент коллизии |
|:----------------:|:---------:|:----------:|
| constant_hash | возвращает константу | 3743.000000 |
| first_letter_hash | возвращает ascii-код первого символа слова | 149.720000 |
| symbol_sum_hash | возвращает сумму ascii-кодов символов слова | 5.191401 |
| string_len_hash | возвращает длину слова | 233.937500 |
| roll_hash | полиномиальный хеш | 1.895190 |
| crc32_hash | реализация crc32 | 1.417266 |

<br>

На основе этих данных можем сделать вывод, что `crc32_hash` лучше всех нам подходит. Для дальнейших оптимизаций будем использовать таблицу с этой хеш-функцией.
<br><br><br>

## Оптимизации
### Введение
Наша задача - оценить время работы хеш-таблицы и попытаться ее ускорить. Для этого нужно выбрать, как и что измерять.

Usecase подразумевает загрузку большого текста с файла и, возможно, дальнейший поиск слов в таблице. Поэтому будем замерять время загрузки файла в таблицу, при этом наша функция будет проверять есть ли слово в таблице, и если нет, то вставлять его туда. Отношение кол-ва поисков в таблице (`find`) к кол-ву вставок (`insert`) назовем `fi_coef`.

В usecase мы подразумеваем использование таблицы для поиска слов, так что кол-во поисков должно быть в разы больше кол-ва вставок. Если после загрузки значение этого коэффициента понадобится увеличить, то будем генерировать рандомные слова и искать их в таблице, при этом замеряя время только поиска в таблице.

Для профилирования программы будем использовать `KCacheGrind`, который будет показывать, как много времени от работы программы заняла какая-то функция.
<br>

### Реализация
Выберем уже знакомый нам текст Шекспира. Также с помощью скрипта уберем все знаки препинания и приведем весь текст к нижнему регистру.

Для замеров времени будем использовать функцию `test_table_speed`, которая будет брать данные из указанного файла. Также ей можно передать нужное нам значение `fi_coef`. От запуска к запуску время работы одного и того же кода может меняться, поэтому будем повторять тест `repeats` раз и считать среднее время работы.

```c++
int test_table_speed(const char* filename, int repeats, double fi_coef) {
    ASSERT_IF(VALID_PTR(filename), "Invalid filename ptr", 0);

    clock_t sum_time = 0;

    LoadContext* context = nullptr;

    for (int i = 0; i < repeats; i++) {
        HashTable* table = CREATE_TABLE(table, crc32_hash_asm, validate::MEDIUM_VALIDATE, CAPACITY_VALUES[1]);

        clock_t start_time = clock();
                context    = load_strings_to_table(table, filename, 0);
        clock_t end_time   = clock();

        sum_time += end_time - start_time;

        if ((context->inserts * fi_coef) > context->finds) {
            int need_inserts = (int) (context->inserts * fi_coef) - context->finds;

            char* word = (char*) calloc(MAX_RANDOM_WORD_LEN, sizeof(char));
            for ( ; need_inserts > 0; need_inserts--) {
                word = random_word(word, rand() % MAX_RANDOM_WORD_LEN + 1);

                start_time = clock();
                table_find(table, word);
                end_time   = clock();

                context->finds++;
                sum_time += end_time - start_time;
            }
            FREE_PTR(word, char);
        }

        table_dtor(table);

        FREE_PTR(context->storage, char);
        if (i + 1 < repeats) FREE_PTR(context, LoadContext);
    }
    
    printf("=============== Speed test ===============\n");
    printf("repeats:  %d\n\n", repeats);
    printf("time avg: %lf sec\n\n", ((double)sum_time / repeats) / CLOCKS_PER_SEC);
    printf("finds:    %d\n",  context->finds);
    printf("inserts:  %d\n",  context->inserts);
    printf("fi coef:  %lf\n", (double)context->finds / context->inserts);
    printf("==========================================\n\n");

    FREE_PTR(context, LoadContext);

    return 1;
}
```
<br>

С самого начала имеем результат:
```
=============== Speed test ===============
repeats:  20

time avg: 1.302874 sec

finds:    37430
inserts:  3743
fi coef:  10.000000
==========================================
```

Приступим к оптимизациям!

![hippo](https://github.com/IvanBrekman/Hash_Table/blob/main/data/images/cat_proger_gif.gif)
<br>

### Оптимизация 1
Посмотрим вывод KCacheGrind.
![image](https://github.com/IvanBrekman/Hash_Table/blob/main/data/images/opt1_checks.png)

Первое, что сразу бросилось в глаза - функции `isbadreadptr` и `table_error`, которые являются проверочными, и в релизе они не нужны. `table_error` отключить очень просто - достаточно передать в таблицу уровень валидации `validate::NO_VALIDATE`, при котором в таблице отключаются проверки.

С `isbadreadptr` сложнее. Эта функция используется для проверок указателей в начале каждой функции с помощью конструкции `ASSERT_IF`.

```c++
#define ASSERT_IF(cond, text, ret) do {                                             \
    assert((cond) && text);                                                         \
    if (!(cond)) {                                                                  \
        PRINT_WARNING(text "\n");                                                   \
        errno = -1;                                                                 \
        return ret;                                                                 \
    }                                                                               \
} while(0)
```

Как видно, просто отключить `assert` не достаточно, поэтому добавим define `NO_CHECKS` при котором будем отключать весь `ASSERT_IF`. Таким образом получим аналог `#define NDEBUG`.

Проделав небольшие изменения в коде смотрим на результаты.
```
=============== Speed test ===============
repeats:  100

time avg: 0.004815 sec

finds:    37430
inserts:  3743
fi coef:  10.000000
==========================================
```

Ускорение в 270 раз? Звучит на нобелевку, но это всего лишь отключение всех проверок. Идем дальше.
<br><br>

### Оптимизация 2
Анализируем новый вывод KCacheGrind.
![image](https://github.com/IvanBrekman/Hash_Table/blob/main/data/images/opt2_split.png)

Сверху списка замечаем вызовы `random`, которые нас не интересуют, так как они используются для генерации рандомных слов для дополнительных поисков в таблице и на ее скорость никак не влияют. После них видим функцию `calloc_s`, которая занимается выделением памяти под указатели.

Внимательно изучим вкладку `Callers`. В ней мы видим, что эту функцию вызывают `test_table_speed` и `load_strings_to_table`, которые выполняют всю работу и в оптимизации не нуждаются, `table_ctor` и `list_ctor`, которые инициализируют объекты этих структур, так что уменьшить кол-во `calloc` в них не получится. Наконец функции `get_text_from_file` и `split`, занимающиеся непосредственно загрузкой текста с файла и разбиением его на слова.

Разберемся немного в их работе. `get_text_from_file` загрузочная функция, которая сгружает в буфер весь текст с файла, после чего проходится по буферу, раcставляя указатели на начала строк, считая параллельно длину этих строк и их кол-во, а также заменяет символы `\n` на `\0`, получая таким образом структуру `Text`. Далее функция `split` проходится по каждой строке и, заменяя пробел на `\0`, сгружает указатели на начала слов в массив, который позже возвращает.

<p align="center" width="100%">
  <img src="http://risovach.ru/upload/2013/09/mem/nu-davay-rasskazhi-mne_30454451_orig_.jpeg"> 
</p>
<br>

Несложно заметить, что функции выполняют похожую работу, при этом вся информативность структуры `Text` нам даже не нужна. Поэтому можно переделать логику `load_strings_to_table`, загружая "сырой" текст в буфер и проходя по нему, расставлять указатели на начала слов и менять пробельные символы на `\0`.

```c++
LoadContext* load_strings_to_table(HashTable* table, const char* filename, int force_insert) {
    ASSERT_OK_HASHTABLE(table,     "Check before load strings_to_table func", nullptr);
    ASSERT_IF(VALID_PTR(filename), "Invalid file ptr",                        nullptr);

    char* data = get_raw_text(filename);

    LoadContext* ctx = NEW_PTR(LoadContext, 1);
    *ctx = { 0, 0, data };

    for (int i = 0; data[i] != '\0'; i++) {
        if (isspace(data[i])) {
            data[i] = '\0';
        } else {
            char* word = data + i;

            while (!isspace(data[i]) && data[i] != '\0') i++;
            data[i] = '\0';

            if (force_insert || table_find(table, word) == NOT_FOUND) {
                table_add(table, word);
                ctx->inserts++;
            }

            ctx->finds += (!force_insert);
        }
    }

    ASSERT_OK_HASHTABLE(table, "Check load to table", nullptr);

    return ctx;
}
```
<br>

Таким образом, последовав советам мудреца ниже, мы упростили логику программы и показали свою приверженность принципу `KISS`.

<p align="center" width="100%">
  <img src="https://gettingclose.ru/wp-content/uploads/2020/08/uslozhnat.jpg"> 
</p>
<br>

Упростить, упростили, а ускорили ли? Смотрим.
```
=============== Speed test ===============
repeats:  100

time avg: 0.003480 sec

finds:    37430
inserts:  3743
fi coef:  10.000000
==========================================
```

Оптимизация сделала таблицу быстрее в 1.4 раза. Отличный результат на таком времени работы! Продолжаем.
<br><br>

### Оптимизация 3
Смотрим в наш любимый вывод KCacheGrind.
![image](https://github.com/IvanBrekman/Hash_Table/blob/main/data/images/opt3_hash.png)

Как видим, все `calloc` отлетели вниз, и теперь в лидерах функция `table_find`, что, вполне, ожидаемо. Изучим узкие места функции во вкладке `Source Code`. Как видно, 2 самых тяжелых места - работа хеш-функции и поиск в списке.

Начнем с оптимизации `crc32_hash`. Возможностей у процессора куда больше, чем может казаться. Так, сейчас мы считаем хеш всех слов по одной букве, хотя могли бы сразу по 8, используя всю мощь регистров. Единственное препятствие - как понять, когда остановиться, ведь слова могут быть разной длины? Раньше признаком окончания был нулевой символ, символизирующий окончание строки, но что делать, если мы считаем сразу 8 символов? Проверять каждый раз среди этих 8 символов `\0` очень не хочется. Здесь мы воспользуемся особенностью наших целевых данный. Это слова, а много ли слов больше 32 букв вы знаете? Я нет. Пользуясь этим знанием, мы можем договориться всегда считать 4 раза по 8 символов, ускорив таким образом подсчет хеша. Однако это требует от нас дополнительных расходов в виде выравнивания всех слов по 32 байта, ведь если в этих ячейках окажутся данные кроме подсчитываемого слова, то у одинаковых слов, лежащих в разных местах будет разный хеш. Такое недопустимо!

Для реализации нашей затеи будем использовать ассемблерную вставку. Она сильно ускорит время работы функции, но понизит переносимость программы, так как на разных машинах может использоваться разный ассемблер.

Таким образом имеем новую реализацию хеш-функции, использующую процессорную команду `crc32`.

```c++
ull crc32_hash_asm(char* string) {
    ull hash = 0;

    __asm__(
        ".intel_syntax noprefix     \n\t"

        "mov rcx, 4                 \n\t"
        "xor %[ret_val], %[ret_val] \n\t"

        "calc_hash:                 \n\t"
            "mov rax, [%[arg_val]]  \n\t"

            "crc32 %[ret_val], rax  \n\t"
            "add %[arg_val], 8      \n\t"
            "loop calc_hash         \n\t"

        ".att_syntax prefix         \n\t"

        : [ret_val]"=r"(hash)
        : [arg_val]"r"(string)
        : "%rax", "%rcx"
    );

    return hash;
}
```

Есть ли от этого польза?
```
=============== Speed test ===============
repeats:  100

time avg: 0.003018 sec

finds:    37430
inserts:  3743
fi coef:  10.000000
==========================================
```

Учитывая накладные расходы мы ускорили программу в 1.15 раз... используя ассемблер! И что вы на это скажете, хейтеры ассемблера?

![hippo](https://journals.ru/smile/users/13/125/12478.gif)
<br><br>

### Оптимизация 4
Вы знаете, с чего мы начинаем.
![image](https://github.com/IvanBrekman/Hash_Table/blob/main/data/images/opt4_find.png)

Как видно нагрузка хеш-функции сильно упала, так что теперь время заняться функцией `list_find`. Начнем с просмотра листинга этой функции.
![image](https://github.com/IvanBrekman/Hash_Table/blob/main/data/images/opt4_strcmp.png)

Становится понятно, что основное время работы функции уходит на `strcmp`. Займемся ее улучшением. Здесь у нас будет схожая логика рассуждений. Мы уже имеем данные, которые точно укладываются в 32 байта. В этот раз, для разнообразия, будем использовать не ассемблерную вставку, а реализацию через `intrinsic` функции. Пользуясь 256-разрядными регистрами, которые содержат `256/8=32` байта мы сможем за 1 команду сравнить сразу 2 полных слова!

Это ли не магия? На самом деле, эти 2 подхода очень похожи, потому что `intrinsic` функция при компиляции превращается в соответствующую команду ассемблера, так что на выходе мы будем иметь почти что ассемблерную вставку (разве что будет чуть больше команд `mov`), но такой код будет более переносимым, чем прямая ассемблерная вставка.

Вашему вниманию новая функция для сравнения строк.

```c++
int strcmp_avx(char* string1, char* string2) {
    __m256i str1 = _mm256_lddqu_si256((const __m256i*) string1);
    __m256i str2 = _mm256_lddqu_si256((const __m256i*) string2);

    __m256i res  = _mm256_cmpeq_epi8(str1, str2);

    int cmp_res  = _mm256_movemask_epi8(res);

    return cmp_res != -1;
}
```

Достаточно простой и, как мне кажется, понятный код (в сравнении с соответствующей ассемблерной вставкой уж точно). Работает все крайне просто. Загружаем строки в `ymm` регистры, сравниваем их значения, после чего сгружаем результат сравнения в переменную типа `int`.

Оценим пользу того, что мы сделали.
```
=============== Speed test ===============
repeats:  100

time avg: 0.002858 sec

finds:    37430
inserts:  3743
fi coef:  10.000000
==========================================
```

Данная оптимизация ускорила программу всего в 1.05 раза. Почему так?

Во-первых, мы оптимизировали самые нагруженные функции, так что логично, что каждый раз польза от оптимизаций будет хуже.

Во-вторых, функция `strcmp` имеет достаточно маленькую нагрузку относительно всей программы, так что и ускорение не очень сильно влияет.

В-третьих, если еще раз внимательно обратиться к листингу программы:
![image](https://github.com/IvanBrekman/Hash_Table/blob/main/data/images/opt4_strcmp.png)

То можно увидеть, что `strcmp` распозналась листингом по адресу `0x00000000004012c0`. Чуть ниже этой функции в списке видна функция `__strcmp_avx2`, что говорит нам о том, что базовая реализация `strcmp` и так ускорена разработчиками языка.

### Итоги (?)
Начинаем как всегда.
![image](https://github.com/IvanBrekman/Hash_Table/blob/main/data/images/opt_res1.png)

После оптимизаций `table_find`, `calloc` снова вылез вверх по нагрузке. Из его листинга мы помним, что он использовался в конструкторах структур. Конструктор таблицы мы увидели в последнем листинге, теперь посмотрим на конструктор листа.
![image](https://github.com/IvanBrekman/Hash_Table/blob/main/data/images/opt_res2.png)

В обоих конструкторах основное время работы уходит на вызовы `calloc`, но от них никуда не деться, да и оптимизировать нет смысла, так как это все работает один раз при создании таблицы (не забывайте, что для усреднения времени таблица создавалась и заполнялась 100 раз). Для наглядности запустим тестирование времени работы с повтором 1 раз, но с `fi_coef=100`, моделируя поиск по созданному словарю, и посмотрим на листинг.
![image](https://github.com/IvanBrekman/Hash_Table/blob/main/data/images/opt_res3.png)

Как видно, теперь в самом верху функции `table_find` и `crc32_hash_asm` (что очевидно), а функции загрузки, конструкторов и `calloc` почти не имеют нагрузки.

Также можем видеть сверху функции `clock`, отвечающие за измерения времени. Их, естественно, оптимизировать не надо :)

Дальнейшие оптимизации уже не имеют смысла, так как дадут слишком мало, при том что потребуют уже немалых усилий, которые, скорее всего, ухудшат переносимость кода.

Поэтому фиксируем результат.
```
=============== Speed test ===============                               =============== Speed test ===============
repeats:  20                                                             repeats:  100
                                                           __   
time avg: 1.302874 sec                                        \          time avg: 0.002858 sec
                                                    ----------|          
finds:    37430                                            __ /          finds:    37430
inserts:  3743                                                           inserts:  3743
fi coef:  10.000000                                                      fi coef:  10.000000
==========================================                               ==========================================
```

Итоговое ускорение в 456 раз! Но давайте сделаем более объективную оценку, так как отключение всех проверок не очень хочется считать за полноценную оптимизацию.

```
=============== Speed test ===============                               =============== Speed test ===============
repeats:  100                                                            repeats:  100
                                                           __   
time avg: 0.004815 sec                                        \          time avg: 0.002858 sec
                                                    ----------|          
finds:    37430                                            __ /          finds:    37430
inserts:  3743                                                           inserts:  3743
fi coef:  10.000000                                                      fi coef:  10.000000
==========================================                               ==========================================
```

Всеми махинациями мы ускорили нашу таблицу в 1.68 раза. Отличный результат! Можно с чистой совестью отдохнуть и посмотреть мемы про котиков, после чего вернуться к любимому программированию!
<br><br>

<p align="center" width="100%">
  <img src="https://postila.ru/resize?w=480&src=%2Fdata%2F54%2F5a%2Ffe%2F65%2F545afe6582cf59575affc16ed610bc57fc7f7ddc938cbcd4ce5b973ffadbba11.gif"> 
</p>
