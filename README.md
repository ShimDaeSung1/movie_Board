# 영화리뷰 게시판 구현

* 기능
      * 영화 리뷰 등록/삭제
      * 평점
      * 페이징처리
      * 영화 이미지(파일 업로드)

* 사용
      * Spring Boot
      * gradle
      * JPA
      * JPQL
      * Lombok
      * Ajax
      * jQuery
      * thymeleaf
* ERD
![영화리뷰](https://user-images.githubusercontent.com/86938974/170615162-eaf6133b-8cb7-49df-a1f3-1617a35daa62.png)
      * 영화와 회원은 M:N 관계 이므로 매핑테이블 '리뷰'를 추가해준다. 
      * 영화와 리뷰는 1:N, 리뷰와 회원은 N:1 관계가 성립한다.
      * 영화와 영화사진은 1:N 관계를 가진다.


* build.gradle에 필요한 라이브러리 추가, MariaDB 드라이버와 Thymeleaf 사용
```
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    annotationProcessor 'org.projectlombok:lombok'
    providedRuntime 'org.springframework.boot:spring-boot-starter-tomcat'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    // https://mvnrepository.com/artifact/org.mariadb.jdbc/mariadb-java-client
    implementation group: 'org.mariadb.jdbc', name: 'mariadb-java-client', version: '2.7.1'
    // https://mvnrepository.com/artifact/org.thymeleaf.extras/thymeleaf-extras-java8time
    implementation group: 'org.thymeleaf.extras', name: 'thymeleaf-extras-java8time', version: '3.0.4.RELEASE'
    // https://mvnrepository.com/artifact/com.querydsl/querydsl-jpa
    implementation group: 'com.querydsl', name: 'querydsl-jpa', version: '5.0.0'
    // https://mvnrepository.com/artifact/com.querydsl/querydsl-apt
    implementation group: 'com.querydsl', name: 'querydsl-apt', version: '5.0.0'
}
```
* application.properties에 DB와 JPA관련 설정 추가
```
server.port=8181
spring.datasource.driver-class-name=org.mariadb.jdbc.Driver
spring.datasource.url=jdbc:mariadb://localhost:3306/bootex
spring.datasource.username=bootuser
spring.datasource.password=bootuser

spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.show-sql=true

spring.thymeleaf.cache=false

```
* BaseEntity와 테이블 4개 생성
![image](https://user-images.githubusercontent.com/86938974/170615861-31f95e49-dec1-4b27-822a-c932a9244c0a.png)

* MovieImage 클래스는 UUID사용, LAZY 방식 사용
```
package org.zerock.mreview.entity;

import lombok.*;

import javax.persistence.*;

@Entity
@Builder
@AllArgsConstructor
@NoArgsConstructor
@Getter
@ToString(exclude = "movie") //연관관계 주의, movie안의 필드들을 들고오면 안 되기 때문
public class MovieImage {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long inum;

    private String uuid;

    private String imgName;

    private String path;

    @ManyToOne(fetch = FetchType.LAZY)
    private Movie movie;
}
```
* Review는 매핑 테이블로, 두 테이블 사이에서 양쪽의 PK를 참조하는 형태로 구성된다.
```
package org.zerock.mreview.entity;

import lombok.*;

import javax.persistence.*;

@Entity
@Builder
@AllArgsConstructor
@NoArgsConstructor
@Getter
@ToString(exclude = {"movie", "member"})
public class Review extends BaseEntity{

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long reviewnum;

    @ManyToOne(fetch = FetchType.LAZY)
    private Movie movie;

    @ManyToOne(fetch = FetchType.LAZY)
    private Member member;

    private int grade;

    private String text;


}

```
* Repository 생성
![image](https://user-images.githubusercontent.com/86938974/170616299-aa7d7bfc-e8d0-406e-bc92-0eccce336f85.png)
      * JPQL 사용한 조인

      * MovieRepository 인터페이스
```
package org.zerock.mreview.repository;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.zerock.mreview.entity.Movie;

import java.util.List;

public interface MovieRepository extends JpaRepository<Movie, Long> {
//    group by 지정을 통해 각 영화의 리뷰 개수와 평점 평균, 영화 이미지 가져옴
    @Query("select m, mi, avg(coalesce(r.grade, 0)), count(distinct r) from Movie m" +
            " left outer join MovieImage mi on mi.movie = m" +
            " left outer join Review r on r.movie = m group by m")
    Page<Object[]> getListPage(Pageable pageable);

    @Query("select m, mi, avg(coalesce(r.grade, 0)), count(r) " +
            " from Movie m left outer join MovieImage mi on mi.movie = m" +
            " left outer join Review r on r.movie = m" +
            " where m.mno = : mno group by mi")
    List<Object[]> getMovieWithAll(Long mno); // 특정 영화 조회

}

```
      * ReviewRepository 인터페이스
```
package org.zerock.mreview.repository;

import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.zerock.mreview.entity.Member;
import org.zerock.mreview.entity.Movie;
import org.zerock.mreview.entity.Review;

import java.util.List;

public interface ReviewRepository extends JpaRepository<Review, Long> {
    
    // FETCH 속성값은 attributePaths에 명시한 속성은 EAGER로 처리하고, 나머지는 LAZY로 처리
    @EntityGraph(attributePaths = {"member"}, type = EntityGraph.EntityGraphType.FETCH)
    List<Review> findByMovie(Movie movie);

    // 특정 회원 삭제시 리뷰도 같이 삭제한다.
    @Modifying // update나 delete를 사용하기 위해서 필요한 어노테이션
    @Query("delete from Review mr where mr.member = :member")
    void deleteByMemer(Member member);
}

```
- @EntityGraph는 attributePaths와 type 속성을 가진다.
- review를 Movie로 조회시 attributePaths에 설정된 member를 EAGER, 동시에 가져온다.
- member를 삭제시 리뷰도 같이 삭제된다.

* 파일 업로드 처리


      


