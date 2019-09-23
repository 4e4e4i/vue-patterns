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

### #1 Cleaner Views

```javascript
data() {
    return {
        loading: false,
        error: null,
        post: null
    }
},
watch: {
    '$route': {
        handler: 'resetData',
        immediate: true
    }
},
methods: {
    resetData() {
        this.loading = false
        this.error = null
        this.post = null
        this.getPost(this.$route.params.id)
    },
    getPost(postId) {
        // ...
    }
}
```

В данном примере мы видем много настроек. Мы устанавливаем дефолтные значения в data() и каждый раз, когда
наш route изменяется мы вызываем метод resetData, который сбрасывает данные из data(), только в том случае, что предыдущий
route использовал тот же компонент, как и текщий route. Например, вы собираетесь перейти со страницы /posts/1 на страницу /posts/2.

Vue увидит, что роуты используют один и тот же компонент и поэтому, чтобы не начинать с нуля, он попытается быть чутка умнее и 
будет рендерить минимально возможные изменения. Например, не будет перерисовывать полностью компонент, а всего-лишь задаст
ему другие данные.

Но вы так же можете упростить данный код. 

```javascript
data() {
    return {
        loading: false,
        error: null,
        post: null
    }
},
created() {
    this.getPost(this.$route.params.id)
}
methods: {
    getPost(postId) {
        // ...
    }
}
```

Как вы видите, мы полностью избавилсь от сброса данных и вотчера. У нас остался только вызов метода на created хук.
И способ, чтобы сделать это - всего-лишь одна строка с добавление key к router-view:

```
<router-view :key="$route.fullPath"></router-view>
```

И ключ, который я часто люблю добавлять - это route.fullPath. Он просто сообщает Vue: если полный путь меняется, даже 
если компонент тот же самый, ты должен отрендерить его заново, поэтому это означает, что у вас может быть немного худше
производительность от перехода маршрута к маршруту, но ваши компоненты Vue или компоненты маршрута будут намного проще
и, скорее всего, с меньшей вероятностью будут содержать ошибки.

Данный подход делает ваши маршруты более предсказуемыми.


### #2 Прозрачные обертки

```
<template>
    <input
        :value="value"
        @input="$emit('input', $event.target.value)"
    >
</template>
```

До этого мы уже рассматривали компоненты для базовых элементов (base-input, base-button и тд). Здесь шаблон для нашего
base-input. В нем мы получаем через Props значение и каждый раз, когда мы вводим что-либо, эмитит событие родителю.

```
<BaseInput @focus.native="doSomething" />
```

Так же когда вы используете данный компонент, вы можете захотеть прослушать какие-то другие события, которые явно не
указали в нем. Так скажем, каждый раз, когда компонент получает focus, вы хотите сделать что-то. И вы должны использовать
.native модификатор, потому что в противном случае @focus будет слушать focus событие, которое эммитится из компонента, 
так как мы сделали в input. Поэтому .native говорит, что мы должны слушать событие на корневом элементе компонента. А
корневым элементом этого комопнента является input. 

Но может возникнуть проблема, если шаблон инпута будет таким:

```
<template>
    <label>
        {{ label  }}
        <input
            :value="value"
            @input="$emit('input', $event.target.value)"
        >
    </label>
</template>
```

Теперь корневым элементом нашего компонента будет label, поэтому наш @focus.native слушает фокус событие на label.
вместо input. И вот мое решение:

```
<template>
    <label>
        {{ label  }}
        <input
            :value="value"
            v-on="listeners"
        >
    </label>
</template>

// ...

computed: {
    listeners() {
        return {
            ...this.$listeners,
            input: event => 
                this.$emit('input', event.target.value)
        }
    }
}
```

Я создаю listeners в вычисляемом свойстве, который возвращает объект со всеми свойствами из listeners (this.$listeners - 
все слушатели из компонента родителя), и затем переопределяет input, чтобы он снова работал с моделью Vue. И для этого в 
шаблоне мы используем v-on без указания какого-либо события, передаем туда объект и он будет слушать всех указанных слушателей
в этом объекте. Это означает, что для base-input мы не должны больше использовать @focus.native, теперь мы можем рассматривать его,
как фактический компонент input

```
<BaseInput @focus="doSomething" />
```

Но есть одна дополнительная проблема.

 ```
 <BaseInput
    placeholder="What's your name?" 
    @focus="doSomething"
 />
 ```

Что произойдет, когда мы добавим плейсхолдер? Он добавит placeholder в label, потому что по умолчанию Vue передает любые
аттрибуты, не являющиеся аттрибутами и не указанными как Props в корневой элемент компонента, в нашем случае - label.

И в данном случае есть решение:

```
inheritAttrs: false
```

Эта опция, которые вы можете добавить в конфигурацию вашего Vue компонента, которая говорит Vue, что компонент не должен
автоматически наследовать аттрибуты корневым элементом, вместо этого мы собираемся явно указать как эти аттрибуты будут
обрабатываться с v-bind="$attrs". attrs - объект содержащий все аттрибуты, которые не были указаны, как props в этом 
компоненте, но были переданы в него. 

```
<template>
    <label>
        {{ label  }}
        <input
            v-bind="$attrs"
            :value="value"
            v-on="listeners"
        >
    </label>
</template>

```


## Разблокированные возможности

