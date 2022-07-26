# BugBane

Набор утилит для аудита безопасности приложений.<br>

Основные принципы и особенности:
1. BugBane образует пайплайн безопасной разработки, этапы которого описаны в виде кода. Без BugBane пайплайн сильно фрагментирован: часть низкоуровневых инструкций описана в Makefile, часть - в shell-скриптах, что-то - в файлах используемого решения для CI/CD, а что-то - в Dockerfile. При подобной фрагментации гораздо проще допустить ошибки.
2. BugBane - вариант стандартизации структуры рабочей директории, структуры и формата отчётных материалов, а также пайплайна безопасной разработки как последовательности определённых действий.
3. BugBane - это набор инструментов, которые могут использоваться как совместно, так и по отдельности.
4. BugBane позволяет выполнить тестирование и подготовить отчётные материалы: в процессе тестирования собираются свидетельства выполняемых операций в виде команд, журналов работы, скриншотов и отчётов. Все материалы соответствуют фактически выполненным действиям и запущенным командам, что защищает пользователя от ошибок при ручном вводе и сборе этих сведений.
5. BugBane является решением, открытым для улучшений с учётом пожеланий сообщества.


Возможности BugBane на текущий момент:
1. Сборка целей для фаззинг-тестирования, в том числе с санитайзерами и сбором покрытия: AFL++, libFuzzer.
2. Фаззинг сборок с использованием [AFL++](https://github.com/AFLplusplus/AFLplusplus), [libFuzzer](https://www.llvm.org/docs/LibFuzzer.html), [dvyukov/go-fuzz](https://github.com/dvyukov/go-fuzz) на заданном количестве ядер до заданного условия остановки.
3. Синхронизация (импорт и экспорт) тестовых примеров между рабочей директорией фаззера и хранилищем (папкой). Включает отсеивание дубликатов (для всех фаззеров) и минимизацию на основе инструментов фаззера (пока только AFL++).
4. Сбор покрытия тестируемого приложения на семплах, полученных в процессе фаззинг-тестирования, а также генерация HTML-отчётов о покрытии (lcov, lcov-llvm, go-tool-cover).
5. Воспроизведение падений и зависаний, обнаруженных фаззером. Извлечение места возникновения ошибки (имя функции, путь к файлу исходного кода, номер строки в файле).
6. Отправка сведений о воспроизводимых багах в систему управления уязвимостями: [Defect Dojo](https://github.com/DefectDojo/django-DefectDojo).
7. Получение скриншотов работы фаззера (tmux + ansifilter + pango-view) и главной страницы отчёта о покрытии исходного кода (WeasyPrint, Selenium).
8. Генерация отчётов на основе шаблонов Jinja2.

# Установка

## Зависимости
UNIX-подобная ОС<br>
Python >= 3.6<br><br>
Зависимости, используемые утилитами BugBane:<br>
**bb-build**: компиляторы используемого фаззера в PATH (afl-g++-fast, clang, ...).<br>
**bb-corpus**: утилита минимизации в соответствии с используемым фаззером в PATH (afl-cmin, ...).<br>
**bb-fuzz**: используемый фаззер в PATH (afl-fuzz, go-fuzz, ...).<br>
**bb-coverage**: используемые средства сбора покрытия в PATH (lcov, genhtml, go, ...).<br>
**bb-reproduce**: утилита `timeout`, отладчик `gdb`.<br>
**bb-send**: python-библиотека `defectdojo_api`.<br>
**bb-screenshot**, **bb-report**: python-библиотеки `Jinja2`, `WeasyPrint`, `Selenium`, приложения `ansifilter` и `pango-view` в PATH, приложение `geckodriver` в PATH и браузер Firefox (необязательно, только для Selenium), шрифты `mono` (могут отсутствовать в базовых докер-образах).<br>
Примечания:
- Python-библиотеки устанавливается автоматически при выполнении дальнейших инструкций;
- в настоящий момент Selenium + geckodriver + Firefox *необходимо* использовать только для отчётов о покрытии, построенных утилитой `go tool cover`, для остальных отчётов достаточно использовать WeasyPrint; при этом скриншоты, сделанные с помощью Selenium, выглядят лучше. Недостаток: размер этих пакетов в некоторых дистрибутивах может занять ~700 мегабайт;
- для просмотра отчётов непосредственно в образе Docker с помощью утилит типа less может потребоваться установка локали с поддержкой UTF-8 и указание переменной LANG.

## Установка и удаление пакета
Установить пакет можно локально с помощью pip:
```
git clone https://github.com/gardatech/bugbane
cd bugbane
pip install .[all]
```
Проверить выполнение тестов:
```
pytest
```
Доступна установка только необходимых Python-зависимостей:
| Группа pip install | Фаззинг\* | Заведение багов в Defect Dojo | Отчёты и скриншоты | Тестирование BugBane | Разработка BugBane |
|-|-|-|-|-|-|
| - | + | - | - | - | - |
| dd | + | + | - | - | - |
| reporting | + | - | + | - | - |
| test | + | - | - | + | - |
| all | + | + | + | + | - |
| dev | + | + | + | + | + |

\* Выполнение сборок, фаззинг, работа с семплами, сбор покрытия и воспроизведение багов.

Таким образом, можно разделить тестирование и работу с его результатами на разные хосты worker и reporter:
```shell
pip install .                  # worker
pip install .[dd,reporting]    # reporter
```
Результат: на хосте worker не требуются зависимости для генерации отчётов, на хосте reporter не требуются окружение для запуска тестируемых приложений и фаззеров.

Для удаления использовать следующую команду:
```
pip uninstall bugbane
```

# Запуск
Рекомендуется использовать BugBane в среде Docker.<br>
Подразумевается последовательный запуск инструментов в определённом порядке, например:
1. bb-build
2. bb-corpus (import)
3. bb-fuzz
4. bb-coverage
5. bb-reproduce
6. bb-corpus (export)
7. bb-send
8. bb-report

При этом этап №1 является опциональным (сборки могут быть выполнены другими способами), а этапы 7 и 8 могут выполняться в отдельном образе Docker или на отдельной машине.

**Большинство инструментов BugBane работают с конфигурационным файлом bugbane.json**: получают входные переменные, обновляют их значения, добавляют новые переменные и дописывают их в существующий файл конфигурации.<br>
Пример исходного файла конфигурации, достаточного для последовательного запуска всех инструментов BugBane:
```json
{
    "fuzzing": {
        "os_name": "Arch Linux",
        "os_version": "Rolling",

        "product_name": "RE2",
        "product_version": "2022-02-01",
        "module_name": "BugBane RE2 Example",
        "application_name": "re2",

        "is_library": true,
        "is_open_source": true,
        "language": [
            "C++"
        ],
        "parse_format": [
            "RegExp"
        ],
        "tested_source_file": "re2_fuzzer.cc",
        "tested_source_function": "TestOneInput",

        "build_cmd": "./build.sh",
        "build_root": "./build",
        "tested_binary_path": "re2_fuzzer",
        "sanitizers": [
            "ASAN", "UBSAN"
        ],
        "builder_type": "AFL++LLVM",
        "fuzzer_type": "AFL++",

        "run_args": null,
        "run_env": null,
        "timeout": null,

        "fuzz_cores": 16
    }
}
```

Утилиты corpus, coverage, reproduce и report поддерживают **альтернативный режим запуска (manual run mode)**, утилита screenshot работает только в этом режиме. Режим запуска manual предназначен для более тонкой настройки параметров или для использования отдельно от других инструментов BugBane.<br>

## bb-build
Выполняет сборку C/C++ приложения с использованием компиляторов фаззера.<br>
На вход приложению подаются:
1. Исходный код, подлежащий сборке
2. Файл с переменными bugbane.json

В файле bugbane.json должны быть заданы переменные: `builder_type`, `build_cmd`, `build_root`, `sanitizers`.<br>

Команда, указанная в переменной `build_cmd`, должна учитывать значения переменных окружения CC, CXX, CFLAGS, CXXFLAGS и при запуске выполнять сборку тестируемого компонента в режиме фаззинг-тестирования. После выполнения одного запуска команды `build_cmd` в папке `build_root` должна оказаться сборка тестируемого приложения. Переменная `sanitizers` должна содержать список санитайзеров, с которыми требуется выполнить сборки. Для каждого санитайзера будет выполнена отдельная сборка.<br>

Приложение последовательно выполняет несколько сборок (с различными санитайзерами + для сбора покрытия + дополнительные сборки для фаззинга) и после каждой сборки сохраняет результаты сборки из папки `build_root` в папку, указанную аргументом запуска `-o`. При этом обновляются некоторые переменные в файле bugbane.json (в частности, `sanitizers` - заполняется названиями санитайзеров, для которых удалось выполнить сборку).<br>

Пример скрипта, путь к которому может быть указан в команде сборки `build_cmd`:
```bash
#!/bin/bash
export CXX="${CXX:=afl-clang-fast++}" &&
mkdir -p build &&
make clean &&
make -j obj/libre2.a &&
$CXX $CXXFLAGS --std=c++11 -I. re2/fuzzing/re2_fuzzer.cc /AFLplusplus/libAFLDriver.a obj/libre2.a -lpthread -o build/re2_fuzzer
```
Таким образом флагами компиляции можно управлять извне и получать сборки с любыми санитайзерами, с инструментацией для сбора покрытия, с отладочной информацией и т.д.

Пример запуска:
```shell
bb-build -i /src -o /fuzz
```
При этом директория /src должна содержать файл bugbane.json.<br>
В результате в пути /fuzz появятся папки с полученными сборками, например: /fuzz/basic, /fuzz/asan, /fuzz/coverage. Также в папке /fuzz сохранится журнал выполнения всех сборок с указанием команд запуска и использованных переменных окружения.

### Соответствие сборок и папок
| Имя папки | Описание | builder_type |
|-|-|-|
| basic | Сборка для фаззинга. Это должна быть наиболее производительная сборка: без санитайзеров, без покрытия | AFL++GCC, AFL++GCC-PLUGIN, AFL++LLVM, AFL++LLVM-LTO, libFuzzer |
| gofuzz | Сборка для фаззинга с использованием dvyukov/go-fuzz (zip-архив). Не поддерживается bb-build, поддерживается остальными утилитами | - |
| laf | Сборка для фаззинга, скомпилированная с переменной окружения AFL_LLVM_LAF_ALL | AFL++LLVM, AFL++LLVM-LTO |
| cmplog | Сборка для фаззинга, скомпилированная с переменной окружения AFL_USE_CMPLOG | AFL++LLVM, AFL++LLVM-LTO |
| asan | Сборка для фаззинга с адресным санитайзером (Address Sanitizer) | AFL++GCC, AFL++GCC-PLUGIN, AFL++LLVM, AFL++LLVM-LTO, libFuzzer 
| ubsan | Сборка для фаззинга с санитайзером неопределённого поведения (Undefined Behavior Sanitizer) | AFL++GCC, AFL++GCC-PLUGIN, AFL++LLVM, AFL++LLVM-LTO, libFuzzer
| cfisan | Сборка для фаззинга с санитайзером целостности потока выполнения (Control Flow Integrity Sanitizer) | AFL++GCC, AFL++GCC-PLUGIN, AFL++LLVM, AFL++LLVM-LTO, libFuzzer
| tsan \* | Сборка для фаззинга с санитайзером потоков (Thread Sanitizer) | AFL++GCC, AFL++GCC-PLUGIN, AFL++LLVM, AFL++LLVM-LTO, libFuzzer
| lsan \* | Сборка для фаззинга с санитайзером утечек памяти (Leak Sanitizer). Этот функционал поддерживается адресным санитайзером, но также может использоваться отдельно | AFL++GCC, AFL++GCC-PLUGIN, AFL++LLVM, AFL++LLVM-LTO, libFuzzer
| msan \* | Сборка для фаззинга с санитайзером памяти (Memory Sanitizer) | AFL++GCC, AFL++GCC-PLUGIN, AFL++LLVM, AFL++LLVM-LTO, libFuzzer
| coverage | Сборка для получения информации о покрытии | AFL++GCC, AFL++GCC-PLUGIN, AFL++LLVM, AFL++LLVM-LTO, libFuzzer

\* Работоспособность не тестировалась.<br>

## Выполнение сборок без инструмента bb-build
Все сборки рекомендуется выполнять компиляторами фаззера, в том числе сборку для получения информации о покрытии.<br>
Все сборки должны выполняться с отладочной информацией, содержащей сведения о строках исходного кода (`-g` для gcc, `-g` или `-gline-tables-only` - для clang).<br>
Все сборки должны выполняться с флагом `-fno-omit-frame-pointer` для получения более точных стеков вызовов в случае обнаружения багов или при ручной отладке.<br>
Если фаззер поддерживает переменные окружения для включения санитайзеров (AFL_USE_ASAN и т.д.), то использование этих переменных предпочтительнее ручного указания флагов компиляции.<br>
Сборки следует размещать в папках под соответствующими названиями. Например, если фаззинг будет запущен из директории /fuzz, то сборка с ASAN должна быть сохранена в папке /fuzz/asan. Сборку, в которой одновременно присутствуют несколько санитайзеров, достаточно разместить в одном экземпляре в любой одной папке для сборки с санитайзером. То есть сборку с ASAN+UBSAN+CFISAN можно разместить в любой из папок: asan, ubsan, cfisan, lsan, tsan или msan - это не снизит эффективность фаззинга и воспроизведения падений. При этом *рекомендуется* создать несколько копий или символьных ссылок в соответствии с санитайзерами (/fuzz/asan, /fuzz/ubsan, ...).<br>
Если процесс сборки в CI занимает время, сопоставимое с временем фаззинг-тестирования, то можно обойтись единственной сборкой, одновременно включающей инструментацию фаззера, покрытия и санитайзеров. Это негативно скажется на скорости фаззинга, а также создаст дополнительную нагрузку на диск в процессе тестирования, но может быть предпочтительнее выполнения нескольких сборок. Например, приложение, собранное компиляторами AFL++ с ASAN и покрытием, можно разместить в папке /fuzz/asan, а затем скопировать его (или создать символьную ссылку) в путь /fuzz/coverage.<br>

## bb-corpus
Синхронизирует тестовые примеры в рабочей директории фаззера с хранилищем.<br>
Поддерживает импорт из хранилища в папку фаззера и экспорт из папки фаззера в хранилище.<br>
Инструмент не создаёт никаких подключений, не взаимодействует с какими-либо базами данных и не выполняет архивацию, вместо этого он работает с хранилищем как с простой папкой в файловой системе. В свою очередь хранилище может быть примонтированным каталогом Samba, NFS и т.д.<br>

Синхронизация происходит в два этапа:
1. Копирование (в случае импорта) или перемещение (в случае экспорта) из папки-источника во временную папку без создания дубликатов по содержимому (проверка sha1).
2. Минимизация семплов из временной папки в конечную папку с использованием инструмента фаззера (afl-cmin, ...).

В конфигурационном файле bugbane.json должна быть объявлена переменная `fuzzer_type`.<br>
Для минимизации с использованием afl-cmin на диске должны присутствовать сборки тестируемого приложения. Наиболее предпочтительной сборкой для минимизации семплов является сборка в папке laf, т.к. она "различает" больше путей выполнения, но если она не будет обнаружена, для минимизации будут использованы другие сборки.

Пример импорта тестовых примеров из хранилища (перед фаззинг-тестированием):
```shell
bb-corpus suite /fuzz import-from /storage
```
Пример экспорта тестовых примеров в хранилище (после фаззинг-тестирования):
```shell
bb-corpus suite /fuzz export-to /storage
```

Имена результирующих файлов будут соответствовать хеш-сумме sha1 их содержимого. При совпадении имён в конечной папке перезапись не происходит.

## bb-fuzz
Запускает фаззинг сборок тестируемого приложения на указанном количестве ядер до наступления указанного условия остановки.<br>
bb-fuzz обнаруживает сборки на диске и распределяет их по разным ядрам процессора:
* сборкам с санитайзерами выделяется по одному ядру;
* вспомогательные сборки (AFL_LLVM_LAF_ALL, AFL_USE_CMPLOG) назначаются на некоторое процентное соотношение от указанного количества ядер;
* сборка basic (без санитайзеров) занимает остальные ядра;
* сборки для определения покрытия исходного кода в фаззинг-тестировании участие не принимают.

В конфигурационном файле bugbane.json должны быть определены переменные `fuzzer_type`, `tested_binary_path`, `fuzz_cores`, `src_root`, `run_args`, `run_env` и `timeout`. Переменная `timeout` указывается в миллисекундах.<br>
На диске должны присутствовать сборки приложения, размещённые в папках, соответствующих названию сборки, точно так же, как их размещает инструмент bb-build.
Также на диске могут присутствовать файлы словарей с расширением ".dict" в папке dictionaries. Они будут объединены в один общий словарь, который будет передан фаззеру при условии поддержки со стороны фаззера.

Доступные значения переменной `fuzzer_type`: AFL++, libFuzzer, go-fuzz.<br>
Переменная `tested_binary_path` содержит путь к тестируемому приложению относительно `build_root` и относительно входной папки (где будет осуществлён поиск сборок).<br>
Переменная `src_root` не используется напрямую, но без её указания потерпят неудачу утилиты, подлежащие запуску после bb-fuzz.<br>
`run_args` - аргументы запуска тестируемого приложения в режиме фаззинг-тестирования. Переменная может включать последовательность "@@", вместо которой фаззер подставит путь к файлу.<br>
`run_env` - переменные окружения, которые необходимо установить для запуска тестируемого приложения (LD_PRELOAD и т.д.).<br>
Пример переменной `run_env` в конфигурационном файле:
```json
"run_env": {
    "LD_PRELOAD": "/src/mylib.so",
    "ENABLE_FUZZ_TARGETS", "1"
}
```

Доступные условия остановки:
* реальная продолжительность фаззинга достигла X секунд (затраченное время независимо от количества ядер / экземпляров фаззера);
* суммарная продолжительность фаззинга достигла X секунд\* (реальная продолжительность, умноженная на количество задействованных в тестировании ядер процессора);
* новые пути выполнения не обнаруживались в течение последних X секунд среди всех экземпляров фаззера.

\* Пока нет способа задать это условие.

Условие остановки задаётся с помощью переменных окружения:<br>
* CERT_FUZZ_DURATION=X - наивысший приоритет, если установлены другие переменные; X определяет количество секунд, в течение которых не должны обнаруживаться новые пути выполнения;
* CERT_FUZZ_LEVEL=X - средний приоритет; X определяет уровень контроля, что в свою очередь определяет время, в течение которого не должны обнаруживаться новые пути выполнения; допустимые значения X: 2, 3, 4.
* FUZZ_DURATION=X - наименьший приоритет; X определяет реальную продолжительность тестирования.

Переменные CERT_FUZZ_\* подходят для сертификационных испытаний, FUZZ_\* - для использования в CI/CD.<br>
Если не объявлена ни одна из указанных переменных, используется FUZZ_DURATION=600.<br>

Количество ядер определяется минимальным значением среди перечисленных:
1. Количество доступных в системе ядер.
2. Значение переменной `fuzz_cores` в файле конфигурации. Если значение не указано, будет выбрано 8 ядер.
3. Аргумент запуска `--max-cpus` (значение по умолчанию: 16).

Пример запуска:
```shell
FUZZ_DURATION=1800 bb-fuzz --suite /fuzz
```
В результате выполнения команды будет запущено несколько экземпляров фаззера в сессии tmux. Инструмент bb-fuzz будет периодически печатать статистику работы фаззера, пока не обнаружит наступление условия остановки, в данном случае, пока не накопится время работы 1800 секунд = 30 минут. Затем с использованием команд tmux capture-pane в папку /fuzz/screens будут сохранены дампы панелей tmux с возможно присутствующими ANSI-последовательностями (цвета, выделение текста жирным шрифтом и т.д.). Эти сохранённые дампы используются на слеующих этапах приложениями bb-report или bb-screenshot.<br>
**Внимание: в настоящий момент после сохранения дампов завершаются ВСЕ процессы фаззера и tmux в пределах операционной системы.**


## bb-coverage
Собирает покрытие тестируемого приложения на семплах, сгенерированных фаззером:
1. Запускает тестируемое приложение на семплах в директории синхронизации фаззера \*
2. Строит отчёт о покрытии

\* Для dvyukov/go-fuzz этап пропускается: подразумевается фаззинг с использованием ключа `-dumpcover` (bb-fuzz использует этот ключ).

В конфигурационном файле bugbane.json должны быть объявлены переменные `tested_binary_path`, `run_args`, `run_env`, `coverage_type`, `fuzzer_type`, `fuzz_sync_dir`, `src_root`.<br>
Переменная `coverage_type` заполняется приложением bb-build и соответствует типу сборки.<br>
`src_root` - путь к исходному коду тестируемого приложения на момент выполнения сборок; не обязан реально существовать в файловой системе: если директория не существует, отчёт о покрытии будет содержать проценты, но не исходный код.

Возможные значения `coverage_type`
| coverage_type | Описание |
|-|-|
| lcov | Для целей, собранных компиляторами GCC с флагом `--coverage` |
| lcov-llvm | Для целей, собранных компиляторами LLVM с флагом `--coverage` |
| go-tool-cover | Для целей golang |

Пример запуска:
```shell
bb-coverage suite /fuzz
```

Результат: в папке /fuzz/coverage_report должны появиться файлы отчёта о покрытии, в том числе /fuzz/coverage_report/index.html - главная страница отчёта о покрытии.


## bb-reproduce
Воспроизводит баги и обобщает результаты работы фаззера:<br>
1. Получает общую статистику работы фаззеров
2. Минимизирует падения и зависания путём их воспроизведения (получает информацию о типе ошибки и месте в коде: функция, файл, номер строки)
3. Формирует JSON-файл с перечисленными выше сведениями
4. Сохраняет тестовые примеры, приводящие к воспроизводимым падениям и зависаниям, на диск

В конфигурационном файле bugbane.json должны быть определены переменные `src_root`, `fuzz_sync_dir`, `fuzzer_type`, `reproduce_specs`, `run_args`, `run_env`. Переменные `fuzz_sync_dir` и `reproduce_specs` добавляются инструментом bb-fuzz.<br>
`fuzz_sync_dir` - директория синхронизации фаззера; bb-fuzz использует директорию "out".<br>
`src_root` - путь к исходному коду тестируемого приложения на момент выполнения сборок; не обязан реально существовать в файловой системе, используется для более точного определения места падений/зависаний в исходном коде.<br>
`reproduce_specs` - словарь, определяющий тип фаззера, и задающий соответствие между сборками приложения и папками, на которых требуется выполнить воспроизведение:
```json
"fuzz_sync_dir": "/fuzz/out",
"reproduce_specs": {
    "AFL++": {
        "/fuzz/basic/app": [
            "test1"
        ],
        "/fuzz/ubsan/app": [
            "test2",
            "test3"
        ]
    }
}
```
В данном случае сборка basic (/fuzz/basic/app) будет запущена на семплах `/fuzz/out/test1/{crashes,hangs}/id*`, а сборка ubsan (/fuzz/ubsan/app) - на семплах `/fuzz/out/test{2,3}/{crashes,hangs}/id*`.<br>
При каждом запуске анализируется вывод приложения в терминал, в том числе сообщения об ошибках от санитайзеров. Каждый баг воспроизводится до успешного воспроизведения, но не более N раз. Число N определяется аргументом запуска bb-reproduce `--num-reruns`, значение по умолчанию: 3. Если при воспроизведении падения не обнаруживается стек вызовов, приложение запускается под отладчиком gdb. Зависания воспроизводятся сразу под отладчиком gdb.<br>

Пример запуска:
```shell
bb-reproduce --hang-timeout 3000 suite /fuzz
```

В результате формируется файл /fuzz/bb_results.json, содержащий статистику работы фаззера и сведения об обнаруженных и воспроизведённых багах, в том числе для каждого бага сохраняются: заголовок issue/бага, место возникновения бага в исходном коде, команда запуска с конкретным семплом, stdout+stderr приложения, переменные окружения. Семплы, соответствующие воспроизводимым багам, сохраняются в папке /fuzz/bug_samples.


## bb-send
Отправляет полученные скриптом bb-reproduce данные в формате JSON в систему управления уязвимостями Defect Dojo.<br>
Один запуск инструмента соответствует созданию одного теста в нужном engagement. В пределах этого теста создаются экземпляры finding на каждый уникальный баг.<br>
Здесь и далее в качестве адреса сервера Defect Dojo используется https://dojo.local:8080.<br>

Приложение bb-send не использует файл конфигурации BugBane. Входные данные берутся из файла bb_results.json.

Пример запуска:
```
bb-send --results-file bb_results.json --host https://dojo.local:8080 \
    --user-name ci_fuzz_user --user-id 2 --token TOKEN \
    --engagement 1 --test-type 141
```
Описание некоторых аргументов запуска bb-send:<br>
`--user-id`: id указанного в `--user-name` пользователя; можно посмотреть в адресной строке Defect Dojo, выбрав нужного пользователя на странице https://dojo.local:8080/user.<br>
`--engagement`: engagement id; также можно посмотреть в адресной строке в браузере (выбрать нужный engagement на странице https://dojo.local:8080/engagement).<br>
`--test-type`: id вида теста; брать также из адресной строки (выбрать нужный тест на странице https://dojo.local:8080/test_type).<br>
`--token`: ключ API; берётся из Defect Dojo по ссылке: https://dojo.local:8080/api/key-v2 (нужно быть авторизованным от имени, указанного в `--user-name`, ключ нужен из раздела "Your current API key is ....").<br>

Если подлинность сертификата сервера Defect Dojo не может быть проверена, то следует добавить аргумент запуска `--no-ssl` и использовать http вместо https.


# bb-report
Создаёт отчёт в формате md на основе указанного Jinja2-шаблона. По умолчанию используется шаблон, подобный протоколу сертификационных испытаний.<br>
Создаёт скриншоты экранов фаззера (из дампов tmux, сохранённых ранее на этапе фаззинг-тестирования) и главной страницы HTML-отчёта о покрытии кода. Скриншоты сохраняются в папку screenshots и вставляются в отчёт в виде ссылок.<br>

В файле конфигурации bugbane.json должны быть объявлены переменные `fuzzer_type`, `coverage_type` и `fuzz_sync_dir`.<br>

Пример запуска:
```shell
bb-report --name fuzzing_re2 suite /fuzz
```
Запуск с использованием Selenium:
```shell
bb-report --html-screener selenium --name fuzzing_re2 suite /fuzz
```

Результат: в папке /fuzz появится папка screenshots и файл с отчётом fuzzing_re2.md. 

# bb-screenshot
Утилита для ручного создания скриншотов. Скриншоты создаются так же, как в приложении bb-report, но пользователь может указать имена входного и выходного файлов.

Примеры запуска:
```shell
bb-screenshot -S pango -i tmux_dump.txt -o tmux_screenshot.png
bb-screenshot -S weasyprint -i index.html -o coverage.png
bb-screenshot -S selenium -i index.html -o coverage2.png
```

# Развитие
Планы по улучшению BugBane:
* Поддержка тестирования разных целей в пределах одной сборки
* Поддержка других фаззеров
* Добавление других утилит
* Генерация отчётов в других форматах и по другим шаблонам

# Для разработчиков
Установка в режиме editable в виртуальное окружение:
```
python -m venv .venv
. .venv/bin/activate
pip install -e .[dev]
```

Запуск тестов:
```
pytest
```

Запуск тестов в среде tox (при этом собирается покрытие кода тестами):
```
tox
```

# Благодарности
Спасибо всем участникам проекта!

Отдельные благодарности:
- [Илья Уразбахтин](https://github.com/donyshow): идеи, консультации, менторство.
