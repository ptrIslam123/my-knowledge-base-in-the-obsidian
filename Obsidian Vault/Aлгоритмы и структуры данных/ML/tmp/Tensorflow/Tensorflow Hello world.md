```python
# Import TensorFlow

import tensorflow as tf
import os

os.environ['TF_CPP_MIN_LOG_LEVEL'] = '3'

# Create a constant tensor
tensor1 = tf.constant(5)
tensor2 = tf.constant(10)

# Add the two tensors
result = tf.add(tensor1, tensor2)

# Print the result
print("TensorFlow version:", tf.__version__)
print("Result:", result)

```

Следующее [[Простейшая модель  на Tensorflow]]