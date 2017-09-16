# tstate-machine
[![Build Status](https://travis-ci.org/SoEasy/tstate-machine.svg?branch=1.1-dev)](https://travis-ci.org/SoEasy/tstate-machine)

Реализация StateMachine на TypeScript

## Attention
Модуль работоспособен, но разработка еще не закончена. API менять не собираюсь

## Установка
Пакет лежит в npm
>npm install --save tstate-machine

Установка с гитхаба
>npm install https://github.com/SoEasy/tstate-machine/tarball/master

## Основная информация
Интерфейс машины состояний подсмотрен тут:
[JS FSM](https://github.com/jakesgordon/javascript-state-machine).

StateMachine представляет собой класс, от которого стоит наследовать свои классы конкретных машин.
Все поля, которые будут описаны в вашем потомке - это начальное состояние машины.

Главный бонус - с типизацией проблем не возникнет.
> Важно! Всем полям класса необходимо задать начальное значение - хоть null, хоть undefined. В пртивном случае машина не запомнит эти поля из-за особенностей компиляции TypeScript

Данная реализация StateMachine не предполагает создания независимых состояний.
Т.е. все состояния машины либо наследуются от initial, либо от других состояний.
Объявлять в состоянии новые поля с данными - технически можно, но не нужно.
Объявление нового состояния должно содержать только те поля, которые отличаются от родительского.

StateMachine берет на себя контроль за переходами из состояния в состояние.
Для этого при описании состояния в декларативном стиле описывается массив других состояний, в которые можно перейти из описываемого.
 При попытке перейти в недозволенное состояние вылетит Error с описанием откуда-куда не получилось перейти
   
StateMachine позволяет регистрировать коллбэки, которые будут вызваны при входе в нужное состояние и при выходе из него.
 Коллбэки, зарегистрированные для входа в состояние способны получать данные, переданные в метод перехода между состояниями.
 Коллбэки выхода из состояния, очевидно, никаких данных не принимают.

## Объяснение работы
Основа реализации машины - метаданные, дескрипторы доступа декораторов и цикл for-in по полям объекта.

Главная идея:
1. В конструкторе класса-потомка вызвать родительский метод `this.rememberInitState();` Который пройдет циклом по всем полям класса и запомнит их как начальное состояние
2. В классе-потомке описать protected-геттер $next, возвращающий массив возможных состояний для перехода из начального
3. С помощью специального декоратора `@StateMachine.extend` зарегистрировать новые состояния, описанные как объекты с изменениями. Хранить их в метаданных класса
4. При переходе из состояния в состояние собрать цепочку наследования состояний, привести объект машины в начальное состояние и накатить на него всю цепочку изменений. *Почему так? Потому что ветвей наследования может быть несколько, и если переходить вдруг из одной в другую - чтобы не описывать все различия - проще поехать от корня дерева состояний. Опыт показывает, что развесистых графов состояния у нас не было - можно не бояться за производительность*
5. Все методы класса оборачиваются декоратором `StateMachine.hide` - он прячет метод от попадания в итератор for-in по объекту. Это важно, чтобы методы не попадали в хранилище initial-состояния и не перезаписывались каждый раз при переходе.

## API
- Поля состояния описывать просто как поля класса.
- `@StateMachine.hide()` - декоратор, которым оборачивать все методы класса-потомка
- `StateMachine.extend(parentState, to)` - объявление состояния, наследованного от parentState и с возможными переходами в to
- `transitTo(targetState, ...args)` - переход в состояние targetState. Опционально - аргументы, которые попадут в коллбэк
- `currentState` - название текущего состояния
- `is(stateName)` - текущее состояние == stateName
- `can(stateName)` - возможно-ли перейти из текущего состояния в stateName
- `transitions()` - получить список состояний, в которые можно перейти из текущего
- `onEnter(stateName: string, cb: (...args: Array<any>) => void): () => void` - повесить коллбэк на вход в состояние
- `onLeave(stateName: string, cb: () => void): () => void` - повесить коллбэк на выход из состояния
## Пример использования
### Описание StateMachine
    export class ChildStateMachine extends StateMachine {
        // Описываем начальное состояние машины
        loading: boolean = false;
        mainWindowVisible: boolean = false;
        paymentWindowVisible: boolean = false;
        successMessageVisible: boolean = false;
        errorMessageVisible: boolean = false;
    
        // Описываем состояния, в которые можно пойти из начального
        @StateMachine.hide()
        protected get $next(): Array<string> { return ['loadingState']; }
    
        // Регистрация состояния loadingState, унаследованного от initial, возможно перейти в mainState
        @StateMachine.extend('initial', ['mainState'])
        private loadingState = {
            loading: true
        };
    
        @StateMachine.extend('initial', ['paymentState'])
        private mainState = {
            mainWindowVisible: true
        };
    
        @StateMachine.extend('mainState', ['mainState', 'successState', 'errorState'])
        private paymentState = {
            paymentWindowVisible: true
        };
    
        @StateMachine.extend('paymentState', ['paymentState', 'mainState'])
        private successState = {
            successMessageVisible: true
        };
    
        @StateMachine.extend('paymentState', ['paymentState', 'mainState'])
        private errorState = {
            errorMessageVisible: true
        };
    
        constructor() {
            super();
            this.rememberInitState();
        }
    }
### Использование StateMachine
    import { ChildFMS } from './child.fsm';
    
    // Создание экземпляра машины
    const f = new ChildStateMachine();
    
    // Регистрируем коллбэк на вход в состояние paymentState
    f.onEnter('paymentState', (paymentSum) => {
        console.log('on enter paymentState', paymentSum);
    });

    f.transitTo('loadingState');
    console.log(f);
    f.transitTo('mainState');
    console.log(f);
    // Переход в состояние с коллбэком, передаем сумму, например
    f.transitTo('paymentState', 1000);
    console.log(f);
    f.transitTo('successState');
    console.log(f.transitions());
    f.transitTo('mainState');
    console.log(f);
    
## Рекомендации
- Не забывать описывать в своих классах-потомках конструктор и $next.
- Делать адекватную цепочку состояний. 
Т.к. создание состояний реализовано с помощью наследования - результат применения какого-либо состояния является цепочкой последовательных мержей предыдущих состояний на родительское состояние машины.
Проще говоря - в initial-состоянии опишите вообще все, что может меняться в контексте состояний и задайте этому дефолтные значения, а каждое новое состояние пусть что-то меняет в предыдущем.
- Не надо делать параллельных ветвей состояний - между ними будет неудобно переходить. См картинку https://monosnap.com/file/KcASX734C1vNcibCd0pDUgpMLymSZo
В этом примере есть три вкладки - release, attach и remove. Так вот для каждой лучше созать свою StateMachine, чем городить такое дерево, где нарисованы далеко не все переходы.

    
## TODO
- Написать тесты
- Написать examples
- initial-состояние хранить так же как все остальные - избавиться от проверок на isInitial внутри реализации
- Добавить в callback входа информацию о состоянии, из которого перешли
- При сборке цепочки родительских состояний приделать проверку на циклы

## LICENCE
MIT