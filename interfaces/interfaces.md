# Interfaces

## 1. Интерфейсы -- что это?
Интерфейс отвечает на вопрос *что это делает?*, то есть проще говоря какие у класса есть методы, а точнее:

1. Как называются методы?
2. Какие параметры принимают?
3. Что эти методы возвращают?

```python
from abc import ABC, abstractclassmethod

class RepositoryInterface(ABC): # <- Интерфейс репозитория(какого-то хранилища)
    @abstractclassmethod
    def find_all(self) -> []Foo: # <- метод `find_all`. Возвращает список всех хранимых объектов. Не принимает никаких параметров
        raise NotImplementedError()

    @abstractclassmethod
    def save(self, foo: Foo): # <- метод `save`. Принимает на вход какой-то объект на вход. Ничего не возвращает
        raise NotImplementedError()

```

Из имеющегося интерфейса мы можем понять следующее:

* Метод `find_all`, исходя из названия, должен возвращать список всех хранимых внутри репозитория объектов класса `Foo`
* Метод `save`, исходя из названия, должен сохранять новый объект класса `Foo`. Принимает на вход тот экземпляр класса, который нужно сохранить.

Интерфейс не имеет под собой никакой реализации, он лишь <u>описывает поведение</u>. 

Реализацию(или, как чаще говорят, имплементацию) интерфейса мы должны написать **сами**

## 2. Имплементация интерфейса
Чтобы имплементировать(реализовать) интерфейс, нам необходимо реализовать каждый метод интерфейса. Например, для реализации интерфейса `RepositoryInterface` из примера выше, мы должны отнаследоваться от него в дочернем классе(унаследовав таким образом методы `find_all` и `save`), после чего необходимо переопределить их в дочернем классе, добавив необходимую логику.

Создадим реализацию `RepositoryInterface` в виде локального репозитория, который будет хранить информацию в питонячьих списках, каждый элемент которого -- объект класса `Foo`

```python
class LocalRepository(RepositoryInterface): # наследуемся от интерфейса
    _storage: list[Foo] # private хранилище. Обращение снаружи класса запрещено

    def __init__(self): # создадим конструктор класса
        self._storage = [] # Проинициализируем пустое хранилище
	# Имплементация методов репозитория
    def find_all(self) -> list[Foo]:
        return self._storage

    def save(self, foo: Foo):
        self._storage.append(foo)
```

## 3. Зачем это нужно?
Интерфейс позволяет скрывать за собой реализацию, просто гарантируя, что у каждой реализации интерфейса точно есть методы с точно такой же сигнатурой(сигнатура метода -- это его название, параметры, которые он принимает и тип возвращаемого значения).

```python
from pydantic import BaseModel

class Application(BaseModel):
    repository: RepositoryInterface

    class Config:
        arbitrary_types_allowed = True
```

		
В нашем приложении есть интерфейс репозитория, и нас не волнует реализация, мы точно знаем, что у репозитория точно есть два метода: `find_all` и `save`

```python
repo = LocalRepository()

app = Application(repository=repo)

app.repository.save(foo1)
app.repository.save(foo2)
```

Интерфейс, кроме всего прочего, может иметь множество реализаций, которые мы можем модульно заменять в приложении без каких-либо особых проблем:

```python
class SQLRepository(RepositoryInterface): 
	# Имплементация методов репозитория
    def find_all(self) -> list[Foo]:
		self.db.cusor.execute("SELECT name, value FROM foo")

    def save(self, foo: Foo):
		self.db.cursor.execute(f"INSERT INTO foo (name, value) VALUES ({foo.name}, {foo.value})")

repo = SQLRepository()

app = Application(repository=repo)

app.repository.save(foo1)
app.repository.save(foo2)
```

## 4. Причем тут gRPC?
gRPC в данной ситуации является языком описания интерфейсов(Interface Definition Language -- IDL) и Data Transfer Object(DTO).

В протофайлах каждый `service` -- есть <u>интерфейс</u>. Каждый `rpc` -- <u>метод</u>, каждый `message` -- <u>DTO</u>

> Data Transfer Object, DTO в питонячьем представлении -- класс без методов, а только с полями. Служит своеобразным "контейнером" для данных

Для популярных языков существуют компиляторы протофайлов(grpcio-tools), которые преобразуют протофайлы в понятные для Python структуры данных, например:

```protobuf
service CommandService {
	rpc GetReply(IncomingMessage) returns ReplyMessage
}
```

Преобразуется во что-то вроде:

```python
class CommandServiceServicer:
	def GetReply(self, request: IncomingMessage, context: grpc.Context) -> ReplyMessage:
		raise NotImplementedError()
```

а сами `message`(DTO):

```protobuf
// Telegram User's message

message IncomingMessage {

// Telegram User's ID

int64 user_id = 1;

// Incoming message text

string text = 2;

}

message ReplyMessage {

// Telegram User's ID

int64 user_id = 1;

// Reply message text from services

string text = 2;

}
```

Преобразуется в самые обыкновенные:

```python
class IncomingMessage:
	user_id: int
	text: str
	
	
class ReplyMessage:
	user_id: int
	text: str
```

И от нас потребуется написать лишь простейшую имплементацию:

```python
class CommandService(CommandServiceServicer):
	def GetReply(self, request: IncomingMessage, context) -> ReplyMessage:
		return ReplyMessage(user_id=request.user_id, text=request.text)
```
