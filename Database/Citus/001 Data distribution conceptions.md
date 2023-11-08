# Архитектура распределения данных
> [Choosing Distribution Column](https://docs.citusdata.com/en/stable/sharding/data_modeling.html#choosing-distribution-column)
------------------------
<details>
  <summary><H2>Принципы распределения данных</H2></summary>
   Существуют следующие основные принципы распределения данных при построении баз данных приложения.
  <H3>Multi-Tenant Apps</H3>
  Расположение всех релевантных данных под одним tenant_id. 
  Связь всех данных в обязательном порядке по ключу шардирования tenant_id.
  Т.е. по сути сведение множества однотипных баз данных заказчиков в одну базу с шардированием их по железу на базе tenant_id.
  Проблема общих данных решается технологией <a href="https://docs.citusdata.com/en/stable/develop/reference_ddl.html#reference-tables">Reference Tables</a>
  <H3>Real-Time Apps</H3>
  <H3>Timeseries Data</H3>
</details>
