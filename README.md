# Решение задачи X5 Retailhero 2020 №2 (рекомендательная система)
- 9 место
- check NMAP: 0,1467
- public NMAP: 0,1286
- private NMAP: 0,143019 

## Описание задачи
https://retailhero.ai/c/recommender_system/overview

Участникам необходимо разработать сервис, 
который сможет отвечать на запросы с предсказаниями будущих покупок клиента
и при этом держать высокую нагрузку. 
По информации о клиенте и его истории покупок необходимо 
построить ранжированный список товаров, 
которые клиент наиболее вероятно купит в следующей покупке. 

## Описание решения

- Данные по транзакциям случайно разделил по клиентам 
на 20 фолдов по ~20к клиентов в каждой (вначале забыл выкинуть клиентов 
из check и локальная валидация плохо коррелировала с публичным скором), 
15 фолдов использовал для обучения, 5 для валидации
- обучение в 2 уровня: генерация кандидатов айтемов и затем их ранжирование бустингом;
- 1 уровень моделей: 
    - бленд по ранкам спсиков айтемов размерами NUM_1LVL из 2 моделей: 
        - implicit.nn.TFIDFRecommender, обученный на user-item матрице 
        частот покупок нормированых по кол-ву транзакций клиента
        - прошлые айтемы клиента ранжировнные по частоте покупок
    (исключая айтемы, встретившиеся в истории транзакций 1 раз)
    - NUM_1LVL пробовал 50/100/200/400/1000 
    (если не хватало до этого кол-ва, то добивал популярным за февраль),
     лучше всего получалось при NUM_1LVL=200
- 2 уровень - бленд по ранкам айтемов 3 LightGBM binary классификаторов, фичи:
    - признаки продукта категориальные (level_{1,2,3,4}, brand, vendor), 
    и прочие (netto, is_own_trademark)
    - признаки клиента: кол-во транзакций, средняя сумма транзакций, 
    возраст клиента, среднее кол-во продуктов у клиента,
    кол-во дней с последней транзакции, 
    кол-во дней между первой и последней транзакцией, 
    среднее кол-во дней между транзакциями,
    отношение кол-ва дней с последней покупки к среднему кол-ву дней между транзакциями
    - аналогичные признаки по датам, но для каждого продукта в отдельности (
    кол-во дней с последней покупки этого продукта и т.п.)
    - разные частоты по продукту: нормированная частота покупок айтема, 
    взвешенная по времени частота покупок айтема (чем ближе к марту, тем больше вес),
    частота покупок бренда, вендора, категории level3, level4
    - ранг продукта после бленда моделей на 1 уровне
    - word2vec ембединги айтемов (1 клиент (его история транзакций) 
    = 1 документ)
- классификаторы 2 уровня обучал по 2 фолда клиентов на 1 классификатор 
(получалось 7 моделей) и из них выбирал 3 лучших LightGBM на валидации, 
затем проверял на валидации бленд из 3 моделей

- что еще попробовал, но не зашло:
    - другие конфигурации на 1 уровне давали результат хуже, 
    например:
        - не нормированне частоты в user-item матрице
        - использование implicit.nn.CosineRecommender
        - вместо простой частоты покупок айтемов - 
        частота с убывающим весом для давних покупок
        - бленды со списком baseline_items
    - фичи по магазинам (средние сумма покупок, число клиентов)
    - implicit.als 
    - LSTM по транзакциям
    - Ранжирование с помщью LightGBM на 2 уровне (вместо классификации)
     `"objective":"lambdarank"`
    - другое кол-во LightGBM моделей (1, 2, 5, 7)
    - не влияло на скор отдельное рассмторение случая, 
    когда кол-во транзакций в истории = 1 (хотя такие клиенты были), 
    оставил там просто прошлые айтемы клиента ранжированные по частоте покупок

## Что в репозитории
- [data](data): check данные
- [rnd](rnd): ноутбуки с исследованием данных 
и обучением моделей
- [solution](solution): финальное решение запускаемое в докере:
    - [model_artifacts](solution/model_artifacts): обученные модели 
    и файлы для построения фичей
    - [get_recs](solution/get_recs.py): предикт моделей
    - [server](solution/server.py): АПИ предикта на flask
    - [settings](solution/settings.py), [utils](solution/utils.py): 
    вспомогательные константы и методы
    - [metadata](solution/metadata.json): метаданные для проверяющей системы
- [Dockerfile](Dockerfile) для сборки образа на основе __geffy/ds-base:retailhero__
- [run_queries](run_queries.py): для локального тестирования решения 
 
## Локальный запуск решения

- Скачиваем docker image __chesnokovmike/python-ds:retailhero2__ 

```text
docker pull chesnokovmike/python-ds:retailhero2
```
- Переходим в папку решения и запускаем сервер
```text
cd solution
docker run \
    -v `pwd`:/workspace \ 
    -v `realpath ../solution`:/workspace/solution \ 
    -w /workspace \
    -p 8000:8000  \  
    chesnokovmike/python-ds:retailhero2 \    
    gunicorn --bind 0.0.0.0:8000 server:app
``` 
- В другом терминале выполняем проверку на check файле
```text
python run_queries.py http://localhost:8000/recommend data/check_queries.tsv
```
- Получаем check score = __0.146726__