# report05

## Результат покрытия

[![Coverage Status](https://coveralls.io/repos/github/mkkazakova/lab05/badge.svg)](https://coveralls.io/github/mkkazakova/lab05)

## Подготовка

```
$ git config --global user.name "Masha"
$ git config --global user.email "mkkazakova@yandex.ru"
```

Вводим логин и токен

Создаём репозиторий и работаем в нём.

## I Часть

Копируем папку banking и её содержимое:

```
$ mkdir banking
$ cd banking
$ wget https://raw.githubusercontent.com/tp-labs/lab05/master/banking/Account.cpp
$ wget https://raw.githubusercontent.com/tp-labs/lab05/master/banking/Account.h
$ wget https://raw.githubusercontent.com/tp-labs/lab05/master/banking/Transaction.cpp
$ wget https://raw.githubusercontent.com/tp-labs/lab05/master/banking/Transaction.h
```

В файле Transaction.cpp допущена ошибка. Меняем Debit(to, sum + fee_) на Debit(from, sum + fee_)

```
$ nano Transaction.cpp
```

Пушим и коммитим:

```
$ cd ..
$ git add banking
$ git commit -m "added banking"
$ git push origin master
```

Вводим логин и токен

Подключим библиотеку gtest. Затем пушим и коммитим

```
$ mkdir third-party
$ git submodule add https://github.com/google/googletest third-party/gtest
$ cd third-party/gtest && git checkout release-1.8.1 && cd ../..
$ git add third-party/gtest
$ git commit -m "added gtest"
$ git push origin master
```

Вводим логин и токен


Создадим CMakeLists.txt для banking:

```
$ cd banking
$ cat >> CMakeLists.txt <<EOF
$ nano CMakeLists.txt
```

Содержимое файла CMakeLists.txt:

```
project(banking_lib)

if (NOT TARGET libbanking)
    add_library(libbanking STATIC
        /Account.cpp
        /Transaction.cpp
    )

    install(TARGETS libbanking
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
    )
endif(NOT TARGET libbanking)

include_directories()
```

```add_library``` Добавляет целевой объект библиотеки с именем libbanking для построения из исходных файлов, перечисленных в вызове командыБиблиотеки ```STATIC``` - это архивы объектных файлов для использования при компоновке других целей.

```install(TARGETS``` указаны правила установки целей из проекта

```ARCHIVE DESTINATION lib``` статическая, ```LIBRARY DESTINATION lib``` общие библиотеки


Пушим и коммитим

```
$ git add CMakeLists.txt
$ git commit -m "added CMakeLists.txt in banking"
$ git push origin master
```



Создадим CMakeLists.txt, который отвечает за работу тестов

```
$ cd ..
$ cat >> CMakeLists.txt << EOF
$ nano CMakeLists.txt 
```

Содержимое файла CMakeLists.txt:

```
cmake_minimum_required(VERSION 3.4)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(BUILD_TEST "Build tests" OFF)

if(BUILD_TESTS)
  add_compile_options(--coverage)
endif()

option(COVERAGE "Check coverage" ON)

project(banking)

add_library(banking STATIC banking/Account.cpp banking/Transaction.cpp)
target_include_directories(banking PUBLIC banking/)

target_link_libraries(banking gcov)

if(BUILD_TESTS)
  enable_testing()
  add_subdirectory(third-party/gtest)
  file(GLOB BANKING_TEST_SOURCES tests/test1.cpp)
  add_executable(check tests/test1.cpp)
  target_link_libraries(check banking gtest_main gmock_main)
  add_test(NAME check COMMAND check)
endif()

if (COVERAGE)
	target_compile_options(check PRIVATE --coverage)
	target_link_libraries(check --coverage)
endif()
```

```target_include_directories``` Указывает включаемые каталоги, которые будут использоваться при компиляции заданной цели banking

```target_link_libraries``` связывание banking и gcov

```GLOB``` сгенерирует список всех файлов, соответствующих выражениям подстановки, и сохранит его в переменной

```add_executable``` Добавляет в проект исполняемый файл, используя указанные исходные файлы

```add_test``` Добавляет тест под названием


Пушим и коммитим

```
$ git add CMakeLists.txt 
$ git commit -m "added CMakeLists.txt for tests"
$ git push origin master
```


## II Часть

Создаём папку tests. В ней файл test1.cpp.
```
$ mkdir tests
$ cd tests
$ cat >> test1.cpp << EOF
$ nano test1.cpp
```
Содержимое файла test1:

```
#include "Account.h"
#include "Transaction.h"

#include <gtest/gtest.h>
#include <gmock/gmock.h>

class AccountMock : public Account {
public:
    AccountMock(int id, int balance) : Account(id, balance) {}
    MOCK_CONST_METHOD0(GetBalance, int());
    MOCK_METHOD1(ChangeBalance, void(int diff)); 
    MOCK_METHOD0(Lock, void());
    MOCK_METHOD0(Unlock, void());
};

class TransactionMock : public Transaction {
public:
    MOCK_METHOD3(Make, bool(Account& from, Account& to, int sum));
};

TEST(Account, Mock) {
    AccountMock acc(1, 666);
    EXPECT_CALL(acc, GetBalance()).Times(1);
    EXPECT_CALL(acc, ChangeBalance(testing::_)).Times(2);
    EXPECT_CALL(acc, Lock()).Times(2);
    EXPECT_CALL(acc, Unlock()).Times(1);
    acc.GetBalance();
    acc.ChangeBalance(100); 
    acc.Lock();
    acc.ChangeBalance(100);
    acc.Lock(); 
    acc.Unlock();
}

TEST(Account, SimpleTest) {
    Account acc(1, 666);
    EXPECT_EQ(acc.id(), 1);
    EXPECT_EQ(acc.GetBalance(), 666);
    EXPECT_THROW(acc.ChangeBalance(200), std::runtime_error);
    EXPECT_NO_THROW(acc.Lock());
    acc.ChangeBalance(200);
    EXPECT_EQ(acc.GetBalance(), 866);
    EXPECT_THROW(acc.Lock(), std::runtime_error);
    EXPECT_NO_THROW(acc.Unlock());
}

TEST(Transaction, Mock) {
    TransactionMock tr;
    Account ac1(1, 100);
    Account ac2(2, 300);
    EXPECT_CALL(tr, Make(testing::_, testing::_, testing::_))
    .Times(5);
    tr.set_fee(200);
    tr.Make(ac1, ac2, 200);
    tr.Make(ac2, ac1, 300);
    tr.Make(ac1, ac1, 0); 
    tr.Make(ac1, ac2, -5); 
    tr.Make(ac2, ac1, 50); 
}

TEST(Transaction, SimpleTest) {
    Transaction tr;
    Account ac1(1, 100);
    Account ac2(2, 300);
    tr.set_fee(10);
    EXPECT_EQ(tr.fee(), 10);
    EXPECT_THROW(tr.Make(ac1, ac2, 40), std::logic_error);
    EXPECT_THROW(tr.Make(ac1, ac2, -5), std::invalid_argument);
    EXPECT_THROW(tr.Make(ac1, ac1, 100), std::logic_error);
    EXPECT_FALSE(tr.Make(ac1, ac2, 400));
    EXPECT_FALSE(tr.Make(ac2, ac1, 300));
    EXPECT_FALSE(tr.Make(ac2, ac1, 290));
    EXPECT_TRUE(tr.Make(ac2, ac1, 150));
}
```

```MOCK_METHOD1(ChangeBalance, void(int diff))``` Первым аргументом идет имя того самого метода, который мы ожидаем что будет выполнен в нашем будущем тесте. 
Далее идет сигнатура этого метода. Цифра 1 в названии макроса означает число аргументов у метода ChangeBalance - один.

```EXPECT_CALL``` позволяет описать, что должно произойти с методом за время теста

```EXPECT_EQ``` EQ - Equal, проверяет равны ли

```EXPECT_THROW(tr.Make(ac1, ac1, 100), std::logic_error);``` Проверяет, что tr.Make(ac1, ac1, 100) выдает исключение типа logic_error

```EXPECT_NO_THROW(acc.Lock());``` Проверяет, что (acc.Lock() не генерирует никаких исключений

```EXPECT_FALSE``` проверяет неверно ли это 

Пушим и коммитим:

```
$ cd ..
$ git add tests
$ git commit -m "added test1.cpp in tests"
$ git push origin master
```

Вводим логин и токен


## III Часть

Связываем репозиторий и средство покрытия кода

Создаём папку coverage. В ней lcov.info
```
$ mkdir coverage
$ cd coverage
$ cat >> lcov.info << EOF
```

Пушим и коммитим

```
$ cd .. 
$ git add coverage
$ git commit -m "add lcov in coverage"
$ git push origin master
```

```lcov``` — графический интерфейс для gcov. Он собирает файлы gcov для нескольких файлов с исходниками и создает комплект HTML-страниц с кодом и сведениями о покрытии. 
Вводим логин и токен

В папке .github/workflows создаём Action.yml и создаём сценарий

```
$ mkdir .github
$ cd .github
$ mkdir workflows
$ cd workflows
$ cat >> Action.yml << EOF
$ nano Action.yml
```

Содержимое файла Action.ymk:

```
name: CMake

on:
 push:
  branches: [master]
 pull_request:
  branches: [master]

jobs:
 build_Linux:

  runs-on: ubuntu-latest

  steps:
  - uses: actions/checkout@v3

  - name: Adding gtest
    run: git clone https://github.com/google/googletest.git third-party/gtest -b release-1.11.0

  - name: Install lcov
    run: sudo apt-get install -y lcov

  - name: Config banking with tests
    run: cmake -H. -B ${{github.workspace}}/build -DBUILD_TESTS=ON

  - name: Build banking
    run: cmake --build ${{github.workspace}}/build

  - name: Run tests
    run: |
      build/check
      cmake --build ${{github.workspace}}/build --target test -- ARGS=--verbose

  - name: Do lcov stuff
    run: lcov -c -d build/CMakeFiles/banking.dir/banking/ --include *.cpp --output-file ./coverage/lcov.info

  - name: Publish to coveralls.io
    uses: coverallsapp/github-action@v1.1.2
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}
```

```sudo apt-get install``` используется для загрузки последней версии нужного вам приложения из онлайн-хранилища программного обеспечения, на которое указывают ваши источники

```${{ secrets.GITHUB_TOKEN }}``` - синтаксис для ссылки на токен

```cmake -H. -B ${{github.workspace}}/build -DBUILD_TESTS=ON``` Подготовка процесса сборки, запись результата в файл

```cmake --build ${{github.workspace}}/build``` Старт сборки

```cmake --build ${{github.workspace}}/build --target test -- ARGS=--verbose``` Начинаем сборку test, которая будет транслировать результаты теста

```
$ git add Action.yml
$ git commit -m "add Action in workflows"
$ git push origin master
```

Вводим логин и токен



Регестрируемся на сайте https://coveralls.io

![image](https://user-images.githubusercontent.com/125077130/227451613-339f8dd1-a56e-4cd2-9612-7f2bfd0e52af.png)

Добавляем ссылку с сайта в README.md. Затем пушим и коммитим

```
$ cd ..
$ nano README.md 
$ git add README.md
$ git commit -m "README changed"
$ git push origin master
```

Вводим логин и токен


## Скрины

![image](https://user-images.githubusercontent.com/125077130/227450998-37198f2e-9ace-4f07-b507-5a4d9e15b726.png)

![image](https://user-images.githubusercontent.com/125077130/227451026-7c039002-ff8f-4f57-86bb-cf79b4643743.png)

![image](https://user-images.githubusercontent.com/125077130/227451089-3df744bd-12a7-43de-b0e6-ecc42732800c.png)

![image](https://user-images.githubusercontent.com/125077130/227451247-4251f7cd-df63-443f-ae17-93fc9310fcc3.png)

![image](https://user-images.githubusercontent.com/125077130/227451396-f7882d59-cd65-4a5f-8e32-0eeb1ac35bde.png)

![image](https://user-images.githubusercontent.com/125077130/227451425-87087044-ff92-418b-9ae3-f18f88f348e9.png)

![image](https://user-images.githubusercontent.com/125077130/227451440-544617ec-7b62-4c29-b527-7db4417bd89c.png)

![image](https://user-images.githubusercontent.com/125077130/227451461-de9ad9cb-827d-48b6-b6a8-8184acb4a973.png)

![image](https://user-images.githubusercontent.com/125077130/227451485-792fe91e-0038-4389-94cb-c253138475a1.png)






