## Оптическое распознавание информации в медицинских справках
Данное OCR-решение разработано для распознавания данных с фото медицинских справок для последующей интеграции в систему [проекта по поиску доноров крови](https://donorsearch.org/).
Было необходимо успешно получить следующую информацию:
* Дата донации
* Факт оплаты донации
* Класс крови донации

---

### Итог работы

Готовый микросервис для распознавания нужной информации, отлично обрабатывающий большинство фотографий, кроме достаточно "размытых". Работает на основе библиотеки [docTR](https://mindee.github.io/doctr/index.html) и с применением постобработки с помощью регулярных выражений. Обработка задач на распознавание реализована асинхронно с использованием [Celery](https://github.com/celery/celery).

Качество решения в рамках метрики *accuracy* по отдельным ячейкам составляет **0.91**.

Качество классификаций по формам справок и по их ориентациям составляет **0.93** для метрики *F-1*.

Описания работы и тестирования, в том числе с интерактивными графиками, можно увидеть в Jupiter-тетрадках на nbviewer:

[OCR](https://nbviewer.org/github/Darveivoldavara/ocr_for_medical_certificates/blob/6714f0dfc44e341a01c33353a9cf2db2719aa032/Doctr/ocr_for_medical_certificates.ipynb)

[Классификация типа справки](https://nbviewer.org/github/Darveivoldavara/ocr_for_medical_certificates/blob/712c639362d1ced47b364f3e92a98b8f0d621017/notebooks/certificates_classifier.ipynb)

[Классификация ориентации изображения](https://nbviewer.org/github/Darveivoldavara/ocr_for_medical_certificates/blob/a5cc34726f549cf390668915e027d6213cbbab3e/notebooks/orientation_classifier.ipynb)

---

### Загрузка сервиса (Docker)

Найти docker-контейнеры для микросервиса можно [здесь](https://hub.docker.com/r/darveivoldavara/ocr_for_medical_certificates).

Скачать:

`docker pull darveivoldavara/ocr_for_medical_certificates:web`

`docker pull darveivoldavara/ocr_for_medical_certificates:redis`

`docker pull darveivoldavara/ocr_for_medical_certificates:worker`

Запустить:

[Compose-файл для самостоятельной сборки](https://github.com/Darveivoldavara/ocr_for_medical_certificates/blob/main/docker-compose.yml).

---

### Клонирование репозитория

Если потребуется самостоятельно собрать / запустить образы (с помощью `docker-compose build` / `docker-compose up`), то при клонировании репозитория необходимо использовать [Git LFS](https://git-lfs.com/) для корректной загрузки моделей. Сначала установить:

[Инструкции по установке](https://github.com/git-lfs/git-lfs#installing)

Далее однократно настроить Git LFS для своей учетной записи: `git lfs install`

После чего можно клонировать репозиторий, как обычно:

`git clone https://github.com/Darveivoldavara/ocr_for_medical_certificates.git`

---

### Handlers (ручки запросов)

- Для отправки изображения в очередь (post-запрос) — `/upload` — подгружает фотографию в сервис и возвращает task_id для использования в get-запросе
  - Пример отправки в сервис вашего конкретного изображения, находящегося по пути *path_to_your_image.jpg* —

    `curl -X POST -F "file=@path_to_your_image.jpg" http://localhost:8000/upload`
- Для получения результата распознавания (get-запрос) — `/result/{task_id}` — возвращает результат в формате JSON

---

### Обновления проекта

| Дата | Значение accuracy | Изменение | 
| :---------------------- | :---------------------- | :---------------------- |
| **04.07.2023** | **0.86** | **изначальный показатель** |
| **08.07.2023** | **0.872** | **заполнение пропусков наиболее частыми соответствующими значениями** |
| **10.07.2023** | **0.879** | **подбор наилучших моделей детекции и распознавания текста** |
| **13.07.2023** | **0.904** | **адаптация скрипта к результатам работы новых моделей** |
| **16.07.2023** | **0.91** | **финальный дебаг скрипта** |
| **17.08.2023** | **0.91** | **реализована асинхронная работа сервиса** |
| **04.09.2023** | **0.91** | **добавлена частичная классификация справок на входе** |
| **08.11.2023** | **0.91** | **оптимизирована файловая структура и сборка контейнера; дебаг обработки некоторых изображений** |
| **21.12.2023** | **0.91** | **внедрена обработка повёрнутых изображений** |

---

### Не сработало

Другие изначально рассмотренные библиотеки для распознавания текста и таблиц, такие как **Tesseract**, **EasyOCR** и **Paddle OCR** давали ощутимо более низкое качество, чем использование для этих целей предобученных нейронных сетей, представленных в **docTR**.
Также были протестированы различные методы по предобработке входящих изображений для улучшения их чёткости, и как следствие, повышения качества распознавания символов. Но все рассмотренные способы в итоге давали максимум визуальное улучшение и повышение чёткости, резкости фото, но многие символы в целом видоизменялись на некорректные, и итоговое качество их распознавания только падало по сравнию с решением без какой-либо предобработки вообще. Протестированы были следующие методы:
* Библиотека `cv2` (методы для [adaptiveThreshold()](https://docs.opencv.org/3.4.0/d7/d1b/group__imgproc__misc.html#ga72b913f352e4a1b1b397736707afcde3) с различными настройками: **ADAPTIVE_THRESH_MEAN_C** и **ADAPTIVE_THRESH_GAUSSIAN_C**) для усиления границ элементов изображения
* Библиотека `PIL` (фильтр **SHARPEN** для [ImageFilter](https://pillow.readthedocs.io/en/stable/reference/ImageFilter.html)) для повышения резкости фото
* Решения, использующие нейронные сети для апскейла и/или улучшения изображения:
  * API [Waifu2x](https://deepai.org/machine-learning-model/waifu2x)
  * Проект [Real-ESRGAN](https://github.com/xinntao/Real-ESRGAN) (дефолтная модель **RealESRGAN_x4plus** с различными настройками)

---

### Не успели

* Поробовать интегрировать в используемый метод библиотеки docTR какие-то другие модели для детекции и распознавания, помимо уже встроенных (например, с [HuggingFace](https://huggingface.co/))
* Отсмотреть возможные улучшения качества работы скрипта на абсолютно всех тестовых примерах (включая обработанные с сильно высоким показателем *accuracy*)
* Протестировать иные методы для предобработки изображений

---

### Статус проекта

В планах реализация обработки дополнительных типов справок с соответствующим классификатором.
