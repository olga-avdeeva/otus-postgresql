* Создала инстанс Google Cloud Engine типа n1-standard-1 с ОС Ubuntu 18.04
* установила PostgreSQL 11
* установила sysbench  

      curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash  
      sudo apt -y install sysbench
  
* скачала скрипты sysbench-tpcc
* подготовила набор данных для тестирования под нагрузкой  

        ./tpcc.lua --pgsql-user=postgres --pgsql-password=test --pgsql-db=sysbanch_test --time=120 --threads=1 --report-interval=1 --tables=10 --scale=10 --db-driver=pgsql prepare
  
* подготовила настройки postgresql нацеленные на максимальную производительность:  

      shared_buffers = 1GB  
      work_mem = 2621kB  
      random_page_cost = 1.1  
      checkpoint_completion_target = 0.9  
      min_wal_size = 2GB  
      max_wal_size = 8GB  
      effective_cache_size = 3GB  
      maintenance_work_mem = 256MB  
      wal_buffers = 16MB  
      effective_io_concurrency = 200  
      
      #Параметры, отключение которых повышает производительность ценой риска потери данных при сбоях  
      synchronous_commit = 'off'  
      full_page_writes = 'off'  
      fsync = 'off'  
  > за основу я взяла настройки, рекомендуемые pgtune  
  >
  > параметр **shared_buffers** определяет колиество памяти, используемое для буферов в разделяемой памяти. Документация рекомендует устанавливать в размере 25% от общего количества памяти, что в данном случае равняется 1Gb  
  >
  > параметр **work_mem** определяет количество памяти, которое будет использоваться каждым запросом для внутренних операций сортировки и хэш-таблиц. Оставила этот параметр таким, как рекомендовал pgtune.  
  >
  > параметр **random_page_cost** задаёт приблизительную стоимость чтения одной произвольной страницы с диска и по умолчанию равен 4. Значение по умолчанию имеет смысл оставлять для hdd дисков, для которых операции произвольного доступа дороже последовательного. Т.к. у моей vm ssd диск, можно уменьшить это значение, поэтому я оставила значение, рекомендованное pgtune.  
  >
  > параметр **checkpoint_completion_target** задает целевое время для завершения процедуры контрольной точки, как коэффициент для общего времени между контрольными точками. Значение 0.9 выбрано для более равномерного распределения нагрузки.  
  >
  > параметр **max_wal_size** задает максимальный размер wal-файла. Т.к. контрольная точка происходит либо по достижении установленного периода, либо при достижении wal-файлов заданного размера имеет смысл установить это значение достаточно большим чтобы контрольные точки под нагрузкой не выполнялись слишком часто.  
  >
  > парамет **effective_cache_size** задает эффективный размер дискового кэша, доступного для одного запроса. Параметр влияет исключительно на оценку стоимости использования индекса и не окажет влияния на тестирование, оставила его значение, рекоменованное pgtune.  
  >
  > параметр **maintenance_work_mem** задаёт максимальный объём памяти для операций обслуживания БД. Это значение важно при подготовке тестового набора данных, но не окажет влияния на тестирование нагрузки. Оставила значение рекомендованное pgtune.
  >
  > параметр **wal_buffers** задает объём разделяемой памяти, который будет использоваться для буферизации данных WAL, ещё не записанных на диск. Установлено максимально возможным, равным размеру одного сегмента WAL 16MB.
  >
  > парметр **effective_io_concurrency** задаёт допустимое число параллельных операций ввода/вывода, которые могут быть выполнены одновременно. Для дисков ssd документация рекомендует устанавливать значение этого параметра равным нескольким сотням. Оставила значение рекомендованное pgtune.
  >
  > параметр **synchronous_commit** определяет будет ли сервер при фиксации транзакции ждать записи на диск, выключение синхронной фиксации транзакций повышает производительность, но появляется вероятность потери данных в случае сбоя.
  >
  > параметр **fsync** отвечает за синхронную запись данных на диск физически. Если этот парамет отключен, имеет смысл отключить параметр **full_page_writes**.
  >
  > когда включён параметр **full_page_writes**, сервер PostgreSQL записывает в WAL всё содержимое каждой страницы при первом изменении этой страницы после контрольной точки.   
  
* Запустила нагрузочное тестирование через sysbench-tpcc для параметров по-умолчанию и своего набора параметров. Установила тестирование на 1 час, 1 потока, для 10 наборов по 10 таблиц

      ./tpcc.lua --pgsql-user=postgres --pgsql-db=sysbanch_test --time=3600 --threads=1 --report-interval=60 --tables=10 --scale=10 --use_fk=0  --trx_level=RC --pgsql-password=test --db-driver=pgsql run  
      
  **Результаты с настройками по-умолчанию**
  
        SQL statistics:
         queries performed:
             read:                            2991850
             write:                           3105289
             other:                           461448
             total:                           6558587
         transactions:                        230723 (64.09 per sec.)
         queries:                             6558587 (1821.82 per sec.)
         ignored errors:                      1036   (0.29 per sec.)
         reconnects:                          0      (0.00 per sec.)
     
       General statistics:
           total time:                          3600.0258s
           total number of events:              230723
       
       Latency (ms):
                min:                                    0.72
                avg:                                   15.60
                max:                                  325.45
                95th percentile:                       38.94
                sum:                              3599231.14
       
       Threads fairness:
           events (avg/stddev):           230723.0000/0.00
           execution time (avg/stddev):   3599.2311/0.00

  **Результаты с измененными настройками**
  
      SQL statistics:
          queries performed:
              read:                            3305455
              write:                           3429458
              other:                           510036
              total:                           7244949
          transactions:                        255017 (70.84 per sec.)
          queries:                             7244949 (2012.47 per sec.)
          ignored errors:                      1037   (0.29 per sec.)
          reconnects:                          0      (0.00 per sec.)
      
      General statistics:
          total time:                          3600.0230s
          total number of events:              255017
      
      Latency (ms):
               min:                                    0.68
               avg:                                   14.11
               max:                                  176.29
               95th percentile:                       33.12
               sum:                              3599041.57
      
      Threads fairness:
          events (avg/stddev):           255017.0000/0.00
          execution time (avg/stddev):   3599.0416/0.00

* Т.к. выигрыш в производительности оказался незначительным, я дополнительно провела тест с помощью pgbench

      pgbench -c1 -P 60 -T 3600 -U postgres postgres
      
  **Результаты с настройками по-умолчанию**
  
      transaction type: <builtin: TPC-B (sort of)>
      scaling factor: 1
      query mode: simple
      number of clients: 1
      number of threads: 1
      duration: 3600 s
      number of transactions actually processed: 2990587
      latency average = 1.204 ms
      latency stddev = 0.175 ms
      tps = 830.718327 (including connections establishing)
      tps = 830.719611 (excluding connections establishing)

  
  **Результаты с измененными настройками**
  
      transaction type: <builtin: TPC-B (sort of)>
      scaling factor: 1
      query mode: simple
      number of clients: 1
      number of threads: 1
      duration: 3600 s
      number of transactions actually processed: 8322113
      latency average = 0.433 ms
      latency stddev = 0.123 ms
      tps = 2311.697643 (including connections establishing)
      tps = 2311.703074 (excluding connections establishing)

**Выводы:** при тесте с помощью pgbench разница в производительности значительно заметней. Считаю что оптимальные настройки необходимо подбирать индивидуально, учитывая аппаратный комплекс, объем бд и нагрузку на нее.
