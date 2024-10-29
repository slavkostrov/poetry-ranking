# poetry-ranking

Проект по курсу "Трансформеры": "Ранжировщик стихов"\
Автор: Костров Вячеслав

## Содержание

- [poetry-ranking](#poetry-ranking)
  - [Содержание](#содержание)
    - [Данные и EDA](#данные-и-eda)
      - [Выводы и мысли по данным](#выводы-и-мысли-по-данным)
      - [Подготовка данных](#подготовка-данных)
    - [Метрика](#метрика)
    - [Варианты моделей](#варианты-моделей)
      - [Вариант 1](#вариант-1)
        - [Преимущества](#преимущества)
        - [Недостатки](#недостатки)
      - [Итоговый вариант](#итоговый-вариант)
    - [Обучение модели](#обучение-модели)

### Данные и EDA

Код построения графиков и сами графики находятся в ноутбуке [1-eda.ipynb](./notebooks/1-eda.ipynb)

#### Наблюдения и мысли по данным

В данных можно увидеть несколько основных колонок:
- __output_text__ - текст произведения/стихотворения;
- __rating__ - рейтинг стихотворения (пока не совсем понятно как он формируется); 
- __views__ - количество просмотров страницы стихотворения;
- __genre__ - жанр стихотворения;
- __url__ - ссылка на страницу стихотворения (в теории можно попробовать добрать оттуда ещё какую-то информацию).

Сэмпл данных:

<img width="520" alt="image" src="https://github.com/user-attachments/assets/8dfbdac2-4bf9-478d-9d27-b2616cfe1fdb">


В первом приближении очевидно, что в качестве входа будет использоваться информация из столбца `output_text`, то есть тексты стихотворений. С целевой переменной сложнее, поскольку это напрямую зависит от того, какой способ решения будет выбран по итогу предварительного анализа - на текущий момент, можно сказать, что на некоторый "идеальный ранг" стихотворения может влиять `rating`, но также может влиять `views`, возможно в итоговом варианте имеет смысл ориентироваться на некоторую комбинацию этих переменных (например, по первым ощущениям напрашивается отношение рейтинга к количеству просмотров, но сначала нужно разобраться в природе этих данных и, по возможности, в механике их формирования).

При этом на `__genre__` также стоит обратить внимание, поскольку кажется, что могут быть жанры, которые сильно отличаются от общей массы и скорее могут мешать при обучении, либо же просто какой-то жанр будет сильно преобладать и тогда есть шанс прийти к чему-то похожему на `mode collapse`, когда наша модель сможет корректно ранжировать только преобладающий класс (жанр), а для оставшихся выдавать что-то около случайного или всегда ранжировать низко. Также сразу приходит мысль, которая состоит в том, чтобы использовать жанр как некоторый известный condition, опираясь на который модели было бы легче понимать какие стихотворения в этом жанре "хорошие", а какие нет.

Распределение стихов по жанрам:
![image](https://github.com/user-attachments/assets/262c4203-fe26-404c-a922-fe31b819ee6d)

По графику видно, что преобладают несколько жанров, а именно: лирика, юмор и песни. При этом юмора более чем в 2 раза меньше, чем лирики, аналогично для песен, но уже относительно юмора. Также можно увидеть, что есть жанры, которые могут хуже подходить под решаемую задачу ранжирования стихов - например, песни (рок, поп, рэп и т.д.) по своей структуре могут сильно отличаться от стихотоворений жанра "лирика", также на их рейтинг может влиять музыка, которая обычно также размещается на сайте ([пример](https://www.chitalnya.ru/work/1670945/)). Итого, по этим данным получается сделать следующие выводы:
- Жанров достаточно много, при это абсолютное большинство содержится в некскольких жанрах - в будущем это может привести, например, к тому, что модель будет плохо работать со стихами, которые относятся к "небольшим" жанров, поскольку не научится выделять паттерны, соответствующие "хорошему" стихотворению в конкретном жанре (иначе говоря, у модели может быть сильное смещение (что-то типо mode collapse) под топ жанров) - при планировании обучения и выборе архитектуры нужно это учесть.
- Некоторые из жанров могут в принципе не подходить под решаемую задачу ранжирования стихов (например, песни) - такие объекты стоит аккуратно учитывать при обучении или в целом исключить их уже на первых этапах.

Далее, перейдем к распределению рейтинга стихотворений:
![image](https://github.com/user-attachments/assets/d91bc823-5fb3-4513-9f0a-480969e6a48d)

Можно увидеть, что:
- бОльшая часть объектов имеет околонулевой рейтинг;
- имеются объекты с рейтингом до примерно 800.

Более наглядной является следующая таблица:

<img width="132" alt="image" src="https://github.com/user-attachments/assets/18d6bb27-daee-40dd-9a71-3e8203cddf00">

То есть 56% объектов имеют нулевой рейтинг, из этого напрашивается **вывод** о том, что в чистом виде использовать данную переменную как целевую (например, для регрессии) скорее будет плохой идей.


### Метрика

### Варианты моделей

#### Вариант 1
##### Преимущества
##### Недостатки

#### Итоговый вариант

### Обучение модели

