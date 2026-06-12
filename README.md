Описание проекта ищите в уроке на платформе

Проект 1.

Описание.
Маркетплейс товаров ручной работы набирает популярность и расширяет свою аудиторию. Чтобы привлечь больше покупателей, приобрели ещё один сайт, который предоставляет похожие услуги. 

Формальная часть сделки завершена, нужно интегрировать данные купленного сайта в хранилище.
Добавление нового источника — тот же процесс, что и загрузка предыдущих источников: нужно интегрировать данные и построить новую витрину. 

Схема подключения, таблицы и модель данных:
В схеме dwh — таблицы измерений для мастеров ручной работы (d_craftsman), заказчиков (d_customer) и товаров (d_product). Также там есть таблица фактов о продажах (f_order), инкрементальная витрина по мастерам за отчётные периоды (craftsman_report_datamart) и MATERIALIZED VIEW с отчётом продаж по всему маркетплейсу (orders_report_materialized_view).
В схеме source1 — широкая таблица первого источника craft_market_wide.
В схеме source2 — две ненормализованные таблицы: таблица мастеров и товаров (craft_market_masters_products) и таблица заказчиков и заказов (craft_market_orders_customers).
В схеме source3 — три таблицы: мастеров (craft_market_craftsmans), заказчиков (craft_market_customers), заказов с товарами (craft_market_orders).
Схема external_source — новая. Там как раз находятся таблицы нового источника, которые нужно подключить к хранилищу.


Шаг 1. Подключение к базе - выполнено.

Шаг 2. Изучение данных нового источника.
Создан файл DDL_craft_market_external_source
В craft_products_orders данные по мастерам, товарам и заказам 
В customers по заказчикам.

Шаг 3. Напишите скрипт переноса данных из источника в хранилище.

Раскладываем данные в dwh по измерениям. 
Источник денормализован: в craft_products_orders в одной строке лежат сразу мастер, товар и заказ, а заказчик вынесен в отдельную customers. В хранилище же действует звезда — три измерения и таблица фактов. Поэтому каждую колонку источника надо отнести к своей сущности.

Раскладка по измерениям и фактам:
dwh.d_craftsman (мастера) — из craft_products_orders:
craftsman_name, craftsman_address, craftsman_birthday, craftsman_email. (Бизнес-ключ источника craftsman_id в измерение не переносится — в dwh у мастера свой суррогатный craftsman_id; уникальность определяется по craftsman_name + craftsman_email.)
dwh.d_product (товары) — из craft_products_orders:
product_name, product_description, product_type, product_price.
dwh.d_customer (заказчики) — из customers:
customer_name, customer_address, customer_birthday, customer_email.
dwh.f_order (факты-заказы) — из craft_products_orders:
order_created_date, order_completion_date, order_status, плюс ссылки на три измерения (product_id, craftsman_id, customer_id), которые проставляются не из источника, а подстановкой суррогатных ключей dwh при джойне с измерениями.
Связующее звено — customer_id: он соединяет craft_products_orders с customers, чтобы к каждому заказу подтянуть данные заказчика.
Что делает скрипт, реализуя эту раскладку: сперва собирает обе таблицы источника в одну плоскую tmp_sources (через JOIN по customer_id) — в том же формате, что и старые источники. Затем MERGE'ами «разносит» её содержимое: уникальные мастера → d_craftsman, товары → d_product, заказчики → d_customer (новые добавляются, существующие обновляются). После этого собирает tmp_sources_fact, где текстовые атрибуты заменяются на суррогатные ключи измерений, и MERGE'ом грузит заказы в f_order. То есть это DML-скрипт загрузки (ELT-перекладка из источника в нормализованное хранилище), а не разовая выгрузка — он рассчитан на регулярный повторный запуск.

Файл SQL_craft_market_loading_to_dwh.sql

Проверяем обновились ли данные:
SELECT 'd_customer'  AS tbl, count(*) FROM dwh.d_customer  WHERE load_dttm::date = current_date
UNION ALL SELECT 'd_craftsman', count(*) FROM dwh.d_craftsman WHERE load_dttm::date = current_date
UNION ALL SELECT 'd_product',   count(*) FROM dwh.d_product   WHERE load_dttm::date = current_date
UNION ALL SELECT 'f_order',     count(*) FROM dwh.f_order     WHERE load_dttm::date = current_date;

Шаг 4. Изучите потребности бизнеса в новой витрине - изучено.

Шаг 5. Напишите DDL новой витрины.
DDL_craft_market_customer_report_datamart.sql: та же структура, что у craftsman_report_datamart, плюс таблица дат загрузок load_dates_customer_report_datamart. В разрезе заказчика: поля customer_*, customer_money (потрачено заказчиком), platform_money (10%), count_order, avg_price_order, median_time_order_completed, top_product_category, и новый top_craftsman_id (любимый мастер) вместо avg_age_customer.


Шаг 6. Напишите скрипт для инкрементального обновления витрины:
SQL_craft_market_customer_datamart_increment.sql: та же CTE-цепочка, что в эталоне — dwh_delta → dwh_update_delta → dwh_delta_insert_result → dwh_delta_update_result → insert_delta / update_delta / insert_load_date → финальный SELECT 'increment datamart'.
Что адаптировано под заказчика и подсказку:

группировка по customer_id (и report_period);
top_craftsman_id добавлен отдельным INNER JOIN через ROW_NUMBER() — самый частый мастер по числу заказов (как и советует подсказка про ROW_NUMBER/DISTINCT ON); top_product_category сделал тем же приёмом;
customer_money = SUM(product_price), platform_money = SUM(product_price)*0.1.

Вывод по проекту
В рамках спринта в хранилище был интегрирован новый источник данных — купленный сайт (схема external_source с таблицами craft_products_orders и customers). Источник денормализован: мастер, товар и заказ лежат в одной широкой таблице, а заказчики вынесены отдельно и связаны по customer_id. Эти данные были разложены по модели «звезда» в схему dwh: атрибуты мастеров, товаров и заказчиков попали в измерения d_craftsman, d_product, d_customer, а сами заказы — в таблицу фактов f_order с подстановкой суррогатных ключей.
Скрипт загрузки SQL_craft_market_loading_to_dwh.sql дополнен четвёртым блоком UNION для нового источника по тому же принципу, что и source1–source3: сборка плоской tmp_sources, наполнение измерений через MERGE, формирование фактов и их загрузка в f_order. Загрузка сделана идемпотентной и рассчитана на регулярный повторный запуск, пока старый сайт ещё работает до завершения ребрендинга. По ходу был устранён сбой MERGE (multi-match по неключевым полям) — источник схлопывается до одной строки на ключ через DISTINCT ON.
По аналогии с витриной по мастерам построена новая инкрементальная витрина dwh.customer_report_datamart (шаг 5 — DDL с типами, комментариями и служебной таблицей дат загрузок) и скрипт её инкрементального наполнения (шаг 6) на той же CTE-логике: дельта по load_dttm → вставка новых строк → пересчёт существующих → фиксация даты загрузки. Витрина даёт по каждому заказчику за отчётный месяц сумму трат, комиссию платформы, число и среднюю стоимость заказов, медианное время выполнения, любимую категорию товаров и любимого мастера (top_craftsman_id через ROW_NUMBER), а также разбивку заказов по статусам.
Практический результат: данные нового источника теперь автоматически попадают в единое хранилище, а витрина по заказчикам готова к использованию для программы лояльности, анализа интересов аудитории, рекомендаций товаров и планирования маркетинговых акций.

