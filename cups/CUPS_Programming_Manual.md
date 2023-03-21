---

title: CUPS Programming Manual
author: Michael R Sweet
copyright: Copyright © 2007-2022 by Apple Inc. All Rights Reserved.
version: 2.3.6

Оригинал документа: https://github.com/apple/cups/blob/master/cups/cupspm.md

The document was translated for personal use [@mikelpsv](https://github.com/mikelpsv)

...

> Please [file issues on Github](https://github.com/apple/cups/issues) to
> provide feedback on this document.

# Вступление

CUPS предоставляет библиотеку "cups"  для взаимодействия с различными частями CUPS и с Internet Printing Protocol (IPP) принтерами. Доступ к функциям библиотеки осуществляется путем включения заголовочного файла `<cups/cups.h`

CUPS основан на протоколе интернет-печати (IPP), который позволяет клиентам
(приложениям) для связи с сервером (планировщик, принтеры и т. д.) получить список адресатов, отправить задания на печать и т. д. Вы определяете сервер, с которым вы хотите общаться, используя указатель на непрозрачную структуру `http_t`. Константа `CUPS_HTTP_DEFAULT` может использоваться, когда вы хотите взаимодействовать с  планировщиком CUPS.

## Указания

При написании программного обеспечения (кроме драйверов принтера), использующего библиотеку «cups»:

- Не используйте недокументированные или устаревшие API,
- Не полагайтесь на предварительно настроенные принтеры,
- Не думайте, что принтеры поддерживают определенные функции или форматы, и
- Не полагайтесь на детали реализации (PPD и т. д.)

CUPS предназначен для того, чтобы оградить пользователей и разработчиков от реализации сведения о принтерах и форматах файлов. Цель состоит в том, чтобы позволить приложению предоставить файл печати в стандартном формате с намерением пользователя ("распечатать четыре копии, двусторонние на носителе формата A4, и каждую копию скрепите скобами"), а система печати управляет связью с принтером и необходимым преобразованием формата.

Точно так же приложения, управляя принтерами и заданиями, могут использовать стандартные запросы операций для получения информации о состоянии в общей форме и использовать стандартные операции управления для контроля состояния этих принтеров и заданий.

> **Примечание:**
> 
> Драйверы принтеров CUPS обязательно зависят от определенных форматов файлов и определенных детали реализации программного обеспечения CUPS. Пожалуйста, ознакомьтесь с постскриптумом и документацию разработчика драйвера растрового принтера на [CUPS.org](https://www.cups.org/documentation.html) для получения дополнительной информации.

## Термины, используемые в этом документе

*Назначение/Destination* — это принтер или очередь печати, которая принимает задания на печать. 
*Задание на печать/Print Job* — это набор из одного или нескольких документов, которые обрабатываются назначения, используя параметры, заданные при создании задания. 
*Документ/Document* — это файл (изображение JPEG, файл PDF и т. д.), пригодный для печати.
*Опция/Option* управление каким-либо аспектом печати, например, используемый носитель.
*Носитель/Media* — листы или рулоны на котором напечатано. 
*Атрибут/Attribite* — параметр, закодированный для запроса протокола интернет печати (IPP).

## Компиляция программ, использующих CUPS API

Библиотеки CUPS можно использовать в любой программе C, C++ или Objective C.
Метод компиляции с использованием библиотек зависит от операционной системы и установки CUPS. В следующих разделах показана компиляции примера в двух распространенных средах.

Следующая простая программа перечисляет доступные назначения (принтеры):

```c
#include <stdio.h>
#include <cups/cups.h>

int print_dest(void *user_data, unsigned flags, cups_dest_t *dest){
  if (dest->instance)
    printf("%s/%s\n", dest->name, dest->instance);
  else
    puts(dest->name);

  return (1);
}

int main(void){
  cupsEnumDests(CUPS_DEST_FLAGS_NONE, 1000, NULL, 0, 0, print_dest, NULL);
  return (0);
}
```

### Компиляция с помощью Xcode

In Xcode, choose *New Project...* from the *File* menu (or press SHIFT+CMD+N),
then select the *Command Line Tool* under the macOS Application project type.
Click *Next* and enter a name for the project, for example "firstcups".  Click
*Next* and choose a project directory. The click *Next* to create the project.

In the project window, click on the *Build Phases* group and expand the
*Link Binary with Libraries* section. Click *+*, type "libcups" to show the
library, and then double-click on `libcups.tbd`.

Finally, click on the `main.c` file in the sidebar and copy the example program
to the file.  Build and run (CMD+R) to see the list of destinations.

### Компиляция с помощью GCC

Cоздайте файл с именем `simple.c`, используя ваш любимый редактор, скопируйте пример в этот файл и сохраните. Затем выполните следующую команду чтобы скомпилировать его с помощью GCC и запустить

From the command-line, create a file called `simple.c` using your favorite
editor, copy the example to this file, and save.  Then run the following command
to compile it with GCC and run it:

```bash
gcc -o simple `cups-config --cflags` simple.c `cups-config --libs`
./simple
```

Команда cups-config предоставляет флаги компилятора (cups-config --cflags).
и библиотеки (`cups-config --libs`), необходимые для локальной системы.

# Работа с назначениями

Назначения, которые в CUPS представляют отдельные принтеры или классы
(коллекции или пулы) принтеров, представлены `cups_dest_t` структурой, которая включает имя \(`name`), экземпляр \(`instance`, сохраненные параметры/настройки), и является ли назначение по умолчанию для пользователя (`is_default`), а также параметры и основную информацию, связанную с этим назначением \(`num_options` и `options`).

Исторически назначения обслуживались вручную администратором системы или сети, но CUPS также поддерживает динамическое обнаружение назначений в текущей сети.

## Поиск доступных назначений

Функция `cupsEnumDests`  находит все доступные назначения:

```c
 int cupsEnumDests(unsigned flags, int msec, int *cancel,
               cups_ptype_t type, cups_ptype_t mask,
               cups_dest_cb_t cb, void *user_data)
```

Параметр `flags` указывает параметры перечисления, которые в настоящее время должны быть`CUPS_DEST_FLAGS_NONE`.

Параметр `msec` указывает максимальное количество времени, которое должно использоватся для поиска назначений в миллисекундах — интерактивные приложения должны сохранять это значение равным 5000 или меньше при запуске в основном потоке

Параметр `cancel` указатель на целочисленную переменную, которая при установке в ненулевое значение значение, приведет к остановке поиска назначений как можно скорее. Может быть `NULL`, если не требуется.

Параметры `type` и `mask` представляют собой битовые поля, которые позволяют вызывающей стороне фильтровать возвращаемые значения

- `CUPS_PRINTER_CLASS`: A collection of destinations.
- `CUPS_PRINTER_FAX`: A facsimile device.
- `CUPS_PRINTER_LOCAL`: A local printer or class.  This constant has the value 0
  (no bits set) and is only used for the `type` argument and is paired with the
  `CUPS_PRINTER_REMOTE` or `CUPS_PRINTER_DISCOVERED` constant passed in the
  `mask` argument.
- `CUPS_PRINTER_REMOTE`: A remote (shared) printer or class.
- `CUPS_PRINTER_DISCOVERED`: An available network printer or class.
- `CUPS_PRINTER_BW`: Can do B&W printing.
- `CUPS_PRINTER_COLOR`: Can do color printing.
- `CUPS_PRINTER_DUPLEX`: Can do two-sided printing.
- `CUPS_PRINTER_STAPLE`: Can staple output.
- `CUPS_PRINTER_COLLATE`: Can quickly collate copies.
- `CUPS_PRINTER_PUNCH`: Can punch output.
- `CUPS_PRINTER_COVER`: Can cover output.
- `CUPS_PRINTER_BIND`: Can bind output.
- `CUPS_PRINTER_SORT`: Can sort output (mailboxes, etc.)
- `CUPS_PRINTER_SMALL`: Can print on Letter/Legal/A4-size media.
- `CUPS_PRINTER_MEDIUM`: Can print on Tabloid/B/C/A3/A2-size media.
- `CUPS_PRINTER_LARGE`: Can print on D/E/A1/A0-size media.
- `CUPS_PRINTER_VARIABLE`: Can print on rolls and custom-size media.

Параметр `cb` задает функцию обратного вызова для каждого найденного назначения:

```c
typedef int (*cups_dest_cb_t)(void *user_data,
                              unsigned flags,
                              cups_dest_t *dest);
```

Функция обратного вызова получает копию параметра `user_data` вместе с
битовым полем \(`flags`) и назначением, которое было найдено. Параметр `flags`
может иметь любое из следующих постоянных (битовых) значений:

- `CUPS_DEST_FLAGS_MORE`: There are more destinations coming.
- `CUPS_DEST_FLAGS_REMOVED`: место назначения исчезло и должно быть удалено.
   из списка назначений, которые пользователь может выбрать.
- `CUPS_DEST_FLAGS_ERROR`: Произошла ошибка. Причину ошибки можно определить, вызвав функции `cupsLastError` и/или `cupsLastErrorString`.

Функция обратного вызова должна вернуть 0 для остановки поиска или 1 для продолжения.

> Функция обратного вызова, скорее всего, будет вызываться несколько раз для
> одного и того же назначения, поэтому вызывающая сторона должна подавить любой дубликат назначения.

Следующий пример показывает, как использовать `cupsEnumDests` для получения отфильтрованного массива назначений:

```c
typedef struct{
  int num_dests;
  cups_dest_t *dests;
} my_user_data_t;

int my_dest_cb(my_user_data_t *user_data, unsigned flags,
           cups_dest_t *dest){
  if (flags & CUPS_DEST_FLAGS_REMOVED){
   /*
    * Удаляем назначение из массива...
    */
    user_data->num_dests =
        cupsRemoveDest(dest->name, dest->instance,
                       user_data->num_dests,
                       &(user_data->dests));
  }else{
   /*
    * Добавляем назначение в массив...
    */
    user_data->num_dests =
        cupsCopyDest(dest, user_data->num_dests,
                     &(user_data->dests));
  }
  return (1);
}

int my_get_dests(cups_ptype_t type, cups_ptype_t mask,
             cups_dest_t **dests){
  my_user_data_t user_data = { 0, NULL };

  if (!cupsEnumDests(CUPS_DEST_FLAGS_NONE, 1000, NULL, type,
                     mask, (cups_dest_cb_t)my_dest_cb,
                     &user_data)){
   /*
    * Произошла ошибка, освобождаем все назначения и выходим...
    */
    cupsFreeDests(user_data.num_dests, user_dasta.dests);

    *dests = NULL;
    return (0);
  }

 /*
  * Возвращаем массив назначений...
  */
  *dests = user_data.dests;
  return (user_data.num_dests);
}
```

## Основная информация о назначении

Поля `num_options` и `options` структуры `cups_dest_t` предоставляют базовые  атрибуты назначения, в дополнение к пользовательским опциям и значениям того же нащначения.  Следующие имена предопределены для различных атрибутов назначения:

- "auth-info-required": Тип аутентификации, необходимый для печати в это назначение: "none", "username,password", "domain,username,password", or
  "negotiate" (Kerberos).
- "printer-info": Удобочитаемое описание назначение, например "My
  Laser Printer".
- "printer-is-accepting-jobs": "true" назначение принемает новые задания,
  "false" в противном случае.
- "printer-is-shared": "true" назначение используется совменстно с другими компьютерами, "false" в противном случае.
- "printer-location":  Удобочитаемое местоположение назначения, например «Лаборатория 4».
- "printer-make-and-model":  Удобочитаемая марка и модель назначения, например
   "ExampleCorp LaserPrinter 4000 Series".
- "printer-state": Состояние: "3" если назначение свободно, "4" выполняется задание, и "5" назначение остановлено
- "printer-state-change-time": Время UNIX установки в текущее состояние.
- "printer-state-reasons": Дополнительные ключевые слова состояния, разделенные запятыми, например «ошибка пустого лотка носителя» и «предупреждение о  низком уровне тонера».
- "printer-type": Значение `cups_ptype_t`, связанное с назначением. 
- "printer-uri-supported": URI, связанный с назначением; если не установлено, это назначение было обнаружено, но еще не настроено как локальный принтер.

Используйте функцию `cupsGetOption` для получения значения. Например, следующий код возвращает марку и модель назначения:

```c
const char *model = cupsGetOption("printer-make-and-model",
                                  dest->num_options,
                                  dest->options);
```

## Детальная информация о назначении

Для каждого назначения можно использовать функцию `cupsCopyDestInfo` для получения подробной информации о назначении.

```c
cups_dinfo_t *cupsCopyDestInfo(http_t *http, cups_dest_t *dest);
```

Параметр `http` определяет соединение с планировщиком CUPS и обычно используется константа `CUPS_HTTP_DEFAULT`.  Параметр `dest` указывает на назначение для запроса.

Возвращаемая структура `cups_dinfo_t` содержит моментальный снимок поддерживаемыx параметров и их  значения и значения по умолчанию. Это также может сообщать об ограничениях между различными параметрами и значениями и рекомендовать изменения чтобы разрешить эти ограничения.

### Получение поддерживаемых параметров и значений

Можно использовать функцию`cupsCheckDestSupported` для проверки поддерживается ли опция или опция и значение:

```c
int cupsCheckDestSupported(http_t *http, cups_dest_t *dest,
                       cups_dinfo_t *info,
                       const char *option,
                       const char *value);
```

Параметр `option` определяет имя проверяемой опции для проверки. Константы, указанные ниже, могут использоваться для проверки различных стандартных опций:

- `CUPS_COPIES`: Controls the number of copies that are produced.
- `CUPS_FINISHINGS`: A comma-delimited list of integer constants that control
  the finishing processes that are applied to the job, including stapling,
  punching, and folding.
- `CUPS_MEDIA`: Controls the media size that is used, typically one of the
  following: `CUPS_MEDIA_3X5`, `CUPS_MEDIA_4X6`, `CUPS_MEDIA_5X7`,
  `CUPS_MEDIA_8X10`, `CUPS_MEDIA_A3`, `CUPS_MEDIA_A4`, `CUPS_MEDIA_A5`,
  `CUPS_MEDIA_A6`, `CUPS_MEDIA_ENV10`, `CUPS_MEDIA_ENVDL`, `CUPS_MEDIA_LEGAL`,
  `CUPS_MEDIA_LETTER`, `CUPS_MEDIA_PHOTO_L`, `CUPS_MEDIA_SUPERBA3`, or
  `CUPS_MEDIA_TABLOID`.
- `CUPS_MEDIA_SOURCE`: Controls where the media is pulled from, typically either
  `CUPS_MEDIA_SOURCE_AUTO` or `CUPS_MEDIA_SOURCE_MANUAL`.
- `CUPS_MEDIA_TYPE`: Controls the type of media that is used, typically one of
  the following: `CUPS_MEDIA_TYPE_AUTO`, `CUPS_MEDIA_TYPE_ENVELOPE`,
  `CUPS_MEDIA_TYPE_LABELS`, `CUPS_MEDIA_TYPE_LETTERHEAD`,
  `CUPS_MEDIA_TYPE_PHOTO`, `CUPS_MEDIA_TYPE_PHOTO_GLOSSY`,
  `CUPS_MEDIA_TYPE_PHOTO_MATTE`, `CUPS_MEDIA_TYPE_PLAIN`, or
  `CUPS_MEDIA_TYPE_TRANSPARENCY`.
- `CUPS_NUMBER_UP`: Controls the number of document pages that are placed on
  each media side.
- `CUPS_ORIENTATION`: Controls the orientation of document pages placed on the
  media: `CUPS_ORIENTATION_PORTRAIT` or `CUPS_ORIENTATION_LANDSCAPE`.
- `CUPS_PRINT_COLOR_MODE`: Controls whether the output is in color
  \(`CUPS_PRINT_COLOR_MODE_COLOR`), grayscale
  \(`CUPS_PRINT_COLOR_MODE_MONOCHROME`), or either
  \(`CUPS_PRINT_COLOR_MODE_AUTO`).
- `CUPS_PRINT_QUALITY`: Controls the generate quality of the output:
  `CUPS_PRINT_QUALITY_DRAFT`, `CUPS_PRINT_QUALITY_NORMAL`, or
  `CUPS_PRINT_QUALITY_HIGH`.
- `CUPS_SIDES`: Controls whether prints are placed on one or both sides of the
  media: `CUPS_SIDES_ONE_SIDED`, `CUPS_SIDES_TWO_SIDED_PORTRAIT`, or
  `CUPS_SIDES_TWO_SIDED_LANDSCAPE`.

Если параметр`value` равен `NULL`, функция `cupsCheckDestSupported` возвращает поддерживается ли опция назначением. В противном случае, функция
возвращает, поддерживается ли указанное значение параметра.

Функция `cupsFindDestSupported` возвращает атрибут IPP, содержащий поддерживаемые значения для данной опции:

```c
 ipp_attribute_t *cupsFindDestSupported(http_t *http, cups_dest_t *dest,
                       cups_dinfo_t *dinfo,
                       const char *option);
```

Например, следующий код печатает поддерживаемые процессы финишной обработки для назначения, если есть, на стандартный вывод:

```c
cups_dinfo_t *info = cupsCopyDestInfo(CUPS_HTTP_DEFAULT,
                                      dest);

if (cupsCheckDestSupported(CUPS_HTTP_DEFAULT, dest, info,
                           CUPS_FINISHINGS, NULL)){
  ipp_attribute_t *finishings =
      cupsFindDestSupported(CUPS_HTTP_DEFAULT, dest, info,
                            CUPS_FINISHINGS);
  int i, count = ippGetCount(finishings);

  puts("finishings supported:");
  for (i = 0; i < count; i ++)
    printf("  %d\n", ippGetInteger(finishings, i));
} else
  puts("finishings not supported.");
```

Параметр «job-creation-attributes» можно запросить, чтобы получить список поддерживаемых параметров. Например, следующий код выводит список поддерживаемых параметров на стандартный вывод:

```c
ipp_attribute_t *attrs =
    cupsFindDestSupported(CUPS_HTTP_DEFAULT, dest, info,
                          "job-creation-attributes");
int i, count = ippGetCount(attrs);

for (i = 0; i < count; i ++)
  puts(ippGetString(attrs, i, NULL));
```

### Получение значений по-умолчанию

Существует два набора значений по умолчанию: значения по умолчанию для пользователя, которые доступны через элементы `num_options` и `options` структуры `cups_dest_t`, и значения по умолчанию для назначения, доступные через структуру `cups_dinfo_t` и функцию `cupsFindDestDefault`, которая возвращает IPP атрибут, содержащий значение(я) по умолчанию для данной опции:

```c
ipp_attribute_t *cupsFindDestDefault(http_t *http, cups_dest_t *dest,
                    cups_dinfo_t *dinfo,
                    const char *option);
```

Пользовательские значения по умолчанию из `cupsGetOption` всегда должны ссылаться на значения по умолчанию назначения. Например, следующий код выводит значения (finishings) по умолчанию на стандартный вывод:

```c
const char *def_value =
    cupsGetOption(CUPS_FINISHINGS, dest->num_options,
                  dest->options);
ipp_attribute_t *def_attr =
    cupsFindDestDefault(CUPS_HTTP_DEFAULT, dest, info,
                        CUPS_FINISHINGS);

if (def_value != NULL) {
  printf("Default finishings: %s\n", def_value);
} else {
  int i, count = ippGetCount(def_attr);

  printf("Default finishings: %d",
         ippGetInteger(def_attr, 0));
  for (i = 1; i < count; i ++)
    printf(",%d", ippGetInteger(def_attr, i));

  putchar('\n');
}
```

### Получение готовых (загруженных) значений

*The finishings and media options* поддерживают запросы на готовые или загруженные значения. Например, на принтере могут быть установлены финишеры для перфорации и сшивания, но скрепок нет — в поддерживаемых значениях будут указаны процессы финишной обработки и перфорации, а в значениях готовности будут перечислены только процессы перфорации. 
Точно так же принтер может поддерживать сотни различных размеров носителей, но момент времени может загружаться только один размер — готовые значения ограничены носителем, который фактически находится в принтере.

Функция `cupsFindDestReady` находит IPP атрибут, содержащий *готовые* значения для данной опции:

```c
ipp_attribute_t *cupsFindDestReady(http_t *http, cups_dest_t *dest,
                  cups_dinfo_t *dinfo, const char *option);
```

Например, в следующем коде перечислены процессы, готовые для  финишной обработки:

```c
ipp_attribute_t *ready_finishings =
    cupsFindDestReady(CUPS_HTTP_DEFAULT, dest, info,
                      CUPS_FINISHINGS);

if (ready_finishings != NULL){
  int i, count = ippGetCount(ready_finishings);

  puts("finishings ready:");
  for (i = 0; i < count; i ++)
    printf("  %d\n", ippGetInteger(ready_finishings, i));
} else
  puts("no finishings are ready.");
```

### Параметры размера носителя

CUPS предоставляет функции для запроса размеров и полей для каждого из поддерживаемых параметров размера носителя. Структура `cups_size_t` используется для описания размера носителя:

```c
typedef struct cups_size_s{
  char media[128];
  int width, length;
  int bottom, left, right, top;
} cups_size_t;
```

Параметры `width` и `length` определяют размеры носителя в сотые доли миллиметра (1/2540 дюйма), a `bottom`, `left`, `right` и `top` определяют поля области печати, также в сотых долях миллиметров.

Функции `cupsGetDestMediaByName` и `cupsGetDestMediaBySize` ищут информацию о размере носителя с использованием имени стандартного размера носителя или размеров в сотые доли миллиметра:

```c
int cupsGetDestMediaByName(http_t *http, cups_dest_t *dest,
                       cups_dinfo_t *dinfo,
                       const char *media,
                       unsigned flags, cups_size_t *size);

int cupsGetDestMediaBySize(http_t *http, cups_dest_t *dest,
                       cups_dinfo_t *dinfo,
                       int width, int length,
                       unsigned flags, cups_size_t *size);
```

Параметры `media`, `width` и `length` указывают размеры для поиска. Параметр `flags` определяет битовое поле, управляющее различными параметрами поиска:

- `CUPS_MEDIA_FLAGS_BORDERLESS`: Find a borderless size.
- `CUPS_MEDIA_FLAGS_DUPLEX`: Find a size compatible with two-sided printing.
- `CUPS_MEDIA_FLAGS_EXACT`: Find an exact match for the size.
- `CUPS_MEDIA_FLAGS_READY`: If the printer supports media sensing or
  configuration of the media in each tray/source, find the size amongst the
  "ready" media.

Если для назначения найден соответствующий размер, информация о размере сохраняется в структуре, на которую указывает аргумент `size`, и возвращается 1. В противном случае возвращается 0.

Например, следующий код печатает данные размеров полей для двусторонней печати на носителе US Letter:

```c
cups_size_t size;

if (cupsGetDestMediaByName(CUPS_HTTP_DEFAULT, dest, info,
                           CUPS_MEDIA_LETTER,
                           CUPS_MEDIA_FLAGS_DUPLEX, &size)){
  puts("Margins for duplex US Letter:");
  printf("  Bottom: %.2fin\n", size.bottom / 2540.0);
  printf("    Left: %.2fin\n", size.left / 2540.0);
  printf("   Right: %.2fin\n", size.right / 2540.0);
  printf("     Top: %.2fin\n", size.top / 2540.0);
} else
  puts("Margins for duplex US Letter are not available.");
```

Вы также можете перечислить все размеры, соответствующие заданному значению `flags`, используя функции `cupsGetDestMediaByIndex` и `cupsGetDestMediaCount`:

```c
int cupsGetDestMediaByIndex(http_t *http, cups_dest_t *dest,
                        cups_dinfo_t *dinfo, int n,
                        unsigned flags, cups_size_t *size);

int cupsGetDestMediaCount(http_t *http, cups_dest_t *dest,
                      cups_dinfo_t *dinfo, unsigned flags);
```

Например, следующий код выводит список используемых носителей и соответствующие отступы:

```c
cups_size_t size;
int i;
int count = cupsGetDestMediaCount(CUPS_HTTP_DEFAULT,
                                  dest, info,
                                  CUPS_MEDIA_FLAGS_READY);

for (i = 0; i < count; i ++){
  if (cupsGetDestMediaByIndex(CUPS_HTTP_DEFAULT, dest, info,
                              i, CUPS_MEDIA_FLAGS_READY,
                              &size)){
    printf("%s:\n", size.name);
    printf("   Width: %.2fin\n", size.width / 2540.0);
    printf("  Length: %.2fin\n", size.length / 2540.0);
    printf("  Bottom: %.2fin\n", size.bottom / 2540.0);
    printf("    Left: %.2fin\n", size.left / 2540.0);
    printf("   Right: %.2fin\n", size.right / 2540.0);
    printf("     Top: %.2fin\n", size.top / 2540.0);
  }
}
```

Наконец, функция `cupsGetDestMediaDefault` возвращает размер носителя по умолчанию:

```c
int cupsGetDestMediaDefault(http_t *http, cups_dest_t *dest,
                        cups_dinfo_t *dinfo, unsigned flags,
                        cups_size_t *size);
```

### Локализация параметров и значений

CUPS предоставляет три функции для получения локализованных, удобочитаемых строк в текущей локали пользователя для параметров и значений: `cupsLocalizeDestMedia`, `cupsLocalizeDestOption` и `cupsLocalizeDestValue`:

```c
const char * cupsLocalizeDestMedia(http_t *http, cups_dest_t *dest,
                      cups_dinfo_t *info, unsigned flags,
                      cups_size_t *size);

const char * cupsLocalizeDestOption(http_t *http, cups_dest_t *dest,
                       cups_dinfo_t *info,
                       const char *option);

const char * cupsLocalizeDestValue(http_t *http, cups_dest_t *dest,
                      cups_dinfo_t *info,
                      const char *option, const char *value);
```

## Отправка задания на печать

Когда вы готовы отправить задание на печать, вы создаете задание с помощью функции `cupsCreateDestJob`:

```c
ipp_status_t cupsCreateDestJob(http_t *http, cups_dest_t *dest,
                  cups_dinfo_t *info, int *job_id,
                  const char *title, int num_options,
                  cups_option_t *options);
```

Параметр title указывает имя задания на печать, например «Мой документ».
Аргументы `num_options` и `options` определяют параметры задания печати, которые устанавливаются с помощью функции `cupsAddOption`.

В случае успеха числовой идентификатор задания сохраняется в виде целого числа, на которое указывает параметр job_id, и возвращается IPP_STATUS_OK. В противном случае возвращается статус ошибки IPP.

Например, следующий код создает новое задание для печати 42 копий двустороннего документа US Letter:

```c
int job_id = 0;
int num_options = 0;
cups_option_t *options = NULL;

num_options = cupsAddOption(CUPS_COPIES, "42",
                            num_options, &options);
num_options = cupsAddOption(CUPS_MEDIA, CUPS_MEDIA_LETTER,
                            num_options, &options);
num_options = cupsAddOption(CUPS_SIDES,
                            CUPS_SIDES_TWO_SIDED_PORTRAIT,
                            num_options, &options);

if (cupsCreateDestJob(CUPS_HTTP_DEFAULT, dest, info,
                      &job_id, "My Document", num_options,
                      options) == IPP_STATUS_OK)
  printf("Created job: %d\n", job_id);
else
  printf("Unable to create job: %s\n",
         cupsLastErrorString());
```

После того, как задание создано, вы отправляете документы в очередь, используя функции `cupsStartDestDocument`, `cupsWriteRequestData` и `cupsFinishDestDocument`:

```c
http_status_t cupsStartDestDocument(http_t *http, cups_dest_t *dest,
                      cups_dinfo_t *info, int job_id,
                      const char *docname,
                      const char *format,
                      int num_options,
                      cups_option_t *options,
                      int last_document);

http_status_t cupsWriteRequestData(http_t *http, const char *buffer,
                     size_t length);

ipp_status_t cupsFinishDestDocument(http_t *http, cups_dest_t *dest,
                       cups_dinfo_t *info);
```

Параметр `docname` указывает имя документа, обычно исходное имя файла. Параметр `format` указывает тип носителя MIME документа, включая следующие константы:

- `CUPS_FORMAT_JPEG`: "image/jpeg"
- `CUPS_FORMAT_PDF`: "application/pdf"
- `CUPS_FORMAT_POSTSCRIPT`: "application/postscript"
- `CUPS_FORMAT_TEXT`: "text/plain"

Аргументы `num_options` и `options` указывают параметры печати для каждого документа, которые в настоящее время должны быть равны 0 и `NULL`. Параметр `last_document` указывает, является ли это последним документом в задании.

Например, следующий код отправляет PDF-файл только что созданному заданию:

```c
FILE *fp = fopen("filename.pdf", "rb");
size_t bytes;
char buffer[65536];

if (cupsStartDestDocument(CUPS_HTTP_DEFAULT, dest, info,
                          job_id, "filename.pdf", 0, NULL,
                          1) == HTTP_STATUS_CONTINUE) {
  while ((bytes = fread(buffer, 1, sizeof(buffer), fp)) > 0)
    if (cupsWriteRequestData(CUPS_HTTP_DEFAULT, buffer,
                             bytes) != HTTP_STATUS_CONTINUE)
      break;

  if (cupsFinishDestDocument(CUPS_HTTP_DEFAULT, dest,
                             info) == IPP_STATUS_OK)
    puts("Document send succeeded.");
  else
    printf("Document send failed: %s\n",
           cupsLastErrorString());
}

fclose(fp);
```

# Отправка IPP запросов

CUPS предоставляет богатый API для отправки запросов IPP планировщику или принтерам, как правило, из управляющих или служебных приложений, основной целью которых не является отправка заданий на печать.

## Подключение к планировщику или принтеру

Соединение с планировщиком или принтером представлено типом соединения `http_t`. С промощью функции `cupsConnectDest` осуществляется подключение к планировщику или принтеру, связанному с назначением:

```c
http_t *cupsConnectDest(cups_dest_t *dest, unsigned flags, int msec,
                int *cancel, char *resource,
                size_t resourcesize, cups_dest_cb_t cb,
                void *user_data);
```

Параметр `dest` указывает назначение для подключения.

Параметр `flags` указывает, хотите ли вы подключиться к планировщику (`CUPS_DEST_FLAGS_NONE`) или устройству/принтеру `CUPS_DEST_FLAGS_DEVICE`), связанному с назначением.

Параметр `msec` указывает, как долго вы готовы ждать соединения должно быть установлено в миллисекундах. Укажите значение `-1`, чтобы ждать бесконечно.

Параметр `cancel` указывает адрес целочисленной переменной, для которой можно установить ненулевое значение, чтобы отменить соединение. Укажите значение `NULL`, чтобы не указывать переменную отмены.

Параметры `resource` и `resourcesize` указывают адрес и размер массива строк символов, в котором хранится путь, используемый при отправке запроса IPP.

Параметры `cb` и `user_data` определяют функцию обратного вызова, которая возвращает 1 для продолжения подключения или 0 для остановки. Обратный вызов назначения работает так же, как и для функции `cupsEnumDests`.

В случае успеха возвращается HTTP-соединение, которое можно использовать для отправки запросов IPP и получения ответов IPP.

Например, следующий код подключается к принтеру, связанному с назначением, с тайм-аутом 30 секунд:

```c
char resource[256];
http_t *http = cupsConnectDest(dest, CUPS_DEST_FLAGS_DEVICE,
                               30000, NULL, resource,
                               sizeof(resource), NULL, NULL);
```

## Создание запроса IPP

Запросы IPP представлены IPP сообщением типа «ipp_t», и каждый атрибут IPP в запросе представлен типом «ipp_attribute_t». Каждый запрос IPP включает код операции (`IPP_OP_CREATE_JOB`, `IPP_OP_GET_PRINTER_ATTRIBUTES` и т. д.) и 32-битный целочисленный идентификатор.

Функция `ippNewRequest` создает новый IPP запрос:

```c
ipp_t * ippNewRequest(ipp_op_t op);
```

Параметр `op` указывает код операции IPP для запроса. Например, следующий код создает запрос IPP Get-Printer-Attributes:

```c
ipp_t *request = ippNewRequest(IPP_OP_GET_PRINTER_ATTRIBUTES);
```

Идентификатору запроса автоматически присваивается уникальное значение для текущего процесса.

Каждый запрос IPP начинается с двух атрибутов, «attributes-charset» и «attributes-natural-language», за которыми следуют атрибуты IPP, которые определяют цель операции. `ippNewRequest` автоматически добавляет правильные атрибуты "attributes-charset" и "attributes-natural-language", но вы должны добавить целевые атрибуты. Например, следующий код добавляет атрибут «printer-uri» в запрос IPP Get-Printer-Attributes, чтобы указать, какой принтер запрашивается:

```c
const char *printer_uri = cupsGetOption("device-uri",
                                        dest->num_options,
                                        dest->options);

ippAddString(request, IPP_TAG_OPERATION, IPP_TAG_URI,
             "printer-uri", NULL, printer_uri);
```

> **Заметка:**
> 
> Если бы мы хотели запросить планировщик, а не устройство, мы бы указали параметр «printer-uri-support» вместо значения «device-uri». ??

Функция `ippAddString` добавляет атрибут «printer-uri» в IPP.
запрос. Аргумент `IPP_TAG_OPERATION` указывает, что атрибут является частью операции. Аргумент `IPP_TAG_URI` указывает, что значение представляет собой строку универсального идентификатора ресурса (URI). Аргумент `NULL` указывает на отсутствие языка (английского, французского, японского и т. д.), связанного со строкой, а аргумент `printer_uri` указывает строковое значение.

Запрос IPP Get-Printer-Attributes также поддерживает параметр IPP «requested-attributes», в котором перечислены интересующие вас атрибуты и значения. Например, следующий код запрашивает атрибуты состояния принтера:

```c
static const char * const requested_attributes[] =
{
  "printer-state",
  "printer-state-message",
  "printer-state-reasons"
};

ippAddStrings(request, IPP_TAG_OPERATION, IPP_TAG_KEYWORD,
              "requested-attributes", 3, NULL,
              requested_attributes);
```

Функция `ippAddStrings` добавляет атрибут с одной или несколькими строками, в данном случае тремя. Аргумент `IPP_TAG_KEYWORD` указывает, что строки являются значениями ключевых слов, которые используются для имен атрибутов. Все строки используют один и тот же язык (`NULL`), а атрибут будет содержать три строки в массиве `requested_attributes`.

CUPS предоставляет множество функций для добавления атрибутов разных типов::

- `ippAddBoolean` adds a boolean (`IPP_TAG_BOOLEAN`) attribute with one value.
- `ippAddInteger` adds an enum (`IPP_TAG_ENUM`) or integer (`IPP_TAG_INTEGER`)
  attribute with one value.
- `ippAddIntegers` adds an enum or integer attribute with one or more values.
- `ippAddOctetString` adds an octetString attribute with one value.
- `ippAddOutOfBand` adds a admin-defined (`IPP_TAG_ADMINDEFINE`), default
  (`IPP_TAG_DEFAULT`), delete-attribute (`IPP_TAG_DELETEATTR`), no-value
  (`IPP_TAG_NOVALUE`), not-settable (`IPP_TAG_NOTSETTABLE`), unknown
  (`IPP_TAG_UNKNOWN`), or unsupported (`IPP_TAG_UNSUPPORTED_VALUE`) out-of-band
  attribute.
- `ippAddRange` adds a rangeOfInteger attribute with one range.
- `ippAddRanges` adds a rangeOfInteger attribute with one or more ranges.
- `ippAddResolution` adds a resolution attribute with one resolution.
- `ippAddResolutions` adds a resolution attribute with one or more resolutions.
- `ippAddString` adds a charset (`IPP_TAG_CHARSET`), keyword (`IPP_TAG_KEYWORD`),
  mimeMediaType (`IPP_TAG_MIMETYPE`), name (`IPP_TAG_NAME` and
  `IPP_TAG_NAMELANG`), naturalLanguage (`IPP_TAG_NATURAL_LANGUAGE`), text
  (`IPP_TAG_TEXT` and `IPP_TAG_TEXTLANG`), uri (`IPP_TAG_URI`), or uriScheme
  (`IPP_TAG_URISCHEME`) attribute with one value.
- `ippAddStrings` adds a charset, keyword, mimeMediaType, name, naturalLanguage,
  text, uri, or uriScheme attribute with one or more values.

## Отправка IPP запроса

Создав запрос IPP, вы можете отправить его с помощью функции `cupsDoRequest`. Например, следующий код отправляет запрос IPP Get-Printer-Attributes получателю и сохраняет ответ:

```c
ipp_t *response = cupsDoRequest(http, request, resource);
```

Для таких запросов, как Send-Document, которые включают файл, следует использовать функцию `cupsDoFileRequest`:

```c
ipp_t *response = cupsDoFileRequest(http, request, resource,
                                    filename);
```

И `cupsDoRequest`, и `cupsDoFileRequest` освобождают запрос IPP. Если получен действительный ответ, он сохраняется в новом сообщении IPP (`ipp_t`) и возвращается вызывающей стороне. В противном случае возвращается `NULL`.

Статус самого последнего запроса можно запросить с помощью функции `cupsLastError`, например:

```c
if (cupsLastError() >= IPP_STATUS_ERROR_BAD_REQUEST)
{
  /* request failed */
}
```

Человекочитаемое сообщение об ошибке также доступно с использованием функции `cupsLastErrorString`:

```c
if (cupsLastError() >= IPP_STATUS_ERROR_BAD_REQUEST)
{
  /* request failed */
  printf("Request failed: %s\n", cupsLastErrorString());
}
```

## Обработка IPP ответа

Каждый ответ на запрос IPP также является сообщением IPP (`ipp_t`) со своими собственными атрибутами (`ipp_attribute_t`), которые включают код состояния (`IPP_STATUS_OK`, `IPP_STATUS_ERROR_BAD_REQUEST` и т. д.) и соответствующее 32-битное целое число идентификатор из запроса.

Например, следующий код находит атрибуты состояния принтера и печатает их значения:

```c
ipp_attribute_t *attr;

if ((attr = ippFindAttribute(response, "printer-state",
                             IPP_TAG_ENUM)) != NULL){
  printf("printer-state=%s\n",
         ippEnumString("printer-state", ippGetInteger(attr, 0)));
} else
  puts("printer-state=unknown");

if ((attr = ippFindAttribute(response, "printer-state-message",
                             IPP_TAG_TEXT)) != NULL){
  printf("printer-state-message=\"%s\"\n",
         ippGetString(attr, 0, NULL)));
}

if ((attr = ippFindAttribute(response, "printer-state-reasons",
                             IPP_TAG_KEYWORD)) != NULL){
  int i, count = ippGetCount(attr);

  puts("printer-state-reasons=");
  for (i = 0; i < count; i ++)
    printf("    %s\n", ippGetString(attr, i, NULL)));
}
```

Функция `ippGetCount` возвращает количество значений в атрибуте.

Функции `ippGetInteger` и `ippGetString` возвращают одно целое или строковое значение из атрибута.

Функция `ippEnumString` преобразует значение перечисления в его эквивалент ключевого слова (строку).

Когда вы закончите использовать ответное сообщение IPP, освободите его с помощью функции `ippDelete`:

```c
ippDelete(response);
```

## Аутентификация

CUPS обычно обрабатывает аутентификацию через консоль. Приложения с графическим интерфейсом должны устанавливать пароль с помощью функции обратного вызова, передавая ее в функцию `cupsSetPasswordCB2`:

```c
void cupsSetPasswordCB2(cups_password_cb2_t cb, void *user_data);
```

Обратный вызов вызова для установки пароля будет вызываться при необходимости при установке текущего имени пользователя с помощью `cupsSetUser` (и возврат строки??):

```c
const char * cups_password_cb2(const char *prompt, http_t *http,
                  const char *method, const char *resource,
                  void *user_data);
```

Параметр `prompt` представляет собой строку из CUPS, которая должна отображаться пользователю.

Параметр `http` — это соединение, на котором размещается аутентифицируемый запрос. Обратный вызов пароля может вызывать функции `httpGetField` и `httpGetSubField` для поиска дополнительных деталей, касающихся запроса аутентификации.

Параметр `method` указывает метод HTTP, используемый для запроса, и обычно это «POST».

Параметр `resource` указывает путь, используемый для запроса.

Параметр `user_data` содержит указатель пользовательских данных из вызова `cupsSetPasswordCB2`.
