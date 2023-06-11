Spring Rest Docs

Intro
시나리오 엔진/에디터의 API 문서는 위키/Swagger를 통해 관리
위키를 통해 API 문서를 관리하는 것은 api가 변경될 때마다 별도로 문서를 업데이트 해야 하는 번거로움이 존재하며, 업데이트 사항이 누락될 가능성이 큼
Swagger를 통해 문서 자동화를 구축하였으나, 스펙이 자세히 설명되어 있지 않고 API가 반드시 동작한다는 보장이 없음
어차피 테스트코드 작성할거면, Spring Rest Docs를 통해 API 문서를 작성하는 것이 어떨까
Spring Rest Docs
Spring Rest Docs란?
API 문서를 자동화 하기 위한 라이브러리
단, 테스트 코드 기반
테스트 코드 기반으로 Restful API 문서 작성의 자동화를 돕는 도구
Asciidoctor를 통해 plain text에서 HTML 등등 다양한 포멧으로 문서를 자동으로 출력할 수 있음
Test Case들을 실행하면서, 해당 테스트에서 사용한 요청,응답,헤더 등을 활용해 조각 (snippet)을 만들고, 이 조각을 모아 문서를 만듬
API Spec, 문서화를 위한 테스트 코드가 일치하지 않으면 테스트 빌드를 실패하게 되어,  API 문서가 테스트코드로 검등되었다는 것을 보장할 수 있음
Spring MVC Test를 기반으로 auto-generated snippets 활용
MockMvc, WebTestClient, REST Assured 등등과 사용 가능
Requirements
Java 8 이상
Spring Framework 5 이상
 Spring Rest Docs vs Swagger




구분	Spring Rest Docs	Swagger
장점	제품 코드에 영향이 없다.	API를 테스트 해 볼 수 있는 화면을 제공한다.

	테스트가 성공해야 문서가 작성된다.	테스트 코드 작성이 필요하지 않으며, 적용하기 쉽다.
단점	적용하기 어렵다.	제품코드에 영향이 있다.

	UI를 직접 구성해야 한다.	문서에 작성된 API의 정상 동작을 보장하지 않는다.
Spring Rest Docs

장점

제품 코드에 영향이 없다
실제 제품 코드와 분리되어 있기 때문에, Swagger같이 Config 설정 코드나 어노테이션이 제품 코드와 섞일 일이 없음
테스트가 성공해야 문서가 작성된다
문서 자체도 테스트 코드를 통과해야 하기 때문에, 휴먼 에러 발생 빈도를 줄일 수 있다. 
테스트 코드를 강제할 수 있다
테스트 코드 역량을 기를 수 있다(?)

단점

테스트가 성공해야 문서가 작성된다
초기 환경 설정이 복잡함
UI를 직접 Asciidoc을 사용해서 구성해야 함 (안예쁨..)
테스트 코드를 작성하는데 진입장벽이 있음
Swagger

장점

적용하기 쉬움
UI를 자동으로 생성해주며, 테스트 해 볼 수 있는 화면을 제공함 (예쁨..)

단점

제품코드에 어노테이션을 추가하는 등의 비즈니스 로직과 상관없는 코드가 추가됨
생성된 문서에 있는 API가 반드시 동작한다고 보장하지 못함




시나리오 엔진 Spring Rest Docs 적용기
준비
환경
Spring Boot 2.5.6
Gradle 7.2
Java13
Junit5
MockMvc
의존성
testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc'
REST Docs 자동 설정
@AutoConfigureRestDocs
RestDocMockMvc 커스터마이징
@TestConfiguration
public class RestDocsConfiguration {

@Bean
public RestDocsMockMvcConfigurationCustomizer restDocsMockMvcConfigurationCustomizer() {
	return configurer -> configurer.operationPreprocessors()
		.withRequestDefaults(prettyPrint())
		.withResponseDefaults(prettyPrint());
	}
}




plugin 설정
plugins {
	...
	id "org.asciidoctor.jvm.convert" version "3.3.2" // gradle 7부터는 org.asciidoctor.convert 사용 X, adoc 파일 변환, build 디렉토리에 복사하기 위한 역할
}

...

ext {
    snippetsDir = file('build/generated-snippets')
}

...

asciidoctor {
	dependsOn test
	inputs.dir snippetsDir
}

asciidoctor.doFirst {
	delete file('src/main/resources/static/docs')// 기존에 존재하는 docs 삭제
}

...

//스니펫을 이용해 문서 작성 후, build/docs/asciidoc 하위에 생기는 html 파일을 BOOT-INF/classes/static/docs로 복사
bootJar {
	enabled = true
	dependsOn asciidoctor
	copy {
		from "${asciidoctor.outputDir}"
		into 'BOOT-INF/classes/static/docs'
	}
}


// build/docs/asciidoc 파일을 src/main/resources/static/docs로 복사
task copyDocument(type: Copy) {
	dependsOn asciidoctor
	from file("build/docs/asciidoc")
	into file("src/main/resources/static/docs")
}

build {
	dependsOn copyDocument
}




테스트 코드 작성
문서화 대상 (코드 참조)
요청 본문 문서화
응답 본문 문서화
요청 헤더 문서화
요청 필드 문서화
응답 헤더 문서화
응답 필드 문서화
문서화
Code Snippet 생성하기
테스트 코드를 실행하면, document()에 사용된 identifier에 따라 build/generated-snippets 하위에 패키지가 생성

템플릿 파일 생성하기
생성된 snippet들을 묶어 html 문서로 만들기 위해, gradle인 경우 src/docs/asciidoc 하위에 adoc 파일 생성
네이밍 무관




snippet을 이용해 문서를 작성한다.

최 상단의 내용을 기입하지 않으면 snippet을 찾지 못하는 오류가 있음
빌드
./gradlew 빌드 후, resources-static-docs 하위에 문서가 생긴다.
해당 url로 접속하여 문서를 확인할 수 있다. (docs/index.html)

참고자료
Spring Rest Docs에서 Open Api Spec 추출
MSA 환경에서 API 문서 관리하기 
Spring Rest Docs와 Swagger 함께 사용하기
우아한 형제들 Spring Rest Docs 전환








