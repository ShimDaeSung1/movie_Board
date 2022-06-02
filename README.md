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
  -----
* 파일 업로드 설정 application.properties
```
spring.servlet.multipart.enabled=true
spring.servlet.multipart.location=C:\\upload
spring.servlet.multipart.max-request-size=30MB
spring.servlet.multipart.max-file-size=10MB
```
- UploadController, UploadResultDTO 생성
- 파일 업로드
![image](https://user-images.githubusercontent.com/86938974/171080145-64c7ae8d-75a1-4f9c-8fbe-ce74528fe73e.png)

* 영화/리뷰 적용
  -----
* 영화의 등록과 수정에는 파일 업로드 기능 활용해서 영화 포스터 등을 등록
* 회원은 기존 회원들이 존재한다고 가정하고 데이터베이스에 존재하는 회원들 이용
* 회원은 특정한 영화 조회 페이지에서 평점과 자신의 감상을 리뷰로 기록
* 조회 화면에서 회원은 자신이 기록한 리뷰의 내용 수정/삭제

 
* 영화 등록 처리
 * MovieController

```
package org.zerock.mreview.controller;

import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import org.zerock.mreview.dto.MovieDTO;
import org.zerock.mreview.dto.PageRequestDTO;
import org.zerock.mreview.service.MovieService;

import java.util.List;

@Controller
@RequestMapping("/movie")
@Log4j2
@RequiredArgsConstructor
public class MovieController {

    private final MovieService movieService; //final

    @GetMapping("/register")
    public void register(){

    }

    @PostMapping("/register")
    public String register(MovieDTO movieDTO, RedirectAttributes redirectAttributes){
        log.info("movieDTO: " + movieDTO);

        Long mno = movieService.register(movieDTO);

        redirectAttributes.addFlashAttribute("msg", mno);

        return "redirect:/movie/list";
    }

    @GetMapping("/list")
    public void list(PageRequestDTO pageRequestDTO, Model model){

        log.info("pageRequestDTO: " + pageRequestDTO);


        model.addAttribute("result", movieService.getList(pageRequestDTO));

    }

    @GetMapping({"/read", "/modify"})
    public void read(long mno, @ModelAttribute("requestDTO") PageRequestDTO requestDTO, Model model ){

        log.info("mno: " + mno);

        MovieDTO movieDTO = movieService.getMovie(mno);

        model.addAttribute("dto", movieDTO);

    }


}

```
 * register.html
```
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">

<th:block th:replace="~{/layout/basic :: setContent(~{this::content} )}">

    <th:block th:fragment="content">

        <h1 class="mt-4">Movie Register Page</h1>

        <form th:action="@{/movie/register}" th:method="post"  >
            <div class="form-group">
                <label >Title</label>
                <input type="text" class="form-control" name="title" placeholder="Enter Title">
            </div>

            <div class="form-group fileForm">
                <label >Image Files</label>
                <div class="custom-file">
                    <input type="file"  class="custom-file-input files" id="fileInput" multiple>
                    <label class="custom-file-label" data-browse="Browse"></label>
                </div>
            </div>

            <div class="box">

            </div>

            <style>
                .uploadResult {
                    width: 100%;
                    background-color: gray;
                    margin-top: 10px;
                }

                .uploadResult ul {
                    display: flex;
                    flex-flow: row;
                    justify-content: center;
                    align-items: center;
                    vertical-align: top;
                    overflow: auto;
                }

                .uploadResult ul li {
                    list-style: none;
                    padding: 10px;
                    margin-left: 2em;
                }

                .uploadResult ul li img {
                    width: 100px;
                }
            </style>

            <div class="uploadResult">
                <ul>

                </ul>
            </div>
            <button type="submit" class="btn btn-primary">Submit</button>
        </form>

        <script>
            $(document).ready(function(e) {

                var regex = new RegExp("(.*?)\.(exe|sh|zip|alz|tiff)$");
                var maxSize = 10485760; //10MB

                function checkExtension(fileName, fileSize){

                    if(fileSize >= maxSize){
                        alert("파일 사이즈 초과");
                        return false;
                    }

                    if(regex.test(fileName)){
                        alert("해당 종류의 파일은 업로드할 수 없습니다.");
                        return false;
                    }
                    return true;
                }

                $(".custom-file-input").on("change", function() {

                    var fileName = $(this).val().split("\\").pop();
                    $(this).siblings(".custom-file-label").addClass("selected").html(fileName);

                    var formData = new FormData();

                    var inputFile = $(this);

                    var files = inputFile[0].files;

                    var appended = false;

                    for (var i = 0; i < files.length; i++) {

                        if(!checkExtension(files[i].name, files[i].size) ){
                            return false;
                        }

                        console.log(files[i]);
                        formData.append("uploadFiles", files[i]);
                        appended = true;
                    }

                    //upload를 하지 않는다.
                    if (!appended) {return;}

                    for (var value of formData.values()) {
                        console.log(value);
                    }

                    //실제 업로드 부분
                    //upload ajax
                    $.ajax({
                        url: '/uploadAjax',
                        processData: false,
                        contentType: false,
                        data: formData,
                        type: 'POST',
                        dataType:'json',
                        success: function(result){
                            console.log(result);
                            showResult(result);
                        },
                        error: function(jqXHR, textStatus, errorThrown){
                            console.log(textStatus);
                        }
                    }); //$.ajax
                }); //end change event


                function showResult(uploadResultArr){

                    var uploadUL = $(".uploadResult ul");

                    var str ="";

                    $(uploadResultArr).each(function(i, obj) {

                        str += "<li data-name='" + obj.fileName + "' data-path='"+obj.folderPath+"' data-uuid='"+obj.uuid+"'>";
                        str + " <div>";
                        str += "<button type='button' data-file=\'" + obj.imageURL + "\' "
                        str += "class='btn-warning btn-sm'>X</button><br>";
                        str += "<img src='/display?fileName=" + obj.thumbnailURL + "'>";
                        str += "</div>";
                        str + "</li>";
                    });

                    uploadUL.append(str);
                }

                $(".uploadResult ").on("click", "li button", function(e){

                    console.log("delete file");

                    var targetFile = $(this).data("file");

                    var targetLi = $(this).closest("li");

                    $.ajax({
                        url: '/removeFile',
                        data: {fileName: targetFile},
                        dataType:'text',
                        type: 'POST',
                        success: function(result){
                            alert(result);

                            targetLi.remove();
                        }
                    }); //$.ajax
                });


                //prevent submit
                $(".btn-primary").on("click", function(e) {
                    e.preventDefault();

                    var str = "";

                    $(".uploadResult li").each(function(i,obj){
                        var target = $(obj);

                        str += "<input type='hidden' name='imageDTOList["+i+"].imgName' value='"+target.data('name') +"'>";

                        str += "<input type='hidden' name='imageDTOList["+i+"].path' value='"+target.data('path')+"'>";

                        str += "<input type='hidden' name='imageDTOList["+i+"].uuid' value='"+target.data('uuid')+"'>";

                    });

                    //태그들이 추가된 것을 확인한 후에 comment를 제거
                    $(".box").html(str);

                    $("form").submit();

                });



            }); //document ready
        </script>

    </th:block>

</th:block>

```
![image](https://user-images.githubusercontent.com/86938974/171573266-e623f173-e67f-41d5-80a7-17fa8e79f358.png)
* MovieDTO, MovieImageDTO
```
package org.zerock.mreview.dto;


import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class MovieDTO {

    private Long mno;

    private String title;

    @Builder.Default
    private List<MovieImageDTO> imageDTOList = new ArrayList<>();

    //영화의 평균 평점
    private double avg;

    //리뷰 수 jpa의 count( )
    private int reviewCnt;

    private LocalDateTime regDate;

    private LocalDateTime modDate;

}

```
```
package org.zerock.mreview.dto;


import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.UnsupportedEncodingException;
import java.net.URLEncoder;

@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class MovieImageDTO {

    private String uuid;

    private String imgName;

    private String path;

    public String getImageURL(){
        try {
            return URLEncoder.encode(path+"/"+uuid+"_"+imgName,"UTF-8");
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return "";
    }
    public String getThumbnailURL(){
        try {
            return URLEncoder.encode(path+"/s_"+uuid+"_"+imgName,"UTF-8");
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return "";
    }
}


```
* MovieService
```
package org.zerock.mreview.service;

import org.zerock.mreview.dto.MovieDTO;
import org.zerock.mreview.dto.MovieImageDTO;
import org.zerock.mreview.dto.PageRequestDTO;
import org.zerock.mreview.dto.PageResultDTO;
import org.zerock.mreview.entity.Movie;
import org.zerock.mreview.entity.MovieImage;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public interface MovieService {

    Long register(MovieDTO movieDTO);

    PageResultDTO<MovieDTO, Object[]> getList(PageRequestDTO requestDTO); //목록 처리

    MovieDTO getMovie(Long mno);

    default MovieDTO entitiesToDTO(Movie movie, List<MovieImage> movieImages, Double avg, Long reviewCnt){
        MovieDTO movieDTO = MovieDTO.builder()
                .mno(movie.getMno())
                .title(movie.getTitle())
                .regDate(movie.getRegDate())
                .modDate(movie.getModDate())
                .build();

        List<MovieImageDTO> movieImageDTOList = movieImages.stream().map(movieImage -> {
            return MovieImageDTO.builder().imgName(movieImage.getImgName())
                    .path(movieImage.getPath())
                    .uuid(movieImage.getUuid())
                    .build();
        }).collect(Collectors.toList());

        movieDTO.setImageDTOList(movieImageDTOList);
        movieDTO.setAvg(avg);
        movieDTO.setReviewCnt(reviewCnt.intValue());



        return movieDTO;

    }

    default Map<String, Object> dtoToEntity(MovieDTO movieDTO){

        Map<String, Object> entityMap = new HashMap<>();

        Movie movie = Movie.builder()
                .mno(movieDTO.getMno())
                .title(movieDTO.getTitle())
                .build();

        entityMap.put("movie", movie);

        List<MovieImageDTO> imageDTOList = movieDTO.getImageDTOList();

        if(imageDTOList != null && imageDTOList.size() > 0 ) { //MovieImageDTO 처리

            List<MovieImage> movieImageList = imageDTOList.stream().map(movieImageDTO ->{

                MovieImage movieImage = MovieImage.builder()
                        .path(movieImageDTO.getPath())
                        .imgName(movieImageDTO.getImgName())
                        .uuid(movieImageDTO.getUuid())
                        .movie(movie)
                        .build();
                return movieImage;
            }).collect(Collectors.toList());

            entityMap.put("imgList", movieImageList);
        }

        return entityMap;
    }

}

```

```
package org.zerock.mreview.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.zerock.mreview.dto.MovieDTO;
import org.zerock.mreview.dto.PageRequestDTO;
import org.zerock.mreview.dto.PageResultDTO;
import org.zerock.mreview.entity.Movie;
import org.zerock.mreview.entity.MovieImage;
import org.zerock.mreview.repository.MovieImageRepository;
import org.zerock.mreview.repository.MovieRepository;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.function.Function;

@Service
@Log4j2
@RequiredArgsConstructor
public class MovieServiceImpl implements MovieService{

    private final MovieRepository movieRepository; //final

    private final MovieImageRepository imageRepository; //final

    @Transactional
    @Override
    public Long register(MovieDTO movieDTO) {

        Map<String, Object> entityMap = dtoToEntity(movieDTO);
        Movie movie = (Movie) entityMap.get("movie");
        List<MovieImage> movieImageList = (List<MovieImage>) entityMap.get("imgList");

        movieRepository.save(movie);

        movieImageList.forEach(movieImage -> {
            imageRepository.save(movieImage);
        });

        return movie.getMno();
    }

    @Override
    public PageResultDTO<MovieDTO, Object[]> getList(PageRequestDTO requestDTO) {

        Pageable pageable = requestDTO.getPageable(Sort.by("mno").descending());

        Page<Object[]> result = movieRepository.getListPage(pageable);

        log.info("==============================================");
        result.getContent().forEach(arr -> {
            log.info(Arrays.toString(arr));
        });


        Function<Object[], MovieDTO> fn = (arr -> entitiesToDTO(
                (Movie)arr[0] ,
                (List<MovieImage>)(Arrays.asList((MovieImage)arr[1])),
                (Double) arr[2],
                (Long)arr[3])
        );

        return new PageResultDTO<>(result, fn);
    }

    @Override
    public MovieDTO getMovie(Long mno) {

        List<Object[]> result = movieRepository.getMovieWithAll(mno);

        Movie movie = (Movie) result.get(0)[0];

        List<MovieImage> movieImageList = new ArrayList<>();

        result.forEach(arr -> {
            MovieImage  movieImage = (MovieImage)arr[1];
            movieImageList.add(movieImage);
        });

        Double avg = (Double) result.get(0)[2];
        Long reviewCnt = (Long) result.get(0)[3];

        return entitiesToDTO(movie, movieImageList, avg, reviewCnt);
    }

}

```
* list.html
```
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">

<th:block th:replace="~{/layout/basic :: setContent(~{this::content} )}">

    <th:block th:fragment="content">

        <h1 class="mt-4">Movie List Page
            <span>
                <a th:href="@{/movie/register}">
                    <button type="button" class="btn btn-outline-primary">REGISTER
                    </button>
                </a>
            </span>
        </h1>

        <form action="/movie/list" method="get" id="searchForm">
            <input type="hidden" name="page" value="1">
        </form>

        <table class="table table-striped">
            <thead>
            <tr>
                <th scope="col">#</th>
                <th scope="col">Picture</th>
                <th scope="col">Review Count</th>
                <th scope="col">AVG Rating</th>
                <th scope="col">Regdate</th>
            </tr>
            </thead>
            <tbody>

            <tr th:each="dto : ${result.dtoList}" >
                <th scope="row">
                    <a th:href="@{/movie/read(mno = ${dto.mno}, page= ${result.page})}">
                        [[${dto.mno}]]
                    </a>
                </th>
                <td><img th:if="${dto.imageDTOList.size() > 0 && dto.imageDTOList[0].path != null }"
                         th:src="|/display?fileName=${dto.imageDTOList[0].getThumbnailURL()}|" >[[${dto.title}]]</td>
                <td><b>[[${dto.reviewCnt}]]</b></td>
                <td><b>[[${dto.avg}]]</b></td>
                <td>[[${#temporals.format(dto.regDate, 'yyyy/MM/dd')}]]</td>
            </tr>

            </tbody>
        </table>

        <ul class="pagination h-100 justify-content-center align-items-center">

            <li class="page-item " th:if="${result.prev}">
                <a class="page-link" th:href="@{/movie/list(page= ${result.start -1})}" tabindex="-1">Previous</a>
            </li>

            <li th:class=" 'page-item ' + ${result.page == page?'active':''} " th:each="page: ${result.pageList}">
                <a class="page-link" th:href="@{/movie/list(page = ${page})}">
                    [[${page}]]
                </a>
            </li>

            <li class="page-item" th:if="${result.next}">
                <a class="page-link" th:href="@{/movie/list(page= ${result.end + 1} )}">Next</a>
            </li>
        </ul>


        <script th:inline="javascript">

        </script>
    </th:block>

</th:block>

```
* ReviewService와 ReviewDTO
```
package org.zerock.mreview.service;

import org.zerock.mreview.dto.ReviewDTO;
import org.zerock.mreview.entity.Member;
import org.zerock.mreview.entity.Movie;
import org.zerock.mreview.entity.Review;

import java.util.List;

public interface ReviewService {

    //영화의 모든 영화리뷰를 가져온다.
    List<ReviewDTO> getListOfMovie(Long mno);

    //영화 리뷰를 추가
    Long register(ReviewDTO movieReviewDTO);

    //특정한 영화리뷰 수정
    void modify(ReviewDTO movieReviewDTO);

    //영화 리뷰 삭제
    void remove(Long reviewnum);

    default Review dtoToEntity(ReviewDTO movieReviewDTO){

        Review movieReview = Review.builder()
                .reviewnum(movieReviewDTO.getReviewnum())
                .movie(Movie.builder().mno(movieReviewDTO.getMno()).build())
                .member(Member.builder().mid(movieReviewDTO.getMid()).build())
                .grade(movieReviewDTO.getGrade())
                .text(movieReviewDTO.getText())
                .build();

        return movieReview;
    }

    default ReviewDTO entityToDto(Review movieReview){

        ReviewDTO movieReviewDTO = ReviewDTO.builder()
                .reviewnum(movieReview.getReviewnum())
                .mno(movieReview.getMovie().getMno())
                .mid(movieReview.getMember().getMid())
                .nickname(movieReview.getMember().getNickname())
                .email(movieReview.getMember().getEmail())
                .grade(movieReview.getGrade())
                .text(movieReview.getText())
                .regDate(movieReview.getRegDate())
                .modDate(movieReview.getModDate())
                .build();

        return movieReviewDTO;
    }
}

```

```
package org.zerock.mreview.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;


@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ReviewDTO {

    //review num
    private Long reviewnum;

    //Movie mno
    private Long mno;

    //Membmer id
    private Long mid;
    //Member nickname
    private String nickname;
    //Member email
    private String email;

    private int grade;

    private String text;

    private LocalDateTime regDate, modDate;


}

```
* 실행 화면

![image](https://user-images.githubusercontent.com/86938974/171574329-261ce049-9a32-402a-b23e-8bccba4fd9d7.png)

![image](https://user-images.githubusercontent.com/86938974/171574470-8e1a69e4-882c-4e1d-805c-ef7c858abcf9.png)

![image](https://user-images.githubusercontent.com/86938974/171577396-7a271b16-0588-4a13-96ac-966bcf9c340c.png)

![image](https://user-images.githubusercontent.com/86938974/171577433-c933f520-fcd9-44f7-b15c-3246848829f7.png)

![image](https://user-images.githubusercontent.com/86938974/171577459-c188e8a9-f171-4db2-aac2-dd873139e128.png)

![image](https://user-images.githubusercontent.com/86938974/171577519-eca53a7c-f76b-45e6-8ec2-d239fd5478ff.png)

![image](https://user-images.githubusercontent.com/86938974/171577548-2eaa76ec-546e-4b0f-a00e-0bbdaf5e9028.png)

* 신세계 영화
![image](https://user-images.githubusercontent.com/86938974/171577828-9bf564b5-5473-4d55-86fe-ce915219fd9b.png)

![image](https://user-images.githubusercontent.com/86938974/171578021-ce30a5b9-c6e4-4ae0-b450-a65524a8bdc1.png)

![image](https://user-images.githubusercontent.com/86938974/171578082-ce289066-0b4b-43e0-a8a6-0310b339fcdb.png)

![image](https://user-images.githubusercontent.com/86938974/171578132-9a47219e-6bc5-44e7-adf4-2a028bd05377.png)




      


