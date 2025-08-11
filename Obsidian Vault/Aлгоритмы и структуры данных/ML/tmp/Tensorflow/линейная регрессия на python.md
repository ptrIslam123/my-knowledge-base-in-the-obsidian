
```python
import numpy as np
from sklearn.linear_model import LinearRegression
from sklearn.metrics import r2_score

# Создаем синтетические данные
np.random.seed(0)

# Генерируем случайные X значения
X = np.random.rand(100, 1) * 10

# Формируем результаты Y с добавлением некоторого случайное отклонение(шума)
# Апроксимируемая функциябудет иметь вид: [y = 4 * x + 3]
Y = 4 * X + 3 + np.random.randn(100, 1) 

# Создаем и обучаем модель линейной регрессии
model = LinearRegression()
model.fit(X, Y)

# Выводим коэффициенты модели
print('(a)=', model.coef_[0][0], ' (b)=', model.intercept_[0])

# Прогнозируем значения для новых данных
x_new = np.array([[2], [5]])
y_pred = model.predict(x_new)

# Новые тестовые даные на которых будем проверять модель
print('X`=', x_new)
# Предикшин модели на уже тестовых данных
print('Y`=', y_pred)
# Реально ожидыные значения
print('Y=', 4 * x_new + 3)
```

Вычисление [[Коэффициент детерминации ( R^2 )]]:
```python
from sklearn.metrics import r2_score

# действительны данные
Y_true = [3, -0.5, 2, 7]
# предсказынне данын нашей моделью, например
Y_pred = [2.5, 0.0, 2, 8]

# расчитываем детерминатность предсказаний нашей модели
r = r2_score(Y_true, Y_pred) # R^2=0.948... - что в говоорит о тесной кореляции

```