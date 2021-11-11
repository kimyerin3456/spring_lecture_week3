# spring_lecture_week3

## 이번강의 목적  

- 기존의 서비스들은 데이터베이스의 의존성이 높았음.</br> 
  => 자바의 객체지향성인 느낌을 잘 살리지 못한 부분이 안타까워서 새로운 기술이 발전하게 됨. (DB는 데이터 저장용도로만 사용하고 자바는 객체지향적인 시스템을 잘 나타낼 수 있도록)


---
### 데이터 영속화
- ORM   
  - O(object)R(Relation)M(Mapping)  
=> 객체지향적/데이터베이스 테이블에 들어가는 것/자바에서의 클래스를 매핑

* 데이터베이스의 벤더와 어플리케이션 프로그램 간의 의존성이 떨어지게 된다.

- H2 Database  
  - 메모리 데이터베이스로 예제 프로젝트에서 주로 사용됨
  - console 설정을 하면 웹 기반으로 DBMS 화면도 볼 수 있음


- JPA, H2 의존성 추가
```groovy
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'com.h2database:h2'
```

- JPA의 관리를 하기 위해 Article 도메인을 엔티티로 변경
```java
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity
public class Article {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  Long id;
  String title;
  String content;

  public void update(String title, String content) {
    this.title = title;
    this.content = content;
  }
}

```
* 룸북 활용
* @AllArgsConstructor -> 빌더때문에 넣어준 것으로 이게 없으면 오류가 남.
* @NoArgsConstructor ->jap가 Article 객체를 새로 만들거나 할 때 기본생성자를 사용하기 때문에 넣어준 것.

---

- JPA 쿼리를 사용하기 위해 Repository 객체 생성
- CrudRepository<T, ID> 에서 T는 엔티티 객체 타입, ID는 T 타입의 ID 타입을 넣는다.

```java
public interface ArticleRepository extends CrudRepository<Article, Long> { }
```

- ArticleService 코드를 ArticleRepository 객체와 연계될 수 있도록 변경
```java

@RequiredArgsConstructor
@Service
public class ArticleService {
  private final ArticleRepository articleRepository;

  public Long save(Article request) {
    return articleRepository.save(request).getId(); //세이브 구현
  }

  public Article findById(Long id) {
    return articleRepository.findById(id).orElse(null); //조회하는 코드 구현
  }

  @Transactional // 도메인이 끝날시, 이 내용을 데이터베이스에 넣어주는 
  public Article update(Article request) {
    Article article = this.findById(request.getId());
    article.update(request.getTitle(), request.getContent()); //가져온 애들을 업데이트하는 함수

    return article;
  }

  public void delete(Long id) {   //지우는 코드 구현
    Article article = this.findById(id); //바로 아이디를 삭제해도 됨
    articleRepository.delete(article); 
  }
}
```

- 엔티티로 등록을 하게 되면 엔티티에 등록된 update 객체는 계속 관리,감독을 하고 있음
- 값이 바뀔 경우, JPA가 알아서 데이터베이스에 있는 내용을 바꿔준다. 
- JPA는 하나의 트랜젝션이 끝날때 이 값을 바꿔준다.

--- 

- Update 함수 코드를 보면 영속화의 징검다리인 Repository 클래스를 활용하지 않고 domain 클래스에 바로 접근해 업데이트 하고 있다.
- JPA는 Persistance Context 라는 논리적 공간에서 엔티티들을 캐싱하고 관리한다.
- JPA는 하나의 트랜잭션이 끝나게 되면 수정된 domain 내용들을 자동으로 확인하며 실제 데이터베이스 벤더에 저장한다


### 트랜잭션

- 시스템을 만들때 원자성이 보존되는 어떤 하나의 업무단위
- update = 조회 + 업데이트 두개의 업무가 존재
- 실무에서 트랜잭션을 정의하고 관리하는 것은 굉장히 중요

--- 
# 정리중!!

### CRUD API 만들기

- ArticleController 객체에 PUT, DELETE API 추가
```java
class ArticleController {
    // ...
  @PutMapping("/{id}")
  public Response<ArticleDto.Res> put(@PathVariable Long id, @RequestBody ArticleDto.ReqPut request) {
    Article article = Article.builder()
            .id(id)
            .title(request.getTitle())
            .content(request.getContent())
            .build();

    Article articleResponse = articleService.update(article);

    ArticleDto.Res response = ArticleDto.Res.builder()
            .id(String.valueOf(articleResponse.getId()))
            .title(articleResponse.getTitle())
            .content(articleResponse.getContent())
            .build();

    return Response.<ArticleDto.Res>builder().code(ApiCode.SUCCESS).data(response).build();
  }

  @DeleteMapping("/{id}")
  public Response<Void> delete(@PathVariable Long id) {
    articleService.delete(id);
    return Response.<Void>builder().code(ApiCode.SUCCESS).build();
  }
}
```

### H2 Database

- 메모리 DB / 사용하기 용이해서 예제 코드 수준에서 많이 사용
- Console 접근하기

```
// application.properties

spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
spring.datasource.url=jdbc:h2:mem:testdb
```

### of pattern

- 정적 팩토리 메서드 패턴이라고도 불린다.
- 특정 객체를 생성하는 코드들이 Controller 에 상당히 중복되고 있다. 이를 개선하기 위함
- 객체를 생성하는 일을 그 객체에 위임함으로써 보다 객체지향스러운 코딩을 할 수 있다.

```java
class Article {
    // ...
  public static Article of(ArticleDto.ReqPost from) {
    return Article.builder()
            .title(from.getTitle())
            .content(from.getContent())
            .build();
  }

  public static Article of(ArticleDto.ReqPut from, Long id) {
    return Article.builder()
            .id(id)
            .title(from.getTitle())
            .content(from.getContent())
            .build();
  }
}
```

```java
class Res {
    // ...
    public static Res of(Article from) {
      return Res.builder()
              .id(String.valueOf(from.getId()))
              .title(from.getTitle())
              .content(from.getContent())
              .build();
    }
}
```

```java
@Getter
@Builder
public class Response<T> {
    private final ApiCode code;
    private final T data;

    public static Response<Void> ok() {
        return Response.<Void>builder().code(ApiCode.SUCCESS).build();
    }
    public static <T> Response<T> ok(T data) {
        return Response.<T>builder().code(ApiCode.SUCCESS).data(data).build();
    }
}
```

```java
class ArticleController {
    //...
  @PostMapping
  public Response<Long> post(@RequestBody ArticleDto.ReqPost request) {
    return Response.ok(articleService.save(Article.of(request)));
  }

  @GetMapping("/{id}")
  public Response<ArticleDto.Res> get(@PathVariable Long id) {
    return Response.ok(ArticleDto.Res.of(articleService.findById(id)));
  }

  @PutMapping("/{id}")
  public Response<ArticleDto.Res> put(@PathVariable Long id, @RequestBody ArticleDto.ReqPut request) {
    return Response.ok(ArticleDto.Res.of(articleService.update(Article.of(request, id))));
  }

  @DeleteMapping("/{id}")
  public Response<Void> delete(@PathVariable Long id) {
    articleService.delete(id);
    return Response.ok();
  }
}
```

### Exception Handling

- NullPointerException 이 발생할 수도 있는 영역을 찾고, null 발생했을 때에 예외를 일으켜 보자

```java
class ArticleSertive {
    // ...
  @Transactional
  public Article update(Article request) {
    Article article = this.findById(request.getId());

    if(Objects.isNull(article)) {
      throw new RuntimeException("article value is not existed.");
    }

    article.update(request.getTitle(), request.getContent());

    return article;
  }
}
```

- 예외를 일으킨 코드를 추가하고 실제 API 테스트를 해보니 뭔가 예외에 대한 내용이 API에 담기지는 않는다.
- 우리가 일으킨 예외를 잡아서 핸들링하는 부분이 없기 때문에 자바 언어 수준에서 제공하는 예외 핸들링이 적용된것.
- API 에서 예외에 대한 정보를 내려주기 위해 Controller 에서 잡아보자

```java
@Getter
@JsonFormat(shape = JsonFormat.Shape.OBJECT)
public enum ApiCode {
  /* COMMON */
  SUCCESS("CM0000", "정상입니다"),
  DATA_IS_NOT_FOUND("CM0001", "데이터가 존재하지 않습니다")
  ;

  private final String name;
  private final String desc;

  ApiCode(String name, String desc) {
    this.name = name;
    this.desc = desc;
  }
}
```

```java
class ArticleController {
  //...
  @PutMapping("/{id}")
  public Response<Object> put(@PathVariable Long id, @RequestBody ArticleDto.ReqPut request) {
    try {
      return Response.ok(ArticleDto.Res.of(articleService.update(Article.of(request, id))));
    } catch (RuntimeException e) {
      return Response.builder().code(ApiCode.DATA_IS_NOT_FOUND).data(e.getMessage()).build();
    }
  }
}
```

- 위 예외처리는 어떠한 오류가 발생하든 다 DATA_IS_NOT_FOUND 오류만을 반환하게 된다.
- 오류에 대한 정보가 Controller 영역에서 Catch 할때 오류 정보를 주는 방법이 message 밖에 없기 때문에 ApiCode 부분이 고정적이다.

```java
@Getter
public class ApiException extends RuntimeException {
    private final ApiCode code;

    public ApiException(ApiCode code) {
        this.code = code;
    }

    public ApiException(ApiCode code, String msg) {
        super(msg);
        this.code = code;
    }
}
```

```java
class ArticleService {
    //...
  @Transactional
  public Article update(Article request) {
    Article article = this.findById(request.getId());

    if (Objects.isNull(article)) {
      throw new ApiException(ApiCode.DATA_IS_NOT_FOUND, "article value is not existed.");
    }

    article.update(request.getTitle(), request.getContent());

    return article;
  }
  //...
}
```

```java
class ArticleController {
  @PutMapping("/{id}")
  public Response<Object> put(@PathVariable Long id, @RequestBody ArticleDto.ReqPut request) {
    try {
      return Response.ok(ArticleDto.Res.of(articleService.update(Article.of(request, id))));
    } catch (ApiException e) {
      return Response.builder().code(e.getCode()).data(e.getMessage()).build();
    }
  }
}
```

- ApiException은 Api 관련된 예외처리의 목적으로 만들었기 때문에 모든 컨트롤러 영역에서 사용이 필요하다
- 예외를 핸들링하는 try-catch 코드가 중복된다.

```java
class ArticleController {
    //...
  @GetMapping("/{id}")
  public Response<Object> get(@PathVariable Long id) {
    try {
      return Response.ok(ArticleDto.Res.of(articleService.findById(id)));
    } catch (ApiException e) {
      return Response.builder().code(e.getCode()).data(e.getMessage()).build();
    }
  }

  @PutMapping("/{id}")
  public Response<Object> put(@PathVariable Long id, @RequestBody ArticleDto.ReqPut request) {
    try {
      return Response.ok(ArticleDto.Res.of(articleService.update(Article.of(request, id))));
    } catch (ApiException e) {
      return Response.builder().code(e.getCode()).data(e.getMessage()).build();
    }
  }
}
```

- ControllerAdvice 를 활용하면 Controller 이 후에 영역에서 공통적으로 예외처리가 가능

```java
@RestControllerAdvice
public class ContollerExceptionHandler {
    @ExceptionHandler(ApiException.class)
    public Response<String> apiException(ApiException e) {
        return Response.<String>builder().code(e.getCode()).data(e.getMessage()).build();
    }
}
```

```java
class ArticleController {
    //...
  @GetMapping("/{id}")
  public Response<ArticleDto.Res> get(@PathVariable Long id) {
    return Response.ok(ArticleDto.Res.of(articleService.findById(id)));
  }

  @PutMapping("/{id}")
  public Response<ArticleDto.Res> put(@PathVariable Long id, @RequestBody ArticleDto.ReqPut request) {
    return Response.ok(ArticleDto.Res.of(articleService.update(Article.of(request, id))));
  }
}
```

- Assert 객체를 만들어서 예외처리를 하면 if 문을 줄여서 보다 가독성을 높일 수 있다.

```java
public class Asserts {
    public static void isNull(@Nullable Object obj, ApiCode code, String msg) {
        if(Objects.isNull(obj)) {
            throw new ApiException(code, msg);
        }
    }
}
```

```java
public class ArticleService {
    //...
  @Transactional
  public Article update(Article request) {
    Article article = this.findById(request.getId());
    Asserts.isNull(article, ApiCode.DATA_IS_NOT_FOUND, "article value is not existed.");

    article.update(request.getTitle(), request.getContent());

    return article;
  }
}
```

- Optional 객체를 사용해서 예외처리를 하면 보다 가독성 높이기
- 비용이 비쌈으로 정말 중요한 비즈니스 로직에 사용하기를 권장

```java
class ArticleService {
    // ...
  public Article findById(Long id) {
    return articleRepository.findById(id)
            .orElseThrow(() -> new ApiException(ApiCode.DATA_IS_NOT_FOUND, "article value is not existed."));
  }

  @Transactional
  public Article update(Article request) {
    Article article = this.findById(request.getId());
    article.update(request.getTitle(), request.getContent());

    return article;
  }
}
```

