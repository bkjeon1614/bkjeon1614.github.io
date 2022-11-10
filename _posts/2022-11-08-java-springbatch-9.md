---
layout: post
title : "Spring Batch 9편(마무리) - ItemProcessor"
subtitle : "2022-11-08-java-springbatch-9.md"
date: 2022-11-08 18:50:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [spring, java, jvm, spring batch]
---

# ItemProcessor
이전글: [Spring Batch 8편 - ItemReader](https://bkjeon1614.tistory.com/746)   
작업코드: [작업코드](https://github.com/bkjeon1614/java-example-code/tree/develop/spring-batch-study/spring-batch-study-jpa)    
    
ItemProcessor 는 데이터를 가공하거나 필터링하는 역할을 하고 있으며 `필수가 아니고 이는 Writer 부분에서도 충분히 구현이 가능` 하다. 그럼에도 사용하는 것은 Reader, Writer 와는 별도의 단계로 분리되었기 때문에 `비즈니스 코드가 섞이는 것을 방지` 해주기 때문이다.   
> Batch 에 비즈니스 로직을 추가할때는 가장 먼저 Processor 를 고려하는 것이 좋다. (각 계층 읽기/처리/쓰기 를 분리할 수 있는 좋은 방법)   
     
   
## 1. ItemProcessor 란
ItemProcessor 는 `Reader 에서 넘겨준 데이터 개별건을 가공/처리 해준다.`   
![springbatch-51](/img/posts/language/java/springbatch/springbatch-51.png)     
- 변환
  - Reader 에서 읽은 데이터를 원하는 타입으로 변환하여 Writer 에 넘겨준다.
- 필터
  - Reader에서 넘겨준 데이터를 Writer로 넘겨줄 것인지를 결정할 수 있다.
  - `null` 을 반환하면 `Writer 에 전달되지 않습니다`
    
     
## 2. 사용법
ItemProcessor 인터페이스는 두 개의 제네릭 타입이 필요하다.   
![springbatch-52](/img/posts/language/java/springbatch/springbatch-52.png)     
Reader 에서 읽은 데이터가 ItemProcessor 의 process 를 통과해서 Writer 에 전달된다.  
   
자 이제 process 메소드를 를 구현해보자. (해당 코드는 Teacher 라는 도메인 클래스를 읽어와 Name 필드(String 타입) 를 Writer 에 넘겨주는 코드이다.)  
[sql]   
```
create table teacher (
 id         bigint not null auto_increment,
 name     varchar(255),
 subject     varchar(255),
 primary key (id)
) engine = InnoDB;

insert into teacher (name, subject) VALUES ('선생님1', '제목1');
insert into teacher (name, subject) VALUES ('선생님2', '제목22');
insert into teacher (name, subject) VALUES ('선생님3', '제목333');
insert into teacher (name, subject) VALUES ('선생님4', '제목4444');
insert into teacher (name, subject) VALUES ('선생님5', '제목55555');
insert into teacher (name, subject) VALUES ('선생님6', '제목666666');
insert into teacher (name, subject) VALUES ('선생님7', '제목7777777');
insert into teacher (name, subject) VALUES ('선생님8', '제목88888888');
insert into teacher (name, subject) VALUES ('선생님9', '제목999999999');
```   
[Teacher.java]   
```
package com.example.entity;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

import lombok.Getter;
import lombok.NoArgsConstructor;

@Getter
@NoArgsConstructor
@Entity
public class Teacher {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;
	private String name;
	private String subject;

	public Teacher(Long id, String name, String subject) {
		this.id = id;
		this.name = name;
		this.subject = subject;
	}

}
```    
[ProcessorConvertJobConfiguration.java]   
```
package com.example.job;

import javax.persistence.EntityManagerFactory;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.JobScope;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemWriter;
import org.springframework.batch.item.database.JpaPagingItemReader;
import org.springframework.batch.item.database.builder.JpaPagingItemReaderBuilder;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.util.Assert;

import com.example.entity.Teacher;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
@Configuration
public class ProcessorConvertJobConfiguration implements InitializingBean {

	public static final String JOB_NAME = "ProcessorConvertBatch";
	public static final String BEAN_PREFIX = JOB_NAME + "_";

	private final JobBuilderFactory jobBuilderFactory;
	private final StepBuilderFactory stepBuilderFactory;
	private final EntityManagerFactory entityManagerFactory;

	@Value("${chunkSize:1000}")
	private int chunkSize;

	@Override
	public void afterPropertiesSet() {
		Assert.notNull(entityManagerFactory, "EntityManagerFactory is required");
	}

	@Bean(JOB_NAME)
	public Job job() {
		return jobBuilderFactory.get(JOB_NAME)
			.preventRestart()
			.start(step())
			.build();
	}

	@Bean(BEAN_PREFIX + "step")
	@JobScope
	public Step step() {
		return stepBuilderFactory.get(BEAN_PREFIX + "step")
			.<Teacher, String>chunk(chunkSize)
			.reader(reader())
			.processor(processor())
			.writer(writer())
			.build();
	}

	@Bean
	public JpaPagingItemReader<Teacher> reader() {
		return new JpaPagingItemReaderBuilder<Teacher>()
			.name(BEAN_PREFIX + "reader")
			.entityManagerFactory(entityManagerFactory)
			.pageSize(chunkSize)
			.queryString("SELECT t FROM Teacher t")
			.build();
	}

	@Bean
	public ItemProcessor<Teacher, String> processor() {
		return teacher -> teacher.getName();
	}

	private ItemWriter<String> writer() {
		return items -> {
			for (String item : items) {
				log.info("Teacher Name={}", item);
			}
		};
	}

}
```  
상기 코드 ProcessorConvertJobConfiguration.java 의 ItemProcessor 에서는 Reader 에서 읽어올 타입이 Teacher 이며, Writer 에서 넘겨줄 타입이 `String` 이기 때문에 제네릭 타입은 `<Teacher, String>` 이 된다.
```
@Bean
public ItemProcessor<Teacher, String> processor() {
	return teacher -> teacher.getName();
}  
```     
여기서 ChunkSize 앞에 선언될 타입역시 Reader,Writer 타입을 따라가기 때문에 아래와 같이 선언된다.   
```
private ItemWriter<String> writer() {
	return items -> {
		for (String item : items) {
			log.info("Teacher Name={}", item);
		}
	};
}
```   

상기 코드를 실행해보면 아래와 같이 log.info() 가 잘 실행되었음을 볼 수 있다. (Teacher class 가 Processor 을 거치면서 String 으로 잘 전환된 것을 확인할 수 있다.)   
![springbatch-53](/img/posts/language/java/springbatch/springbatch-53.png)     
 

## 3. 필터   
Writer 에 값을 넘길지 말지를 Processor 에서 판단하는 것이다.   
하기 예시 코드를 참고하자 (Teacher 의 id 값이 짝수일 경우 null 을 return 하여 Writer 에 넘기지 않도록 작성)   
[ProcessorNullJobConfiguration.java]   
```
package com.example.job;

import javax.persistence.EntityManagerFactory;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.JobScope;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemWriter;
import org.springframework.batch.item.database.JpaPagingItemReader;
import org.springframework.batch.item.database.builder.JpaPagingItemReaderBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.example.entity.Teacher;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
@Configuration
public class ProcessorNullJobConfiguration {

	public static final String JOB_NAME = "processorNullBatch";
	public static final String BEAN_PREFIX = JOB_NAME + "_";

	private final JobBuilderFactory jobBuilderFactory;
	private final StepBuilderFactory stepBuilderFactory;
	private final EntityManagerFactory emf;

	@Value("${chunkSize:1000}")
	private int chunkSize;

	@Bean(JOB_NAME)
	public Job job() {
		return jobBuilderFactory.get(JOB_NAME)
			.preventRestart()
			.start(step())
			.build();
	}

	@Bean(BEAN_PREFIX + "step")
	@JobScope
	public Step step() {
		return stepBuilderFactory.get(BEAN_PREFIX + "step")
			.<Teacher, Teacher>chunk(chunkSize)
			.reader(pagingItemReader())
			.processor(ItemProcessor())
			.writer(writer())
			.build();
	}

	@Bean
	public JpaPagingItemReader<Teacher> pagingItemReader() {
		return new JpaPagingItemReaderBuilder<Teacher>()
			.name(BEAN_PREFIX+"reader")
			.entityManagerFactory(emf)
			.pageSize(chunkSize)
			.queryString("SELECT t FROM Teacher t")
			.build();
	}

	@Bean
	public ItemProcessor<Teacher, Teacher> ItemProcessor() {
		return teacher -> {

			boolean isIgnoreTarget = teacher.getId() % 2 == 0L;
			if(isIgnoreTarget){
				log.info(">>>>>>>>> Teacher name={}, isIgnoreTarget={}", teacher.getName(), isIgnoreTarget);
				return null;
			}

			return teacher;
		};
	}

	private ItemWriter<Teacher> writer() {
		return items -> {
			for (Teacher item : items) {
				log.info("Teacher Name={}", item.getName());
			}
		};
	}

}
```   
하기 이미지와 같이 실제 코드를 실행해보면 홀수인 Teacher 만 출력되는 것을 볼 수 있다.   
![springbatch-54](/img/posts/language/java/springbatch/springbatch-54.png)     


## 4. 트랜잭션 범위
Spring Batch 에서는 `트랜잭션 범위는 Chunk 단위` 이다. 그래서 Reader 에서 Entity 를 반환해주었다면 `Entity 간의 Lazy Loading 이 가능` 하다. 이는 Processor 뿐만 아니라 Writer 에서도 가능하다.

### 4-1. Processor 에서의 Lazy Loading  
하기 코드는 Reader 에서 Teacher Entity 를 반환하고 Processor 에서 Entity 의 하위 자식들인 Student 를 Lazy Loading 한다.   
[sql]   
```
create table student (
 id          bigint not null auto_increment,
 teacher_id  bigint not null,
 name        varchar(255),
 primary key (id)
) engine = InnoDB;

insert into student (name, teacher_id) VALUES ('전봉근', 1);
insert into student (name, teacher_id) VALUES ('홍길동', 2);
insert into student (name, teacher_id) VALUES ('정도전', 3);
insert into student (name, teacher_id) VALUES ('이성계', 4);
```   
[ClassInformation.java]   
```
package com.example.entity;

import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.ToString;

@ToString
@Getter
@NoArgsConstructor
public class ClassInformation {

	private String teacherName;
	private int studentSize;

	public ClassInformation(String teacherName, int studentSize) {
		this.teacherName = teacherName;
		this.studentSize = studentSize;
	}
}
```   
[Teacher.java]    
```
package com.example.entity;

import java.util.ArrayList;
import java.util.List;

import javax.persistence.CascadeType;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.OneToMany;

import lombok.Getter;
import lombok.NoArgsConstructor;

@Getter
@NoArgsConstructor
@Entity
public class Teacher {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;
	private String name;
	private String subject;

	@OneToMany(mappedBy = "teacher", cascade = CascadeType.ALL)
	private List<Student> students = new ArrayList<>();

	public Teacher(Long id, String name, String subject) {
		this.id = id;
		this.name = name;
		this.subject = subject;
	}

}
```    
[Student.java]   
```
package com.example.entity;

import lombok.AccessLevel;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

import javax.persistence.*;

@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Student {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	private String name;

	@ManyToOne
	@JoinColumn(name = "teacher_id", foreignKey = @ForeignKey(name = "fk_student_teacher"))
	private Teacher teacher;

	@Builder
	public Student(String name) {
		this.name = name;
	}

	public void setTeacher(Teacher teacher) {
		this.teacher = teacher;
	}
}
```   
[TransactionProcessorJobConfiguration.java]   
```
package com.example.job;

import javax.persistence.EntityManagerFactory;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.JobScope;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemWriter;
import org.springframework.batch.item.database.JpaPagingItemReader;
import org.springframework.batch.item.database.builder.JpaPagingItemReaderBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.example.entity.ClassInformation;
import com.example.entity.Teacher;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
@Configuration
public class TransactionProcessorJobConfiguration {

	public static final String JOB_NAME = "transactionProcessorBatch";
	public static final String BEAN_PREFIX = JOB_NAME + "_";

	private final JobBuilderFactory jobBuilderFactory;
	private final StepBuilderFactory stepBuilderFactory;
	private final EntityManagerFactory emf;

	@Value("${chunkSize:1000}")
	private int chunkSize;

	@Bean(JOB_NAME)
	public Job job() {
		return jobBuilderFactory.get(JOB_NAME)
			.preventRestart()
			.start(step())
			.build();
	}

	@Bean(BEAN_PREFIX + "step")
	@JobScope
	public Step step() {
		return stepBuilderFactory.get(BEAN_PREFIX + "step")
			.<Teacher, ClassInformation>chunk(chunkSize)
			.reader(itemReader())
			.processor(itemProcessor())
			.writer(itemWriter())
			.build();
	}


	@Bean
	public JpaPagingItemReader<Teacher> itemReader() {
		return new JpaPagingItemReaderBuilder<Teacher>()
			.name(BEAN_PREFIX + "reader")
			.entityManagerFactory(emf)
			.pageSize(chunkSize)
			.queryString("SELECT t FROM Teacher t")
			.build();
	}

	public ItemProcessor<Teacher, ClassInformation> itemProcessor() {
		return teacher -> new ClassInformation(teacher.getName(), teacher.getStudents().size());
	}

	private ItemWriter<ClassInformation> itemWriter() {
		return items -> {
			log.info(">>>>>>>>>>> Item Write");
			for (ClassInformation item : items) {
				log.info("반 정보= {}", item);
			}
		};
	}

}
```   
상기 코드를 보면 Processor 부분에서 teacher.getStudents() 로 가져오는것을 볼 수 있다. 여기서 Processor 이 트랜잭션 범위 밖이면 오류가 날텐데 해당 코드를 실행해보면 성공적으로 배치가 실행되는것을 볼 수 있다. 즉, `Processor 는 트랜잭션 범위 안이며, Entity 의 Lazy Loading 이 가능` 하다는 것을 확인할 수 있다.   
![springbatch-55](/img/posts/language/java/springbatch/springbatch-55.png)     

### 4-2. Writer
Writer 에서의 Lazy Loading 을 확인해보자. (하기 코드는 Reader 에서 Teacher Entity 를 반환하고 Processor 를 거치지 않고 Writer 로 바로 넘겨 Writer 에서 Entity 의 하위 자식들인 Student 를 Lazy Loading 한다.)   
[Teacher.java]   
```
package com.example.entity;

import java.util.ArrayList;
import java.util.List;

import javax.persistence.CascadeType;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.OneToMany;

import lombok.AccessLevel;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Teacher {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;
	private String name;
	private String subject;

	@OneToMany(mappedBy = "teacher", cascade = CascadeType.ALL)
	private List<Student> students = new ArrayList<>();

	@Builder
	public Teacher(String name, String subject) {
		this.name = name;
		this.subject = subject;
	}

	public Teacher(Long id, String name, String subject) {
		this.id = id;
		this.name = name;
		this.subject = subject;
	}

}
```    
[Student.java]     
```  
package com.example.entity;

import lombok.AccessLevel;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

import javax.persistence.*;

@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Student {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	private String name;

	@ManyToOne
	@JoinColumn(name = "teacher_id", foreignKey = @ForeignKey(name = "fk_student_teacher"))
	private Teacher teacher;

	@Builder
	public Student(String name) {
		this.name = name;
	}

	public void setTeacher(Teacher teacher) {
		this.teacher = teacher;
	}

}
```    
[TransactionWriterJobConfiguration.java]   
```
package com.example.job;

import javax.persistence.EntityManagerFactory;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.JobScope;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.item.ItemWriter;
import org.springframework.batch.item.database.JpaPagingItemReader;
import org.springframework.batch.item.database.builder.JpaPagingItemReaderBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.example.entity.Teacher;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
@Configuration
public class TransactionWriterJobConfiguration {

	public static final String JOB_NAME = "transactionWriterBatch";
	public static final String BEAN_PREFIX = JOB_NAME + "_";

	private final JobBuilderFactory jobBuilderFactory;
	private final StepBuilderFactory stepBuilderFactory;
	private final EntityManagerFactory emf;

	@Value("${chunkSize:1000}")
	private int chunkSize;

	@Bean(JOB_NAME)
	public Job job() {
		return jobBuilderFactory.get(JOB_NAME)
			.preventRestart()
			.start(step())
			.build();
	}

	@Bean(BEAN_PREFIX + "step")
	@JobScope
	public Step step() {
		return stepBuilderFactory.get(BEAN_PREFIX + "step")
			.<Teacher, Teacher>chunk(chunkSize)
			.reader(jpaItemReader())
			.writer(teacherItemWriter())
			.build();
	}

	@Bean
	public JpaPagingItemReader<Teacher> jpaItemReader() {
		return new JpaPagingItemReaderBuilder<Teacher>()
			.name(BEAN_PREFIX + "reader")
			.entityManagerFactory(emf)
			.pageSize(chunkSize)
			.queryString("SELECT t FROM Teacher t")
			.build();
	}

	private ItemWriter<Teacher> teacherItemWriter() {
		return items -> {
			log.info(">>>>>>>>>>> Item Write");
			for (Teacher item : items) {
				log.info("teacher={}, student Size={}", item.getName(), item.getStudents().size());
			}
		};
	}

}
```   
상기 코드 또한 실행하면 Lazy Loading 이 발생하는 것을 확인할 수 있다. (`Writer 에서 조회시 Student 조회 쿼리 발생` -> 하기 이미지 참고)   
![springbatch-56](/img/posts/language/java/springbatch/springbatch-56.png)     
   
> 4-1, 4-2 2개의 테스트로 Processor와 Writer는 트랜잭션 범위 안이며, Lazy Loading이 가능하다는 것을 확인하였다. (JPA 를 편하게 사용할 수 있다.)


## 5. ItemProcessor 구현체
Spring Batch 에서는 자주 사용하는 용도의 Processor 를 미리 클래스로 만들어서 제공해주는것은 아래와 같다.  
- ItemProcessorAdapter
- ValidatingItemProcessor
- CompositeItemProcessor  
여기서 ItemProcessorAdapter, ValidatingItemProcessor는 거의 사용하지 않는다. 왜냐하면 최근에는 대부분 Processor 를 직접 구현할때가 많고, 아니면 람다식으로 빠르게 구현하는 경우가 많기 때문이다. 즉, 커스텀하게 직접 구현해도 되기 때문이다.  
   
단, CompositeItemProcessor는 간혹 사용할떄가 있는데 해당 Processor 는 `ItemProcessor간의 체이닝을 지원하는 Processor라고 보면 된다.`   
     
Processor 는 변환 또는 필터의 역할을 하지만 변환이 2번 필요할 경우 어떻게 처리하는지 하기 코드를 통해 알아보자. (Teacher 의 이름을 가져와 getName() 앞/뒤 문장에 다른 문자를 붙여 Writer 에 전달하는 코드이다.)  
[ProcessorCompositeJobConfiguration.java]  
```
package com.example.job;

import java.util.ArrayList;
import java.util.List;

import javax.persistence.EntityManagerFactory;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.JobScope;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemWriter;
import org.springframework.batch.item.database.JpaPagingItemReader;
import org.springframework.batch.item.database.builder.JpaPagingItemReaderBuilder;
import org.springframework.batch.item.support.CompositeItemProcessor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.example.entity.Teacher;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
@Configuration
public class ProcessorCompositeJobConfiguration {

	public static final String JOB_NAME = "processorCompositeBatch";
	public static final String BEAN_PREFIX = JOB_NAME + "_";

	private final JobBuilderFactory jobBuilderFactory;
	private final StepBuilderFactory stepBuilderFactory;
	private final EntityManagerFactory emf;

	@Value("${chunkSize:1000}")
	private int chunkSize;

	@Bean(JOB_NAME)
	public Job job() {
		return jobBuilderFactory.get(JOB_NAME)
			.preventRestart()
			.start(step())
			.build();
	}

	@Bean(BEAN_PREFIX + "step")
	@JobScope
	public Step step() {
		return stepBuilderFactory.get(BEAN_PREFIX + "step")
			.<Teacher, String>chunk(chunkSize)
			.reader(jpaItemreader())
			.processor(compositeProcessor())
			.writer(writer())
			.build();
	}

	@Bean
	public JpaPagingItemReader<Teacher> jpaItemreader() {
		return new JpaPagingItemReaderBuilder<Teacher>()
			.name(BEAN_PREFIX+"reader")
			.entityManagerFactory(emf)
			.pageSize(chunkSize)
			.queryString("SELECT t FROM Teacher t")
			.build();
	}

	@Bean
	public CompositeItemProcessor compositeProcessor() {
		List<ItemProcessor> delegates = new ArrayList<>(2);
		delegates.add(processor1());
		delegates.add(processor2());

		CompositeItemProcessor processor = new CompositeItemProcessor<>();
		processor.setDelegates(delegates);

		return processor;
	}

	public ItemProcessor<Teacher, String> processor1() {
		return Teacher::getName;
	}

	public ItemProcessor<String, String> processor2() {
		return name -> "안녕하세요. "+ name + "입니다.";
	}

	private ItemWriter<String> writer() {
		return items -> {
			for (String item : items) {
				log.info("Teacher Name={}", item);
			}
		};
	}

}
```   
CompositeItemProcessor에 ItemProcessor List인 delegates을 할당하면 된다. (서로 다른 클래스 타입으로 변환해도 가능)   
단, 제네릭 타입을 사용하지 못하는데 이유는 사용하게 되면 `delegates에 포함된 모든 ItemProcessor는 같은 제네릭 타입을 가져야 한다.` processor1 은 <Teacher, String> 을, processor2 는 <String, String> 이므로 같은 제네릭 타입을 쓰지 못하기 때문에 따로 다른 클래스로 선언된 것이다.    
   
상기 코드를 실행하면 하기 이미지와 같이 잘 실행되었음을 확인할 수 있다.    
![springbatch-57](/img/posts/language/java/springbatch/springbatch-57.png)     


## 참고
- https://jojoldu.tistory.com/