# Алгоритм генерации множества Мандельброта

## Входные параметры:
- Количество потоков $ {\text{nthreads}} $
- Количество точек $ {\text{npoints}} $

1. Задать границы для действительной и мнимой частей:
   - $ \text{real}_\text{min} = -2 $
   - $ \text{real}_\text{max} = 1 $
   - $ \text{img}_\text{min} = -1 $
   - $ \text{img}_\text{max} = 1 $

2. Разделить общее количество точек между потоками:
   - Для каждого потока $ i $ от 0 до $ {\text{nthreads}} - 1 $:
     - Вычислить количество точек для генерации одним потоком: $npoints_i =  {\text{npoints}}\ / \ {\text{nthreads}}$
     - Передать соответствующий диапазон индексов потоку:
       - $ \text{start} = i \times \text{points\_per\_thread} $
       - $ \text{end} = (i + 1) \times \text{points\_per\_thread} $

3. Для каждой точки в диапазоне:
   - Генерировать случайные $ c_{\text{real}} $ и $ c_{\text{img}} $ в заданных границах.
   - Установить начальные значения $ z_{\text{real}} = 0 $ и $ z_{\text{img}} = 0 $.

4. Итеративно проверять, принадлежит ли точка множеству Мандельброта:
   - Для каждой точки выполнять до фиксированного максимального числа шагов $ \text{iterations\_to\_include} $:
     - Вычислять:
       $$
       z_{\text{next}} = z_{\text{real}}^2 - z_{\text{img}}^2 + c_{\text{real}}
       $$
       $$
       z_{\text{img\_next}} = 2 \cdot z_{\text{real}} \cdot z_{\text{img}} + c_{\text{img}}
       $$
     - Проверять условие:
       $$
       z_{\text{real}}^2 + z_{\text{img}}^2 \geq 4 \text{}
       $$

5. Если после всех итераций точка не вышла за границы, записать её в результирующий массив.

6. Повторить шаги 3-5 для всех точек в диапазоне для каждого потока.

7.  Сохранить результат в файл `mandelbrot.csv`.

# Алгоритм для вычисления числа $\pi$ с использованием многопоточности

1. **Инициализация:**
   - Создать глобальную переменную `incircle` для подсчета точек внутри четверти единичной окружности.
   - Инициализировать мьютекс `mutex` для защиты доступа к `incircle`.

2. **Определение функции `pi`:**
   - Каждый поток выполняет следующее:
     - Получить количество точек, которые будет генерировать поток.
     - Инициализировать локальную переменную `incircle_thread` для подсчета точек внутри окружности.
     - Генерировать случайные координаты $ (x, y) $ в диапазоне от 0 до 1.
     - Проверить, попадает ли точка в единичную окружность: если $ x^2 + y^2 < 1 $, увеличить `incircle_thread`.
     - Заблокировать мьютекс и добавить `incircle_thread` к глобальной переменной `incircle`.
     - Разблокировать мьютекс.

3. **Создание потоков:**
   - Разделить общее количество испытаний `ntrials` на количество потоков `nthreads` для определения количества точек, генерируемых каждым потоком.
   - Создать массив потоков `threads`.
   - Запустить первый поток с дополнительным количеством точек (остаток от деления) и остальные потоки с равным количеством точек.

4. **Ожидание завершения потоков:**
   - Для каждого потока в массиве `threads`, дождаться его завершения.

5. **Вывод результата:**
   - Аппроксимировать число $\pi$ по формуле $ \hat{\pi} = 4 \times \frac{\text{incircle}}{\text{ntrials}} $.
   - Вывести значение числа $\hat{\pi}$ и время выполнения.

6. **Завершение:**
   - Уничтожить мьютекс.

### Описание реализации `rwlock`

1. **Заголовочный файл:**
   - `#ifndef MY_RWLOCK` / `#define MY_RWLOCK`: Предотвращает многократное включение заголовочного файла.
   - `#include <pthread.h>`: Подключает библиотеку POSIX threads.
   - `#include <stdbool.h>`: Подключает поддержку булевых типов.

2. **Структура `rwlock_t`:**
   - `pthread_mutex_t mutex`: Мьютекс для синхронизации доступа к rwlock.
   - `pthread_cond_t readers`: Условная переменная для читателей.
   - `pthread_cond_t writers`: Условная переменная для писателей.
   - `int readers_count`: Количество активных читателей.
   - `int writers_waiting`: Количество ожидающих писателей.
   - `int writer_active`: Флаг, указывающий на активного писателя.

3. **Функция инициализации:**
   - `void rwlock_init(rwlock_t* lock)`:
     - Инициализирует rwlock.
     - Создает мьютекс и условные переменные.
     - Устанавливает начальные значения для `readers_count`, `writers_waiting` и `writer_active`.

4. **Функция блокировки на чтение:**
   - `void rwlock_rdlock(rwlock_t* lock)`:
     - Блокирует rwlock для чтения.
     - Ждет, если писатель активен или есть ожидающие писатели.
     - Увеличивает счетчик активных читателей после получения блокировки.

5. **Функция блокировки на запись:**
   - `void rwlock_wrlock(rwlock_t* lock)`:
     - Блокирует rwlock для записи.
     - Увеличивает счетчик ожидающих писателей.
     - Ждет, если есть активные читатели или другой активный писатель.
     - Активирует флаг `writer_active` после получения блокировки.

6. **Функция разблокировки:**
   - `void rwlock_unlock(rwlock_t* lock)`:
     - Разблокирует rwlock.
     - Если писатель был активен, отключает его и оповещает всех ожидающих читателей.
     - Если читатель разблокируется, уменьшает его счетчик и уведомляет ожидающего писателя, если он стал нулевым.

7. **Функция разрушения:**
   - `void rwlock_destroy(rwlock_t* lock)`:
     - Освобождает ресурсы, связанные с rwlock.
     - Уничтожает мьютекс и условные переменные.

8. **Общая характеристика:**
   - Реализация rwlock позволяет безопасно выполнять операции чтения и записи в многопоточном окружении.
   - Использует мьютексы и условные переменные для управления доступом к общим ресурсам.
   - Поддерживает параллельное чтение и ограничивает запись до тех пор, пока не завершатся все операции.