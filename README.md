Да, ошибка понятная:

Argument #1 ($note) must be of type Note, string given

Значит контейнер увидел параметр маршрута {note}, но передал его как строку "5", а не превратил в объект Note.

Причина почти точно в порядке внутри Container::resolveParameters():

if (array_key_exists($name, $parameters)) {
    $dependencies[] = $parameters[$name];
    continue;
}

Этот блок срабатывает раньше, чем Route Model Binding. Поэтому $note сразу становится строкой.


---

Исправляем /local/mvc/Core/Container.php

Открой файл:

/local/mvc/Core/Container.php

Найди метод:

private function resolveParameters(...)

И замени его полностью на этот:

private function resolveParameters(ReflectionMethod|ReflectionFunctionAbstract $reflection, array $parameters = []): array
{
    $dependencies = [];

    foreach ($reflection->getParameters() as $parameter) {
        $name = $parameter->getName();
        $type = $parameter->getType();

        /**
         * 1. Сначала обрабатываем классы.
         *
         * Это важно для Route Model Binding:
         *
         * public function destroy(Note $note)
         *
         * Если сначала проверить route params,
         * то $note станет строкой из URL, а не объектом Note.
         */
        if ($type instanceof ReflectionNamedType && !$type->isBuiltin()) {
            $className = $type->getName();

            /**
             * Laravel-like Route Model Binding.
             *
             * Пример:
             * маршрут: /notes/{note}
             * метод: destroy(Note $note)
             */
            if (is_subclass_of($className, Model::class)) {
                $modelId = null;

                if (array_key_exists($name, $parameters)) {
                    $modelId = $parameters[$name];
                } elseif (array_key_exists('id', $parameters)) {
                    $modelId = $parameters['id'];
                } elseif (count($parameters) === 1) {
                    $modelId = reset($parameters);
                }

                if ($modelId !== null) {
                    $model = $className::findModel($modelId);

                    if (!$model instanceof Model) {
                        throw new ModelNotFoundException($className, $modelId);
                    }

                    $dependencies[] = $model;
                    continue;
                }
            }

            /**
             * Обычная зависимость:
             * UserService $service
             * StoreNoteRequest $request
             */
            $object = $this->make($className);

            if ($object instanceof FormRequest) {
                $object->validateResolved();
            }

            $dependencies[] = $object;
            continue;
        }

        /**
         * 2. Потом обычные route-параметры.
         *
         * Пример:
         * public function show(string $id)
         */
        if (array_key_exists($name, $parameters)) {
            $dependencies[] = $parameters[$name];
            continue;
        }

        /**
         * 3. Значение по умолчанию.
         */
        if ($parameter->isDefaultValueAvailable()) {
            $dependencies[] = $parameter->getDefaultValue();
            continue;
        }

        throw new RuntimeException(
            'CONTAINER_CANNOT_RESOLVE_PARAMETER: $' . $name . ' in ' . $reflection->getName()
        );
    }

    return $dependencies;
}


---

Проверь, что сверху в Container.php есть нужный use

В начале файла должно быть:

use ReflectionClass;
use ReflectionFunctionAbstract;
use ReflectionMethod;
use ReflectionNamedType;
use RuntimeException;

Model, FormRequest, ModelNotFoundException дополнительно импортировать не надо, потому что они в том же namespace:

namespace Local\Mvc\Core;


---

Сбрось OPcache

opcache_reset();
echo 'OPcache reset OK';

Потом снова попробуй удалить заметку.

Теперь цепочка должна быть такая:

DELETE /notes/5
↓
Router получает {note} = 5
↓
Container видит destroy(Note $note)
↓
Container делает Note::findModel(5)
↓
В контроллер приходит объект Note
↓
$note->delete()
↓
soft delete через deleted_at

После удаления заметка должна исчезнуть из обычного списка и появиться в корзине.