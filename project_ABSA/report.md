## Структура проекта

- [https://github.com/thddbptnsndshs/nlp_homeworks/blob/main/project_ABSA/ner%2Babsa.ipynb](ner+absa.ipynb): эксперименты с обучением моделей, анализ ошибок
- [https://github.com/thddbptnsndshs/nlp_homeworks/blob/main/project_ABSA/train_on_whole_dataset.ipynb](train_on_whole_dataset.ipynb) - создание второй версии модели, обученной на train+dev
- [https://github.com/thddbptnsndshs/nlp_homeworks/blob/main/project_ABSA/evaluation.ipynb](evaluation.ipynb) - тетрадка для подсчета метрик на тестовых данных

## Инструкция по запуску на тестовых данных

Для подсчета метрик на тестовых данных нужно запустить тетрадку [evaluation.ipynb](https://github.com/thddbptnsndshs/nlp_homeworks/blob/main/project_ABSA/evaluation.ipynb), поменяв в ячейке "Пути к файлам" названия файлов на необходимые.

## Описание корпусов и моделей 

Для обучения моделей использовался корпус [train_split_aspects](https://github.com/named-entity/hse-nlp/blob/master/4th_year/Project/train_split_aspects.txt), для валидации - [dev_aspects](https://github.com/named-entity/hse-nlp/blob/master/4th_year/Project/dev_aspects.txt). Дополнительные корпуса и разметка не использовались.

Модель, которую мы обучили под задачу NER - [cointegrated/rubert-tiny2](https://huggingface.co/cointegrated/rubert-tiny2). 

## Методы

Данные корпуса были конвертированы в набор токенов и BIO-тегов. Формат тегов: ```O``` и

```{B|I}-{Whole|Food|Interior|Price|Service}_{positive|negative|neutral|both}```.

Сначала мы заморозили все веса модели, кроме последнего полносвязного слоя, и обучили со следующей конфигурацией гиперпараметров:

| hyperparameter | value |
| ------ | ------ |
| train batch size | 16 |
| learning rate | 2e-4 |
| weight decay | 0.01 |
| epochs | 50 |

Затем разморозили модель полностью, уменьшили learning rate в 2 раза и дообучили на большем количестве эпох:

| hyperparameter | value |
| ------ | ------ |
| train batch size | 16 |
| learning rate | 1e-4 |
| weight decay | 0.01 |
| epochs | 70 |

Обучение всей модели сразу дает более низкий F1-score на валидации (~0.443), чем заморозка и дообучение (~0.490).
Эксперимент с увеличением вероятности дропаута (0.1 → 0.5) ухудшил качество модели, поэтому мы оставили дропаут 0.1.

После NER задачу ABSA мы решили как в бейзлайне, присвоив категориям следующие значения:

-  ```absence``` - если нет упоминаний данной категории
- ```both``` - если есть упоминания с разной тональностью
- ```positive/neutral/negative``` - если все упоминания одной тональности

Модели, которые у нас получились:
[https://huggingface.co/bert-base/rubert-tiny2-ner-absa-v1](rubert-tiny2-ner-absa-v1) 
и [https://huggingface.co/bert-base/rubert-tiny2-ner-absa-v2](rubert-tiny2-ner-absa-v2). 
v1 обучалась только на train-части датасета, а v2 на train+dev на меньшем количестве эпох (30 и 60 вместо 50 и 70), чтобы избежать переобучения.

## Результаты тестирования

Аccuracy по выделению упоминаний с категориями:

| Метрика | Бейзлайн | rubert-tiny2-ner-absa-v1 |
| ------ | ------ | ------ |
| full match precision | 0.480 | 0.653 |
| full match recall | 0.716 | 0.697 |
| full match accuracy | 0.46 | 0.62 |
| partial match ratio | 0.619 | 0.804 |
| partial category accuracy | 0.603 | 0.790 |

Аccuracy по тональности упоминаний:

| Метрика | Бейзлайн | rubert-tiny2-ner-absa-v1 |
| ------ | ------ | ------ |
| full match accuracy | 0.677 | 0.820 |
| partial match accuracy | 0.637 | 0.736 |

Аccuracy по тональности категории:

| Метрика | Бейзлайн | rubert-tiny2-ner-absa-v1 |
| ------ | ------ | ------ |
| overall sentiment accuracy | 0.524 | 0.586 |

## Анализ ошибок

Количественный анализ ошибок показал, во-первых, переобучение: метрики качества на тренировочном датасете гораздо выше, чем на валидационном. Мы думаем, что причина в относительно небольшом размере тренировочного датасета. Модель обучалась всего на 213 отзывах и валидировались на 71, что немного для такой относительно сложной модели, как дистиллированный rubert. 

Качественные ошибки обусловлены, опять же, спецификой модели: во-первых, из-за wordpiece-токенизатора, который некоторые слова разбивает на несколько токенов, выделенные сущности часто содержат обрывки слов, и из-за этого заведомо снижается качество по точным соответствиям. Также ground truth разметка такова, что в самих выделенных аспектах упоминания сентимента не содержится, и для определения тональности модель должна использовать контекст. Трансформер с этим хорошо справляется за счет внимания, но ошибается в деталях, например, вместе с наименованием блюда захватывается сказуемое (“пили вино” вместо “вино”). Грубо говоря, предположим, что поскольку контекст важен, он иногда попадает в выделенную сущность.
