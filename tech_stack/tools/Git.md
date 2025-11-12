Объединение коммитов, до Initial commit

```ignorelang
# Объединяем последние 3 коммита (все кроме Initial commit)
git reset --soft 602b95f  # reset до Initial commit

# Создаем один объединенный коммит
git commit -m "Все задачи завершены: финальная версия"

# Принудительно отправляем новую историю
git push --force-with-lease
```