# Отчет по курсовому проекту
## по курсу "Логическое программирование"

### студент: Инютин М. А.

## Результат проверки

| Преподаватель     | Дата         |  Оценка       |
|-------------------|--------------|---------------|
| Сошников Д.В. |  21.12.2020            |      5-         |
| Левинская М.А.|              |               |

> *Хороший реферат. В задании не находится кратчайшее отношение родства.*

## Введение

В ходе выполнения курсового проекта я познакомлюсь с форматом родословных деревьев, научусь преобразовывать этот фарм в понятный логическому языку, реализую естественно-языковый интерфейс для ответа на вопросы о степени родства двух людей в дереве. Для ответов на вопросы я буду использовать алгоритм поиска путей в графе.

## Задание

 1. Создать родословное дерево своего рода на несколько поколений (3-4) назад в стандартном формате GEDCOM с использованием сервиса MyHeritage.com 
 2. Преобразовать файл в формате GEDCOM в набор утверждений на языке Prolog, с использованием предикатов father(отец, потомок) и mother(мать, потомок)
 3. Реализовать предикат проверки/поиска деверя.
 4. Реализовать программу на языке Prolog, которая позволит определять степень родства двух произвольных индивидуумов в дереве
 5. [На оценки хорошо и отлично] Реализовать естественно-языковый интерфейс к системе, позволяющий задавать вопросы относительно степеней родства, и получать осмысленные ответы. 

## Получение родословного дерева

Я нашёл в сети несколько семейных деревьев Дональда Трампа, модифицировал их и экпортировал в формат GEDCOM.
Получилось дерево на 5 поколений из 30 человек.

## Конвертация родословного дерева

Я использовал `GNU bash` для решения этой задачи, потому что язык позволяет легко обрабатывать строки и файлы. Создадим пять дополнительных файлов, в которые будем писать только имена людей и их номера, предикаты father(отец, потомок) и предикаты mother(мать, потомок), предикаты male(человек) и female(человек) (это нужно для предиката деверя). Будем анализировать каждую строку файла. Если мы нашли идентификатор человека или его имя, то просто перепишем эту строку в первый файл. Если найдём поле вида *HUSB*, *WIFE* или *CHIL*, то найдём их имена из первого файла (искать по файлу из 50 строк лучше, чем по файлу из 500). Когда мы нашли всех троих членов семьи, то запишем во второй файл отца, а в третий мать (Prolog не любит чередующиеся предикаты). Не забудем сбросить флаг *CHIL*, чтобы можно было сделать семью многодетной. После нахождения нового идентификатора семьи сбросим все флаги. Теперь нужно просто объединить файлы с предикатами `father(отец, потомок)`, `mother(мать, потомок)`, `female(человек)` и `male(человек)` в чистый файл base.txt

## Предикат поиска родственника

Деверь - это брат мужа. Нам необходимо определить пол человека, чтобы неженатый человек тоже считался деверем (хотя это не предусмотрено вариантом). Реализуем поиск мужа таким предикатом:
```Prolog
husb(X, Y) :-
	mother(Y, Z),
	father(X, Z).
```
Теперь через мать мужа найдём его братьев. Конечно, самому себе нельзя стать братом, поэтому нужно делать дополнительную проверку.
```Prolog
brother(X, Y) :-
	male(X),
	mother(Z, X),
	mother(Z, Y),
	X \== Y.

brotherInLaw(X, Y) :-
	husb(Z, X),
	brother(Y, Z).
```
Примеры запросов:
```
?- brotherInLaw(X, 'MelanijaKnavs').
X = 'FrederickCTrumpJr' ;
X = 'RobertSTrump' ;
false.
```
Если у семьи несколько детей, то деверь будет выведен несколько раз.
```
?- brotherInLaw(X, 'IvanaMZelnickova').
X = 'FrederickCTrumpJr' ;
X = 'RobertSTrump' ;
X = 'FrederickCTrumpJr' ;
X = 'RobertSTrump' ;
X = 'FrederickCTrumpJr' ;
X = 'RobertSTrump' ;
false.
```

## Определение степени родства

Для нашождения родственников будем использовать поиск в глубину. Дополнительно определим предикат брата и сестры, опишем предикаты `move` для следующих связей:
```Prolog
%% Father
move(X, Y) :- father(X, Y).
%% Mother
move(X, Y) :- mother(X, Y).
%% Son
move(X, Y) :- father(Y, X).
%% Daughter
move(X, Y) :- mother(Y, X).
%% Brother
move(X, Y) :- brother(X, Y).
%% Sister
move(X, Y) :- sister(X, Y).
```

Поиск будет возвращать найденный путь, состоящий из людей, поэтому нужно восстановить родство по каждой паре людей в списке.
```Prolog
analyse_path([_], []) :- !.

analyse_path([X, Y | T], ['mother' | R]) :-
	mother(Y, X),
	analyse_path([Y | T], R).
```

Сам поиск в глубину:
```Prolog
prolong([X | T], [Y, X | T]) :-
	move(X, Y),
	not(member(Y, [X | T])).

dfs([X | T], X, [X | T]).

dfs(X, Y, R) :-
	prolong(X, Z),
	dfs(Z, Y, R).
```

Примеры запросов:
```
?- relative(['brother'], 'DonaldJTrump', X).
X = 'MarryaneTrump' ;
X = 'FrederickCTrumpJr' ;
X = 'ElizabethJTrump' ;
X = 'RobertSTrump' ;
false.

?- relative(['son', 'daughter', 'daughter'], 'DonaldJTrump', X).
X = 'DonaldSmith' ;
X = 'MaryMacauley' ;
false.

?- relative(['son'], X, 'DonaldJTrump').
X = 'BarronWTrump' ;
X = 'DonaldJTrumpJr' ;
X = 'EricFTrump' ;
false.

?- relative(['son', 'son', 'son', 'son'], 'BarronWTrump', X).
X = 'JohannesDrumpf' ;
X = 'KatherinaKober' ;
false.

?- relative(['brother-in-law'], X, 'MelanijaKnavs').
X = 'FrederickCTrumpJr' ;
X = 'RobertSTrump' ;
false.
```

Из-за сильного ветвления дерева (например, у Дональда Трампа много сестёр и братьев) поиск может выдавать одинаковые пути, связанные с братьями и сёстрами.
```
?- relative(X, 'ElizabethJTrump', 'DonaldJTrump').
X = [daughter, father, daughter, mother, brother] ;
X = [daughter, father, daughter, mother, brother, brother] ;
X = [daughter, father, daughter, mother] ;
X = [daughter, father, daughter, mother, brother, brother] ;
X = [daughter, father, daughter, mother, brother] ;
...
X = [sister, brother, brother, daughter, mother] ;
X = [sister, brother, brother, sister] ;
X = [sister, brother, brother] ;
X = [sister] ;
```

## Естественно-языковый интерфейс

Будем обрабатывать три типа предложений со следующей грамматикой:
```
Предложение --> вопрос объект сказуемое подлежащее
Предложение --> вопрос сказуемое объект подлежащее
Предложение --> сказуемое объект подлежащее
вопрос     |--> 'How', 'many'
вопрос     |--> 'Who'
объект     |--> степень родства
сказуемое  |--> глагол to have вопросом
сказуемое  |--> глагол to be
подлежащее |--> местоимение
подлежащее |--> имя
подлежащее |--> 'of', местоимение
подлежащее |--> 'of', имя
```

Грамматика использует словарь для анализа фразы. Разбор выражения реализуем с помощью предикатов `append`, а отвечать на вопросы будем с помощью `relative`:
```Prolog
%% How many brothers does Vasya have?
%% How many sisters does he have?
ask(L, RES) :-
	append([QUESTION1], L1, L),
	append([QUESTION2], L2, L1),
	append(OBJECT, L3, L2),
	append([ACTION1], L4, L3),
	append(AGENT, [ACTION2], L4),
	anQuest([QUESTION1, QUESTION2], R_QUEST),
	anObject(OBJECT, R_OBJ),
	anAction([ACTION1, ACTION2], R_ACT),
	anAgent(AGENT, R_AGENT),
	request(R_QUEST, R_OBJ, R_ACT, R_AGENT, RES).
```
Из-за сильного ветвления `findall` может считать одинаковых людей, поэтому нужно использовать `setof` для подсчёта количество родственников.
```Prolog
request('quest_count', OBJ, 'have', PERSON, RES) :-
	setof(X, relative([OBJ], X, PERSON), L),
	length(L, RES).
```

Примеры запросов:
```Prolog
?- ask(['How', 'many', 'sisters', 'does', 'DonaldJTrump', 'have'], R).
R = 2 ;
false.

?- ask(['Who', 'are', 'sisters', 'of', 'him'], R).
R = 'MarryaneTrump' ;
R = 'ElizabethJTrump' ;
false.

?- ask(['How', 'many', 'sons', 'did', 'MaryAMacleod', 'have'], R).
R = 3 ;
false.

?- ask(['Is', 'she', 'mother', 'of', 'DonaldJTrump']).
true.

?- ask(['How', 'many', 'brother-in-laws', 'does', 'IvanaMZelnickova', 'have'], R).
R = 2 ;
false.

?- ask(['Is', X, 'father', 'of', 'BarronWTrump']).
X = 'DonaldJTrump'.

?- ask(['Who', 'is', X, 'of', 'AlexanderMacleod'], R).
X = sons,
R = 'MalcolmMacleod' ;
X = son,
R = 'MalcolmMacleod' ;
false.
```

## Выводы

В ходе выполнения курсвого проекта я опробовал разные способы преобразования родословной GEDCOM в формат, читаемый для Prolog и реализовал один из них на GNU Bash. Я сделал естественно-языковой интерфейс для ответа на вопросы пользователя о степени родства двух людей в родословной.

По мере выполнения курсовой я узнавал новые факты о логическом языке (даже успел перейти с GNU Prolog на SWI Prolog) и улучшал свою технику программирования на нём. Я уверен, этот опыт пригодится при изучении и применении функционального программирования.
