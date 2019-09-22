# Полезные паттерны Vue

## Повышение производительности

### #1 Умные наблюдатели

```javascript
created() {
    this.fetchUserList()    
},
watch: {
    searchText() {
        this.fetchUserList()
    }
}
```
Fetch на created, затем watch. 
Как только компонент создается, мы вызываем, например, метод запроса списка пользователей. Так же,
когда происходит изменение каких-либо данных в data, например searchText, мы вызываем тот же метод запроса.

Допустим вы набираете какого-либо пользователя в поисковой строке и хотите отфильтровать всех пользователей.

Многие сталкивались с данным подходом, но давайте посмотрим, как можно оптимизировать и немного улучшить наш код.

Во-первых, наблюдатели могут принимать названия методов в виде строки, как значение вместо функции.

```javascript
created() {
    this.fetchUserList();
}
watch: {
    searchText: 'fetchUserList'
}
```

Другая оптимизаця - immediate: true.

```javascript
watch() {
    searchText: {
        handler: 'fetchUserList',
        immediate: true
    }
}
```
 
До этого значение наблюдателя searchText была строка, теперь это объект. И обработчик(handler) может иметь
значения так же либо строки с названием метода, либо функции. А поле immediate со значением true, означает,
что мы не должны создавать created хук больше, потому что обработчик будет вызван сразу как компонент будет готов.

Данный подход не сделает ваш код намного чище, но в случае, когда ваш метод отслеживается путем консолидации в
наблюдателе, вы можете уменьщить площадь поверхности для ошибок в вашем приложении.

### #2 Регистрация компонента

```javascript
import BaseButton from './base-button'
import BaseIcon from './base-icon'
import BaseInput from './base-input'

export default {
    components: {
        BaseButton,
        BaseIcon,
        BaseInput
    }
}
```

Часто импортируемые компоненты.

Часто мы встречаем подход, когда у нас есть много основных компонентов, иногда как обертка для других элементов, чтобы
добавить некоторые стили, которые используются именно в вашем приложении. Мы импортируем постоянно кучу компонентов...
baseIcon, baseButton, baseInput и затем регистрируем их, для того чтобы написать небольшой код в шаблоне.

```javascript
<BaseInput
    v-model="searchText"
    @keydown.enter="search"
/>
<BaseButton @click="search">
    <BaseIcon name="search"/>
</BaseButton>
```

Согласитесь иногда это слишком надоедает, постоянно импортировать и регистрировать.

Давайте рассмотрим новый подход к регистрации компонентов, которые используются у нас очень часто на страницах приложения

```javascript
import Vue from 'vue'
import upperFirst from 'lodash/upperFirst'
import camelCase from 'lodash/camelCase'

// Require in a base component context
const requireComponent = require.context(
    '.', false, /base-[\w-]+\.vue$/
)

requireComponent.keys().forEach(fileName => {
    // Get component config
    const componentConfig = requireComponent(fileName)
    
    // Get PascalCase name of component
    const componentName = upperFirst(
        camelCase(fileName.replace(/^\.\//, '').replace(/\.\w+$/, ''))
    )
    
    // Register component globally
    Vue.component(componentName, componentConfig.default || componentConfig)
})
```

Это выглядит очень большим кодом, на первый взгляд, но давайте пройдемся по нему по-порядку.

```javascript
const requireComponent = require.context(
    '.', false, /base-[\w-]+\.vue$/
)
```

Так requier context. requier служебная команда Node.js, здесь мы используем ее для webpack, чтобы
программно потребовать группы из целой группы компонентов. Поэтому как вы видите, здесь мы получаем
все компоненты из текущего каталог, которые соответсвуют шаблону base- и оканчиваются на .vue. 
base- это общий префикс, который мы собираемся использовать в приложении для компонентов, которые
используются везде и они часто всего-лишь оборачивают элемент и так же часто имееют имя элемента в своем
названии. 

```javascript
requireComponent.keys().forEach(fileName => {
    // Get component config
    const componentConfig = requireComponent(fileName)
    
    // Get PascalCase name of component
    const componentName = upperFirst(
        camelCase(fileName.replace(/^\.\//, '').replace(/\.\w+$/, ''))
    )
    
    // Register component globally
    Vue.component(componentName, componentConfig.default || componentConfig)
})
```

Затем мы проходим по всем компонентам, которые соответсвуют данному шаблону и получаем componentConfig и затем
получаем версию этого компонента в Pascal Case, а в самом низу регистрируем его через Vue.component. Регистрация через
Vue.component - глобальная регистрация и это означает, что компонент будет доступен в любом месте нашего
приложения. Для того, чтобы использовать код, вы можете просто вставить его прямо в main.js или входной файл приложения.
Но я часто храню его в файле, как например, 'component/_global.js' и затем импортирую его в main.js. И все эти компоненты
регистрируются до того как инициализируется новый экземпляр представления. 

Я хотел бы еще рассказать о еще одном подходе в данном случае, которые больше связан с webpack, чем со vue6 но многие люди
не знают о нем. Так вот в самом низу, когда мы регистрируем компонент 

```javascript
Vue.component(componentName, componentConfig.default || componentConfig)
```

Зачем мы пишем componentConfig.default || componentConfig? 
Если вы используете компонент .vue при экспорте по умолчанию, это означает, что параметры вашего компонента будут
находиться в componentConfig.default, находится в свойстве default на экспортируемом модуле, если вы импортируете что-то откуда-то
используя es6 синтаксис это автоматически будет искать default. А когда вы используете require для этого, то он не будет искать
default. Таким образом мы должны указать, что хотим использовать .default или если вы не экспортировали по умолчанию 
свой компонент vue, например, вы используете модуль, который экспортируется в js синтаксисе, параметры будут находится в componentConfig,
а так же если вы используете компонент, просто как шаблон или стили и вы не используете тэг script в нем, конфигурация
компонента будет находится в componentConfig вместо .default.

### #3 Регистрация модулей

```javascript
import auth from './modules/auth'
import posts from './modules/posts'
import comments from './modules/comments'
// ...

export default new Vuex.Store({
    modules: {
        auth,
        posts,
        comments,
        // ...
    }
})
```

Этот паттерн очень похож с предыдущим. Vuex модули - маленькие кусочки state management'а. И этот паттерн подходит
для любых других фреймворков, которые реализуют state management и webpack. 

Так когда вы создаете новые модули, вы постоянно должны их импортировать и регистрировать в поле modules. И это утомительно!

Но есть решение)

```javascript
import camelCase from 'lodash/camelCase'
const requireModule = require.context('.', false, /\.js$/)
const modules = {}

requireModule.keys().forEach(filename => {
    // Don't register this file as a Vuex module
    if (fileName === './index.js') return
    
    const moduleName = camelCase(
        fileName.replace(/(\.\/|\.js)/g, '')
    )
    modules[moduleName] = requireModule(filename)
})

export default modules
```

Это выглядет очень похоже с тем, что мы делали до этого с регистрацией компонентов. 

Так мы делаем require всех наших модулей, обычно я храню их в modules/index.js, все js файлы
внутри текущей папки, а затем мы проходим по каждому файлу и если имя файла index.js, то тогда не нужно ничего делать.
Затем мы получаем имя модуля с camel case названия файла и затем добавляем в объект модулей и экспортируем все эти модули.

Так же хочу показать еще одну оптимизацию, которую я часто делаю здесь.

```javascript
modules[moduleName] = {
    namespaced: true,
    ...requireModule(filename),
}
```

Если вам нравятся модули с пространством имен, или вы хотите установить какие-нибудь другие опции по-умолчанию для всех
ваших модулей, вы можете сделать это здесь. 

Теперь наш index.js в сторе будет выглядеть так:

```javascript
import modules from './modules'

export default new Vuex.Store({
    modules
})
```

## Радикальная настройка

## Разблокированные возможности

