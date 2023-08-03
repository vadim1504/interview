Всего в хибернейте существует 4 способа управлять загрузкой дочерних коллекций:

SELECT
JOIN
SUBSELECT
BATCH

Каждая из стратегий не лишена недостатков и достоинств. Рассмотрим их на примере интернет магазина. Представим, что наш интернет магазин продает только книги, а пользователи могут оставлять к ним комментарии. В базе будет всего 2 таблицы book и comment.
Доменные объекты опишем так:

@Entity
@Table(name="book")
public class SelectBook implements java.io.Serializable {
    @Id
    @SequenceGenerator(name = "book_seq")
    @Column(name = "book_id")
    Long id;    
    String title;    
    String author;    
    Set<SelectComment> comments;
    

@Entity
@Table(name="comment")
public class SelectComment {

    @Id
    @SequenceGenerator(name = "comment_seq")
    @Column(name = "comment_id")
    Long id;

    @ManyToOne
    SelectBook book;
    String text;    
    

Данные которые лежат в БД:

Таблица book

book_id	author	title
46	J. K. Rowling	Garry Potter
47	Joseph Heller	Catch-22: 50th Anniversary Edition
48	J.R.R. Tolkien	The Lord of the Rings

Таблица comment

comment_id	text	comment_id
1	Best book I have ever read!	46
2	Whoever wants a lovely interesting…	46
3	I’ve loved Harry Potter since it first came out…	46
4	Great read even for the second time!	47
5	I love it	47
6	Life is a comedy to those who think, and a tragedy…	47


Select

Это самая распространенная стратегия загружки дочерних данных, именно ее используется хибернейт по умолчанию, если стратегия загрузки не указана явно. Она выполняет загрузку дочерней коллекций с помощью отдельного select запроса.

Пример (укажем только связь с комментариями аннотацией @OneToMany):

    @OneToMany(fetch = FetchType.EAGER, mappedBy = "book")
    Set<SelectComment> comments;

Получим все книги с помощью HQL:

List<Book> books = session.createQuery("from Book b").list();
Что приведет к следующим запросам в БД:

Hibernate: 
    select
        selectbook0_.book_id as book_id1_0_,
        selectbook0_.author as author2_0_,
        selectbook0_.title as title3_0_ 
    from
        book selectbook0_ 
Hibernate: 
    select
        comments0_.book_book_id as book_boo3_1_0_,
        comments0_.comment_id as comment_1_1_0_,
        comments0_.comment_id as comment_1_1_1_,
        comments0_.book_book_id as book_boo3_1_1_,
        comments0_.text as text2_1_1_ 
    from
        comment comments0_ 
    where
        comments0_.book_book_id=?
Hibernate: 
    select
        comments0_.book_book_id as book_boo3_1_0_,
        comments0_.comment_id as comment_1_1_0_,
        comments0_.comment_id as comment_1_1_1_,
        comments0_.book_book_id as book_boo3_1_1_,
        comments0_.text as text2_1_1_ 
    from
        comment comments0_ 
    where
        comments0_.book_book_id=?
Hibernate: 
    select
        comments0_.book_book_id as book_boo3_1_0_,
        comments0_.comment_id as comment_1_1_0_,
        comments0_.comment_id as comment_1_1_1_,
        comments0_.book_book_id as book_boo3_1_1_,
        comments0_.text as text2_1_1_ 
    from
        comment comments0_ 
    where
        comments0_.book_book_id=?

Итого выполнилось 4 запроса: 1 запрос на получение всех книг и 3 запроса комментариев для каждой книги. Чем больше книг, тем больше паразитных запросов будет выполнено к БД. Это проблема называется проблемой N+1 запроса (см. https://secure.phabricator.com/book/phabcontrib/article/n_plus_one/). Такой способ выборки данных удобен, если комментарии запрашиваются для некоторых книг. Если же необходимо получить комментарии для всех книг участвующих в запросе – стратегия SELECT самый плохой способ.

Достоинства
Рабочая стратегия без подводных камней для данных небольшого объема.
Недостатки
Проблема N+1 запроса, которая при больших объемах данных может существенно снизить скорость.


JOIN

Чтобы избавиться от проблемы N+1 запроса, можно использовать стратегию JOIN, которая объединяет родительскую и дочернюю таблицы с помощью оператора join.

Пример:

session.createQuery("select b from JoinBook b join fetch b.comments comments")
Итого в логе видим только 1 запрос к БД:

Hibernate: 
    select
        joinbook0_.book_id as book_id1_0_0_,
        joinbook0_.author as author2_0_0_,
        joinbook0_.title as title3_0_0_,
        comments1_.book_book_id as book_boo3_1_1_,
        comments1_.comment_id as comment_1_1_1_,
        comments1_.comment_id as comment_1_1_2_,
        comments1_.book_book_id as book_boo3_1_2_,
        comments1_.text as text2_1_2_ 
    from
        book joinbook0_ 
    left outer join
        comment comments1_ 
            on joinbook0_.book_id=comments1_.book_book_id 
    where
        joinbook0_.book_id=?

Особенности использования
Объявление данной стратегии применит режим джойнов только для запросов поиска сущностей по id: session.find(class, id).

Для остальных запросов установленный режим будет игнорироваться (http://stackoverflow.com/questions/36796798/why-hibernate-sometimes-ignores-fetchmode-join), поэтому в hql необходимо явно указывать таблицы для джойна (более того, когда хибернейт встречает join в запросе, он игнорирует любую установленную ранее стратегию и забирает данные в режиме JOIN).

Работа стратегия Join подразумевает энергичную загрузку данных, поэтому даже если указать ленивую (FetchType = LAZY), параметр будет проигнорирован и данные загрузятся сразу же.

Недостатки стратегии
Большой паразитный размер ответа БД. Основным недостатком данной стратегии является особенность работы оператора join, который при соединении второй таблицы выполняет декартово произведение строк. Это означает, что если у нас есть 2 книги по 3 комментария, результат выборки будет содержать 6 строк. Что влечет за собой существенное увеличение размера ответа БД.

Отсутствие пейджинации в sql-запросе. Исходя из первого недостатка, в выборке будут дублирующиеся книги, что не позволяет использовать пейджинацию на уровне запросов. Поэтому при указании setMaxResults(), setFirstResult(), хибернейт сначала скачает две таблицы полностью (book и comments), а только потом самостоятельно отделит дубли и применит пейджинацию. Если представить, что в базе содержится 50 000 книг, то запрос 10 книг с пейджинацией, приведет к скачиванию всей таблицы book, то есть 50 000 книг!


Batch

Один из способов получения данных без недостатков стратегии JOIN - использование режима SELECT с пакетной выборкой зависимых коллекций. Это способ отличается от стратегии SELECT только тем, что зависимые коллекции загружаются не одиночными запросами, а пакетно. Где размером пакета управляет аннотация @BatchSize:

    @OneToMany(fetch = FetchType.EAGER, mappedBy = "book")
    @Fetch(FetchMode.SELECT)
    @BatchSize(size = 5)
    Set<BatchComment> comments;

Запросы:

Hibernate: 
    select
        batchbook0_.book_id as book_id1_0_,
        batchbook0_.author as author2_0_,
        batchbook0_.title as title3_0_ 
    from
        book batchbook0_
Hibernate: 
    select
        comments0_.book_book_id as book_boo3_1_1_,
        comments0_.comment_id as comment_1_1_1_,
        comments0_.comment_id as comment_1_1_0_,
        comments0_.book_book_id as book_boo3_1_0_,
        comments0_.text as text2_1_0_ 
    from
        comment comments0_ 
    where
        comments0_.book_book_id in (
            ?, ?, ?
        )
Данная стратегия позволяет использовать пейджинацию, т.к. первым выполняется запрос на получение родительских объектов, где возможно применение интервалов к выборке. А затем, для полученных объектов, запрашиваются дочерние коллекции.

Достоинства
Решает проблему N+1 не создавая других: нет проблемы Декартова произведения, работает пейджинация в самом sql запросе.
Недостатки
Необходимо вручную устанавливать размер пакета. Не всегда ясно чему он должен быть равен. Это скорее не недостаток, а небольшое неудобство.


Subselect

Как в предыдущем, для данного режима ленивая загрузка игнорируется. Всего формируется 2 запроса, первый для получения книг, второй - для получения комментариев с подзапросом.

Включение режима subselect:

    @OneToMany(fetch = FetchType.EAGER, mappedBy = "book")
    @Fetch(FetchMode.SUBSELECT)
    Set<SelectComment> comments;
Пример:

Hibernate: 
    select
        selectbook0_.book_id as book_id1_0_,
        selectbook0_.author as author2_0_,
        selectbook0_.title as title3_0_ 
    from
        book selectbook0_ limit ?
Hibernate: 
    select
        comments0_.book_book_id as book_boo3_1_1_,
        comments0_.comment_id as comment_1_1_1_,
        comments0_.comment_id as comment_1_1_0_,
        comments0_.book_book_id as book_boo3_1_0_,
        comments0_.text as text2_1_0_ 
    from
        comment comments0_ 
    where
        comments0_.book_book_id in (
            select
                selectbook0_.book_id 
            from
                book selectbook0_
        )
Недостатки
Отсутствие пейджинации. Хотя стратегии Subselect ничто не мешает использовать в подзапросе пейджинацию из родительского запроса, в хибернейте это не работает. Баг заведен в 2007 году https://hibernate.atlassian.net/browse/HHH-2666 и не исправлен даже в версии 5.2.3.Final. Более подробно можно почитать здесь http://www.christophbrill.de/de_DE/hibernate-fetch-subselect-performance/
Достоинства
Всего 2 запроса к БД.
Нет проблемы Декартова произведения.
Динамический выбор стратегии загрузки
Установленный один раз режим запросов будет распространяться на все существующие и новые запросы. На практике же часто необходима использовать в разных запросах различные стратегии загрузки. Небольшую свободу, конечно, предоставляет режим JOIN, позволяя определять коллекции для загрузки прямо в запросах, но в целом удобного выбора стратегий в хибернейте нет.

https://dshuplyakov.github.io/hibernate-loading-strategies