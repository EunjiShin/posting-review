# 1. OneToMany 단방향 매핑

## JPA 에서 연관관계 설정

- 객체지향관점: 일대다 단방향이 깔끔 → 부가 업데이트 쿼리가 무조건 나가게 됨.
- 기술 JPA 관점: 다대일 단방향이 깔끔 → 객체의 입장에서 참조관계가 DB 테이블 구조 처럼 됨.
- 합의: 다대일 양방향 → 편의 메소드에 대한 고려를 해야함.

## 배경

- 양방향을 설정하면 편의 메소드를 고려해야하며, 순환참조에 빠지지 않도록 주의해야 한다.
- 양방향을 설정하는 것(서로간의 단방향을 2개를 설정하는 것)은 객체지향의 입장에서도 약간 이상하다고 느껴짐
- 개발자가 신경써야할 포인트가 늘어나기 때문에 실수할 여지가 많아짐
- 따라서 단방향 설정이 좋다고 하는데 다대일 단방향은 객체처럼 객체를 사용하기 어려움
- 그렇다면 일대다 단방향은 성능 면에서 안 좋다고 하는데 왜일까?
- 어떤 부가쿼리가 나가는 것일까?

## 테스트 내용

- OneToMany 단방향 설정시 `JoinColumn(name = "")`을 설정한 경우와 설정하지 않은 경우를 비교
- 같은 테스트로 진행하지만 쿼리문이 다르게 동작함

## 참고

- 쿼리문에서 user, delete_history 테이블 등 불필요한 정보는 배제하고 글을 작성함
- `@OneToMany`의 fetch 기본설정은 LAZY 이다.

## 테스트 코드

- Question 객체를 하나 만들고 Question에 answer1, answer2를 등록하는 테스트 설정
- 나중에 answer1의 답변 내용을 `답변1`에서 `답변1을 바꿈`으로 수정
- 통과하는 테스트

```java
@DataJpaTest
class QuestionTest {
    /* ...각종 테스트 위한 설정 ... */
    @DisplayName("단방향 조회")
    @Test
    void oneToManyOneWay() {
        // 기존 설정 데이터를 저장
        Question question = new Question("제목1", "질문1");
        User user = new User("유저아이디", "0000", "유저1", "유저메일");
        Answer answer = new Answer(user, "답변1");
        Answer answer2 = new Answer(user, "답변2");
        answerRepository.save(answer);
        answerRepository.save(answer2);
        questionRepository.save(question);
        question.addAnswer(answer);
        question.addAnswer(answer2);

        System.out.println("--------저장----------");
        testEntityManager.flush();
        testEntityManager.detach(question); // 중간 select 쿼리문을 보기 위한 설정
        System.out.println("--------매핑정보확인----------");

        // 가저올 때, 연관관계가 매핑되어있는지 확인 및 답변을 수정
        Question persistQuestion = questionRepository.findById(question.getId()).get();
        List<Answer> answers = persistQuestion.getAnswers();
        assertThat(answers).hasSize(2);

        Answer firstAnswer = answers.get(0);
        String previousContent = firstAnswer.getContents();
        System.out.println("----------수정---------");
        answer.setContents("답변1을 바꿈");
        assertThat(previousContent).isEqualTo("답변1");

        testEntityManager.flush();
        testEntityManager.clear();

        // 최종 변경된 내용을 다시 조회
        System.out.println("-----------다시조회----------");
        Question changedQuestion = questionRepository.findById(question.getId()).get();
        List<Answer> changedAnswers = changedQuestion.getAnswers();
        Answer changedFirstAnswer = changedAnswers.get(0);
        String changedContent = changedFirstAnswer.getContents();
        assertThat(changedContent).isEqualTo("답변1을 바꿈");
        testEntityManager.flush();
    }
}
```

## 1-1. JoinColumn 을 설정하지 않은 경우

```java
@Entity
public class Question {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    private String contents;
    private Long writerId;
    private boolean deleted = false;
    @OneToMany
    //@JoinColumn(name = "answer_id") 
    private List<Answer> answers = new ArrayList<>();
}
```

### 테이블 생성시

- answer, question 테이블의 각각의 id를 FK로 설정하여 question_answers 테이블을 생성

```sql
create table answer (
    id bigint generated by default as identity,
    contents varchar(255),
    deleted boolean not null,
    question_id bigint,
    writer_id bigint,
    primary key (id)
)

create table question (
    id bigint generated by default as identity,
    contents varchar(255),
    deleted boolean not null,
    title varchar(255),
    writer_id bigint,
    primary key (id)
)

create table question_answers (
    question_id bigint not null,
    answers_id bigint not null
)

alter table question_answers
    add constraint UK_4qtn1pf4ea4ougou3ewipk9qx unique (answers_id)
    Hibernate:

alter table question_answers
    add constraint FKnr1xcvup15w03kboejfervq1y
        foreign key (answers_id)
            references answer
    Hibernate:

alter table question_answers
    add constraint FKlglw0r110cw97aje0b0pa4q51
        foreign key (question_id)
            references question
```

### 저장시

- answer, question에 각 정보를 저장하고 List<Answer>의 갯수만큼 question_answers 테이블에 추가

```sql
 insert 
    into
        answer
        (id, contents, deleted, question_id, writer_id) 
    values
        (null, ?, ?, ?, ?)
insert 
    into
        question
        (id, contents, deleted, title, writer_id) 
    values
        (null, ?, ?, ?, ?)
insert
    into
        question_answers
        (question_id, answers_id)
    values
        (?, ?)
insert
    into
        question_answers
        (question_id, answers_id)
    values
        (?, ?)
```

### Question 조회시

- 2번의 select 쿼리문이 발생
  1. question 을 조회하는 쿼리문
  2. question_answers 테이블에서 answer 과 inner join 으로 `List<Answer>` 을 만드는 쿼리문

```sql
--------매핑정보확인----------
    select
        question0_.id as id1_2_0_,
        question0_.contents as contents2_2_0_,
        question0_.deleted as deleted3_2_0_,
        question0_.title as title4_2_0_,
        question0_.writer_id as writer_i5_2_0_
    from
        question question0_
    where
        question0_.id=?

    select
        answers0_.question_id as question1_3_0_,
        answers0_.answers_id as answers_2_3_0_,
        answer1_.id as id1_0_1_,
        answer1_.contents as contents2_0_1_,
        answer1_.deleted as deleted3_0_1_,
        answer1_.question_id as question4_0_1_,
        answer1_.writer_id as writer_i5_0_1_ 
    from
        question_answers answers0_ 
    inner join
        answer answer1_ 
            on answers0_.answers_id=answer1_.id 
    where
        answers0_.question_id=?
```

### 답변1을 수정시

- 업데이트 쿼리문 1개가 발생

```sql
----------수정---------
    update
        answer 
    set
        contents=?,
        deleted=?,
        question_id=?,
        writer_id=? 
    where
        id=?
```

## 정리

- question 과 answer 를 조합할 수 있는 또다른 추가 테이블이 생성됨 (question_answers 테이블)
- `List<Anwser>`의 수만큼 update 쿼리가 부가적으로 발생함
- question 조회시 2개의 쿼리문이 발생함

## 1-2. JoinColumn(name = ) 을 설정한 경우

```java

@Entity
public class Question {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  private String title;
  private String contents;
  private Long writerId;
  private boolean deleted = false;
  @OneToMany
  @JoinColumn(name = "answer_id")
  private List<Answer> answers = new ArrayList<>();
}
```

### 테이블 생성시

- question_answer 테이블을 생성하지 않고 answer 을 question 의 외래키로 설정함

```sql
    create table answer (
       id bigint generated by default as identity,
        contents varchar(255),
        deleted boolean not null,
        question_id bigint,
        writer_id bigint,
        answer_id bigint,
        primary key (id)
    )
    
    create table question (
       id bigint generated by default as identity,
        contents varchar(255),
        deleted boolean not null,
        title varchar(255),
        writer_id bigint,
        primary key (id)
    )
    
    alter table answer 
       add constraint FKthayxt9ie8c2q6lflw7n1mbdn 
       foreign key (answer_id) 
       references question
```

### 저장시

- answer 에 2개의 insert 쿼리문 발생
- question 에 1개의 insert 쿼리문 발생
- `List<Answer>`의 수 만큼 update 쿼리문이 발생

```sql
insert 
    into
        answer
        (id, contents, deleted, question_id, writer_id) 
    values
        (null, ?, ?, ?, ?)
 
insert 
    into
        answer
        (id, contents, deleted, question_id, writer_id) 
    values
        (null, ?, ?, ?, ?)

insert 
    into
        question
        (id, contents, deleted, title, writer_id) 
    values
        (null, ?, ?, ?, ?)

update
    answer
set
    answer_id=?
where
        id=?

update
    answer
set
    answer_id=?
where
    id=?
```

### Question 조회시

- question 을 조회하는 쿼리문이 발생
- answer 을 조회하는 쿼리문이 발생

```sql
--------매핑정보확인----------
    select
        question0_.id as id1_2_0_,
        question0_.contents as contents2_2_0_,
        question0_.deleted as deleted3_2_0_,
        question0_.title as title4_2_0_,
        question0_.writer_id as writer_i5_2_0_ 
    from
        question question0_ 
    where
        question0_.id=?

    select
        answers0_.answer_id as answer_i6_0_0_,
        answers0_.id as id1_0_0_,
        answers0_.id as id1_0_1_,
        answers0_.contents as contents2_0_1_,
        answers0_.deleted as deleted3_0_1_,
        answers0_.question_id as question4_0_1_,
        answers0_.writer_id as writer_i5_0_1_ 
    from
        answer answers0_ 
    where
        answers0_.answer_id=?
```

### 수정시

- answer 을 수정하는 쿼리문 1개 발생

```sql
----------수정---------
    update
        answer 
    set
        contents=?,
        deleted=?,
        question_id=?,
        writer_id=? 
    where
        id=?
```

## 정리

- 부가 테이블을 생성되지 않고 answer 을 외래키로 설정함
- 하지만 `List<Answer>`의 수만큼 부가적인 update 쿼리문이 발생함
- 조회시 question 을 만드는 쿼리문 1개, `List<Answer>`을 만드는 쿼리문 1개가 발생함
  - 여기서는 `@OneToMany`기본 설정인 LAZY 때문에 2개의 쿼리문이 발생
  - EAGER 로 변경하면 left join 이 발생하는 1개의 쿼리문이 발생