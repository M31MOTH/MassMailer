# MassMailer

MassMailer выполнен в виде классической Windows службы и предназначен для отправки больших объемов писем по указанному шаблону.
> Данный проект является частью внутреннего сервиса рассылок известной организации и может содержать в себе жесткие связи с другими узлами.

## Основные возможности

- Поддержка HTML шаблонов писем с использованием синтаксиса С# (Razor).
- Поддержка шаблонов темы письма.
- Возможность отправки писем по приоритетам.
- Различные DKIM подписи для различных адресов отправителя.
- Отправка основных показателей жизнедеятельности в Zabbix.
- Возможность запускать параллельно множество копий данной службы.
- Возможность отправки писем с вложениями.

## Структура

Система рассылки состоит из 2-х основным частей: БД для хранения заданий на рассылку, шаблонов, данных, истории, DB API в виде набора хранимых процедур и сервисной части, выполняющей отправку, DKIM подпись писем и запись результатов.

### База данных

Основной таблицей БД является таблица ActiveQueue – в ней содержатся письма, ожидающие отправки, а также недавно отправленные письма. Данные из этой таблицы архивируются каждые сутки процедурой RobotArchivator в таблицу Archive. Таблица ActiveQueue допускает одновременное обращение из нескольких потоков для получения порции данных через процедуру GetBatchFromQueue.
Для массовых рассылок (маркетинговых акций или прочих оповещений) есть таблицы Mailing – содержит рассылки, Mission – содержит задания на отправку в рамках рассылок, Sets – группы клиентов для рассылки.
> Предполагается, что массовые рассылки отправляются клиентам компании, использующей службу. Клиенты должны находиться в отдельной БД (в плоском виде). Данная связка не конфигурируется и должна быть переписана.

### Служба

Главный рабочий процесс службы выполнен в виде DataFlow конвейера и содержит в себе 4 основных блока: блок подготовки данных для шаблона _parseXmlDataBlock, блок подготовки и отправки письма _sendEmailsBlock, блок накопления результатов _batchResultBlock и блок записи _writeResultsBlock. Количество копий каждого блока (степень параллелизма) настраиваются в конфигурационной файле и подбираются опытным путем под конкретную систему.
> Так как самое узкое место системы – отправка почты, рекомендуется использовать настройки 0,5х; 2х;1;1 для указанных блоков (где «х» - количество процессорных ядер).

Каждый из блоков оперирует порциями данных – batch – размер которых также доступен в настройках.
Данные, для подстановки в письмо – модель – получается из БД в виде обычной XML структуры и в первом блоке преобразуется в dynamic объект. Затем, во втором блоке, текст шаблона подгружается из БД или берется из внутреннего кэша, если он там присутствует. Далее рендерится тело письма, а затем и заголовок если это необходимо. Если в письме присутствуют вложения, оно загружается из БД или из кэша. Далее генерируется готовое письмо, из кэша доступных DKIM подписей выбирается нужная и письмо отправляется. Результаты работы данного блока записываются во временную таблицу и в статические счетчики успехов/ошибок. Временные таблицы аккумулируются в 3 блоке, а затем сливаются и записываются в БД в 4 блоке.

## Стресс-тесты

Производительность системы зависит от ряда параметров:
- Количество ядер серверной машины.
- Ширина internet канала.
- Производительность БД.
- Настройки степени параллелизма службы.
- Количество копий службы.
- Размер письма.

Если допустить, что smtp-шлюз, используемый для отправки не ограничен по пропускной способности и ширина канала до него порядка 300 Мб/сек, то можно добиться производительности в 300-350 тыс. писем в час при следующих параметрах:
- Сервер Xeon E5-2695 (8 ядер).
- 1 копия службы.
- Настройки параллелизма 16;32;1;1
- Размер письма – 49 Кбайт.
