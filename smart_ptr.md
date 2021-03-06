# Умные указатели
В языке Си управление памятью осуществляется с помощью операторов new и delete. Программисту необходимо следить самостоятельно за тем чтобы выделенная память была освобождена. Работа с памятью может быть опасна и приводить к ошибкам:
* утечки памяти
* разыменовывание нулевого указателя, либо обращение к неициализированной области памяти
* удаление уже удаленного объекта

Существует техника управления ресурсами посредством локальных объектов, называемая RAII. То есть, при получении какого-либо ресурса, его инициализируют в конструкторе, а, поработав с ним в функции, корректно освобождают в деструкторе. Ресурсом может быть что угодно, к примеру файл, сетевое соединение, блок памяти.
Представим себе класс, который содержит в себе кучу ресурсов:
```cpp
class Foo {
    int *data1;
    double *data2;
    char *data3;
public:
    Foo() {
        data1 = new int(5);
        ...
    }
    ~Foo() {
        delete data1;
        ...
    }
}
```
Нужно проконтролировать чтобы это все успешно создалось и удалилось... u

Теперь представьте, что полей не 3, а 30, а значит в деструкторе придется для всех них вызывать delete. А если мы второпях добавим новое поле, но забудем его убить в деструкторе, то последствия будут негативными. В итоге получается класс, нагруженный операциями выделения\освобождения памяти, да еще и непонятно, все ли было правильно удалено.
Поэтому предлагается использовать умные указатели: это объекты, которые хранят указатели на динамически аллоцированные участки памяти произвольного типа. Причем они автоматически очищают память по выходу из области видимости.
 
## unique_ptr
Простой тип умного указателя.
* Объект уничтожается при выходе из области видимости
* Нельзя создать копию unique_prt

```cpp
std::unique_ptr<int> x_ptr(new int(42));
std::unique_ptr<int> y_ptr;

// ошибка при компиляции
y_ptr = x_ptr;
// ошибка при компиляции
std::unique_ptr<int> z_ptr(x_ptr);
```

```cpp
#include <iostream>
#include <memory>
using namespace std;
class Element
{
public:
    Element()
    {
        cout << "Element ctor" << endl;
    }
    ~Element()
    {
        cout << "Element dtor" << endl;
    }
    void someFunction()
    {
        cout << "Hello" << endl;
    }
};

int main()
{
    {
        //первый способ
        unique_ptr<Element> element1(new Element());
        //второй способ exception safe предпочтительней
        unique_ptr<Element> element2 = make_unique<Element>();
        element1->someFunction();
        element2->someFunction();
    }

    return 0;
}
```


## shared_ptr
Умный указатель с подсчетом ссылок. Что это значит. Это значит, что где-то есть некая переменная, которая хранит количество указателей, которые ссылаются на объект. Если эта переменная становится равной нулю, то объект уничтожается. Счетчик инкрементируется при каждом вызове либо оператора копирования либо оператора присваивания. Так же у shared_ptr есть оператор приведения к bool, что в итоге дает нам привычный синтаксис указателей, не заботясь об освобождении памяти.

![](https://habrastorage.org/getpro/habr/post_images/ede/09d/f71/ede09df7175a46d8097a7613a576c9b3.png)


```cpp
int main()
{
    {
        shared_ptr<Element> sh_el0;
        {
            shared_ptr<Element> sh_el1 = make_shared<Element>();
            weak_ptr<Element> weak_el = sh_el1;
            sh_el0 = sh_el1;
        }
    }
    return 0;
}
```

Ещё пример использования:
```cpp
#include <iostream>
#include <memory>
int test() 
{
    std::shared_ptr<MyObject> p1(new MyObject);
    std::shared_ptr<MyObject> p2;    
    p2 = p1;    
    if (p2)
        std::cout << "Hello, world!\n";
}
```

 Мы можем передавать этот указатель в функцию:
```cpp
int test(std::shared_ptr<MyObject> p1) 
{
    // Делаем что-то
}
```
Если вы передаете указатель по ссылке, то счетчик не будет увеличен. Вы должны быть уверены, что объект MyObject будет жив, пока будет выполняться функция test.


## weak_prt

Этот указатель также, как и shared_ptr начал свое рождение в проекте boost, затем был включен в C++ Technical Report 1 и, наконец, пришел в новый стандарт.

Данный класс позволяет разрушить циклическую зависимость, которая, несомненно, может образоваться при использовании shared_ptr. Предположим, есть следующая ситуация (переменные-члены не инкапсулированы для упрощения кода)
```cpp
class Bar;

class Foo
{
public:
    Foo() { std::cout << "Foo()" << std::endl; }
    ~Foo() { std::cout << "~Foo()" << std::endl; }
    std::shared_ptr<Bar> bar;
};

class Bar
{
public:
    Bar() { std::cout << "Bar()" << std::endl; }
    ~Bar() { std::cout << "~Bar()" << std::endl; }
    std::shared_ptr<Foo> foo;
};

int main()
{
    auto foo = std::make_shared<Foo>();
    foo->bar = std::make_shared<Bar>();
    foo->bar->foo = foo;
    return 0;
}
```
Как видно, объект foo ссылается на bar и наоборот. Образован цикл, из-за которого не вызовутся деструкторы объектов. Для того чтобы разорвать этот цикл, достаточно в классе Bar заменить shared_ptr на weak_ptr.
