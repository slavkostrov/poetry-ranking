# poetry-ranking

Проект по курсу "Трансформеры": "Ранжировщик стихов"\
Автор: Костров Вячеслав

## Содержание

- [poetry-ranking](#poetry-ranking)
  - [Содержание](#содержание)
    - [Данные и EDA](#данные-и-eda)
      - [Наблюдения и мысли по данным](#наблюдения-и-мысли-по-данным)
    - [Метрика](#метрика)
    - [Варианты моделей](#варианты-моделей)
      - [Вариант 1](#вариант-1)
        - [Преимущества](#преимущества)
        - [Недостатки](#недостатки)
      - [Итоговый вариант](#итоговый-вариант)
    - [Обучение модели](#обучение-модели)

### Данные и EDA

Код построения графиков и сами графики находятся в ноутбуке [1-eda.ipynb](./notebooks/1-eda.ipynb).

#### Наблюдения и мысли по данным

В данных можно увидеть несколько основных колонок:
- __output_text__ - текст произведения/стихотворения;
- __rating__ - рейтинг стихотворения (пока не совсем понятно как он формируется); 
- __views__ - количество просмотров страницы стихотворения;
- __genre__ - жанр стихотворения;
- __url__ - ссылка на страницу стихотворения (в теории можно попробовать добрать оттуда ещё какую-то информацию).


Сэмпл данных:

<img width="520" alt="image" src="https://github.com/user-attachments/assets/8dfbdac2-4bf9-478d-9d27-b2616cfe1fdb">

В первом приближении очевидно, что в качестве входа будет использоваться информация из столбца `output_text`, то есть тексты стихотворений. С целевой переменной сложнее, поскольку это напрямую зависит от того, какой способ решения будет выбран по итогу предварительного анализа - на текущий момент, можно сказать, что на некоторый "идеальный ранг" стихотворения может влиять `rating`, но также может влиять `views`: возможно в итоговом варианте имеет смысл ориентироваться на некоторую комбинацию этих переменных (например, по первым ощущениям напрашивается отношение рейтинга к количеству просмотров, но сначала нужно разобраться в природе этих данных и, по возможности, в механике их формирования на стороне сайта).

При этом на `genre` также стоит обратить внимание, поскольку кажется, что могут быть жанры, которые сильно отличаются от общей массы и скорее могут мешать при обучении, либо же просто какой-то жанр будет сильно преобладать и тогда есть шанс прийти к чему-то похожему на `mode collapse`, когда наша модель сможет корректно ранжировать только преобладающий класс (жанр), а для оставшихся выдавать что-то около случайного или всегда ранжировать низко. Также сразу приходит мысль, которая состоит в том, чтобы использовать жанр как некоторый известный condition, опираясь на который модели было бы легче понимать какие стихотворения в этом жанре "хорошие", а какие нет.

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

То есть 56% объектов имеют нулевой рейтинг, из этого напрашивается **вывод** о том, что в чистом виде использовать данную переменную как целевую (например, для регрессии) скорее будет плохой идей. Также полезно посмотреть на оставшиеся 44% объектов и то, как распределено значений рейтинга:

![image](https://github.com/user-attachments/assets/da086206-1741-4143-a2c2-cb86a868a3f4)

Из этой картинки, как и из таблицы выше, подозрительно выбивается, например, рейтинг 7, в связи с чем напрашивается вопрос о том, как формируется рейтинг на рассматриваемом сайте. Захотелось разобраться как строится рейтинг - первым делом проверялась гипотеза, что рейтинг является количеством пользовательских "голосов" - оказалось нет, поскольку есть стихотворения с рейтингом бОльшим, чем количество просмотров. Далее удалось найти [обсуждение на сайте](https://www.chitalnya.ru/commentary/23042/), где автор объясняет, что в формировании рейтинга "задействовано много факторов, в том числе и качество отзыва, и авторитет автора, который оставляет отзыв" и эта скрытая формула обновляется ежегодно. Это скорее всего является проблемой, поскольку а) высокий рейтинг может объясняться большим количеством просмотров б) низкий рейтинг может объясняться маленьким количеством просмотрв, а не тем, что стих плохой.

С учетом полученной (и в то же время ограниченной) информации о том, как формируется рейтинг, появилось ощущение, что количество просмотров уже должно быть включено в рейтинг, при этом на количество просмотров также интересно посмотреть:

![image](https://github.com/user-attachments/assets/ab2dbdb8-c194-4d7b-8136-303beba5d934)

В целом видно, что очень много стихов с околонулевым значением просмотров, насколько "адекватный" рейтинг у таких объектов - вопрос открытые. Зависимость количества просмотров и рейтинга:

![image](https://github.com/user-attachments/assets/af07fe2e-eae0-4cc9-a796-951f0eb06233)

Весь код, все графики и некоторые другие представлены в ноутбуке [1-eda.ipynb](./notebooks/1-eda.ipynb).

### Метрика

Перед решением задачи необходимо зафиксировать набор метрик (или метрику). В целом, кажется, что стандартные метрики ранжирования в определенной конфигурации могут подойти - например, этой стандартной метрикой может быть NDCG. Более важным возможно является то, в каких разрезах и на каких объектах считать метрику, поскольку:
- a) много нулевых рейтингов;
- б) много разных жанров с потенциально разной структурой стихотворений.

Например, вариантом может быть алгоритм генерации выборок определенного размера, которые бы в равной мере содержали нулевые (или околонулевые рейтинги), "средние" рейтинги и "высокие" рейтинги.

Также стоит вспомнить о том, что у нас есть данные по просмотрам - в таком случае более правильным кажется вариант, когда мы разбиваем просмотры на некоторые бины и выборки для замера качества формируем уже внутри этих бинов, чтобы попробовать не попасть в ситуацию, когда у одного стихотворения рейтинг выше лишь из-за того, что больше просмотров (напомню, что точная механика формирования рейтинга неизвестна). Если выкидывать жанры (например, песни) нельзя, то имеет смысл замеряться внутри жанров или внутри групп жанров (понадобится дополнительная "околоручная" разметка), которые должны быть похожи между собой.

Помимо этого, список метрик может быть расширен, например, метриками регрессии (MSE/MAE), чтобы оценивать качество попадания в rating (или другой вариант комбинации rating и views) - правда по первичному анализу сложилось впечатление, что решение через регрессию может быть плохим вариантом.

Также есть опция собрать некоторый эталонный список классических произведений и использовать его как некоторые референс, то есть при таком подходе хотим проверять, что эталонные произведения ранжируются выше, чем "средние" и ниже по рейтингу стихотворения из выборки.

Итак, для расчета метрик(и) предалагается:
- формировать выборки с использованием информации о жанре и количестве просмотров;
- использовать метрики, которые будут основываться на стандартных метриках ранжирования (например, ap@k, ndcg);
- использовать набор эталонных произведений для проверки того, что в среднем модель ранжирует их выше;
- расширять список метрик, в зависимости от предложенного варианта решения задачи (например, если будем рещать через регрессию).

Выбор одной конкретной метрики на этапе предварительного анализа показался сложным и неочевидным, предполагается, что это может стать ясней в рамках экспериментов и тестирования разных метрик на стабильность, адекватность и так далее.

### Токенезация

Кажется, что в случае, когда работаем со стихотворениями, этап токенезации очень важен (для случая когда мы хотим обучать модель с нуля). Например, напрашивается учёт либо же какая-то дополнительная обработка переносов строк, разделений между частями стихотворения. Стандартные предобученные трансфоремы возможно лучше "вытаскиваю" именно семантику, при этом меньше обращяя внимания на структуру. В общем и целом даже при выборе предобученной модели стоит изучить как устроен токенизатор, поскольку в этой задаче это может быть более важным.

### Варианты моделей

Далее предалагается рассмотреть некоторые варианты обучения моделей и по возможности выделить преимущества и недостатки каждой из моделей. Пока что кажется, что общей идеей для всех вариантов может быть двухуровневая модель, то есть:
- "верхняя" модель будет распределять по группа, например: топ рейтинга, середина рейтинга, низ рейтинга. При этом "бинов" может быть и больше, чем 3;
- "нижняя" модель будет ранжировать стихи внутри выделенной на первой этапе группе (в теории это может быть и несколько моделей для каждой из групп).

В целом в зависимости от сценария, в котором в итоге нужно будет применять модель, может быть достаточным "верхняя" модель, которая будет выделять "хорошие" стихи безотносительно их порядкового отношения между собой.

Также во всех вариантах предполагается, что могут быть использованы идеи из прошлых разделов (например, можно использовать информацию о просмотрах, взвешивая объекты в соответствии с ними).

#### Вариант 1. Решение через задачу регрессии

Учим предсказывать рейтинг, или некоторую трансформацию рейтинга (возможно с "подмешиванием" количества просмотров). Берём предобученный энкодер, прогоняем стихотворение через него, получаем эмбединг и дальше навешиваем fully-connected сеть, которая будет выдавать на выходе одно число - по сути рейтинг или же score, по которому уже можно будет ранжировать стихотворения.

Поскольку у рейтинга большой разброс, мне показалось, что правильней в качестве таргета брать отношение рейтинга к просмотрам. При этом такой подход все равно не спасет от большого количества нулей - тут предлагается попробовать добавить в `dataloader` `WeightedSampler`, который будет с меньшей вероятность брать в батч нулевые таргеты.

##### Преимущества
- "простое" и понятное обучение с точки зрения того, что хотим предсказывать;
- потенциально быстрый инференс;
- универсальность, поскольку будем иметь скор, отражающий качество стихотворения.

##### Недостатки

- у нас много нулей;
- при наивном обучении (например, без учета количества просмотров) можем свалиться во что-то неправильное.

В целом, в качестве относительно бейзлайна, кажется, что можно попробовать.

#### Вариант 2. Обучение с triplet loss

В этом подходе предлагается формировать тройки:
- anchor - некоторое стихотворение со средним показателем рейтинга (или отношения рейтинга к просмотрам);
- negative - объект с околонулевым рейтингом (но с "нормальным" количеством просмотров);
- positive - объект с хорошим рейтингом.

При таком подходе модель будет выдавать на выходе не скор, а вектор, далее можно будет считать близость этого вектора к некоторому эталонному вектору (например взять средний вектор по стихам с высоким рейтингам или даже попробовать взять что-то из литературы) и на основе этого ранжировать.

##### Преимущества
- уходим от необходимости закладывать rating в таргет модели, оставляя его использовании лишь в формировании троек;

##### Недостатки 
- менее очевидный инференс, поскольку он зависит от того, как мы выберем эталонные векторы стихов;
- формирование троек также непростая задача.

#### Вариант 3. Обучение cross-endoder

#### Доп. улучшения

Список доп. идей, которые приходили во время работы над задачей:
- попробовать более явно научить attention обращать внимания на рифмы или другие структурные особенности стихотворений - тут сходу приходит в голову что-то наподобии dilated window attention;
- ...

#### Итоговый вариант

TBD

