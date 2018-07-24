# Начало работы с Webpack 4 #

[Официальная страница документации](https://webpack.js.org/guides/)

## Установка

```
npm install webpack webpack-cli --save-dev
```

Webpack 4 требует для работы webpack-cli. Если последний пакет не установить, то при запуске webpack будет просить его установить.

Webpack можно установить и глобально, но это не рекомендуется, так как является плохой практикой. Глобальная установка завязывает все проекты на использование конкретной версии Webpack, что может привести к проблемам, если в настоящее время, либо в будущем, разные проекты используют, либо будут использовать различные версии Webpack.

## Запуск

Запуск просто командой `webpack`.

## Файл конфигурации

Webpack 4 не требует обязательного использования конфигурационного файла, но при этом необходимо четкое размещение и именование файла, который будет являться точкой входа: `./src/index.js`. При этом выходной бандл будет `./dist/main.js`.

Если в текущем каталоге имеется файл `webpack.config.js`, то Webpack автоматически будет использовать его как конфигурационный файл. Если же необходимо указать другой файл конфигурации, то можно это сделать через соответсвующий ключ:
```
webpak --config other.webpack.config.js
```
Это может быть полезно для сохранения различной конфигурации в разных файлах.

Пример простейшего файла конфигурации:
```javascript
const path = require('path');
module.exports = {
    entry: './src/app.js',
    output: {
        path: path.resolve(__dirname, './dist'),
        filename: 'app.js'
    },
}
```
здесь всё очевидно:

`entry` - точка входа в приложение. Отталкиваясь от этого файла Webpack ,будет стройть граф зависимостей (dependency graph). Подробнее: [https://webpack.js.org/concepts/entry-points/](https://webpack.js.org/concepts/entry-points/);

`output.path` - **абсолютный** путь к каталогу, где будет размещён результат обработки. Для того, чтобы обеспечить независимость от платформы и места расположения проекта в файловой системе, путь к каталогу строится динамически с использованием родного пакета Node.js `path`. Подробнее: [https://webpack.js.org/concepts/output/](https://webpack.js.org/concepts/output/).



### Параметр `mode`

При запуске Webpack с описанным выше простейшим файлом конфигурации, выводится предупреждение, что параметр `mode` не задан и Webpack будет автоматически использовать значение `production`:

```
WARNING in configuration
The 'mode' option has not been set, webpack will fallback to 'production' for this value. Set 'mode' option to 'development' or 'production' to enable defaults for each environment.
You can also set it to 'none' to disable any default behavior. Learn more: https://webpack.js.org/concepts/mode/
```

Задать значение параметра можно либо в файле конфигурации:

```javascript
mode: 'production'
```

либо передать в командной строке при запуске Webpack:

```
webpack --mode=production
```

Параметр `mode` определяет, какой способ встроенной оптимизации кода будет использован при сборке проекта. Допустимые значения приведены в таблице ниже.

| Значение параметра | Описание |
|---|---|
| development | Устанавливает переменную среды `process.env.NODE_ENV` для плагина `DefinePlugin` в значение `development`. Задействует плагины `NamedChunksPlugin` и `NamedModulesPlugin`.|
| production | Устанавливает переменную среды `process.env.NODE_ENV` для плагина `DefinePlugin` в значение `production`. Задействует плагины `FlagDependencyUsagePlugin`, `FlagIncludedChunksPlugin`, `ModuleConcatenationPlugin`, `NoEmitOnErrorsPlugin`, `OccurrenceOrderPlugin`, `SideEffectsFlagPlugin` и `UglifyJsPlugin`.
| none | Не использует ни какие оптимизации.

Подробнее о параметре `mode` - в [документации](https://webpack.js.org/concepts/mode/).



### Загрузчики (Loaders)

[Загрузчики (Lodaders)](https://webpack.js.org/concepts/loaders/) - это трансформации, которые применяются для исходников модуля. Они позволяют предварительно обрабатывать файлы по мере их импорта или загрузки. Таким образом, загрузчики - это что-то наподобие задач (tasks) в других системах сборки, которые предоставляют большие возможности по обработке при сборке фронтенда.

Для применения загрузчика описываются правила в файле конфигурации. Для этого используется параметр `module.rules`, в котором можно описать несколько правил для загрузчиков.

Правило ([Rule](https://webpack.js.org/configuration/module/#rule)) состоит из трёх частей: условие, результат и вложенное правило.

Условие ([Rule Condition](https://webpack.js.org/configuration/module/#rule-conditions)) может иметь два значения:

1. ресурс - абсолютный путь к требуемому файлу. Для указания ресурса применяются свойства `test`, `include`, `exclude` и `resource`.
2. issuer - абсолютный путь к файлу модуля, запросившего ресурс. Применяется свойство `issuer`.

Если условие сработало, то используются результаты. Для них существует два значения:

1. Применяемые загрузчики (массив загрузчиков, применяемых к ресурсу). Для загрузчиков используются свойства `loader`, `options`, `use`. Для совместимости также применяются `query`, `loaders`. Здесь же может применятся свойство `enforce`, определяющее будет ли это нормальный, пред- либо пост-загрузчик.
2. Параметры парсера, используемые для создания парсера модуля. Используется свойство `parser`.


Доступные загрузчики можно найти в [разделе Loaders](https://webpack.js.org/loaders/).

Пример описания правила для загрузчика [`babel-loader`](https://webpack.js.org/loaders/babel-loader/):

```javascript
module: {
  rules: [
    {
      test: /\.js$/,
      exclude: '/node_modules',
      use: {
        loader: 'babel-loader',
        options: {
          presets: ['es2015']
        }
      }
    }
  ]
}
```

Здесь правило применяется для всех файлов `*.js` (описано регулярным выражением в параметре `test`) за исключением файлов, находящихся в папке `node_modules`. При нахождении `*.js` файла используется (`use`) загрузчик `babel-loader` с параметрами, указанными в свойстве `options`.