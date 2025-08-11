Импотрируем нужные билиотека
```python
import tensorflow as tf
import matplotlib.pyplot as plt
import numpy as np
import os

from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Flatten, Dense

```

Определяем глобавльные переменные и константы
```python
module_name = "model.out"
target_size = (500, 500)
train_epochs = 30
train_images_limits = 1000
test_images_limits = train_images_limits / 10

```

Функция загрузки изображеений с диска в рам и вспомогательны функции
```python
def is_image_file(file_path: str):
	_, file_extension = os.path.splitext(file_path)
	return file_extension.lower() in ['.jpg', '.jpeg', '.png', '.gif']

  

def load_image(image_path: str):
	image = (tf.keras.preprocessing.image.load_img(image_path))
	return image

def load_images(images_dir_path: str, image_limit: int):
	images = []
	labels = []
	loaded_images_count = 0
	
	for filename in os.listdir(images_dir_path):
		full_image_path = os.path.join(images_dir_path, filename)
		if os.path.isfile(full_image_path) and is_image_file(full_image_path):
		
			try:
				image = load_image(full_image_path)
				images.append(image)
				labels.append(get_category(filename))
				loaded_images_count += 1
				  			
				if loaded_images_count >= image_limit:
					break
	
			except Exception as e:
				print(f"WARNING: Could not load the image with full path={full_image_path} because={str(e)}. Just pass this image!")
	
	return images, labels

```

Покготовка изображений и конвертация их в обучающий dataset
```python
def get_category(filename):
	return category_to_int(os.path.splitext(filename)[0].split('_')[0])

def category_to_int(category: str):
	if category == "cat":
		return 0
	else:
		return 1

def category_to_str(category: int):
	if category == 0:
		return "cat"
	else:
		return "dog"

def prepare_dataset(images: list, target_size: tuple):
	dataset = []
	for image in images:
		image_array = np.array(image)
		image_array.resize(target_size)
		image_array = image_array / 255.0
		dataset.append(image_array)
	
	return np.array(dataset)

```

Процесс загруки обучающей и тестовой выборки в память
```python
(train_images, train_labels) = load_images(train_dir, train_images_limits)
(test_images, test_labels) = load_images(test_dir, test_images_limits)
print("Loaded train and test images successfully")

```

Процесс подготовки обучающей и тестовой выборки
```python
(train_dataset_x,train_dataset_y) = prepare_dataset(train_images, target_size), np.array(train_labels)

(test_dataset_x, test_dataset_y) = prepare_dataset(test_images, target_size), np.array(test_labels)
print("Prepared dataset for training successfully")

```

Создание и нейроклассификатора
```python
model = Sequential([
	Flatten(input_shape=target_size),
	Dense(128, activation='relu'),
	Dense(524, activation='relu'),
	Dense(128, activation='relu'),
	Dense(1, activation='sigmoid')

])

model.compile(optimizer='adam', loss='binary_crossentropy', metrics=['accuracy'])
print("Created the model successfully")

```

Обучение модели
```python
model.fit(train_dataset_x, train_dataset_y, epochs=train_epochs, validation_data=(test_dataset_x, test_dataset_y), batch_size=64)

```

Оченка модели на тестовой выборки для проверки
```python
test_loss, test_acc = model.evaluate(test_dataset_x, test_dataset_y)
print('Test accuracy=', test_acc, ' Test loss=', test_loss)

```