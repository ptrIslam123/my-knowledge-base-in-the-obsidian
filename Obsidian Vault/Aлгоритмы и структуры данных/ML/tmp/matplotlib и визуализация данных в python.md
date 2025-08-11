Визуализация данных является важным этапом в анализе и понимании информации. Библиотека `matplotlib` в Python предоставляет широкие возможности для создания различных типов графиков. Давайте рассмотрим визуализацию одномерных и двумерных данных подробно.

### Визуализация одномерных данных

Одномерные данные — это данные, которые можно представить в виде одного списка или массива. Рассмотрим несколько типов графиков для визуализации таких данных.

#### 1. Гистограмма (Histogram)
Гистограмма используется для отображения распределения данных.

```python
import matplotlib.pyplot as plt
import numpy as np

# Пример данных
data = np.random.randn(1000)

# Создание гистограммы
plt.hist(data, bins=30, edgecolor='black')

# Добавление подписей
plt.xlabel('Values')
plt.ylabel('Frequency')
plt.title('Histogram')

# Отображение графика
plt.show()
```

![[Figure_1.png]]


#### 2. Ящик с усами (Box Plot)
Ящик с усами используется для отображения статистических данных, таких как медиана, квартили и выбросы.

```python
import matplotlib.pyplot as plt
import numpy as np

# Пример данных
data = [np.random.normal(0, std, 100) for std in range(1, 4)]

# Создание ящика с усами
plt.boxplot(data)

# Добавление подписей
plt.xlabel('Groups')
plt.ylabel('Values')
plt.title('Box Plot')

# Отображение графика
plt.show()
```

#### 3. График плотности (Density Plot)
График плотности используется для визуализации распределения данных с помощью сглаженной кривой.

```python
import matplotlib.pyplot as plt
import numpy as np
from scipy.stats import gaussian_kde

# Пример данных
data = np.random.randn(1000)

# Создание графика плотности
density = gaussian_kde(data)
xs = np.linspace(min(data), max(data), 200)
plt.plot(xs, density(xs), label='Density')

# Добавление подписей
plt.xlabel('Values')
plt.ylabel('Density')
plt.title('Density Plot')

# Отображение графика
plt.show()
```

![[Figure_2.png]]

### Визуализация двумерных данных

Двумерные данные — это данные, которые можно представить в виде пар значений (x, y). Рассмотрим несколько типов графиков для визуализации таких данных.

#### 1. Линейный график (Line Plot)
Линейный график используется для отображения изменения данных во времени или последовательности.

```python
import matplotlib.pyplot as plt

# Пример данных
x = [1, 2, 3, 4, 5]
y = [2, 3, 5, 7, 11]

# Создание линейного графика
plt.plot(x, y, marker='o')

# Добавление подписей
plt.xlabel('X-axis')
plt.ylabel('Y-axis')
plt.title('Line Plot')

# Отображение графика
plt.show()
```

![[Figure_3.png]]

#### 2. Столбчатый график (Bar Plot)
Столбчатый график используется для сравнения значений различных категорий.

```python
import matplotlib.pyplot as plt

# Пример данных
categories = ['A', 'B', 'C', 'D']
values = [3, 7, 5, 4]

# Создание столбчатого графика
plt.bar(categories, values)

# Добавление подписей
plt.xlabel('Categories')
plt.ylabel('Values')
plt.title('Bar Plot')

# Отображение графика
plt.show()
```

![[Figure_4.png]]

#### 3. Диаграмма рассеяния (Scatter Plot)
Диаграмма рассеяния используется для отображения зависимости между двумя переменными.

```python
import matplotlib.pyplot as plt
import numpy as np

# Пример данных
x = np.random.rand(100)
y = np.random.rand(100)

# Создание диаграммы рассеяния
plt.scatter(x, y)

# Добавление подписей
plt.xlabel('X-axis')
plt.ylabel('Y-axis')
plt.title('Scatter Plot')

# Отображение графика
plt.show()
```

![[Figure_5.png]]

#### 4. Тепловая карта (Heatmap)
Тепловая карта используется для отображения данных в виде цветных матриц.

```python
import matplotlib.pyplot as plt
import numpy as np
import seaborn as sns

# Пример данных
data = np.random.rand(10, 10)

# Создание тепловой карты
sns.heatmap(data, annot=True, cmap='YlGnBu')

# Добавление подписей
plt.xlabel('X-axis')
plt.ylabel('Y-axis')
plt.title('Heatmap')

# Отображение графика
plt.show()
```

![[Figure_6.png]]

### Заключение

Визуализация данных — это мощный инструмент для анализа и понимания информации. Библиотека `matplotlib` предоставляет широкие возможности для создания различных типов графиков, начиная от простых линейных графиков и заканчивая сложными тепловыми картами. Выбор подходящего типа графика зависит от характера данных и задачи, которую вы хотите решить.