# План демонстрации на защите проектной работы

1. Показать развёрнутые ВМ в Яндекс-Облаке, показать интерфейс мониторинга.
2. Подключиться к БД - к мастеру и реплике, выполнить изменение данных на мастере, показать, что изменения появились на реплике. Показать, как это отразилось в мониторинге.
3. Показать управление кластером с помощью patronictl: просмотр информации о кластере, switchover
4. Показать просмотр информации с помощью запросов к REST Api
5. Показать добавление новой реплики в кластер.
6. Показать failover: завершить процесс patroni на мастере сигналом SIGKILL, потом вновь его запустить. Показать, что в логах и мониторинге, как отработал HAProxy.