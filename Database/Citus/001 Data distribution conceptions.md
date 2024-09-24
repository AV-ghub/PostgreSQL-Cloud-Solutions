# Архитектура распределения данных
> [Choosing Distribution Column](https://docs.citusdata.com/en/stable/sharding/data_modeling.html#choosing-distribution-column)
------------------------
<details>
  <summary><H2>Принципы распределения данных</H2></summary>
  
  <H3>Основные принципы распределения данных при построении баз данных приложения</H3>
  
  <H4>Multi-Tenant Apps</H4>
  <p>
    Расположение всех релевантных данных под одним <b>tenant_id</b>. 
    Связь всех данных в обязательном порядке по ключу шардирования tenant_id.
    Т.е. по сути сведение множества однотипных баз данных заказчиков в одну базу с шардированием их по железу на базе tenant_id.
    Проблема <b>общих данных</b> решается технологией <a href="https://docs.citusdata.com/en/stable/develop/reference_ddl.html#reference-tables">Reference Tables</a>
  </p>
  
  <H4>Real-Time Apps</H4>
  <p>
    Цель подхода - обеспечение <b>мультипроцессинга</b> по максимальному количеству нод с макисмально равномерным рапределением.
    Сбор данных осуществляет координатор, с соотвествующими накладными расходами.
    Как и при рапараллеливании запросов, проблема целесообразности при данной конкретной нагрузке и сопоставимости затрат на сведение результатов с объемом обрабатываемых данных.
    Утверждается, что итоговая эффективность почти прямо пропорциональна количеству участвующих в обработке запроса нод.
  </p>
  <p>
    Ключом шардирования выбирается поле с наивысшей кардинальностью, используемое в:
  <ul>
    <li>группировках</li>
    <li>объединениях</li>
  </ul>
  </p>
  <p>
    Типовые ключи - пользователь, хост, устройство. Опять же предлагается не мучаться попытками разделения таблиц, содержащих измерения. а сделать из них то же самое <a href="https://docs.citusdata.com/en/stable/develop/reference_ddl.html#reference-tables">Reference Tables</a>
  </p>
  <p>
  Для примера - аналитические <a href="https://docs.citusdata.com/en/stable/use_cases/realtime_analytics.html#real-time-dashboards">Real-Time Dashboards</a>
  </p>
  
  <H4>Timeseries Data</H4>
  В этом типе приложения как раз указывается на основную типовую ошибку при проектировании - указание timestamp в качестве ключа.
  Упоминается про hash для рандомного распределения данных.
  Но вместе с тем указывается на то, что основные запросы запрашивают данные по диапазонам, что скорее всего повлечет за собой накладные расходы на транспортировку данных с узлов.
  Здесь рекоментуется использовать связку tenant_id с партиционированием Postgres.
  И практическая рекомендация про построение приложения тут <a href="https://docs.citusdata.com/en/stable/use_cases/timeseries.html#timeseries">Timeseries Data</a>
  <div class="tab">
    <p>
      Старая идея про соспоставление натурального ключа с ключом секционирования <a href="https://stackoverflow.com/questions/401480/converting-guid-to-integer-and-back">Converting GUID to Integer and Back</a>
    </p>
  </div>

  <H3>Table Co-Location</H3>
  <p>
    Основная проблема при масштабировании релялционных баз - при попытке их шардирования мы получаем критический импакт на объединениях.
    Движок БД не предназначен для объединения распределенных данных.
    <b>Co-location</b> как раз является техникой тактического распределения данных, когда все необходимые реляционые данные находятся на одном физическом узле.
    А вот то что подлежит группировке, размазано по разным узлам.
    Т.о. внутри группировок происходит подготовка связанных реляционных данных, а затем объединение готовых результатов как схожих наборов.
  </p>
  <p>
    Каждая нода в Citus представляет из себя отдельную базу.
    Но это расширение фактически добавляет в каческтве надстройки базу-координатор.
    Эта база позволяет реализовать практически похожую отдельной базе Postgres функциональность.
    При этом, если данные по ключу шардирования располагаются на одной ноде, то SQL-функциональность поддерживается в полном виде.
  </p>
  <p>
    Общая техника работы выглядит следующим образом.
    На верхнем уровне работает оптимизатор Citus.
    Он готовит подзапросы к шардам и отправляет их туда.
    На уровне шарды оптимизатор Citus оформляет запрос соответствующим образом и исполняет его уже при помощи оптимизатора Postgres.
    Т.о. все внешние оптимизации для запроса на уровне шарды будут работать.
  </p>
</details>