
В новых версия Linux дистрибутивов нельзя устанавливать глобально питоновские модули, что бы это обойти нужно использовать:

```bash
pip install <module-name> --break-system-packages

# пример
pip install bs4 --break-system-packages
```
